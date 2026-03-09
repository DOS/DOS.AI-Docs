# DOSafe 3-Tier Quota System Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement a unified 3-tier quota system (anonymous/free/paid) for DOSafe endpoints across extension, Telegram bot, and direct API consumers, leveraging Supabase as the primary data layer.

**Architecture:** All quota tracking lives in Supabase (`dosafe` schema). The API gateway (`api.dos.ai`) acts as a thin proxy — it reads `X-Client-Type`/`X-Client-ID` headers for anonymous users or validates Supabase JWT for authenticated users, then calls a Supabase RPC to atomically check+consume quota. Extension authenticates via `id.dos.me` OAuth redirect flow (no Firebase SDK).

**Tech Stack:** Supabase (Postgres + Auth + RPC), Cloudflare Workers (D1 + proxy), Chrome Extension Manifest V3

---

## Existing Infrastructure (DO NOT recreate)

- **Supabase `dosafe.bot_quota`** — Telegram bot daily quota (chat_id, used, reset_at)
- **Supabase `dosai.dosafe_usage`** — Web app monthly quota (user_id, usage_date, endpoint, check_count)
- **D1 `rate_limit_log`** — Per-minute rate limiting at gateway
- **D1 `billing` + `api_keys`** — API key auth + balance for paid API users
- **`dosafe-quota.ts`** — Web app quota logic (anonymous in-memory, authenticated via Supabase)
- **`id.dos.me`** — Supabase Auth with redirect whitelist (dosafe.io already whitelisted)

---

## Task 1: Create unified quota tables in Supabase

**Files:**
- Create migration: `d:/Projects/DOSafe/supabase/migrations/YYYYMMDD_unified_quota.sql`

### What to build:

**Table `dosafe.quota_config`** — Configurable limits per client_type × tier:

```sql
CREATE TABLE IF NOT EXISTS dosafe.quota_config (
  id            SERIAL PRIMARY KEY,
  client_type   TEXT NOT NULL,           -- 'extension', 'telegram', 'zalo', 'api'
  tier          TEXT NOT NULL,           -- 'anonymous', 'free', 'paid'
  daily_limit   INTEGER NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(client_type, tier)
);

INSERT INTO dosafe.quota_config (client_type, tier, daily_limit) VALUES
  ('extension',  'anonymous', 10),
  ('extension',  'free',      30),
  ('extension',  'paid',      -1),   -- -1 = unlimited (billed per-request)
  ('telegram',   'anonymous', 10),
  ('telegram',   'free',      30),
  ('telegram',   'paid',      -1),
  ('zalo',       'anonymous', 10),
  ('zalo',       'free',      30),
  ('zalo',       'paid',      -1),
  ('api',        'anonymous',  5),
  ('api',        'free',      20),
  ('api',        'paid',      -1);
```

**Table `dosafe.client_quota`** — Unified daily usage tracking:

```sql
CREATE TABLE IF NOT EXISTS dosafe.client_quota (
  client_type   TEXT NOT NULL,
  client_id     TEXT NOT NULL,
  user_id       UUID REFERENCES auth.users(id),  -- NULL for anonymous
  used          INTEGER NOT NULL DEFAULT 0,
  quota_date    DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (client_type, client_id, quota_date)
);

CREATE INDEX idx_client_quota_user ON dosafe.client_quota(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_client_quota_date ON dosafe.client_quota(quota_date);
```

**RPC `dosafe.consume_quota`** — Atomic check + increment:

```sql
CREATE OR REPLACE FUNCTION dosafe.consume_quota(
  p_client_type TEXT,
  p_client_id   TEXT,
  p_user_id     UUID DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_tier       TEXT;
  v_limit      INTEGER;
  v_used       INTEGER;
  v_today      DATE := CURRENT_DATE;
BEGIN
  -- Determine tier
  IF p_user_id IS NOT NULL THEN
    -- Check if user has paid balance (billing_accounts in public schema)
    IF EXISTS (
      SELECT 1 FROM public.billing_accounts
      WHERE user_id = p_user_id AND balance_cents > 0
    ) THEN
      v_tier := 'paid';
    ELSE
      v_tier := 'free';
    END IF;
  ELSE
    v_tier := 'anonymous';
  END IF;

  -- Get daily limit from config
  SELECT daily_limit INTO v_limit
  FROM dosafe.quota_config
  WHERE client_type = p_client_type AND tier = v_tier;

  -- Default fallback
  IF v_limit IS NULL THEN
    v_limit := 10;
  END IF;

  -- Unlimited (paid tier)
  IF v_limit = -1 THEN
    -- Upsert usage row for tracking (no limit enforcement)
    INSERT INTO dosafe.client_quota (client_type, client_id, user_id, used, quota_date)
    VALUES (p_client_type, p_client_id, p_user_id, 1, v_today)
    ON CONFLICT (client_type, client_id, quota_date)
    DO UPDATE SET
      used = dosafe.client_quota.used + 1,
      user_id = COALESCE(EXCLUDED.user_id, dosafe.client_quota.user_id),
      updated_at = now()
    RETURNING used INTO v_used;

    RETURN jsonb_build_object(
      'allowed', true,
      'used', v_used,
      'limit', -1,
      'tier', v_tier,
      'reset_at', (v_today + 1)::text
    );
  END IF;

  -- Check current usage
  SELECT used INTO v_used
  FROM dosafe.client_quota
  WHERE client_type = p_client_type
    AND client_id = p_client_id
    AND quota_date = v_today;

  IF v_used IS NULL THEN
    v_used := 0;
  END IF;

  -- Check if over limit
  IF v_used >= v_limit THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'used', v_used,
      'limit', v_limit,
      'tier', v_tier,
      'reset_at', (v_today + 1)::text
    );
  END IF;

  -- Consume one unit
  INSERT INTO dosafe.client_quota (client_type, client_id, user_id, used, quota_date)
  VALUES (p_client_type, p_client_id, p_user_id, 1, v_today)
  ON CONFLICT (client_type, client_id, quota_date)
  DO UPDATE SET
    used = dosafe.client_quota.used + 1,
    user_id = COALESCE(EXCLUDED.user_id, dosafe.client_quota.user_id),
    updated_at = now()
  RETURNING used INTO v_used;

  RETURN jsonb_build_object(
    'allowed', true,
    'used', v_used,
    'limit', v_limit,
    'tier', v_tier,
    'reset_at', (v_today + 1)::text
  );
END;
$$;
```

**RPC `dosafe.get_quota`** — Read current usage without consuming:

```sql
CREATE OR REPLACE FUNCTION dosafe.get_quota(
  p_client_type TEXT,
  p_client_id   TEXT,
  p_user_id     UUID DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_tier       TEXT;
  v_limit      INTEGER;
  v_used       INTEGER := 0;
  v_today      DATE := CURRENT_DATE;
BEGIN
  -- Determine tier (same logic as consume_quota)
  IF p_user_id IS NOT NULL THEN
    IF EXISTS (
      SELECT 1 FROM public.billing_accounts
      WHERE user_id = p_user_id AND balance_cents > 0
    ) THEN
      v_tier := 'paid';
    ELSE
      v_tier := 'free';
    END IF;
  ELSE
    v_tier := 'anonymous';
  END IF;

  SELECT daily_limit INTO v_limit
  FROM dosafe.quota_config
  WHERE client_type = p_client_type AND tier = v_tier;

  IF v_limit IS NULL THEN v_limit := 10; END IF;

  SELECT cq.used INTO v_used
  FROM dosafe.client_quota cq
  WHERE cq.client_type = p_client_type
    AND cq.client_id = p_client_id
    AND cq.quota_date = v_today;

  RETURN jsonb_build_object(
    'allowed', v_limit = -1 OR COALESCE(v_used, 0) < v_limit,
    'used', COALESCE(v_used, 0),
    'limit', v_limit,
    'tier', v_tier,
    'reset_at', (v_today + 1)::text
  );
END;
$$;
```

**Cleanup cron** (optional, add later): Delete rows older than 90 days from `client_quota`.

**Grant permissions:**
```sql
GRANT USAGE ON SCHEMA dosafe TO service_role;
GRANT ALL ON dosafe.quota_config TO service_role;
GRANT ALL ON dosafe.client_quota TO service_role;
GRANT EXECUTE ON FUNCTION dosafe.consume_quota TO service_role;
GRANT EXECUTE ON FUNCTION dosafe.get_quota TO service_role;
```

### Migrate existing bot_quota data:

```sql
-- Migrate existing telegram bot quota data
INSERT INTO dosafe.client_quota (client_type, client_id, used, quota_date)
SELECT 'telegram', chat_id::text, used, CURRENT_DATE
FROM dosafe.bot_quota
WHERE used > 0
ON CONFLICT DO NOTHING;
```

---

## Task 2: Update API Gateway to support anonymous quota

**Files:**
- Modify: `d:/Projects/DOSafe/packages/api-gateway/src/worker.ts`

### What to build:

Update the DOSafe proxy handler (currently at the `dosafeMatch` block, ~line 294) to:

1. **Check for JWT first** (existing `Authorization: Bearer` flow)
2. **If no JWT**, read `X-Client-Type` and `X-Client-ID` headers
3. **Call Supabase RPC** `dosafe.consume_quota` with appropriate params
4. **If quota exceeded**, return 429 with quota info
5. **If allowed**, proxy to dosafe.io and return response with `X-Quota-*` headers

```typescript
// New headers to read for anonymous clients
const clientType = request.headers.get('X-Client-Type') || 'api';
const clientId = request.headers.get('X-Client-ID') || clientIp;

// Call Supabase RPC for quota check
const quotaRes = await fetch(`${env.SUPABASE_URL}/rest/v1/rpc/consume_quota`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'apikey': env.SUPABASE_SERVICE_KEY,
    'Authorization': `Bearer ${env.SUPABASE_SERVICE_KEY}`,
    'Content-Profile': 'dosafe',
  },
  body: JSON.stringify({
    p_client_type: clientType,
    p_client_id: clientId,
    p_user_id: userId || null,  // from JWT if present
  }),
});
```

Add response headers:
```typescript
response.headers.set('X-Quota-Used', quota.used);
response.headers.set('X-Quota-Limit', quota.limit);
response.headers.set('X-Quota-Tier', quota.tier);
response.headers.set('X-Quota-Reset', quota.reset_at);
```

**Important:** Keep existing D1 rate limiting (per-minute) as Layer 1. Supabase quota is Layer 2 (daily).

### Also add a `/v1/dosafe/quota` endpoint:

GET endpoint that returns current quota status without consuming:
- Reads same headers (`X-Client-Type`, `X-Client-ID`, or JWT)
- Calls `dosafe.get_quota` RPC
- Returns JSON quota object

---

## Task 3: Update extension for anonymous + auth quota

**Files:**
- Modify: `d:/Projects/DOSafe/apps/extension/background.js`
- Modify: `d:/Projects/DOSafe/apps/extension/popup.html` (add login/quota UI)

### What to build:

**3a. Anonymous mode (default, no login needed):**

```javascript
// Generate and persist instance ID on first install
chrome.runtime.onInstalled.addListener(async () => {
  const { clientId } = await chrome.storage.local.get('clientId');
  if (!clientId) {
    await chrome.storage.local.set({ clientId: crypto.randomUUID() });
  }
});

// API calls include client headers
async function callDosafeApi(endpoint, body) {
  const { clientId } = await chrome.storage.local.get('clientId');
  const res = await fetch(`https://api.dos.ai/v1/dosafe/${endpoint}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Client-Type': 'extension',
      'X-Client-ID': clientId,
    },
    body: JSON.stringify(body),
  });
  return res;
}
```

**3b. Authenticated mode (login via id.dos.me):**

When user clicks "Login" in side panel:
1. Open `id.dos.me/login?redirect=https://dosafe.io/auth/extension-callback`
2. User logs in on id.dos.me (Supabase Auth)
3. Redirect to `dosafe.io/auth/extension-callback#access_token=...&refresh_token=...`
4. The callback page sends tokens back to extension via `chrome.runtime.sendMessage` (requires `externally_connectable` in manifest) OR stores in a page the extension can read

**Simpler approach:** Use `chrome.identity.launchWebAuthFlow` if available, or just:
1. Extension opens tab to `dosafe.io/auth/extension?ext_id=<extension-id>`
2. Page redirects to `id.dos.me/login?redirect=dosafe.io/auth/extension-callback`
3. After login, callback page extracts tokens from URL hash
4. Callback page calls `chrome.runtime.sendMessage(extensionId, { tokens })` via `externally_connectable`
5. Extension stores tokens in `chrome.storage.local` (session tokens, not API keys)
6. Extension refreshes tokens via Supabase token refresh endpoint

**API calls with auth:**
```javascript
async function callDosafeApi(endpoint, body) {
  const { clientId, authToken } = await chrome.storage.local.get(['clientId', 'authToken']);
  const headers = {
    'Content-Type': 'application/json',
    'X-Client-Type': 'extension',
    'X-Client-ID': clientId,
  };
  if (authToken) {
    headers['Authorization'] = `Bearer ${authToken}`;
  }
  // ... fetch
}
```

**Remove** the old `dosApiKey` storage approach — replace with this client ID + optional JWT approach.

---

## Task 4: Create extension auth callback page on dosafe.io

**Files:**
- Create: `d:/Projects/DOSafe/apps/web/src/app/auth/extension-callback/page.tsx`

### What to build:

A simple page that:
1. Reads tokens from URL hash (set by id.dos.me redirect)
2. Sends tokens to extension via `chrome.runtime.sendMessage`
3. Shows success/failure UI

```tsx
'use client';
import { useEffect, useState } from 'react';

export default function ExtensionCallbackPage() {
  const [status, setStatus] = useState('Connecting to extension...');

  useEffect(() => {
    const hash = window.location.hash.substring(1);
    const params = new URLSearchParams(hash);
    const accessToken = params.get('access_token');
    const refreshToken = params.get('refresh_token');

    if (!accessToken) {
      setStatus('Login failed. Please try again.');
      return;
    }

    // Send to extension via externally_connectable
    // Extension ID will be known after publishing
    const EXTENSION_ID = chrome.runtime?.id; // won't work from web page

    // Alternative: use BroadcastChannel or postMessage
    // The extension's content script on dosafe.io/* can listen for this
    window.postMessage({
      type: 'DOSAFE_AUTH_TOKENS',
      accessToken,
      refreshToken,
    }, '*');

    setStatus('Login successful! You can close this tab.');
    // Auto-close after delay
    setTimeout(() => window.close(), 2000);
  }, []);

  return (
    <div className="min-h-screen flex items-center justify-center">
      <p className="text-lg">{status}</p>
    </div>
  );
}
```

**Extension content script** (on dosafe.io) listens for the message:
```javascript
// content-dosafe-auth.js - runs on dosafe.io/auth/extension-callback
window.addEventListener('message', (event) => {
  if (event.data?.type === 'DOSAFE_AUTH_TOKENS') {
    chrome.runtime.sendMessage({
      type: 'DOSAFE_AUTH_TOKENS',
      accessToken: event.data.accessToken,
      refreshToken: event.data.refreshToken,
    });
  }
});
```

**Update manifest.json:**
```json
{
  "content_scripts": [
    // existing facebook content script...
    {
      "matches": ["https://dosafe.io/auth/extension-callback*"],
      "js": ["content-dosafe-auth.js"],
      "run_at": "document_idle"
    }
  ]
}
```

---

## Task 5: Add quota display to extension side panel

**Files:**
- Modify: `d:/Projects/DOSafe/apps/extension/popup.html`
- Modify or create: `d:/Projects/DOSafe/apps/extension/popup.js`

### What to build:

Add to the side panel UI:
1. **Quota bar** showing `used / limit` with progress indicator
2. **Login button** (if anonymous) — "Sign in for 3x more scans"
3. **User info** (if logged in) — email/name + logout button
4. **Upgrade button** (if free tier) — links to dosafe.io pricing

Read quota from response headers (`X-Quota-*`) after each API call and update the UI.

Also fetch quota on panel open via `GET /v1/dosafe/quota`.

---

## Task 6: Update Telegram bot to use unified quota

**Files:**
- Modify: Telegram bot code (identify location in `d:/Projects/DOSafe/`)

### What to build:

Update bot to call `dosafe.consume_quota` RPC with:
- `p_client_type: 'telegram'`
- `p_client_id: chatId.toString()`
- `p_user_id: null` (or linked user_id if bot user has linked DOS.Me account)

Replace direct `bot_quota` table queries with the new RPC.

---

## Task 7: Cleanup and migration

**Files:**
- Migration file to deprecate old `dosafe.bot_quota` table
- Update `dosafe-quota.ts` in web app to align with new system (optional, web app can keep its own quota logic for now since it tracks word-level usage)

### Notes:
- Keep `dosai.dosafe_usage` for web app's word-level tracking (different granularity)
- The unified `dosafe.client_quota` is for check-count-based quota (extension, bot, API)
- Old `bot_quota` data migrated in Task 1, table kept for backward compat during transition

---

## Deployment Order

1. Task 1 — Supabase migration (no breaking changes)
2. Task 2 — API gateway update (backward compatible, new headers optional)
3. Task 3 + 4 — Extension update (new version)
4. Task 5 — UI polish
5. Task 6 — Bot migration
6. Task 7 — Cleanup

## Testing

After each task:
- Verify quota counting via Supabase dashboard SQL
- Test anonymous flow (no auth headers)
- Test authenticated flow (with JWT)
- Test quota exhaustion (429 response)
- Test daily reset (change date in DB)
