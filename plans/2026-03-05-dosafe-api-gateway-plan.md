# DOSafe API Gateway Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Expose DOSafe safety endpoints (`url-check`, `entity-check`, `detect`) through the DOS.AI API gateway with per-request billing.

**Architecture:** Cloudflare Worker at `api.dos.ai` proxies `/v1/dosafe/*` requests to `dosafe.io/api/*`, reusing existing API key auth, rate limiting, and billing. New `fixed_cost_cents` column in `model_pricing` enables per-request pricing instead of token-based.

**Tech Stack:** Cloudflare Workers (TypeScript), D1 database, Supabase billing

**Design doc:** `docs/plans/2026-03-05-dosafe-api-gateway-design.md`

---

### Task 1: Add `fixed_cost_cents` column to D1 + insert pricing rows

**Files:**
- Create: `packages/api-gateway/migrations/003_dosafe_pricing.sql`

**Step 1: Write the migration SQL**

```sql
-- Add fixed_cost_cents column for per-request pricing (DOSafe endpoints)
ALTER TABLE model_pricing ADD COLUMN fixed_cost_cents REAL DEFAULT 0;

-- DOSafe endpoint pricing
INSERT OR REPLACE INTO model_pricing (model_id, display_name, input_price_per_million, output_price_per_million, is_active, fixed_cost_cents)
VALUES
  ('dosafe/url-check', 'DOSafe URL Check', 0, 0, 1, 0.10),
  ('dosafe/entity-check', 'DOSafe Entity Check', 0, 0, 1, 0.05),
  ('dosafe/detect', 'DOSafe AI Detection', 0, 0, 1, 0.50);
```

**Step 2: Apply migration to D1**

```bash
cd d:/Projects/DOSafe/packages/api-gateway
npx wrangler d1 execute dos-api --remote --file=migrations/003_dosafe_pricing.sql
```

Expected: Migration applied, 3 rows inserted.

**Step 3: Verify**

```bash
npx wrangler d1 execute dos-api --remote --command="SELECT model_id, fixed_cost_cents FROM model_pricing WHERE model_id LIKE 'dosafe/%'"
```

Expected: 3 rows with correct costs.

**Step 4: Commit**

```bash
git add packages/api-gateway/migrations/003_dosafe_pricing.sql
git commit -m "feat: add DOSafe per-request pricing to D1 model_pricing"
```

---

### Task 2: Update `ModelPricing` interface and `deductBalance()` for fixed cost

**Files:**
- Modify: `packages/api-gateway/src/worker.ts` (lines 165-169 ModelPricing interface, lines 419-498 deductBalance function)

**Step 1: Update `ModelPricing` interface**

At `worker.ts:165-169`, add `fixed_cost_cents`:

```typescript
interface ModelPricing {
  model_id: string;
  input_price_per_million: number;
  output_price_per_million: number;
  fixed_cost_cents: number;
}
```

**Step 2: Update `deductBalance()` query to include `fixed_cost_cents`**

At `worker.ts:431-432`, change the SELECT:

```typescript
const pricing = await db.prepare(
  'SELECT input_price_per_million, output_price_per_million, fixed_cost_cents FROM model_pricing WHERE model_id = ?'
).bind(model).first<ModelPricing>();
```

**Step 3: Add fixed-cost branch in `deductBalance()`**

After the pricing lookup (around line 436), add:

```typescript
// Fixed per-request cost (DOSafe endpoints)
const fixedCost = pricing?.fixed_cost_cents || 0;
let totalCostCents: number;
let inputCostRounded = 0;
let outputCostRounded = 0;

if (fixedCost > 0) {
  totalCostCents = fixedCost;
} else {
  // Token-based pricing (existing logic)
  const inputPricePerMillion = pricing?.input_price_per_million || 10;
  const outputPricePerMillion = pricing?.output_price_per_million || 10;
  const inputCostCents = (inputTokens / 1_000_000) * inputPricePerMillion;
  const outputCostCents = (outputTokens / 1_000_000) * outputPricePerMillion;
  totalCostCents = inputCostCents + outputCostCents;
  inputCostRounded = Math.round(inputCostCents * 10000) / 10000;
  outputCostRounded = Math.round(outputCostCents * 10000) / 10000;
}

if (totalCostCents < 0.00001) return;
const costInCentsRounded = Math.round(totalCostCents * 10000) / 10000;
```

Remove the old cost calculation block (lines 436-451) and replace with above.

**Step 4: Commit**

```bash
git add packages/api-gateway/src/worker.ts
git commit -m "feat: support fixed per-request pricing in deductBalance"
```

---

### Task 3: Add DOSafe proxy routing in worker

**Files:**
- Modify: `packages/api-gateway/src/worker.ts` (add `DOSAFE_BACKEND_URL` to Env, add routing before vLLM forward)

**Step 1: Add `DOSAFE_BACKEND_URL` to Env interface**

At `worker.ts:1-8`:

```typescript
export interface Env {
  DB: D1Database;
  VLLM_BACKEND_URL: string;
  VLLM_API_KEY: string;
  DASHBOARD_SECRET: string;
  SUPABASE_URL: string;
  SUPABASE_SERVICE_KEY: string;
  DOSAFE_BACKEND_URL: string; // https://dosafe.io
}
```

**Step 2: Add DOSafe routing map**

After the `TOKENS_PER_CREDIT` constant (line 172), add:

```typescript
// DOSafe endpoint mapping: /v1/dosafe/<name> -> /api/<name>
const DOSAFE_ROUTES: Record<string, string> = {
  'url-check': 'dosafe/url-check',
  'entity-check': 'dosafe/entity-check',
  'detect': 'dosafe/detect',
};
```

**Step 3: Add DOSafe handler before vLLM forward**

In the `fetch()` handler, after the rate limit section (around line 292) and before the "Forward request to vLLM" comment (line 294), insert:

```typescript
// DOSafe proxy endpoints
const dosafePath = url.pathname.match(/^\/v1\/dosafe\/(.+)$/)?.[1];
if (dosafePath && DOSAFE_ROUTES[dosafePath]) {
  const modelId = DOSAFE_ROUTES[dosafePath];
  const backendEndpoint = dosafePath; // url-check, entity-check, detect

  if (request.method !== 'POST') {
    return errorResponse(405, 'DOSafe endpoints only accept POST requests');
  }

  const startTime = Date.now();
  const dosafeBacked = `${env.DOSAFE_BACKEND_URL}/api/${backendEndpoint}`;

  let backendResponse: Response;
  try {
    backendResponse = await fetch(dosafeBacked, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: request.body,
    });
  } catch (error) {
    return errorResponse(502, 'DOSafe backend unavailable');
  }

  const latencyMs = Date.now() - startTime;
  const responseBody = await backendResponse.text();

  // Log usage and deduct fixed cost (async)
  if (backendResponse.ok) {
    ctx.waitUntil(
      Promise.all([
        logUsage(env.DB, {
          apiKeyId: keyRecord.id,
          userId: keyRecord.user_id,
          model: modelId,
          promptTokens: 0,
          completionTokens: 0,
          totalTokens: 0,
          endpoint: url.pathname,
          statusCode: backendResponse.status,
          latencyMs,
        }),
        deductBalance(env.DB, supabase, keyRecord.user_id, keyRecord.id, modelId, 0, 0),
      ])
    );
  }

  return new Response(responseBody, {
    status: backendResponse.status,
    headers: {
      'Content-Type': 'application/json',
      ...corsHeaders(),
    },
  });
}
```

**Step 4: Commit**

```bash
git add packages/api-gateway/src/worker.ts
git commit -m "feat: add /v1/dosafe/* proxy routes to API gateway"
```

---

### Task 4: Set DOSAFE_BACKEND_URL secret and deploy

**Step 1: Add the secret**

```bash
cd d:/Projects/DOSafe/packages/api-gateway
printf "https://dosafe.io" | npx wrangler secret put DOSAFE_BACKEND_URL
```

**Step 2: Add DOSAFE_BACKEND_URL to wrangler.toml vars (for local dev)**

In `wrangler.toml`, add under `[vars]`:

```toml
DOSAFE_BACKEND_URL = "https://dosafe.io"
```

**Step 3: Deploy worker**

```bash
npx wrangler deploy
```

**Step 4: Test the endpoints**

```bash
# Get an API key (use existing dos_sk_* key)
# Test url-check
curl -X POST https://api.dos.ai/v1/dosafe/url-check \
  -H "Authorization: Bearer dos_sk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://google.com"}'

# Test entity-check
curl -X POST https://api.dos.ai/v1/dosafe/entity-check \
  -H "Authorization: Bearer dos_sk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"entityType":"domain","entityId":"google.com"}'

# Test detect (may be slow — calls vLLM)
curl -X POST https://api.dos.ai/v1/dosafe/detect \
  -H "Authorization: Bearer dos_sk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text":"This is a test text for AI detection","lang":"en","task":"ai_detection"}'
```

Expected: All 3 return valid JSON responses with riskLevel/scores.

**Step 5: Verify billing deduction**

```bash
npx wrangler d1 execute dos-api --remote --command="SELECT * FROM usage_transactions ORDER BY created_at DESC LIMIT 5"
```

Expected: Rows with `model = 'dosafe/url-check'` etc.

**Step 6: Commit wrangler.toml**

```bash
git add packages/api-gateway/wrangler.toml
git commit -m "feat: add DOSAFE_BACKEND_URL config and deploy"
```

---

### Task 5: Update Chrome extension to use gateway API

**Files:**
- Modify: `apps/extension/background.js`
- Modify: `apps/extension/manifest.json` (add api.dos.ai to host_permissions if needed)

**Step 1: Check manifest permissions**

Read `apps/extension/manifest.json` and ensure `host_permissions` includes `https://api.dos.ai/*`. Add if missing.

**Step 2: Add API config and URL check to background.js**

Add auto URL-check on tab navigation using the gateway:

```javascript
// DOSafe API Gateway config
const DOSAFE_API = {
  urlCheck: "https://api.dos.ai/v1/dosafe/url-check",
  entityCheck: "https://api.dos.ai/v1/dosafe/entity-check",
};

// Get stored API key
async function getApiKey() {
  const { dosApiKey } = await chrome.storage.sync.get("dosApiKey");
  return dosApiKey || null;
}

// URL check with gateway auth
async function checkUrl(url) {
  const apiKey = await getApiKey();
  if (!apiKey) return null;

  try {
    const res = await fetch(DOSAFE_API.urlCheck, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${apiKey}`,
      },
      body: JSON.stringify({ url }),
    });
    if (!res.ok) return null;
    return await res.json();
  } catch {
    return null;
  }
}

// Badge colors by risk level
const BADGE_COLORS = {
  safe: "#22c55e",
  low: "#86efac",
  medium: "#facc15",
  high: "#f97316",
  critical: "#ef4444",
};

// Check URL on tab navigation
chrome.tabs.onUpdated.addListener(async (tabId, changeInfo, tab) => {
  if (changeInfo.status !== "complete" || !tab.url) return;
  if (!tab.url.startsWith("http")) return;

  const result = await checkUrl(tab.url);
  if (!result) return;

  const color = BADGE_COLORS[result.riskLevel] || "#9ca3af";
  const text = result.riskLevel === "safe" ? "" : result.riskLevel.charAt(0).toUpperCase();

  chrome.action.setBadgeBackgroundColor({ tabId, color });
  chrome.action.setBadgeText({ tabId, text });
});
```

**Step 3: Commit**

```bash
git add apps/extension/background.js apps/extension/manifest.json
git commit -m "feat: extension auto URL-check via DOS.AI gateway"
```
