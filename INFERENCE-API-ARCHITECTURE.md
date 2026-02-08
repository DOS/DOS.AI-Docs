# Inference API Architecture (Serverless Inference)

## Overview

This document describes the architecture for DOS.ai Serverless Inference API, allowing users to access LLM models via OpenAI-compatible API with API key authentication and usage-based billing.

## Current Infrastructure (February 2026)

### Components

| Component | Location | Description |
|-----------|----------|-------------|
| **vLLM Server** | Local PC (localhost:8000) | Docker container running LLM models with GPU |
| **Cloudflare Tunnel** | cloudflared service | Exposes local vLLM at `inference.dos.ai` |
| **API Gateway** | Cloudflare Worker | `api.dos.ai` - Auth, billing, rate limiting |
| **D1 Database** | Cloudflare Edge | API keys, rate limiting, model pricing (edge-fast) |
| **Supabase** | PostgreSQL | Billing accounts, usage logs (single source of truth) |
| **Dashboard** | Vercel (apps/app) | `app.dos.ai` - User management |

### Database Architecture

```
┌─ Supabase (PostgreSQL) ─────────────────────────────────────────┐
│                                                                  │
│  public schema (DOS-Me platform, shared across all products)     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ auth.users           │ profiles          │ organizations │   │
│  │ billing_accounts ◄───┼── Balance R/W (single source)     │   │
│  │ credit_transactions  │── Deposits/refunds                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  dosai schema (DOS.AI-specific data)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ models             ◄─┼── Model info + pricing reference    │   │
│  │ usage_transactions ◄─┼── Per-request usage logs           │   │
│  │ user_settings      ◄─┼── User preferences (default model) │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌─ Cloudflare D1 (Edge) ──────────────────────────────────────────┐
│  api_keys          ◄── API key validation (every request)        │
│  rate_limit_log    ◄── Rate limiting                             │
│  model_pricing     ◄── Pricing cache (sub-ms reads)              │
│  usage_transactions◄── Dashboard usage queries                   │
└──────────────────────────────────────────────────────────────────┘
```

**Schema rule**: Product-specific data goes in its own schema (e.g., `dosai`), NOT in `public`. Shared platform data stays in `public`.

**PostgREST config**: Exposed schemas = `public, graphql_public, dosai`

### Pricing Table

Prices in USD cents per 1M tokens. DOS.AI prices at ~50% of provider rates.

| Model | Model ID | Input ¢/1M | Output ¢/1M |
|-------|----------|-----------|------------|
| Llama 3.1 8B | `llama-3.1-8b` | 10 | 20 |
| Llama 3.1 70B | `llama-3.1-70b` | 50 | 150 |
| Llama 3.3 70B | `llama-3.3-70b` | 60 | 180 |
| Qwen 2.5 72B | `qwen-2.5-72b` | 50 | 150 |
| DeepSeek V3 | `deepseek-v3` | 27 | 110 |
| Qwen3 VL 30B | `qwen3-vl-30b` | 10 | 80 |
| Default | `default` | 10 | 10 |

Pricing stored in **D1 `model_pricing`** (read at edge for billing) and **Supabase `dosai.models`** (source of truth with full model info). Worker reads from D1 for sub-ms latency.

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
│   (API Client)  │     │ (CF Worker)     │     │ (CF Tunnel)     │     │  localhost:8000  │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                        ┌──────┴──────┐
                        ▼             ▼
                  ┌──────────┐  ┌──────────┐
                  │    D1    │  │ Supabase │
                  │ (Edge)   │  │ (Billing)│
                  └──────────┘  └──────────┘

┌─────────────────┐     ┌─────────────────┐
│   User          │────▶│  Dashboard      │────▶ Supabase (Users, Orgs, Billing)
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
   d. Check Supabase billing_accounts balance
   e. If insufficient balance → 402 Payment Required

3. Forward to inference backend
   POST https://inference.dos.ai/v1/chat/completions
   (No auth needed - tunnel is internal)

4. Cloudflare Tunnel routes to local vLLM
   POST http://localhost:8000/v1/chat/completions

5. vLLM processes and returns response
   - Includes usage: { prompt_tokens, completion_tokens, total_tokens }

6. Worker calculates and deducts cost (async, non-blocking)
   - Get model pricing from D1.model_pricing (edge-fast)
   - Calculate: cost = (tokens / 1,000,000) × price_per_million
   - Exact precision: 4 decimal places (0.0001 cents), no minimum charge
   - Deduct from Supabase public.billing_accounts
   - Log transaction to Supabase dosai.usage_transactions

7. Return response to client
```

---

## Billing System

### Token-Based Billing Flow

Located in `packages/api-gateway/src/worker.ts`:

```typescript
// Supabase is single source of truth for billing
async function deductBalance(
  db: D1Database,
  supabase: SupabaseClient | null,
  userId: string,
  apiKeyId: string,
  model: string,
  inputTokens: number,
  outputTokens: number
): Promise<void> {
  // Get model pricing from D1 (edge-fast, sub-ms)
  const pricing = await db.prepare(
    'SELECT input_price_per_million, output_price_per_million FROM model_pricing WHERE model_id = ?'
  ).bind(model).first();

  const inputPricePerMillion = pricing?.input_price_per_million || 10;
  const outputPricePerMillion = pricing?.output_price_per_million || 10;

  // Calculate cost in cents (exact - no rounding until final)
  const inputCostCents = (inputTokens / 1_000_000) * inputPricePerMillion;
  const outputCostCents = (outputTokens / 1_000_000) * outputPricePerMillion;
  const totalCostCents = inputCostCents + outputCostCents;

  // Skip if cost is essentially zero
  if (totalCostCents < 0.00001) return;

  // Round to 4 decimal places for storage (0.0001 cents precision)
  const costInCentsRounded = Math.round(totalCostCents * 10000) / 10000;

  // ===== SUPABASE WRITE (single source of truth) =====
  // Update billing_accounts in public schema
  await supabase.from('billing_accounts').eq('user_id', userId).update({
    balance_cents: currentBalance - costInCentsRounded,
    total_spent_cents: currentSpent + costInCentsRounded,
    lifetime_tokens_used: currentTokens + inputTokens + outputTokens,
  });

  // Log usage to dosai schema
  await supabase.from('usage_transactions', 'dosai').insert({
    user_id: userId,
    api_key_id: apiKeyId,
    model, input_tokens: inputTokens, output_tokens: outputTokens,
    input_cost_cents: inputCostRounded,
    output_cost_cents: outputCostRounded,
    total_cost_cents: costInCentsRounded,
  });
}
```

**Key design decisions:**
- **No minimum charge**: Exact per-token pricing with 4 decimal precision
- **Pricing from D1**: Sub-ms reads for every API request
- **Billing writes to Supabase**: Single source of truth, NUMERIC(20,4) columns
- **Usage logs to `dosai` schema**: Product-specific data separated from platform
- **Worker uses custom REST client**: CF Workers can't use npm Supabase SDK, uses `Content-Profile: dosai` header for dosai schema

### Adding Credits

```bash
# Via Supabase SQL editor or dashboard API:
UPDATE public.billing_accounts
SET balance_cents = balance_cents + 10000,
    total_deposited_cents = total_deposited_cents + 10000
WHERE user_id = '<user-uuid>';

# Also log the credit transaction:
INSERT INTO public.credit_transactions (user_id, amount_cents, type, description)
VALUES ('<user-uuid>', 10000, 'deposit', 'Manual credit addition');
```

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
| `/api/billing` | GET | Get billing info (from Supabase) |
| `/api/billing/transactions` | GET | Get credit transactions |

### Dashboard Endpoints (via Worker)

| Route | Method | Description |
|-------|--------|-------------|
| `/dashboard/billing/usage` | GET | Usage stats (from D1) |
| `/dashboard/billing/add-credits` | POST | Add credits (admin) |
| `/dashboard/models/pricing` | GET | Model pricing (from D1) |

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

```sql
-- Via Supabase SQL editor
SELECT ba.*, au.email
FROM public.billing_accounts ba
JOIN auth.users au ON au.id = ba.user_id
WHERE au.email = 'user@example.com';
```

### Find User ID by Email

```sql
-- Supabase SQL editor
SELECT id, email FROM auth.users WHERE email = 'user@example.com';
```

### Update Model Pricing

```sql
-- Update in Supabase (dosai.models - source of truth)
UPDATE dosai.models
SET input_price_per_million = 50, output_price_per_million = 150, updated_at = NOW()
WHERE model_id = 'llama-3.1-70b';

-- Also update D1 (edge pricing cache - worker reads from here)
-- cd packages/api-gateway
-- npx wrangler d1 execute dos-api --remote --command="UPDATE model_pricing SET input_price_per_million = 50, output_price_per_million = 150 WHERE model_id = 'llama-3.1-70b'"
```

---

## File Structure

```
packages/
└── api-gateway/
    ├── src/
    │   └── worker.ts          # Main Worker with billing logic
    ├── migrations/             # D1 migration SQL files
    ├── wrangler.toml          # D1 bindings, env vars
    └── package.json

apps/app/
├── src/app/
│   ├── api/
│   │   ├── keys/              # API key management
│   │   └── billing/           # Billing endpoints (reads Supabase)
│   ├── (app)/
│   │   ├── api-keys/          # API Keys UI
│   │   ├── usage/             # Usage dashboard
│   │   └── billing/           # Billing UI
│   └── ...
├── supabase/
│   └── migrations/            # Supabase migration SQL files
└── ...

C:\Users\JOY\.cloudflared\
├── 04915ff2-6f0d-49ab-8115-c36c74edbff9.json  # Tunnel credentials
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
- [Supabase Custom Schemas](https://supabase.com/docs/guides/api/using-custom-schemas)
