# DOS.AI API Gateway — Product Plan

Updated: 2026-03-14
Status: Draft
Owner: DOS.AI
Related: `INFERENCESENSE_LIKE_ALPHA_MVP.md`, `API_CONTRACT_V1.md`

## Vision

A unified LLM API gateway at `api.dos.ai` — users call one endpoint, the system
routes to the cheapest/fastest available backend. Like OpenRouter, but with a
self-hosted GPU fleet as first-class capacity (100% margin).

```
User app
  │
  ▼
api.dos.ai  (Cloudflare Worker — auth, routing, metering)
  │
  ├─► Self-hosted vLLM fleet  (cost $0, margin 100%)
  │     ├── joy-pc: RTX Pro 6000 — Qwen3.5-35B-A3B
  │     ├── node-2: future GPU — model TBD
  │     └── (InferenceSense nodes — spare capacity)
  │
  ├─► Alibaba Cloud  (Qwen3.5-flash $0.10/$0.40 per 1M tokens)
  ├─► Google AI  (Gemini 3 Flash $0.50/$3.00)
  ├─► OpenAI  (GPT-4o-mini $0.15/$0.60)
  └─► Any OpenAI-compatible provider
```

## Why This Works

1. **Self-hosted = unfair advantage**: When vLLM fleet has capacity, every request
   is pure profit. OpenRouter can't compete on margin for models we self-host.
2. **Paid providers = infinite scale**: When self-hosted is down/full, overflow to
   paid APIs. User never sees downtime.
3. **Already built the hard parts**: vLLM infra, Cloudflare Worker at api.dos.ai,
   DOS.Me auth + credits system, InferenceSense node agent design.
4. **Vietnamese market gap**: No local LLM API gateway. Vietnamese devs use
   OpenRouter (USD, no local payment) or direct provider APIs (fragmented).

## Relationship to InferenceSense

InferenceSense (existing docs) defines how to manage a **fleet of self-hosted GPU
nodes** with spare capacity routing. The API Gateway is the **commercial layer on
top**:

```
┌─────────────────────────────────────────────┐
│  API Gateway (this doc)                     │
│  - User-facing API, auth, billing, routing  │
│  - Model catalog, pricing, rate limits      │
├─────────────────────────────────────────────┤
│  InferenceSense (existing docs)             │
│  - Node agent, heartbeat, spare capacity    │
│  - GPU fleet management, draining           │
├─────────────────────────────────────────────┤
│  Paid Provider Backends                     │
│  - Alibaba, Google, OpenAI, Anthropic       │
│  - OpenAI-compatible API forwarding         │
└─────────────────────────────────────────────┘
```

InferenceSense becomes one of several backends. The API Gateway doesn't replace
it — it wraps it and adds paid fallback + billing.

## Core Architecture

### 1. Gateway (Cloudflare Worker at api.dos.ai)

Already exists. Currently proxies to vLLM. Extend to:

- **Auth**: Validate API key → look up user, tier, credits
- **Route**: Pick backend based on model + availability + cost
- **Meter**: Log tokens in/out, cost, latency, provider used
- **Bill**: Deduct credits from user's DOS.Me account
- **Stream**: Pass through SSE streaming from any backend

### 2. Provider Registry (D1 or KV)

```
providers:
  - id: self-hosted
    base_url: https://inference.dos.ai  (or direct vLLM IPs)
    api_key: (internal)
    models: [dos-ai]  # served-model-name in vLLM
    priority: 1  # try first
    cost_input: 0
    cost_output: 0
    max_concurrent: 50
    health_check: GET /v1/models

  - id: alibaba
    base_url: https://dashscope-intl.aliyuncs.com/compatible-mode
    api_key: sk-...
    models: [qwen3.5-flash, qwen3.5-plus, qwen3.5-35b-a3b]
    priority: 2
    cost_input: 0.10  # per 1M tokens
    cost_output: 0.40
    max_concurrent: 100

  - id: google
    base_url: https://generativelanguage.googleapis.com
    api_key: AIza...
    models: [gemini-2.5-flash, gemini-3-flash-preview]
    priority: 3
    cost_input: 0.30
    cost_output: 2.50
    max_concurrent: 100
```

### 3. Model Catalog (user-facing)

Users request models by alias. Gateway maps to best available backend:

```
model aliases:
  "dos-ai"       → self-hosted Qwen3.5-35B → alibaba qwen3.5-flash
  "qwen-fast"    → alibaba qwen3.5-flash (always paid, fastest)
  "qwen-plus"    → alibaba qwen3.5-plus
  "gemini-flash" → google gemini-2.5-flash
  "auto"         → cheapest available model matching request constraints
```

### 4. Routing Logic

```
function selectProvider(model, request):
  candidates = providers.filter(p =>
    p.models.includes(model) &&
    p.healthy &&
    p.current_load < p.max_concurrent
  )
  // Sort by: priority (self-hosted first), then cost, then latency
  candidates.sort(by: priority ASC, cost ASC, avg_latency ASC)
  return candidates[0]  // fallback chain is implicit
```

### 5. Billing Model

```
user_cost = (input_tokens / 1M) * sell_price_input
          + (output_tokens / 1M) * sell_price_output

our_cost  = (input_tokens / 1M) * provider_cost_input
          + (output_tokens / 1M) * provider_cost_output

margin    = user_cost - our_cost
```

Sell prices (proposed, ~2-3x markup on paid, 100% on self-hosted):

| Model alias     | Sell input/1M | Sell output/1M | Self-hosted margin | Paid margin |
|-----------------|---------------|----------------|--------------------|-------------|
| dos-ai          | $0.15         | $0.60          | 100%               | ~33%*       |
| qwen-fast       | $0.20         | $0.80          | -                  | 50%         |
| qwen-plus       | $0.50         | $3.00          | -                  | 48%         |
| gemini-flash    | $0.50         | $4.00          | -                  | 37%         |

*When self-hosted is down, dos-ai falls back to alibaba qwen3.5-flash at cost.

### 6. Auth & Credits — Two-Layer Billing Model

The gateway serves two distinct consumer types with different billing models:

```
┌──────────────────────────────────────────────────────────┐
│  Application Layer (request-based billing)                │
│                                                          │
│  DOSafe Telegram bot  →  consume_quota() per request     │
│  DOSafe web app       →  dosafe_usage per request        │
│  DOS.AI app features  →  per-feature billing             │
│                                                          │
│  Auth: INTERNAL_API_KEY (bypass gateway billing)         │
│  Why: billing already handled upstream per user/tier     │
└──────────────────────┬───────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│  api.dos.ai Gateway (token-based billing)                │
│                                                          │
│  INTERNAL_API_KEY  →  skip billing + rate limit          │
│  dos_sk_xxx        →  deductBalance per token            │
│                                                          │
│  Why: external consumers call gateway directly,          │
│  no application layer to handle billing for them         │
└──────────────────────────────────────────────────────────┘
```

**Rule:** If a product has its own user-facing billing (Telegram quota, web app quota), it uses `INTERNAL_API_KEY` to skip gateway billing. If a consumer calls `api.dos.ai` directly (developers, partners), the gateway handles token-based billing with `dos_sk_xxx` keys.

**LLM fallback cost tracking:** When self-hosted vLLM is unavailable, application-layer services fall back to paid providers (Alibaba Cloud). Fallback usage is logged as structured JSON (`event: llm_fallback_used`) with token count and estimated cost, for internal cost monitoring only — user-facing billing stays request-based.

Leverage DOS.Me existing infrastructure:

- **API keys**: `dosai.api_keys` table (key_hash, user_id, tier, rate_limit)
- **Credits**: DOS.Me credit system (credit_transactions table)
- **Tiers**: Free (rate limited, dos-ai only), Pro (all models, higher limits)
- **Top-up**: VNPay, Stripe (via DOS.Me billing)

### 7. Usage Tracking

```sql
-- dosai.usage_log
CREATE TABLE dosai.usage_log (
  id            UUID DEFAULT gen_random_uuid(),
  api_key_id    UUID REFERENCES dosai.api_keys(id),
  model         TEXT NOT NULL,
  provider_id   TEXT NOT NULL,       -- which backend served it
  input_tokens  INT NOT NULL,
  output_tokens INT NOT NULL,
  latency_ms    INT,
  cost_usd      NUMERIC(10,6),      -- our cost
  revenue_usd   NUMERIC(10,6),      -- what we charged user
  status        TEXT DEFAULT 'success',  -- success/error
  created_at    TIMESTAMPTZ DEFAULT now()
);
```

## Phased Rollout

### Phase 0 — Already Done (2026-03-14)

- [x] api.dos.ai Cloudflare Worker (proxy to vLLM)
- [x] Self-hosted vLLM with Qwen3.5-35B-A3B
- [x] DOS.Me auth + credits infrastructure
- [x] Fallback chain in DOSafe entity-check (vLLM → Alibaba Cloud qwen3.5-flash)
- [x] Alibaba Cloud API key provisioned (DashScope International)
- [x] INTERNAL_API_KEY bypass for DOS internal services (skip billing/rate limit)
- [x] Two-layer billing model: application-layer (request-based) + gateway (token-based)
- [x] Fallback usage logging (structured JSON with token count + cost estimate)

### Phase 1 — Internal Gateway (1-2 weeks)

Turn the existing Cloudflare Worker into a proper gateway:

- [ ] Provider registry in D1 (self-hosted + alibaba)
- [ ] Routing logic: try self-hosted → fallback to paid
- [ ] Token counting (tiktoken-compatible, estimate from char count for speed)
- [ ] Usage logging to D1
- [ ] API key auth (INTERNAL_API_KEY for internal, dos_sk_xxx for external)
- [ ] Health check endpoint: `GET /health` returns provider status

Deliverable: DOSafe entity-check uses api.dos.ai gateway instead of direct
vLLM + hardcoded fallback. Same functionality, centralized routing.

### Phase 2 — Multi-Model + Billing (2-3 weeks)

- [ ] Model catalog endpoint: `GET /v1/models` returns available models + pricing
- [ ] Multiple model aliases (dos-ai, qwen-fast, qwen-plus, gemini-flash)
- [ ] Per-request billing: deduct credits from DOS.Me account
- [ ] Usage dashboard in app.dos.ai (tokens used, cost, by model)
- [ ] Rate limiting per tier (free: 10 RPM, pro: 100 RPM)
- [ ] Streaming support (SSE passthrough)

Deliverable: External users can sign up, get API key, call multiple models,
pay with credits.

### Phase 3 — InferenceSense Integration (3-4 weeks)

- [ ] Node agent from InferenceSense docs → register self-hosted nodes
- [ ] Multi-node routing (multiple GPUs, spare capacity)
- [ ] Dynamic model loading (node reports which model it's serving)
- [ ] Draining support (operator reclaim without dropping requests)
- [ ] Node health in `/v1/models` response

Deliverable: Multiple self-hosted GPU nodes contribute capacity. Gateway
routes across fleet + paid providers seamlessly.

### Phase 4 — Public Launch

- [ ] Pricing page on dos.ai
- [ ] Self-serve API key creation
- [ ] Documentation (OpenAI SDK compatible — just change base_url)
- [ ] VNPay top-up for Vietnamese market
- [ ] Analytics dashboard (for users)
- [ ] Admin dashboard (for us — revenue, costs, margins)

## Key Design Decisions

### Why Cloudflare Worker (not FastAPI)?

- Already deployed at api.dos.ai
- Edge network = low latency globally
- D1 for structured data, KV for hot config
- Free tier generous (100k requests/day)
- Streaming via ReadableStream works well
- Can always add FastAPI origin for complex logic later

### Why not just use OpenRouter?

- OpenRouter charges ~15-30% markup
- We have self-hosted GPUs = $0 cost for significant traffic
- Vietnamese market needs local payment (VNPay)
- We control the model selection and can optimize for our use cases
- Revenue stays in ecosystem (DOS.Me credits)

### Token Counting Strategy

For Phase 1, estimate tokens from character count (chars / 4 for English,
chars / 2 for Vietnamese/CJK). Accurate enough for billing at our scale.
Switch to tiktoken if precision matters later.

### Pricing Philosophy

- **Self-hosted models**: Price at ~50% of cheapest paid alternative.
  Users get a discount, we get 100% margin. Win-win.
- **Paid models**: 2-3x markup. Competitive with OpenRouter.
- **Free tier**: Rate-limited access to dos-ai model only. Acquisition funnel.

## Technical Notes

### Existing Infrastructure to Reuse

| Component | Location | Purpose |
|-----------|----------|---------|
| Cloudflare Worker | `api.dos.ai` (DOS-AI repo) | Gateway, already proxies to vLLM |
| D1 Database | Cloudflare | Request logging, API keys |
| vLLM | joy-pc RTX Pro 6000 | Self-hosted inference |
| DOS.Me API | `api-v2.dos.me` | User auth, credits, billing |
| Supabase | `gulptwduchsjcsbndmua` | Usage data (dosai schema) |

### Request Flow (Phase 1)

```
1. User → POST api.dos.ai/v1/chat/completions
2. Worker validates API key (D1 lookup)
3. Worker checks provider health (KV cache, 30s TTL)
4. Worker selects provider (priority order)
5. Worker forwards request to provider
6. Provider responds (streaming or batch)
7. Worker logs usage to D1
8. Worker returns response to user
```

### Streaming Architecture

```
User ←SSE── Worker ←SSE── Provider
               │
               └── (buffer last chunk for token count, then log)
```

Count tokens from the final `usage` field in the provider response.
If provider doesn't return usage (some don't for streaming), estimate
from accumulated content length.
