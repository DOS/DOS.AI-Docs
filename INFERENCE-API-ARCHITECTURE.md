# Inference API Architecture (Serverless Inference)

## Overview

This document describes the architecture for DOS.ai Serverless Inference API, allowing users to access LLM models via OpenAI-compatible API with API key authentication and usage-based billing.

## Current Infrastructure (January 2025)

### Components

| Component | Location | Description |
|-----------|----------|-------------|
| **vLLM Server** | Local PC (localhost:8000) | Docker container running LLM models with GPU |
| **Cloudflare Tunnel** | cloudflared service | Exposes local vLLM at `inference.dos.ai` |
| **API Gateway** | Cloudflare Worker | `api.dos.ai` - Auth, billing, rate limiting |
| **D1 Database** | Cloudflare Edge | API keys, billing balances, usage logs |
| **Supabase** | PostgreSQL | Users, orgs, Stripe customers, payments |
| **Dashboard** | Vercel (apps/app) | `app.dos.ai` - User management |

### Tunnel Configuration

The vLLM server runs locally and is exposed via Cloudflare Tunnel:

- **Tunnel Name**: AI API
- **Tunnel ID**: `04915ff2-6f0d-49ab-8115-c36c74edbff9`
- **Account**: `5f2a58925e790423dfafa0e6bee46b28`
- **Public Hostname**: `inference.dos.ai` → `http://localhost:8000`
- **Protocol**: QUIC (with HTTP/2 fallback)

**Service Installation:**
```powershell
# Token-based service installation
$token = "<tunnel-token-from-dashboard>"
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" service install $token
Start-Service cloudflared
```

**Known Issues:**
1. **QUIC blocked by firewall**: If UDP port 7844 is blocked, cloudflared will timeout for ~2 minutes before falling back to HTTP/2. Log shows: `INF Switching to fallback protocol http2`
2. **Service hang on restart**: `Restart-Service cloudflared` can hang indefinitely. Use `Get-Process cloudflared | Stop-Process -Force` before starting.

---

## Architecture Design

### High-Level Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Bexly/Apps    │────▶│ api.dos.ai      │────▶│ inference.dos.ai│────▶│  vLLM (Local)   │
│   (API Client)  │     │ (CF Worker)     │     │ (CF Tunnel)     │     │  localhost:8000 │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │  Cloudflare D1  │
                        │  (Keys, Billing)│
                        └─────────────────┘
                               │
                               │ (Future: migrate billing)
                               ▼
┌─────────────────┐     ┌─────────────────┐
│   User          │────▶│  Dashboard      │────▶ Supabase (Users, Orgs, Stripe)
│   (Browser)     │     │  app.dos.ai     │
└─────────────────┘     └─────────────────┘
```

### Request Flow with Billing

```
1. Client sends request
   POST https://api.dos.ai/v1/chat/completions
   Authorization: Bearer dos_sk_xxx

2. Cloudflare Worker (api.dos.ai)
   a. Extract API key from Authorization header
   b. Hash key with SHA-256
   c. Query D1 to validate key and get user_id
   d. Check billing balance (must have sufficient credits)
   e. If insufficient balance → 402 Payment Required

3. Forward to inference backend
   POST https://inference.dos.ai/v1/chat/completions
   (No auth needed - tunnel is internal)

4. Cloudflare Tunnel routes to local vLLM
   POST http://localhost:8000/v1/chat/completions

5. vLLM processes and returns response
   - Includes usage: { prompt_tokens, completion_tokens, total_tokens }

6. Worker calculates and deducts cost
   - Get model pricing from D1.model_pricing table
   - Calculate: cost = (tokens / 1M) × price_per_million
   - Apply minimum charge of $0.01
   - Deduct from D1.billing.balance_cents
   - Log transaction to D1.usage_transactions

7. Return response to client
```

---

## Billing System (Current State)

### Token-Based Billing Flow

Located in `packages/api-gateway/src/worker.ts`:

```typescript
async function deductBalance(
  db: D1Database,
  userId: string,
  apiKeyId: string,
  model: string,
  inputTokens: number,
  outputTokens: number
): Promise<{ success: boolean; newBalance?: number; error?: string }> {
  // Get model pricing from D1
  const pricing = await db.prepare(
    'SELECT input_price_per_million, output_price_per_million FROM model_pricing WHERE model_id = ?'
  ).bind(model).first();

  // Default pricing if model not in table
  const inputPricePerMillion = pricing?.input_price_per_million || 10;   // $0.10/1M
  const outputPricePerMillion = pricing?.output_price_per_million || 10; // $0.10/1M

  // Calculate cost in cents
  const inputCostCents = Math.ceil((inputTokens / 1_000_000) * inputPricePerMillion * 100) / 100;
  const outputCostCents = Math.ceil((outputTokens / 1_000_000) * outputPricePerMillion * 100) / 100;
  const totalCostCents = inputCostCents + outputCostCents;

  // Minimum charge: 1 cent
  const finalCostCents = Math.max(1, Math.round(totalCostCents * 100) / 100);

  // Deduct from balance
  const result = await db.prepare(`
    UPDATE billing
    SET balance_cents = balance_cents - ?, updated_at = datetime('now')
    WHERE user_id = ? AND balance_cents >= ?
  `).bind(finalCostCents, userId, finalCostCents).run();

  if (result.changes === 0) {
    return { success: false, error: 'Insufficient balance' };
  }

  // Log the transaction
  await db.prepare(`
    INSERT INTO usage_transactions
    (user_id, api_key_id, model, input_tokens, output_tokens, cost_cents, created_at)
    VALUES (?, ?, ?, ?, ?, ?, datetime('now'))
  `).bind(userId, apiKeyId, model, inputTokens, outputTokens, finalCostCents).run();

  return { success: true };
}
```

### Pricing Table

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Llama 3.3 70B | $0.20 | $0.20 |
| Llama 3.1 8B | $0.05 | $0.05 |
| Qwen 2.5 72B | $0.25 | $0.25 |
| Default | $0.10 | $0.10 |

### Adding Credits (Admin Only)

Via Wrangler CLI:
```bash
# Find user_id from Supabase by email
# Then update D1 directly:
wrangler d1 execute dos-inference --command="UPDATE billing SET balance_cents = balance_cents + 100000 WHERE user_id = 'xxx'"
# 100000 cents = $1000
```

---

## Database Architecture (Hybrid)

### Cloudflare D1 (Edge-Optimized)

**Purpose**: Low-latency operations at the edge (API key validation, balance checks, usage logging)

**Tables**:
- `api_keys` - API key storage with SHA-256 hashes
- `billing` - User balance in cents
- `usage_transactions` - Per-request usage logs
- `usage_logs` - Legacy usage tracking
- `model_pricing` - Per-model pricing configuration

**Why D1**:
- Sub-millisecond latency at edge
- Co-located with API Gateway Worker
- Handles high-frequency reads (every API request)

### Supabase (PostgreSQL)

**Purpose**: Business data, authentication, Stripe integration

**Tables**:
- `auth.users` - Supabase Auth users
- `public.organizations` - User organizations
- `public.organization_members` - Org membership
- `public.stripe_customers` - Stripe customer IDs
- `public.payment_transactions` - Payment history
- `public.subscription_plans` - Available plans

**Why Supabase**:
- Row-level security (RLS) for multi-tenant data
- Direct Stripe webhook integration
- Full-featured PostgreSQL for complex queries
- Already handles auth and user management

### Future Migration

See [BILLING-MIGRATION-ROADMAP.md](./BILLING-MIGRATION-ROADMAP.md) for the plan to:
1. Keep API keys and rate limiting in D1 (edge-fast)
2. Migrate billing/credits to Supabase (single source of truth)
3. Remove duplicate data and inconsistency risk

---

## API Endpoints

### Public Inference API (api.dos.ai)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat completions |
| `/v1/completions` | POST | Text completions |
| `/v1/models` | GET | List available models |
| `/v1/embeddings` | POST | Text embeddings |
| `/health` | GET | Health check |

### Request Example

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

### Error Responses

| Status | Error | Description |
|--------|-------|-------------|
| 401 | `invalid_api_key` | API key missing or invalid |
| 402 | `insufficient_balance` | Not enough credits |
| 429 | `rate_limit_exceeded` | Too many requests |
| 500 | `internal_error` | Backend error |

---

## Dashboard API Routes (apps/app)

### Key Management

| Route | Method | Description |
|-------|--------|-------------|
| `/api/keys` | GET | List user's API keys |
| `/api/keys` | POST | Create new API key |
| `/api/keys/[id]` | DELETE | Revoke API key |

### Usage & Billing

| Route | Method | Description |
|-------|--------|-------------|
| `/api/usage` | GET | Get usage statistics |
| `/api/billing` | GET | Get billing info |
| `/api/billing/add-credits` | POST | Add credits (admin) |

---

## Security

### API Key Security
- Keys hashed with SHA-256 before storage
- Full key shown only once at creation
- Keys can be revoked instantly
- Format: `dos_sk_<32 random chars>`

### Tunnel Security
- Cloudflare Tunnel handles TLS termination
- No exposed ports on local machine
- Token-based authentication to Cloudflare

### Rate Limiting
- Per-key rate limits configurable
- Default: 100 requests/minute
- Uses D1 for counter storage

---

## Operational Runbook

### Check Tunnel Status

```powershell
# Check service status
Get-Service cloudflared

# View recent logs
Get-EventLog -LogName Application -Source cloudflared -Newest 10

# Test endpoint
curl https://inference.dos.ai/health
```

### Restart Tunnel

```powershell
# Force kill and restart (safe)
Get-Process cloudflared -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 3
Start-Service cloudflared

# Wait for connection (QUIC timeout can take 2 min)
Start-Sleep -Seconds 30
curl https://inference.dos.ai/health
```

### Check User Balance

```bash
# Via Wrangler
wrangler d1 execute dos-inference --command="SELECT * FROM billing WHERE user_id = 'xxx'"
```

### Add Credits

```bash
# Add $100 credit (10000 cents)
wrangler d1 execute dos-inference --command="UPDATE billing SET balance_cents = balance_cents + 10000 WHERE user_id = 'xxx'"
```

---

## File Structure

```
packages/
└── api-gateway/
    ├── src/
    │   └── worker.ts          # Main Worker with billing logic
    ├── wrangler.toml          # D1 bindings, env vars
    └── package.json

apps/app/src/
├── app/
│   ├── api/
│   │   ├── keys/              # API key management
│   │   ├── usage/             # Usage statistics
│   │   └── billing/           # Billing endpoints
│   ├── api-keys/
│   │   └── page.tsx           # API Keys UI
│   └── usage/
│       └── page.tsx           # Usage dashboard
└── lib/
    └── cloudflare.ts          # D1 client helper

C:\Users\JOY\.cloudflared\
├── 04915ff2-6f0d-49ab-8115-c36c74edbff9.json  # Tunnel credentials
├── ai-api-config.yml                           # Tunnel config (if using config file)
├── fix-and-install-service.ps1                 # Service installation script
└── force-restart-service.ps1                   # Service restart script
```

---

## References

- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [vLLM OpenAI Server](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [BILLING-MIGRATION-ROADMAP.md](./BILLING-MIGRATION-ROADMAP.md) - Plan to migrate billing to Supabase
