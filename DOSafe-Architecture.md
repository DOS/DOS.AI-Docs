# DOSafe — System Architecture

**Updated:** 2026-03-10
**Status:** AI Detection COMPLETE. Threat Intel Pipeline COMPLETE (1.52M+ entries, 13 sources). Risk Scoring V2 COMPLETE. Chrome Extension Protection v0.5.4 COMPLETE. Audio/Video TODO.

**Implementation ownership:** Claude is the primary coding agent for architecture changes; this document is the handoff/reference source for Claude-first implementation.

---

## DOS Ecosystem Overview

DOSafe is one product in the **DOS ecosystem**, sharing infrastructure with DOS.Me, DOS.AI, DOS Chain, Rate.Box, and Bexly.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DOS ECOSYSTEM                                       │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   DOS.Me      │  │   DOSafe     │  │   DOS.AI     │  │  Rate.Box     │  │
│  │  id.dos.me    │  │  dosafe.io   │  │  app.dos.ai  │  │  Bexly, etc. │  │
│  │  Next.js      │  │  Next.js     │  │  Next.js     │  │  Various     │  │
│  │  Vercel       │  │  Vercel      │  │  Vercel      │  │              │  │
│  └──────┬────────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                  │                  │                  │           │
│         └──────────────────┴──────────────────┴──────────────────┘           │
│                                      │                                       │
│                    ┌─────────────────▼─────────────────┐                    │
│                    │   DOS-Me API  (api-v2.dos.me)      │                    │
│                    │   NestJS 11 · Cloud Run · asia-se1 │                    │
│                    │                                    │                    │
│                    │  /auth      — login, OAuth, JWT    │                    │
│                    │  /users     — profiles, accounts   │                    │
│                    │  /wallet    — multi-chain wallets  │                    │
│                    │  /trust     — cross-product safety │                    │
│                    │  /attest    — on-chain stamps      │                    │
│                    │  /ens-gw    — ENS CCIP-Read        │                    │
│                    └─────────────────┬─────────────────┘                    │
│                                      │                                       │
│          ┌───────────────────────────┼──────────────────────┐               │
│          │                           │                       │               │
│  ┌───────▼──────┐      ┌────────────▼──────────┐   ┌───────▼──────┐        │
│  │   Supabase   │      │      DOS Chain         │   │  api.dos.ai  │        │
│  │  PostgreSQL  │      │  EVM Testnet (3939)    │   │  CF Worker   │        │
│  │  Shared DB   │      │  EAS Attestations      │   │  API Gateway │        │
│  │  (gulptw...) │      │  Schema 6: entity.flag │   │  Enterprise  │        │
│  └──────────────┘      └────────────────────────┘   └──────────────┘        │
│                                                                              │
│  ┌─────────────────────────────────────────────────┐                        │
│  │   vLLM Inference (self-hosted, RTX Pro 6000)    │                        │
│  │   api.dos.ai       — Qwen3.5-35B-A3B-GPTQ-Int4 │                        │
│  │   inference-ref.dos.ai — Qwen3-8B (observer)   │                        │
│  └─────────────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Product Roles

| Product | Domain | Stack | Role |
|---------|--------|-------|------|
| **DOS.Me** | `id.dos.me` | Next.js + NestJS | Identity platform — auth, profiles, wallets, social graph |
| **DOS-Me API** | `api-v2.dos.me` | NestJS · Cloud Run | Main backend — auth server, Trust gateway, attestation |
| **DOSafe** | `dosafe.io` | Next.js · Vercel | Safety product — AI detection, scam lookup, threat intel |
| **DOS.AI** | `app.dos.ai` | Next.js · Vercel | AI marketplace — model access, enterprise API key management |
| **api.dos.ai** | Cloudflare Worker | CF Workers · D1 | API gateway — auth + billing + rate limit for enterprise |
| **DOS Chain** | RPC `test.doschain.com` | EVM (3939) | On-chain safety flags via EAS attestations |
| **Rate.Box** | — | — | Rating platform — consumes Trust API |
| **Bexly** | — | — | Browser extension — consumes Trust API |

---

## Shared Infrastructure

### Supabase (`gulptwduchsjcsbndmua`)

Single Supabase project shared across all DOS products.

| Schema | Owner | Key Tables |
|--------|-------|------------|
| `public` | Shared | `profiles`, `billing_accounts`, `credit_transactions`, `safety_flags`, `bot_quota`, `organizations`, `fed_profiles`, `posts` |
| `dosafe` | DOSafe | `threat_intel`, `threat_clusters`, `raw_imports`, `sync_log`, `api_keys` |
| `dosai` | DOS.AI | `dosafe_usage`, `user_settings`, `usage_transactions` |

**PostgREST exposed schemas:** `public, graphql_public, dosai`

**Auth:** Supabase handles JWT issuance. All products validate tokens against the same Supabase instance (`SUPABASE_JWT_SECRET`).

### DOS Chain (EVM Testnet 3939)

On-chain safety attestation layer via [EAS](https://attest.sh).

| Schema | UID | Purpose |
|--------|-----|---------|
| `dos.entity.flag` | `0x0b0565...` | Flag a scammer entity (wallet, url, phone, etc.) |

Written to by: DOS-Me Trust API (`/trust/flags/:id/attest`). Read by: DOSafe (`doschain.ts`), DOS-Me attestation module.

### vLLM Inference

| Endpoint | Model | Hardware | Purpose |
|----------|-------|----------|---------|
| `api.dos.ai` | Qwen3.5-35B-A3B-GPTQ-Int4 | RTX Pro 6000 (96GB) | Scorer — LLM rubric + image analysis |
| `inference-ref.dos.ai` | Qwen3-8B base | RTX 5090 (32GB) | Observer — Binoculars cross-entropy |

Both models are natively multimodal (text + image). Auth: `DOS_INFERENCE_API_KEY`.

---

## Cross-Product API Patterns

### Auth — Who validates tokens?

```
User logs in → DOS-Me API (/auth/login)
  → Issues Supabase JWT
  → All products validate JWT with same Supabase instance

DOS.AI exception: uses Firebase Auth (legacy) + Supabase JWT for billing
```

### DOSafe calls DOS-Me API

| When | Endpoint | Purpose |
|------|----------|---------|
| (Planned) Sync confirmed flags | `POST /trust/flags` | Push high-confidence threat_intel entries to shared Trust DB |
| (Future) Flag lifecycle | `POST /trust/flags/:id/confirm` | Move flag to confirmed → ready for on-chain |

**Auth:** M2M API key (`X-Api-Key`), scopes: `trust.check`, `trust.flag`

### External services call DOSafe API

| Product | Endpoint | Scopes |
|---------|----------|--------|
| Bexly | `POST /api/check` | `check` |
| Rate.Box | `POST /api/check/bulk` | `bulk` |
| Any developer | `POST /api/check`, `/api/detect`, `/api/url-check`, `/api/entity-check` | per key |

**Auth:** `X-Api-Key: dsk_xxxx...` — one key per service, scopes stored in `dosafe.api_keys`.

**Note:** Rate.Box and Bexly now call DOSafe directly. DOS.Me Trust API is deprecated (sunset 2026-11-01).

### DOS.AI calls api.dos.ai (Worker)

DOS.AI dashboard manages API keys via `api.dos.ai/dashboard/*` (internal `X-Dashboard-Secret`). Enterprise clients use `dos_sk_*` keys to call `api.dos.ai/v1/*` which routes to inference.

### DOSafe consumes Supabase directly

DOSafe reads/writes `dosafe.*` schema tables directly with `SUPABASE_SERVICE_ROLE_KEY`. The `dosai.dosafe_usage` table (quota tracking) is also written by DOSafe.

---

## Unified Billing (Planned — DOS.Me owns)

### Ownership

| Responsibility | Owner |
|----------------|-------|
| Subscription management (create, upgrade, cancel) | DOS.Me |
| Payment processing (Stripe webhooks) | DOS.Me |
| Credit top-up flows | DOS.Me |
| `public.subscriptions` table (new) | DOS.Me |
| `public.billing_accounts` (existing) | DOS.Me |
| Credit deduction per API call | DOSafe / DOS.AI (write to Supabase directly) |

Products **never call DOS.Me per request** — they read/write Supabase directly to avoid latency.

### Two billing models

**Consumer subscription** — for end users of DOSafe, DOS.AI, DOS.Me products:
- Monthly plan with included quota
- One subscription unlocks all consumer products
- Quota enforced per product via `dosafe.client_quota` / `dosai.dosafe_usage`

**Credits (B2B / developer)** — for API key holders (Bexly, Rate.Box, developers):
- Buy credits, spend per request, no monthly reset
- `dosafe.api_keys` linked to `public.billing_accounts`
- Deduction written to `public.credit_transactions`

### Credit pricing (DOSafe)

| Endpoint | Credits/request |
|----------|----------------|
| `/check`, `/check/bulk` (per entity) | 1 |
| `/url-check` | 1 |
| `/entity-check` | 1 |
| `/detect` (AI text) | 10 |
| `/detect-image` | 20 |

### Tables needed from DOS.Me

```sql
-- public schema (DOS.Me to create)
public.subscriptions (
  id, profile_id, plan, status,
  period_start, period_end, stripe_subscription_id
)

-- public.billing_accounts already exists
-- public.credit_transactions already exists
```

---

## Safety Data Flow

```
External threat sources (13 feeds)
  │  ScamSniffer, MetaMask, Phishing.Database, URLhaus,
  │  OpenPhish, checkscam.vn, admin.vn, scam.vn, etc.
  │
  ▼
dosafe.raw_imports (staging)
  │  pg_cron: sync-threats-6h, sync-phishing-db, daily-scraped
  │
  ▼
dosafe.threat_intel (1.52M+ entries)
dosafe.threat_clusters (89k+ scammer groups)
  │
  ├── DOSafe entity-check / url-check  ← direct query (dosafe.threat_intel)
  │
  └── (Planned) POST api-v2.dos.me/trust/flags
            ↓
      public.safety_flags (confirmed flags only)
            ↓
      Rate.Box / Bexly / dos.me  ← via POST /trust/check
            ↓
      (optional) DOS Chain EAS  ← via POST /trust/flags/:id/attest
```

---

## DOSafe — Architecture Detail

### What is DOSafe?

Safety platform that detects AI-generated content and identifies online scams. Serves users through: web app (`dosafe.io`), Telegram bot, Chrome extension, and mobile app.

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         INTERFACES                               │
│                                                                  │
│  dosafe.io (web)   Telegram Bot   Chrome Extension   Mobile     │
│  Next.js 16        Supabase Edge  Content script     Flutter    │
│  Vercel            Deno 2         Manifest V3        iOS/Android│
│       │                 │                │               │       │
│       └─────────────────┼────────────────┘               │       │
│                         ▼                                 │       │
│                  API Routes (Next.js)              ←──────┘       │
│                  dosafe.io/api/*                                   │
├──────────────────────────────────────────────────────────────────┤
│                         API LAYER                                │
│                                                                  │
│  /api/detect          AI text detection (multi-tier)             │
│  /api/detect-image    AI image detection (C2PA + EXIF + LLM)    │
│  /api/url-check       URL/domain scam check (DB + runtime)      │
│  /api/entity-check    Phone/wallet/email risk check              │
│  /api/extract-text    Document text extraction (PDF/DOCX)        │
│  /api/quota           Quota status query                         │
├──────────────────────────────────────────────────────────────────┤
│                    DATA & INTELLIGENCE                            │
│                                                                  │
│  Supabase PostgreSQL (shared instance)                           │
│  ├── dosafe schema: threat_intel, threat_clusters, raw_imports   │
│  ├── dosai schema: dosafe_usage (quota tracking)                 │
│  └── public schema: profiles, billing_accounts, bot_quota       │
│                                                                  │
│  DOS-Me Trust API (api-v2.dos.me/trust)                         │
│  └── Source of confirmed community flags (M2M, read + write)    │
│                                                                  │
│  DOS Chain (EAS attestations)                                    │
│  └── Schema 6: dos.entity.flag (on-chain safety flags)           │
│                                                                  │
│  External APIs                                                   │
│  ├── vLLM (api.dos.ai) — Qwen3.5-35B scorer                    │
│  ├── vLLM (inference-ref.dos.ai) — Qwen3-8B observer            │
│  ├── Google Safe Browsing, Cloud Vision (reverse image)          │
│  ├── Serper / SerpApi (web search + reverse image fallback)      │
│  └── RDAP/WHOIS (domain registration date)                       │
└──────────────────────────────────────────────────────────────────┘
```

### Project Structure

```
DOSafe/
├── apps/
│   ├── web/                    # Main web app (dosafe.io) — Next.js 16 + React 19
│   │   └── src/
│   │       ├── app/
│   │       │   ├── api/        # API routes
│   │       │   ├── page.tsx    # Landing page
│   │       │   └── ...         # Static pages (about, pricing, privacy, terms)
│   │       └── lib/            # Shared libraries
│   │           ├── entity-scoring.ts   # V2 risk scoring engine (tiers, freshness, corroboration)
│   │           ├── entity-web-search.ts # Web search + LLM entity analysis
│   │           ├── threat-intel.ts     # Threat DB lookup/upsert
│   │           ├── doschain.ts         # DOS Chain EAS queries (viem)
│   │           ├── dosme-trust.ts      # DOS.Me Identity API client
│   │           ├── dosafe-quota.ts     # Quota management
│   │           ├── trusted-domains.ts  # Whitelisted domains (44 entries)
│   │           ├── url-normalize.ts    # URL normalization + hashing
│   │           └── dosai-session.ts    # Auth session validation
│   └── extension/              # Chrome extension (Manifest V3) — v0.5.4
│       ├── manifest.json
│       ├── popup.html/css      # Side panel UI
│       ├── background.js       # Service worker + icon badge management
│       ├── protection-content.js  # Real-time URL protection overlay
│       ├── content-facebook.js    # Facebook-specific content script
│       └── content-dosafe-auth.js # Extension auth callback
├── supabase/
│   ├── functions/
│   │   ├── dosafe-telegram/    # Telegram bot (Deno 2 Edge Function)
│   │   ├── sync-threats/       # Threat intel sync from external sources (6h)
│   │   ├── sync-scraped-sources/ # Incremental scraper sync (daily)
│   │   └── _shared/            # Shared utilities (detect, quota, llm, telegram, language)
│   ├── migrations/             # DB schema migrations
│   └── config.toml             # Supabase CLI config
├── scripts/
│   ├── scrape-admin-vn.ts         # Bulk scraper for admin.vn (963 entries)
│   ├── scrape-checkscam-vn.ts     # Bulk scraper for checkscam.vn (62k posts)
│   └── deep-scrape-checkscam-vn.ts # Deep scraper: fetch HTML, extract entities + images
├── benchmark/raid/             # RAID benchmark submission scripts
└── docs/
    ├── DOSafe-Architecture.md  # This file
    ├── threat-intel.md         # Threat intelligence system (detailed)
    └── plans/                  # Design docs and implementation plans
```

---

## Core Features

### 1. AI Text Detection (`/api/detect`)

Multi-tier pipeline combining statistical analysis, zero-shot model comparison, and LLM judgment.

```
Input text (50–5000 chars)
  │
  ├── Tier 1: Statistical Analysis (<10ms)
  │   ├── Perplexity (via vLLM logprobs)
  │   └── Burstiness (sentence-level perplexity variance)
  │
  ├── Tier 2: Binoculars — Zero-Shot Model Comparison (~2s)
  │   ├── Scorer: Qwen3.5-35B → compute perplexity
  │   ├── Observer: Qwen3-8B → compute cross-entropy
  │   └── Score = PPL(scorer) / CE(observer)
  │       AI text → both agree → ratio ≈ 1.0
  │       Human text → disagree → ratio >> 1.0
  │
  ├── Tier 3: LLM-as-Judge (~3s)
  │   ├── 19-criteria rubric (A–S)
  │   ├── Vietnamese-specific signals (particles, code-switching)
  │   ├── ESL de-biasing (R, S criteria)
  │   └── Paraphrase defense (P, Q criteria)
  │
  ├── Source Matching (parallel)
  │   └── Serper → SerpApi fallback (verbatim phrase search)
  │
  └── Score blending: 35% statistical + 65% rubric
      → ai_probability (0–100), verdict (AI/Human/Mixed), confidence
```

**Quota:** 10k words/day (anonymous), 50k/month (free), 500k/month (paid). Image = 500 words flat.

### 2. AI Image Detection (`/api/detect-image`)

```
Input image (≤10MB, JPEG/PNG/WEBP/GIF)
  │
  ├── Tier 0: C2PA Content Credentials (<50ms, deterministic)
  │   └── Cryptographic manifest → camera origin OR AI tool (100% accurate)
  │
  ├── Tier 1: Metadata Analysis
  │   ├── EXIF parsing (camera model, GPS, AI tool detection)
  │   ├── JPEG quantization / DCT analysis
  │   └── Reverse image search (Google Cloud Vision → Serper Lens)
  │
  └── Tier 3: LLM Visual Analysis (~3s)
      └── Multimodal analysis (texture, lighting, anatomy, edges)
      → Blend: 30% metadata + 70% visual rubric
      → Trusted source cap: 2+ matches → cap ≤ 35%, 1 match → cap ≤ 50%
```

### 3. URL/Domain Scam Check (`/api/url-check`)

Hybrid DB-first + runtime check pipeline. Uses V2 scoring engine with source tiers, freshness decay, and corroboration bonus. See [threat-intel.md](threat-intel.md) for DB details.

```
Input URL
  │
  ├── 0. Typosquatting detection (sync, <1ms)
  │   └── Levenshtein distance + homoglyph normalization vs 44 trusted domains
  │
  ├── 1. DB lookup (dosafe.threat_intel) — <10ms
  │   ├── Hash URL → lookup_threats()
  │   └── Hash domain → lookup_threats() (skip for trusted domains)
  │
  ├── 2. Phase 1: Runtime checks (parallel)
  │   ├── Trusted domain whitelist (44 major platforms)
  │   ├── Google Safe Browsing API v4
  │   ├── WHOIS/RDAP domain age
  │   ├── Web search via searchEntityWeb() (Serper → SerpApi, 8 results)
  │   └── DOS Chain on-chain attestations
  │
  ├── 3. Phase 2: LLM Analysis (sequential, after Phase 1)
  │   └── analyzeEntityLLM() — Qwen3.5-35B analyzes web results
  │       + threat intel summary → structured signals + Vietnamese summary
  │
  ├── 4. V2 Risk Scoring (computeRiskScoreV2)
  │   ├── Signal weights × source tier multiplier × freshness factor
  │   ├── Corroboration bonus (2+ independent sources)
  │   ├── Confidence level (low / medium / high)
  │   └── → riskScore (0–100) + riskLevel + confidence
  │
  └── 5. Cache runtime result (fire-and-forget, 7-day TTL)

  Extension fast path (X-Client-Type: extension):
    Skips: WHOIS, web search, LLM, on-chain, session check, quota
    Only: DB lookup + Safe Browsing + trusted domain + typosquatting
```

**Response:** `riskScore`, `riskLevel`, `confidence`, `riskSignals[]`, `webAnalysis`, `llmSummary`, `typosquatting`, `threatIntel`, `onChain`

### 4. Entity Risk Check (`/api/entity-check`)

Check risk for phone numbers, wallets, emails, bank accounts, etc. Uses same V2 scoring engine.

```
Input: { entityType, entityId }
  │
  ├── Phase 1 (parallel)
  │   ├── dosafe.threat_intel DB lookup    — 1.52M+ entries
  │   ├── DOS Chain EAS query              — on-chain attestations (15s timeout)
  │   ├── DOS.Me Identity API              — member trust score, flags, DOS ID
  │   └── Web search (Serper → SerpApi)    — 8 results (skip for bulk)
  │
  ├── Phase 2 (sequential)
  │   └── LLM Analysis (analyzeEntityLLM)  — structured signals + Vietnamese summary
  │
  └── V2 Scoring (computeRiskScoreV2)
      → riskScore + riskLevel + confidence + riskSignals + llmSummary
```

**Supported types:** phone, email, wallet, url, domain, bank_account, national_id, telegram, facebook, organization

### 5. Telegram Bot (`supabase/functions/dosafe-telegram`)

Commands: `/detect`, `/scam`, `/phone`, `/quota`, `/help`, `/start`
Auto-detection: URLs, phone numbers, plain text, photos
Bilingual: Vietnamese + English (auto-detected)
Quota: 20 checks/day per chat
Calls: `dosafe.io/api/*` internally (via `DOSAFE_API_URL` secret)

### 6. Chrome Extension (`apps/extension`) — v0.5.4

- **Real-time URL protection**: Auto-checks current tab URL via `/api/url-check` with `X-Client-Type: extension` fast path
- **Icon badge**: Green ✓ (safe/low), Yellow ! (medium), Red ✗ (high/critical) on extension icon
- **Warning overlay**: Full-page overlay for high/critical risk — localized (vi/en), human-readable signal names, real DB sources
- AI text detection on selected text or full page
- Facebook profile analysis (content script)
- Side panel results display
- Auth via `dosafe.io/auth/extension-callback`

### 7. Mobile App (Flutter — `DOSafe-Mobile`)

- Calls `dosafe.io/api/entity-check` for phone/entity lookups
- Base URL: `https://dosafe.io` (via `kDosafeBaseUrl`)

---

## Threat Intelligence Pipeline

Detailed in [threat-intel.md](threat-intel.md).

**Summary:**
- **1.52M+ entries** from 13 sources: Phishing.Database (~450k), ScamSniffer (346k), nTrust/NCA (252k), MetaMask (234k), Tín Nhiệm Mạng (97k), TrueCaller (40k), ChongLuaDao (34k), checkscam.vn (27k), URLhaus (26k), scam.vn (10k), admin.vn (963), OpenPhish
- **Schema:** `dosafe.threat_intel` + `dosafe.threat_clusters` (89k+ cluster links)
- **Entity types:** domain, url, wallet, phone, bank_account, facebook, email, name
- **Sync schedules (pg_cron):**
  - `sync-threats-6h` — structured feeds (ScamSniffer, MetaMask, URLhaus, etc.)
  - `sync-phishing-db-p1/p2` (03:00/03:05 UTC) — Phishing.Database (~711k domains)
  - `daily-scraped-sources-sync` (02:00 UTC) — checkscam.vn, scam.vn
- **Storage:** `evidence` bucket (Supabase Storage) for scam screenshots

### Sync to DOS-Me Trust API (Planned)

```
dosafe.threat_intel
  → filter: category IN ('phishing','scam','malware'), confidence >= threshold
  → POST api-v2.dos.me/trust/flags  (M2M, scope: trust.flag)
  → public.safety_flags (confirmed, visible to Rate.Box / Bexly)
```

---

## Risk Scoring Engine V2

**Implementation:** `src/lib/entity-scoring.ts` — shared engine used by both `/api/url-check` and `/api/entity-check`.

### Concept

The V1 scoring system used flat signal weights — every source contributing the same amount regardless of quality, freshness, or corroboration. This led to:

- **False positives**: Crowdsourced Vietnamese sources (scam.vn, checkscam.vn) flagging major platforms like google.com
- **Stale data inflation**: Year-old reports from a single unverified source scoring as high-risk
- **No confidence signal**: Users couldn't tell if a "medium risk" was backed by 5 authoritative sources or 1 crowdsourced report

V2 solves these with 5 mechanisms: **source tiers**, **freshness decay**, **corroboration bonus**, **confidence levels**, and **typosquatting detection**.

### Design Principles

1. **No single source determines verdict alone** — base score is 15 (low), signals add/subtract
2. **Higher-quality sources have more influence** — Google Safe Browsing (tier 1) carries more weight than runtime_cache (tier 4)
3. **Recent reports weigh more** — a phishing report from yesterday is more relevant than one from 2 years ago
4. **Multiple independent sources increase confidence** — 3 sources saying "scam" is more trustworthy than 1
5. **Trusted domains bypass domain-level DB lookup** — prevents false positives from crowdsourced reports on major platforms

### Source Tiers

Each threat intel source is classified into a quality tier. The tier multiplier scales the signal's base weight.

| Tier | Multiplier | Sources | Rationale |
|------|-----------|---------|-----------|
| **1 — Authoritative** | 1.0 | Google Safe Browsing, Tín Nhiệm Mạng (NCSC VN), tinnhiemmang_trusted_webs/orgs | Government/industry authority, low false positive rate |
| **2 — Curated Security** | 0.85 | MetaMask, Phishing.Database, ScamSniffer, URLhaus, OpenPhish | Security teams with review processes |
| **3 — Crowdsourced VN** | 0.70 | checkscam.vn, scam.vn, admin.vn, nTrust/NCA, TrueCaller, ChongLuaDao | Community-driven, higher noise, valuable for VN-specific threats |
| **4 — Heuristic** | 0.50 | runtime_cache | Our own cached results, may be stale or from incomplete checks |

**Scoring formula for DB signals:**
```
effective_weight = base_weight × tier_multiplier × freshness_factor
```

For non-DB signals (on-chain, DOS.Me, web search, URL-specific), base weight is used directly.

### Freshness Decay

Reports decay in influence as they age, measured by `last_seen_at`:

| Age | Factor | Rationale |
|-----|--------|-----------|
| ≤ 30 days | 1.0 | Fresh, highly relevant |
| ≤ 90 days | 0.85 | Recent, still relevant |
| ≤ 365 days | 0.70 | Aging, may no longer be active |
| ≤ 2 years | 0.50 | Old, likely inactive but retain historical context |
| > 2 years | 0.30 | Very old, minimal influence — scam sites almost certainly dead |

If `last_seen_at` is null: factor = 0.70 (treat as aging).

### Corroboration Bonus

Multiple independent risk sources increase the score and confidence:

| Unique risk sources | Bonus |
|---------------------|-------|
| 4+ | +20 |
| 3 | +15 |
| 2 | +10 |
| 1 | 0 |

Sources counted: all DB sources (excluding `google_safe_browsing` for "clean" results and `runtime_cache`), plus `onchain` and `google_safe_browsing` when they report actual threats.

### Confidence Levels

Returned alongside riskScore to help UIs and downstream consumers calibrate trust:

| Level | Criteria |
|-------|----------|
| **high** | 3+ unique risk sources, OR tier 1 source + 2+ total sources |
| **medium** | 2+ unique risk sources, OR 1 tier 1 source |
| **low** | 0–1 risk sources, no tier 1 |

### Typosquatting Detection

Detects domains that visually resemble trusted brands:

1. **Levenshtein distance**: Compares domain base name against 44 trusted domains. Distance 1–2 for domains ≥ 5 chars, distance 1 for shorter.
2. **Homoglyph normalization**: Cyrillic `а` → `a`, `е` → `e`, `0` → `o`, `1` → `l`, etc. — catches `gооgle.com` (Cyrillic o's).
3. **Signal**: `typosquatting_suspected` (+25 weight) with `similarTo` field identifying the targeted brand.

### Signal Weights Reference (V2)

#### Threat DB Signals (× tier × freshness)

| Signal | Base Weight |
|--------|------------|
| `db_flagged_phishing` | +85 |
| `db_flagged_malware` | +80 |
| `db_flagged_fraud` | +75 |
| `db_flagged_scam` | +70 |
| `db_flagged_spam` | +50 |
| `db_flagged_robocall` | +45 |
| `db_flagged_unwanted` / `db_flagged_political` | +40 |
| `db_verified_legitimate` | −40 |
| `db_verified_clean` | −25 |
| `db_very_high_report_count` (≥1000 reports) | +35 |
| `db_high_report_count` (≥100 reports) | +20 |
| `cluster_linked` | +12 |

#### On-Chain Signals (no tier reduction)

| Signal | Weight |
|--------|--------|
| `onchain_flagged_phishing` | +85 |
| `onchain_flagged_scam` | +70 |
| `onchain_high_risk` (≥80) | +60 |
| `onchain_medium_risk` (50–79) | +30 |
| `onchain_verified_legitimate` | −40 |
| `onchain_trusted` (<20) | −25 |

#### DOS.Me Identity Signals

| Signal | Weight |
|--------|--------|
| `dosme_member_flagged` | +40 |
| `dosme_trust_passing` | −20 |
| `dosme_high_trust` | −15 |
| `dosme_multi_verified` | −10 |
| `dosme_medium_trust` / `dosme_has_dosid` | −5 |

#### Web Search + LLM Signals

| Signal | Weight |
|--------|--------|
| `web_identified_scam` | +55 |
| `web_scam_reports` | +35 |
| `web_spam_reports` | +25 |
| `web_identified_brand` | −15 |
| `web_identified_government` | −25 |
| `web_mixed_signals` | +8 |

#### URL-Specific Signals (extra weights in url-check only)

| Signal | Weight |
|--------|--------|
| `safe_browsing_malware` / `safe_browsing_social_engineering` | +90 |
| `safe_browsing_unwanted_software` | +70 |
| `safe_browsing_potentially_harmful_application` | +60 |
| `new_domain` (<30 days) | +25 |
| `young_domain` (<90 days) | +10 |
| `trusted_domain` | −40 |
| `typosquatting_suspected` | +25 |

### Score → Risk Level

| Score | Level |
|-------|-------|
| 0–19 | `safe` |
| 20–49 | `low` |
| 50–74 | `medium` |
| 75–89 | `high` |
| 90–100 | `critical` |

### Web Search + LLM Pipeline

**Implementation:** `src/lib/entity-web-search.ts`

The V1 URL pipeline used simple keyword matching (`hasScamTerms()`) on web search results. V2 replaces this with a full LLM analysis pipeline:

```
1. searchEntityWeb(entityType, entityId)
   └── Builds entity-type-aware query (phone: local+intl format, domain: +scam/review terms)
   └── Serper → SerpApi fallback, 8 results

2. analyzeEntityLLM(entityType, entityId, webResults, threatIntelSummary)
   └── Qwen3.5-35B-A3B-GPTQ-Int4 (api.dos.ai)
   └── Compact prompt: entity + top 5 results (120-char snippets) + threat intel summary
   └── Returns: entity_identity, risk_level, risk_score, signals[], summary (Vietnamese)
   └── max_tokens=250, temperature=0.1

3. getWebSignals(webAnalysis)
   └── Filters LLM signals to valid set: web_identified_scam, web_scam_reports, etc.
```

**Performance:** Web search runs parallel with Phase 1 (DB/on-chain/DOS.Me). Only LLM analysis waits for Phase 1 results. Total added latency: ~3–5s for full path (skipped on extension fast path).

---

## Database Layout

### Supabase Schemas

| Schema | Owner | Tables |
|--------|-------|--------|
| `dosafe` | DOSafe | `threat_intel`, `threat_clusters`, `raw_imports`, `sync_log` |
| `dosai` | DOS.AI | `dosafe_usage`, `user_settings`, `usage_transactions` |
| `public` | Shared | `profiles`, `billing_accounts`, `credit_transactions`, `safety_flags`, `bot_quota`, `organizations`, `fed_profiles` |

### Key Tables

| Table | Schema | Purpose |
|-------|--------|---------|
| `threat_intel` | dosafe | Unified threat data (1.52M+ entries, all sources) |
| `threat_clusters` | dosafe | Scammer group linking (89k+ clusters) |
| `raw_imports` | dosafe | Staging for scraped reports |
| `sync_log` | dosafe | Sync health monitoring |
| `api_keys` | dosafe | DOSafe API key registry (SHA-256 hashed, scoped) |
| `safety_flags` | public | Confirmed flags shared with other products via Trust API |
| `bot_quota` | public | Telegram bot daily limits |
| `dosafe_usage` | dosai | User quota tracking (reads: `billing_accounts` for paid check) |

---

## Infrastructure

### Compute

| Service | Host | Purpose |
|---------|------|---------|
| Scorer (Qwen3.5-35B-A3B-GPTQ-Int4) | `api.dos.ai` (RTX Pro 6000, 96GB) | LLM inference — text scoring + image analysis |
| Observer (Qwen3-8B base) | `inference-ref.dos.ai` (RTX 5090, 32GB) | Binoculars cross-entropy |
| Web app | Vercel (serverless) | Next.js API routes + static pages |
| Edge Functions | Supabase (Deno 2) | Telegram bot + threat sync (pg_cron) |
| Database | Supabase PostgreSQL 17 | All data storage |
| DOS-Me API | Google Cloud Run (asia-southeast1) | NestJS main backend |
| DOS Chain | DOS Testnet (3939) | EVM — on-chain safety attestations |
| API Gateway | Cloudflare Workers | `api.dos.ai` — enterprise key auth + billing |

### Key Environment Variables

```env
# Supabase (shared instance)
NEXT_PUBLIC_SUPABASE_URL=https://gulptwduchsjcsbndmua.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# AI Inference
DOS_INFERENCE_API_KEY=...
VLLM_OBSERVER_URL=https://inference-ref.dos.ai

# External APIs (all optional, degrade gracefully)
GOOGLE_SAFE_BROWSING_API_KEY=...
GOOGLE_CLOUD_VISION_API_KEY=...
SERPER_API_KEY=...
SERPAPI_API_KEY=...

# DOS Chain
DOS_TESTNET_RPC=https://test.doschain.com/...

# Telegram bot
TELEGRAM_BOT_TOKEN=...
DOSAFE_API_URL=https://dosafe.io

# DOS-Me Trust API (M2M — planned)
DOS_ME_API_URL=https://api-v2.dos.me
DOS_ME_TRUST_API_KEY=...
```

---

## AI Detection Research

### Text Detection — 19-Criteria Rubric (A–S)

**AI indicators (+):** Structure (A), Transitions (B), Consistent register (C), Even emotion (D), Generic examples (E), Hedging overuse (F), Uniform complexity (G), Repetitive structures (H), Paraphrase markers (P), Bypasser artifacts (Q)

**Human indicators (−):** Unexpected detail (I), Authentic messiness (J), Natural errors (K), Emotional spikes (L), ESL markers (R), Formulaic academic (S)

**Vietnamese-specific (−):** Particle usage (M), Code-switching (N), Informal markers (O)

### Key Papers

| Method | Paper | Use in DOSafe |
|--------|-------|---------------|
| Binoculars | ICML 2024 | Tier 2 — PPL ratio between scorer/observer |
| C2PA | c2pa.org v2.1 | Tier 0 — deterministic image provenance |
| DivEye | 2025 | Informed rubric design (surprisal variability) |

### RAID Benchmark

| | |
|---|---|
| **Script** | `benchmark/raid/raid_submit.py` |
| **Total** | 672,000 texts |
| **Train AUROC** | **0.852** (target: 0.80+) |
| **Status** | COMPLETE — PR #90 submitted |

### vLLM Model Selection

| Model | VRAM | Throughput @50 | Scam F1 | Status |
|-------|------|----------------|---------|--------|
| Qwen3.5-35B-A3B-GPTQ-Int4 | 21GB | 3,373 tok/s | 0.970 | **Production (scorer)** |
| Qwen3-8B base | ~16GB | — | — | **Production (observer)** |
| Qwen3.5-27B-GPTQ-Int4 | 27.5GB | 1,518 tok/s | 0.880 | Evaluated, not used |

### Image Detection Methods

- **C2PA:** Cryptographic content credentials (~40% of AI images have C2PA in 2026)
- **EXIF:** Camera metadata + AI tool detection in Software field
- **DCT:** JPEG quantization table analysis (camera-specific vs AI generic)
- **Reverse search:** Google Cloud Vision WEB_DETECTION → Serper Lens fallback
- **LLM visual:** Multimodal rubric analysis — Qwen3.5-35B (natively multimodal, no separate VL model needed)

---

## Roadmap

### Completed
- [x] Multi-tier text detection (statistical + Binoculars + LLM rubric)
- [x] Image detection (C2PA + EXIF/DCT + reverse search + LLM)
- [x] Paraphrase shield + ESL de-biasing
- [x] URL/domain scam check with on-chain integration
- [x] Telegram bot with bilingual support
- [x] Chrome extension with real-time URL protection (v0.5.4)
- [x] Mobile app (Flutter, calls dosafe.io/api/*)
- [x] Threat intelligence pipeline (1.52M+ entries, 13 sources)
- [x] Quota system (anonymous + authenticated)
- [x] Vietnamese scraper pipeline (admin.vn + checkscam.vn + scam.vn → raw_imports → threat_intel)
- [x] nTrust (NCA) + TrueCaller phone DB extraction (292k entries)
- [x] Deep scraping with evidence image extraction + Supabase Storage upload
- [x] Entity clustering (89k+ clusters)
- [x] RAID benchmark submission (672k texts, AUROC 0.852)
- [x] **Risk Scoring V2**: source tiers, freshness decay, corroboration bonus, confidence levels
- [x] **Typosquatting detection**: Levenshtein + homoglyph normalization
- [x] **LLM web search pipeline**: replaced keyword matching with Qwen3.5-35B analysis
- [x] **Extension protection overlay**: localized (vi/en), human-readable signals, real DB sources
- [x] **Partner API / DOS Shield Gateway**: `/api/v1/check`, `/check/bulk`, `/report`
- [x] **DOS.Me Identity integration**: member trust score, flags, verified providers

### Planned
- [ ] Sync confirmed flags to DOS-Me Trust API (`POST /trust/flags`)
- [ ] Enterprise API gateway (api.dos.ai Worker proxying dosafe.io endpoints)
- [ ] User report command (/report) with LLM entity extraction
- [ ] LLM evidence image analysis (classify screenshots as chat/transfer/profile)
- [ ] Caller ID / spam phone lookup (iCallMe-like feature)
- [ ] Additional Vietnamese sources (kiemtraluadao.vn, etc.)
- [ ] Audio detection pipeline (TTS/voice cloning)
- [ ] Video detection pipeline (deepfake)
- [ ] Binoculars threshold calibration (Vietnamese + English corpus)
