# DOS.AI — System Architecture

**Updated:** 2026-03-14
**Status:** API Gateway Phase 0 complete. InferenceSense MVP draft.

---

## What is DOS.AI?

DOS.AI is the AI infrastructure layer of the DOS ecosystem. It provides:

1. **Self-hosted LLM inference** — GPU fleet running open-source models (Qwen3.5)
2. **API Gateway** — unified endpoint at `api.dos.ai` for all LLM consumers
3. **Enterprise API** — key management, billing, usage tracking for external developers

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        CONSUMERS                                  │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │  DOSafe      │  │  DOS.AI App │  │  External Developers     │ │
│  │  (internal)  │  │  (internal) │  │  (dos_sk_xxx keys)       │ │
│  └──────┬───────┘  └──────┬──────┘  └──────────┬───────────────┘ │
│         │                  │                     │                 │
│    INTERNAL_API_KEY   INTERNAL_API_KEY       dos_sk_xxx           │
│    (bypass billing)   (bypass billing)   (token-based billing)   │
│         │                  │                     │                 │
│         └──────────────────┼─────────────────────┘                │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              api.dos.ai — Cloudflare Worker                 │  │
│  │                                                             │  │
│  │  Auth:     INTERNAL_API_KEY → skip billing                  │  │
│  │            dos_sk_xxx → D1 key lookup + token billing       │  │
│  │  Route:    self-hosted first → paid fallback                │  │
│  │  Meter:    log tokens, latency, provider per request        │  │
│  │  Bill:     deductBalance for dos_sk_xxx keys only           │  │
│  └──────────────────────┬─────────────────────────────────────┘  │
│                          │                                        │
│         ┌────────────────┼────────────────────┐                  │
│         ▼                ▼                    ▼                   │
│  ┌──────────────┐ ┌──────────────┐  ┌──────────────────────┐    │
│  │  Self-hosted  │ │  Alibaba     │  │  Google / OpenAI     │    │
│  │  vLLM fleet   │ │  Cloud       │  │  (future providers)  │    │
│  │  (priority 1) │ │  (priority 2)│  │  (priority 3+)       │    │
│  │  cost: $0     │ │  $0.10/$0.40 │  │  varies              │    │
│  └──────────────┘ └──────────────┘  └──────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Two-Layer Billing Model

The gateway serves two consumer types with different billing strategies:

### Layer 1 — Application Layer (request-based)

Internal DOS products (DOSafe, DOS.AI app) handle their own user-facing billing **before** calling the gateway.

| Product | Billing method | Implementation |
|---------|---------------|----------------|
| DOSafe Telegram bot | `consume_quota()` RPC per request | `dosafe.bot_quota` + `dosafe.client_quota` |
| DOSafe web app | `dosafe_usage` per request | Supabase RPC |
| DOSafe Chrome extension | Anonymous IP-based quota | Supabase RPC |
| DOS.AI app features | Per-feature billing (TBD) | — |

These products authenticate with `INTERNAL_API_KEY` → gateway skips billing, rate limiting, and usage tracking. The gateway is pure infrastructure for them.

### Layer 2 — Gateway Layer (token-based)

External developers and partners call `api.dos.ai` directly with `dos_sk_xxx` keys.

| Auth | Billing | Rate limit | Use case |
|------|---------|------------|----------|
| `INTERNAL_API_KEY` | Skip (app layer handles) | Skip | DOSafe, DOS.AI internal services |
| `dos_sk_xxx` | Per-token deduction from credits | Per-key (D1) | External developers, partners, enterprise |

**Why two layers?** Application-layer products know their users (Telegram chat ID, web session, etc.) and can enforce business-level quotas (free tier: 10/day, paid: 100/day). The gateway only knows API keys, which is the right abstraction for external developers.

---

## Self-Hosted GPU Fleet

### Current Hardware

| Node | GPU | VRAM | Model | Endpoint |
|------|-----|------|-------|----------|
| joy-pc | RTX Pro 6000 (Blackwell) | 96 GB | Qwen3.5-35B-A3B-GPTQ-Int4 (scorer) | `inference.dos.ai` |
| joy-pc | RTX 5090 | 32 GB | Qwen3-8B base (observer) | `inference-ref.dos.ai` |

### vLLM Configuration

- **Scorer**: `--served-model-name dos-ai`, `--gpu-memory-utilization 0.90`, prefix caching + chunked prefill enabled
- **Observer**: `--served-model-name observer`, `--enforce-eager` (CUDA graph fails on RTX 5090)
- **Qwen3.5**: Thinking model — requires `chat_template_kwargs: {"enable_thinking": false}` for non-chat workloads
- **Docker**: Named volume `vllm-compile-cache` for CUDA graph persistence

### Performance (Qwen3.5-35B-A3B-GPTQ-Int4)

| Metric | Value |
|--------|-------|
| Single request throughput | 184 tok/s |
| @10 concurrent | 800 tok/s |
| @50 concurrent | 3,373 tok/s |
| TPOT (after warmup) | 6.6ms |
| TTFT median @1 rps | 48ms |
| VRAM usage | 23 GiB |
| JSON compliance | 95% |
| Scam detection F1 | 0.970 |

---

## LLM Fallback Chain

When self-hosted vLLM is unavailable, requests fall back to paid providers:

```
1. Self-hosted vLLM (api.dos.ai → inference.dos.ai)
   ↓ fail (timeout, 502, model not loaded)
2. Alibaba Cloud DashScope International
   Model: qwen3.5-flash ($0.10/$0.40 per 1M tokens)
   ↓ fail
3. (Future) Google Gemini 3 Flash, OpenAI, etc.
```

### Fallback Cost Tracking

When a paid provider is used, structured JSON is logged to Vercel console:

```json
{
  "event": "llm_fallback_used",
  "provider": "qwen3.5-flash",
  "prompt_tokens": 4800,
  "completion_tokens": 180,
  "cost_usd": 0.000552,
  "entity_type": "phone"
}
```

Filter by `llm_fallback_used` in Vercel logs to monitor paid API spend.

User-facing billing is **not affected** by which provider served the request — it stays request-based at the application layer.

---

## API Gateway (Cloudflare Worker)

### Endpoint: `api.dos.ai`

| Route | Purpose |
|-------|---------|
| `POST /v1/chat/completions` | OpenAI-compatible chat API |
| `GET /v1/models` | List available models |
| `GET /dashboard/*` | Internal dashboard API (X-Dashboard-Secret) |

### Key Types

| Key format | Purpose | Billing |
|------------|---------|---------|
| `INTERNAL_API_KEY` (env secret) | DOS internal services | None (bypass) |
| `dos_sk_xxx` | External developers/partners | Token-based, credits deducted |

### Infrastructure

| Component | Service | Purpose |
|-----------|---------|---------|
| Worker | Cloudflare Workers | Auth, routing, metering, billing |
| D1 | Cloudflare D1 | API keys, rate limit log, usage log |
| Supabase | `dosai` schema | Usage transactions, user settings |
| DOS.Me | `public` schema | Billing accounts, credit transactions |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [API_GATEWAY_PRODUCT_PLAN.md](API_GATEWAY_PRODUCT_PLAN.md) | Full product plan with phased rollout, pricing, design decisions |
| [INFERENCESENSE_LIKE_ALPHA_MVP.md](INFERENCESENSE_LIKE_ALPHA_MVP.md) | GPU spare capacity routing — fleet management layer |
| [API_CONTRACT_V1.md](API_CONTRACT_V1.md) | API contract for InferenceSense alpha endpoints |
| [DOSafe-Architecture.md](DOSafe-Architecture.md) | DOSafe system architecture (consumer of api.dos.ai) |
