# DOSafe API via DOS.AI Gateway

**Date:** 2026-03-05
**Status:** Approved

## Overview

Expose DOSafe safety endpoints through the DOS.AI API gateway (`api.dos.ai`) so that Chrome extensions, third-party apps, and other DOS products can call them using the existing `dos_sk_*` API key system with per-request billing.

## Routes

```
POST api.dos.ai/v1/dosafe/url-check      -> dosafe.io/api/url-check
POST api.dos.ai/v1/dosafe/entity-check   -> dosafe.io/api/entity-check
POST api.dos.ai/v1/dosafe/detect         -> dosafe.io/api/detect
```

## Architecture

Approach A: Proxy through Cloudflare Worker (chosen over shared middleware).

```
Client (Extension / 3rd party / DOS products)
  -> api.dos.ai/v1/dosafe/* (Cloudflare Worker)
     1. Validate API key (dos_sk_*)
     2. Check balance > 0
     3. Rate limit check
     4. Forward POST body -> dosafe.io/api/*
     5. Return response to client
     6. Async: log usage + deduct per-request cost
```

Extra hop latency (~10-20ms) is negligible since DOSafe endpoints already take 1-5s (WHOIS, Safe Browsing, vLLM inference, etc.)

## Pricing

Per-request fixed cost, deducted from the user's `billing_accounts.balance_cents`.

| Endpoint | model_id | Cost/request | $/1000 requests |
|---|---|---|---|
| `/v1/dosafe/url-check` | `dosafe/url-check` | 0.10 cents | $1.00 |
| `/v1/dosafe/entity-check` | `dosafe/entity-check` | 0.05 cents | $0.50 |
| `/v1/dosafe/detect` | `dosafe/detect` | 0.50 cents | $5.00 |

Detect costs more because it calls vLLM inference internally.

## Implementation Changes

### 1. Worker routing (`packages/api-gateway/src/worker.ts`)

Add a handler for `/v1/dosafe/*` paths that:
- Reuses existing auth, rate limit, and balance check logic
- Proxies the request body to `${DOSAFE_BACKEND_URL}/api/${endpoint}`
- Deducts a fixed cost per request (not token-based)
- Logs usage to `usage_transactions` with the dosafe model_id

### 2. New env var

- `DOSAFE_BACKEND_URL` = `https://dosafe.io` (Cloudflare Worker secret)

### 3. D1 model_pricing rows

```sql
INSERT INTO model_pricing (model_id, display_name, input_price_per_million, output_price_per_million, is_active, fixed_cost_cents)
VALUES
  ('dosafe/url-check', 'DOSafe URL Check', 0, 0, 1, 0.10),
  ('dosafe/entity-check', 'DOSafe Entity Check', 0, 0, 1, 0.05),
  ('dosafe/detect', 'DOSafe AI Detection', 0, 0, 1, 0.50);
```

Note: `fixed_cost_cents` is a new column added to `model_pricing`. When present and > 0, the worker uses this instead of token-based pricing.

### 4. Billing logic update

In `deductBalance()`, check if `fixed_cost_cents > 0` for the model. If so, use that directly instead of calculating from token counts.

### 5. Extension update (`apps/extension/background.js`)

Update default endpoints:
```javascript
urlCheckEndpoint: "https://api.dos.ai/v1/dosafe/url-check"
entityCheckEndpoint: "https://api.dos.ai/v1/dosafe/entity-check"
```

Add API key header to requests:
```javascript
headers: {
  "Content-Type": "application/json",
  "Authorization": "Bearer ${apiKey}"
}
```

## Request/Response Format

Unchanged — the gateway proxies request and response bodies as-is. See existing endpoint docs:

**POST /v1/dosafe/url-check**
- Request: `{ "url": "https://example.com" }`
- Response: `{ url, domain, riskLevel, riskSignals, checks, threatIntel, onChain }`

**POST /v1/dosafe/entity-check**
- Request: `{ "entityType": "phone", "entityId": "0912345678" }`
- Response: `{ entityType, entityId, riskLevel, riskSignals, threatIntel, onChain }`

**POST /v1/dosafe/detect**
- Request: `{ "text": "...", "lang": "vi", "task": "ai_detection" }`
- Response: AI detection results with probability scores
