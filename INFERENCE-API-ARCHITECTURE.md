# Inference API Architecture (Serverless Inference)

## Overview

This document describes the architecture for DOS.ai Serverless Inference API, allowing users to access LLM models via OpenAI-compatible API with API key authentication and usage-based billing.

## Current Infrastructure

- **vLLM** container running with models loaded
- **Cloudflare Tunnel** exposing vLLM at `api.dos.ai`
- **Next.js Dashboard** (apps/app) for user management
- **Next.js Landing** (apps/web) for marketing

---

## Architecture Design

### High-Level Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   User App      │────▶│ Cloudflare      │────▶│  vLLM Container │
│   (curl, SDK)   │     │ Worker (Gateway)│     │  (GPU Server)   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │  Cloudflare D1  │
                        │  (Keys, Usage)  │
                        └─────────────────┘
                               ▲
                               │
┌─────────────────┐     ┌─────────────────┐
│   User          │────▶│  Dashboard      │
│   (Browser)     │     │  (apps/app)     │
└─────────────────┘     └─────────────────┘
```

### Request Flow

```
1. User calls API
   POST https://api.dos.ai/v1/chat/completions
   Authorization: Bearer dos_sk_xxx

2. Cloudflare Worker receives request
   - Extract API key from header
   - Query D1 to validate key
   - If invalid → 401 Unauthorized

3. Worker forwards to vLLM
   - Strip/replace auth header
   - Forward request body as-is

4. vLLM processes and responds
   - Returns OpenAI-format response
   - Includes usage (prompt_tokens, completion_tokens)

5. Worker logs usage
   - Extract token counts from response
   - Insert into D1 usage table (async)

6. Return response to user
```

---

## Components

### 1. Cloudflare D1 Database

**Schema:**

```sql
-- Users table (if not using external auth)
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- API Keys table
CREATE TABLE api_keys (
  id TEXT PRIMARY KEY,              -- 'key_xxxx'
  user_id TEXT NOT NULL,
  key_prefix TEXT NOT NULL,         -- 'dos_sk_abc...' (first 8 chars for display)
  key_hash TEXT NOT NULL,           -- SHA256 hash of full key
  name TEXT DEFAULT 'Default',      -- User-friendly name
  status TEXT DEFAULT 'active',     -- 'active', 'revoked'
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_used_at DATETIME,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Usage logs table
CREATE TABLE usage_logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  api_key_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  model TEXT NOT NULL,
  prompt_tokens INTEGER NOT NULL,
  completion_tokens INTEGER NOT NULL,
  total_tokens INTEGER NOT NULL,
  endpoint TEXT NOT NULL,           -- '/v1/chat/completions'
  status_code INTEGER NOT NULL,
  latency_ms INTEGER,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (api_key_id) REFERENCES api_keys(id)
);

-- Daily usage aggregation (for faster queries)
CREATE TABLE usage_daily (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  date DATE NOT NULL,
  total_requests INTEGER DEFAULT 0,
  total_prompt_tokens INTEGER DEFAULT 0,
  total_completion_tokens INTEGER DEFAULT 0,
  total_tokens INTEGER DEFAULT 0,
  cost_usd REAL DEFAULT 0,
  UNIQUE(user_id, date)
);

-- Indexes
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_user ON api_keys(user_id);
CREATE INDEX idx_usage_logs_key ON usage_logs(api_key_id);
CREATE INDEX idx_usage_logs_user_date ON usage_logs(user_id, created_at);
CREATE INDEX idx_usage_daily_user ON usage_daily(user_id, date);
```

### 2. Cloudflare Worker (API Gateway)

**Location:** `packages/api-gateway/` or separate repo

**worker.ts:**

```typescript
export interface Env {
  DB: D1Database;
  VLLM_BACKEND_URL: string;  // Internal vLLM URL
  RATE_LIMIT_KV: KVNamespace; // Optional: for rate limiting
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // Health check endpoint
    if (url.pathname === '/health') {
      return new Response('OK', { status: 200 });
    }

    // Only handle /v1/* endpoints
    if (!url.pathname.startsWith('/v1/')) {
      return new Response('Not Found', { status: 404 });
    }

    // Extract API key
    const authHeader = request.headers.get('Authorization');
    if (!authHeader?.startsWith('Bearer ')) {
      return errorResponse(401, 'Missing API key. Include Authorization: Bearer dos_sk_xxx');
    }

    const apiKey = authHeader.replace('Bearer ', '');

    // Validate key format
    if (!apiKey.startsWith('dos_sk_')) {
      return errorResponse(401, 'Invalid API key format');
    }

    // Validate key against D1
    const keyHash = await hashKey(apiKey);
    const keyRecord = await env.DB.prepare(
      'SELECT id, user_id, status FROM api_keys WHERE key_hash = ?'
    ).bind(keyHash).first();

    if (!keyRecord) {
      return errorResponse(401, 'Invalid API key');
    }

    if (keyRecord.status !== 'active') {
      return errorResponse(401, 'API key has been revoked');
    }

    // Update last_used_at (async, don't block)
    ctx.waitUntil(
      env.DB.prepare('UPDATE api_keys SET last_used_at = ? WHERE id = ?')
        .bind(new Date().toISOString(), keyRecord.id)
        .run()
    );

    // Forward request to vLLM
    const startTime = Date.now();
    const backendUrl = `${env.VLLM_BACKEND_URL}${url.pathname}`;

    const backendResponse = await fetch(backendUrl, {
      method: request.method,
      headers: {
        'Content-Type': 'application/json',
      },
      body: request.body,
    });

    const latencyMs = Date.now() - startTime;
    const responseBody = await backendResponse.json();

    // Log usage (async)
    if (backendResponse.ok && responseBody.usage) {
      ctx.waitUntil(
        logUsage(env.DB, {
          apiKeyId: keyRecord.id,
          userId: keyRecord.user_id,
          model: responseBody.model,
          promptTokens: responseBody.usage.prompt_tokens,
          completionTokens: responseBody.usage.completion_tokens,
          totalTokens: responseBody.usage.total_tokens,
          endpoint: url.pathname,
          statusCode: backendResponse.status,
          latencyMs,
        })
      );
    }

    // Return response with CORS headers
    return new Response(JSON.stringify(responseBody), {
      status: backendResponse.status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Authorization, Content-Type',
      },
    });
  },
};

async function hashKey(key: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(key);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

async function logUsage(db: D1Database, usage: UsageLog): Promise<void> {
  await db.prepare(`
    INSERT INTO usage_logs
    (api_key_id, user_id, model, prompt_tokens, completion_tokens, total_tokens, endpoint, status_code, latency_ms)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).bind(
    usage.apiKeyId,
    usage.userId,
    usage.model,
    usage.promptTokens,
    usage.completionTokens,
    usage.totalTokens,
    usage.endpoint,
    usage.statusCode,
    usage.latencyMs
  ).run();
}

function errorResponse(status: number, message: string): Response {
  return new Response(JSON.stringify({ error: { message, type: 'invalid_request_error' } }), {
    status,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

**wrangler.toml:**

```toml
name = "dos-api-gateway"
main = "src/worker.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "dos-inference"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

[vars]
VLLM_BACKEND_URL = "https://vllm-internal.dos.ai"
```

### 3. Dashboard API Routes (apps/app)

**File structure:**

```
apps/app/src/app/api/
├── keys/
│   ├── route.ts           # GET (list), POST (create)
│   └── [id]/
│       └── route.ts       # DELETE (revoke)
└── usage/
    └── route.ts           # GET usage stats
```

**apps/app/src/app/api/keys/route.ts:**

```typescript
import { getD1 } from '@/lib/cloudflare';
import { getUser } from '@/lib/auth';
import { nanoid } from 'nanoid';

// GET /api/keys - List user's API keys
export async function GET() {
  const user = await getUser();
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  const db = getD1();
  const keys = await db.prepare(`
    SELECT id, key_prefix, name, status, created_at, last_used_at
    FROM api_keys
    WHERE user_id = ? AND status = 'active'
    ORDER BY created_at DESC
  `).bind(user.id).all();

  return Response.json({ keys: keys.results });
}

// POST /api/keys - Create new API key
export async function POST(request: Request) {
  const user = await getUser();
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  const { name = 'Default' } = await request.json();

  // Generate key: dos_sk_<32 random chars>
  const keyId = `key_${nanoid(12)}`;
  const secretPart = nanoid(32);
  const fullKey = `dos_sk_${secretPart}`;
  const keyPrefix = `dos_sk_${secretPart.substring(0, 4)}...`;
  const keyHash = await hashKey(fullKey);

  const db = getD1();
  await db.prepare(`
    INSERT INTO api_keys (id, user_id, key_prefix, key_hash, name)
    VALUES (?, ?, ?, ?, ?)
  `).bind(keyId, user.id, keyPrefix, keyHash, name).run();

  // Return full key only once - user must save it
  return Response.json({
    id: keyId,
    key: fullKey,  // Only shown once!
    name,
    created_at: new Date().toISOString(),
  });
}

async function hashKey(key: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(key);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}
```

**apps/app/src/app/api/keys/[id]/route.ts:**

```typescript
import { getD1 } from '@/lib/cloudflare';
import { getUser } from '@/lib/auth';

// DELETE /api/keys/:id - Revoke API key
export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await getUser();
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  const db = getD1();

  // Verify ownership and revoke
  const result = await db.prepare(`
    UPDATE api_keys
    SET status = 'revoked'
    WHERE id = ? AND user_id = ?
  `).bind(params.id, user.id).run();

  if (result.changes === 0) {
    return Response.json({ error: 'Key not found' }, { status: 404 });
  }

  return Response.json({ success: true });
}
```

**apps/app/src/app/api/usage/route.ts:**

```typescript
import { getD1 } from '@/lib/cloudflare';
import { getUser } from '@/lib/auth';

// GET /api/usage?period=7d
export async function GET(request: Request) {
  const user = await getUser();
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 });

  const url = new URL(request.url);
  const period = url.searchParams.get('period') || '7d';

  const days = period === '30d' ? 30 : period === '24h' ? 1 : 7;
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);

  const db = getD1();

  // Get aggregated usage
  const usage = await db.prepare(`
    SELECT
      DATE(created_at) as date,
      COUNT(*) as requests,
      SUM(prompt_tokens) as prompt_tokens,
      SUM(completion_tokens) as completion_tokens,
      SUM(total_tokens) as total_tokens
    FROM usage_logs
    WHERE user_id = ? AND created_at >= ?
    GROUP BY DATE(created_at)
    ORDER BY date DESC
  `).bind(user.id, startDate.toISOString()).all();

  // Get totals
  const totals = await db.prepare(`
    SELECT
      COUNT(*) as total_requests,
      SUM(total_tokens) as total_tokens
    FROM usage_logs
    WHERE user_id = ? AND created_at >= ?
  `).bind(user.id, startDate.toISOString()).first();

  return Response.json({
    usage: usage.results,
    totals,
    period,
  });
}
```

### 4. Dashboard UI (apps/app)

**Pages to create:**

```
apps/app/src/app/
├── api-keys/
│   └── page.tsx          # API Keys management
├── usage/
│   └── page.tsx          # Usage dashboard
└── docs/
    └── page.tsx          # Quick start guide
```

**API Keys Page Features:**
- List all active keys (showing prefix only: `dos_sk_abc1...`)
- Create new key (show full key once, copy button)
- Revoke key
- Rename key

**Usage Page Features:**
- Chart: requests/tokens over time
- Total tokens used this month
- Cost estimate
- Per-key breakdown

---

## API Endpoints (User-Facing)

### OpenAI-Compatible Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat completions |
| `/v1/completions` | POST | Text completions |
| `/v1/models` | GET | List available models |
| `/v1/embeddings` | POST | Text embeddings |

### Request Format

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.3-70b",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "max_tokens": 100
  }'
```

### Response Format

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "llama-3.3-70b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 8,
    "total_tokens": 18
  }
}
```

### Error Format

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```

---

## Pricing Model

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Llama 3.3 70B | $0.20 | $0.20 |
| Llama 3.1 8B | $0.05 | $0.05 |
| Qwen 2.5 72B | $0.25 | $0.25 |

**Billing calculation:**

```
cost = (prompt_tokens * input_price + completion_tokens * output_price) / 1_000_000
```

---

## Security

1. **API Key Security**
   - Keys hashed with SHA-256 before storage
   - Full key shown only once at creation
   - Keys can be revoked instantly

2. **Rate Limiting** (optional)
   - Per-key rate limits using Cloudflare KV
   - Default: 100 requests/minute

3. **Request Validation**
   - Validate JSON body
   - Max request size
   - Timeout handling

---

## Implementation Phases

### Phase 1: MVP (Week 1)
- [ ] Create D1 database with schema
- [ ] Deploy Cloudflare Worker (basic auth + forwarding)
- [ ] API routes for key management
- [ ] Simple API Keys page in dashboard

### Phase 2: Usage Tracking (Week 2)
- [ ] Log all requests to D1
- [ ] Usage API endpoint
- [ ] Usage dashboard page with charts

### Phase 3: Polish (Week 3)
- [ ] Rate limiting
- [ ] Better error handling
- [ ] API documentation page
- [ ] SDK examples (Python, Node.js)

### Phase 4: Billing (Future)
- [ ] Stripe integration
- [ ] Prepaid credits system
- [ ] Usage alerts

---

## File Structure

```
packages/
└── api-gateway/
    ├── src/
    │   └── worker.ts
    ├── wrangler.toml
    └── package.json

apps/app/src/
├── app/
│   ├── api/
│   │   ├── keys/
│   │   │   ├── route.ts
│   │   │   └── [id]/route.ts
│   │   └── usage/
│   │       └── route.ts
│   ├── api-keys/
│   │   └── page.tsx
│   └── usage/
│       └── page.tsx
└── lib/
    └── cloudflare.ts     # D1 client helper

```

---

## Commands

```bash
# Create D1 database
wrangler d1 create dos-inference

# Run migrations
wrangler d1 execute dos-inference --file=./schema.sql

# Deploy worker
cd packages/api-gateway && wrangler deploy

# Local development
wrangler dev
```

---

## References

- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)
- [vLLM OpenAI Server](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
