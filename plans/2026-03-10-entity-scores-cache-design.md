# Entity Scores Materialized Cache

**Date:** 2026-03-10
**Status:** Approved

## Problem

Every API request (url-check, entity-check) re-computes risk scores from scratch:
- Queries threat_intel DB (free, fast)
- Calls 5+ external APIs: Safe Browsing, WHOIS, on-chain EAS, DOS.Me, web search (paid/slow)
- Runs LLM analysis on web results (compute-heavy)

Total latency: up to 30s. External APIs cost money per call. Same entity checked multiple times wastes both time and money.

## Solution

Add `dosafe.entity_scores` table as a materialized cache layer. Store computed scores + external API results with separate TTLs. Subsequent lookups return cached score in < 5ms. Only refresh when data is stale.

## Design Principles

1. **Internal DB queries are free** — always query threat_intel fresh
2. **External APIs cost money** — cache aggressively with long TTLs
3. **Different sources change at different rates** — separate TTLs per source
4. **Extension needs < 50ms response** — single-row DB lookup, no external calls
5. **Score quality must stay high** — re-compute when underlying data changes

## Database Schema

```sql
CREATE TABLE dosafe.entity_scores (
  entity_type       TEXT NOT NULL,
  entity_hash       TEXT NOT NULL,
  entity_id         TEXT NOT NULL,

  -- Computed score
  risk_score        SMALLINT NOT NULL,
  risk_level        TEXT NOT NULL,        -- safe/low/medium/high/critical
  confidence        TEXT NOT NULL,        -- low/medium/high
  signals           TEXT[] NOT NULL DEFAULT '{}',

  -- Expensive cached results
  llm_summary       TEXT,
  web_analysis      JSONB,               -- { webResults, llmAnalysis }
  external_results  JSONB NOT NULL DEFAULT '{}',
  -- {
  --   safeBrowsing: { threats, checkedAt },
  --   whois: { ageDays, registrar, checkedAt },
  --   onChain: { attestationCount, latestRiskScore, checkedAt },
  --   dosMe: { found, trustScore, isFlagged, checkedAt },
  --   webSearch: { resultCount, checkedAt }
  -- }

  -- Change detection
  threat_intel_hash TEXT,                -- MD5 of sorted entry hashes+scores

  -- Timestamps
  computed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at        TIMESTAMPTZ NOT NULL,

  PRIMARY KEY (entity_type, entity_hash)
);

CREATE INDEX idx_entity_scores_expires ON dosafe.entity_scores (expires_at);
```

## External API TTLs

| Source | TTL | Rationale |
|--------|-----|-----------|
| Safe Browsing | 7 days | Rarely changes for same URL |
| WHOIS | 30 days | Domain age is static |
| On-chain EAS | 24h | New attestations are rare |
| DOS.Me Identity | 24h | Profile changes infrequently |
| Web search | 7 days | Paid API (Serper/SerpApi) — future: free via SearXNG |
| LLM analysis | 24h | Re-analyze when new threat data arrives |

Score TTL: **6 hours** — re-compute from fresh DB data + cached external results.

## Cache Lookup Flow

```
Request
│
├─ 1. Query entity_scores WHERE entity_type + entity_hash
│   ├─ HIT + expires_at > NOW() → return cached (< 5ms)
│   ├─ HIT + expired → step 2 (smart refresh)
│   └─ MISS → step 3 (full compute)
│
├─ 2. Smart Refresh
│   ├─ Query threat_intel → compute threat_intel_hash
│   ├─ Compare hash with cached
│   │   ├─ SAME + all external TTLs fresh
│   │   │   → bump expires_at +6h, return same score
│   │   ├─ DIFFERENT (DB changed) + external fresh
│   │   │   → re-compute score from new DB + cached external_results
│   │   └─ External TTL expired
│   │       → call ONLY expired external APIs, re-compute
│   └─ UPSERT entity_scores
│
└─ 3. Full Compute (first time ever)
    ├─ Query threat_intel (free)
    ├─ Parallel: SB + WHOIS + on-chain + DOS.Me + web search
    ├─ Sequential: LLM analysis (after web results)
    ├─ computeRiskScoreV2()
    ├─ INSERT entity_scores
    └─ return score
```

## Consumer Integration

### Extension Fast Path
- Query `entity_scores` only (1 row)
- If MISS → fallback: query threat_intel + computeRiskScoreV2() (no external APIs)
- NEVER calls external APIs

### Full Path (bot/app/web)
- Query `entity_scores`
- HIT + fresh → return cached
- Stale/MISS → smart refresh (only call expired externals)

### Bulk Endpoint
- Batch query `entity_scores` for up to 50 entities
- Return cached results
- Queue MISSes for background refresh (phase 2)

## Change Detection

`threat_intel_hash` = MD5 of sorted `(entry_id, risk_score)` pairs for this entity.

When scrapers add/update entries (every 6h), the hash changes → triggers score re-computation on next request. But external API results are preserved if still within their TTL.

## Cleanup

```sql
-- pg_cron: remove stale scores not refreshed in 30 days
DELETE FROM dosafe.entity_scores
WHERE expires_at < NOW() - INTERVAL '30 days';
```

## Future: Background Refresh (Phase 2)

- pg_cron job every 6h: refresh top 1000 most-queried entities
- Ensures extension always gets a HIT
- Can track query_count on entity_scores for popularity ranking

## Future: Self-hosted Search (SearXNG)

Replace Serper/SerpApi with self-hosted SearXNG meta-search engine:
- Docker container, zero API cost
- Aggregates Google/Bing/DuckDuckGo
- Web search TTL can drop from 7 days to 24h once free
- Full pipeline: SearXNG → LLM analyze → cached

## Implementation Files

- `supabase/migrations/YYYYMMDD_entity_scores.sql` — table + RPC functions
- `apps/web/src/lib/entity-score-cache.ts` — cache lookup, smart refresh, TTL logic
- `apps/web/src/app/api/url-check/route.ts` — integrate cache layer
- `apps/web/src/app/api/entity-check/route.ts` — integrate cache layer
- `apps/web/src/lib/entity-scoring.ts` — add threat_intel_hash computation
