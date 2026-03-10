# Entity Scores Cache Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add materialized score cache (`dosafe.entity_scores`) so repeated lookups return pre-computed results in < 5ms instead of re-running expensive external API calls.

**Architecture:** New `entity_scores` table stores computed risk scores + external API results with separate TTLs. A `entity-score-cache.ts` lib handles cache lookup, smart refresh (only call expired externals), and upsert. Both `url-check` and `entity-check` routes integrate this cache layer. Extension fast path becomes a single-row DB read.

**Tech Stack:** Supabase (PostgreSQL), Next.js API routes, TypeScript

---

### Task 1: Database Migration — `entity_scores` table + RPC

**Files:**
- Create: `supabase/migrations/20260311000000_entity_scores.sql`

**Step 1: Write migration SQL**

```sql
-- Materialized cache for computed entity risk scores
-- Stores V2 scores + external API results with separate TTLs
-- Extension fast path: single-row lookup (< 5ms)

CREATE TABLE IF NOT EXISTS dosafe.entity_scores (
  entity_type       TEXT NOT NULL,
  entity_hash       TEXT NOT NULL,
  entity_id         TEXT NOT NULL,

  -- Computed score
  risk_score        SMALLINT NOT NULL,
  risk_level        TEXT NOT NULL,
  confidence        TEXT NOT NULL DEFAULT 'low',
  signals           TEXT[] NOT NULL DEFAULT '{}',

  -- Expensive cached results
  llm_summary       TEXT,
  web_analysis      JSONB,
  external_results  JSONB NOT NULL DEFAULT '{}',

  -- Change detection: MD5 of sorted threat_intel entry IDs + scores
  threat_intel_hash TEXT,

  -- Timestamps
  computed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at        TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '6 hours'),

  PRIMARY KEY (entity_type, entity_hash)
);

CREATE INDEX IF NOT EXISTS idx_entity_scores_expires
  ON dosafe.entity_scores (expires_at);

-- Fast lookup RPC: returns cached score if not expired
CREATE OR REPLACE FUNCTION dosafe.get_cached_score(
  p_entity_type TEXT,
  p_entity_hash TEXT
)
RETURNS TABLE (
  entity_type TEXT,
  entity_hash TEXT,
  entity_id TEXT,
  risk_score SMALLINT,
  risk_level TEXT,
  confidence TEXT,
  signals TEXT[],
  llm_summary TEXT,
  web_analysis JSONB,
  external_results JSONB,
  threat_intel_hash TEXT,
  computed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ
)
LANGUAGE sql STABLE
AS $$
  SELECT entity_type, entity_hash, entity_id,
         risk_score, risk_level, confidence, signals,
         llm_summary, web_analysis, external_results,
         threat_intel_hash, computed_at, expires_at
  FROM dosafe.entity_scores
  WHERE entity_type = p_entity_type
    AND entity_hash = p_entity_hash;
$$;

-- Upsert RPC: insert or update cached score
CREATE OR REPLACE FUNCTION dosafe.upsert_entity_score(
  p_entity_type TEXT,
  p_entity_hash TEXT,
  p_entity_id TEXT,
  p_risk_score SMALLINT,
  p_risk_level TEXT,
  p_confidence TEXT,
  p_signals TEXT[],
  p_llm_summary TEXT,
  p_web_analysis JSONB,
  p_external_results JSONB,
  p_threat_intel_hash TEXT,
  p_expires_at TIMESTAMPTZ
)
RETURNS VOID
LANGUAGE sql
AS $$
  INSERT INTO dosafe.entity_scores (
    entity_type, entity_hash, entity_id,
    risk_score, risk_level, confidence, signals,
    llm_summary, web_analysis, external_results,
    threat_intel_hash, computed_at, expires_at
  ) VALUES (
    p_entity_type, p_entity_hash, p_entity_id,
    p_risk_score, p_risk_level, p_confidence, p_signals,
    p_llm_summary, p_web_analysis, p_external_results,
    p_threat_intel_hash, NOW(), p_expires_at
  )
  ON CONFLICT (entity_type, entity_hash)
  DO UPDATE SET
    entity_id = EXCLUDED.entity_id,
    risk_score = EXCLUDED.risk_score,
    risk_level = EXCLUDED.risk_level,
    confidence = EXCLUDED.confidence,
    signals = EXCLUDED.signals,
    llm_summary = EXCLUDED.llm_summary,
    web_analysis = EXCLUDED.web_analysis,
    external_results = EXCLUDED.external_results,
    threat_intel_hash = EXCLUDED.threat_intel_hash,
    computed_at = NOW(),
    expires_at = EXCLUDED.expires_at;
$$;

-- Cleanup: pg_cron job to remove stale scores (not refreshed in 30 days)
-- Run daily at 04:00 UTC
SELECT cron.schedule(
  'cleanup-stale-entity-scores',
  '0 4 * * *',
  $$DELETE FROM dosafe.entity_scores WHERE expires_at < NOW() - INTERVAL '30 days'$$
);
```

**Step 2: Apply migration**

Run: `npx supabase db push` or apply via Supabase dashboard SQL editor.

**Step 3: Verify table exists**

```sql
SELECT * FROM dosafe.entity_scores LIMIT 0;
SELECT * FROM dosafe.get_cached_score('domain', 'test');
```

**Step 4: Commit**

```bash
git add supabase/migrations/20260311000000_entity_scores.sql
git commit -m "feat(db): add entity_scores materialized cache table + RPCs"
```

---

### Task 2: Cache Library — `entity-score-cache.ts`

**Files:**
- Create: `apps/web/src/lib/entity-score-cache.ts`
- Read (reference): `apps/web/src/lib/threat-intel.ts` (for Supabase client pattern, `hashEntity`, `lookupThreats`)
- Read (reference): `apps/web/src/lib/entity-scoring.ts` (for `computeRiskScoreV2`, `assessEntity`, types)

**Step 1: Create cache library**

This file handles:
- `getCachedScore()` — single-row DB lookup
- `computeThreatIntelHash()` — MD5 of threat entries for change detection
- `isExternalExpired()` — check individual external result TTLs
- `saveCachedScore()` — upsert to DB
- `getOrComputeScore()` — main entry: cache hit → return, miss → compute + save

```typescript
import { createClient } from '@supabase/supabase-js'
import { lookupThreats, type ThreatLookupResult, type ThreatEntry } from './threat-intel'
import { computeRiskScoreV2, type ScoringResult } from './entity-scoring'
import { createHash } from 'crypto'

// ─── Supabase client (same pattern as threat-intel.ts) ───

const SUPABASE_URL = process.env.NEXT_PUBLIC_SUPABASE_URL ?? process.env.SUPABASE_URL ?? ''
const SUPABASE_SERVICE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY ?? ''

let _client: ReturnType<typeof createClient> | null = null
function getClient() {
  if (!_client) {
    _client = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY, {
      auth: { autoRefreshToken: false, persistSession: false },
    })
  }
  return _client
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function dosafe(): any {
  return (getClient() as any).schema('dosafe')
}

// ─── Types ───

export interface CachedScore {
  entity_type: string
  entity_hash: string
  entity_id: string
  risk_score: number
  risk_level: 'safe' | 'low' | 'medium' | 'high' | 'critical'
  confidence: 'low' | 'medium' | 'high'
  signals: string[]
  llm_summary: string | null
  web_analysis: Record<string, unknown> | null
  external_results: ExternalResults
  threat_intel_hash: string | null
  computed_at: string
  expires_at: string
}

export interface ExternalResults {
  safeBrowsing?: { threats: string[]; checkedAt: string }
  whois?: { ageDays: number | null; registrar: string | null; checkedAt: string }
  onChain?: { attestationCount: number; latestRiskScore: number; checkedAt: string }
  dosMe?: { found: boolean; trustScore: number | null; isFlagged: boolean; checkedAt: string }
  webSearch?: { resultCount: number; checkedAt: string }
  llmAnalysis?: { checkedAt: string }
}

// ─── TTL Configuration (hours) ───

const SCORE_TTL_HOURS = 6
const EXTERNAL_TTLS: Record<string, number> = {
  safeBrowsing: 7 * 24,   // 7 days
  whois: 30 * 24,          // 30 days
  onChain: 24,             // 24h
  dosMe: 24,               // 24h
  webSearch: 7 * 24,       // 7 days
  llmAnalysis: 24,         // 24h
}

// ─── Hash entity value (same as threat-intel.ts) ───

function hashEntity(value: string): string {
  const normalized = value.toLowerCase().trim()
  return '0x' + createHash('sha256').update(normalized).digest('hex')
}

// ─── Threat intel change detection ───

export function computeThreatIntelHash(entries: ThreatEntry[]): string {
  if (!entries || entries.length === 0) return 'empty'
  const sorted = [...entries]
    .sort((a, b) => (a.entity_hash ?? '').localeCompare(b.entity_hash ?? '') || (a.source ?? '').localeCompare(b.source ?? ''))
    .map(e => `${e.source}:${e.risk_score}:${e.last_seen_at ?? ''}`)
    .join('|')
  return createHash('md5').update(sorted).digest('hex')
}

// ─── Check if external result TTL expired ───

export function isExternalExpired(checkedAt: string | undefined, source: string): boolean {
  if (!checkedAt) return true
  const ttlHours = EXTERNAL_TTLS[source] ?? 24
  const age = Date.now() - new Date(checkedAt).getTime()
  return age > ttlHours * 60 * 60 * 1000
}

// ─── Get list of expired external sources ───

export function getExpiredExternals(external: ExternalResults): string[] {
  const expired: string[] = []
  for (const [source, ttlHours] of Object.entries(EXTERNAL_TTLS)) {
    const data = external[source as keyof ExternalResults] as { checkedAt?: string } | undefined
    if (!data?.checkedAt || isExternalExpired(data.checkedAt, source)) {
      expired.push(source)
    }
  }
  return expired
}

// ─── DB Operations ───

export async function getCachedScore(entityType: string, entityId: string): Promise<CachedScore | null> {
  const hash = hashEntity(entityId)
  const { data, error } = await dosafe()
    .rpc('get_cached_score', { p_entity_type: entityType, p_entity_hash: hash })

  if (error || !data || data.length === 0) return null
  return data[0] as CachedScore
}

export async function saveCachedScore(
  entityType: string,
  entityId: string,
  score: ScoringResult & { signals: string[] },
  llmSummary: string | null,
  webAnalysis: Record<string, unknown> | null,
  externalResults: ExternalResults,
  threatIntelHash: string,
): Promise<void> {
  const hash = hashEntity(entityId)
  const expiresAt = new Date(Date.now() + SCORE_TTL_HOURS * 60 * 60 * 1000).toISOString()

  await dosafe().rpc('upsert_entity_score', {
    p_entity_type: entityType,
    p_entity_hash: hash,
    p_entity_id: entityId,
    p_risk_score: score.score,
    p_risk_level: score.riskLevel,
    p_confidence: score.confidence,
    p_signals: score.signals,
    p_llm_summary: llmSummary,
    p_web_analysis: webAnalysis,
    p_external_results: externalResults,
    p_threat_intel_hash: threatIntelHash,
    p_expires_at: expiresAt,
  })
}

// ─── Bump expiry without re-computing ───

export async function bumpScoreExpiry(entityType: string, entityId: string): Promise<void> {
  const hash = hashEntity(entityId)
  const expiresAt = new Date(Date.now() + SCORE_TTL_HOURS * 60 * 60 * 1000).toISOString()

  await dosafe()
    .from('entity_scores')
    .update({ expires_at: expiresAt })
    .eq('entity_type', entityType)
    .eq('entity_hash', hash)
}

// ─── Score freshness check ───

export function isScoreFresh(cached: CachedScore): boolean {
  return new Date(cached.expires_at) > new Date()
}
```

**Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit src/lib/entity-score-cache.ts` (or just build)

**Step 3: Commit**

```bash
git add apps/web/src/lib/entity-score-cache.ts
git commit -m "feat(cache): add entity-score-cache lib with TTL-based smart refresh"
```

---

### Task 3: Integrate Cache into `entity-check` Route

**Files:**
- Modify: `apps/web/src/app/api/entity-check/route.ts`
- Read (reference): `apps/web/src/lib/entity-score-cache.ts`

**Step 1: Add cache imports**

At top of `entity-check/route.ts`, add:

```typescript
import {
  getCachedScore, saveCachedScore, isScoreFresh,
  computeThreatIntelHash, getExpiredExternals,
  bumpScoreExpiry, type ExternalResults
} from '@/lib/entity-score-cache'
```

**Step 2: Add cache layer before `assessEntity()`**

After input validation and phone normalization, before `try { const { ... } = await assessEntity(...)`:

```typescript
// ─── Cache lookup ───
const cached = await getCachedScore(entityType, entityId).catch(() => null)

if (cached && isScoreFresh(cached)) {
  // Cache HIT — return immediately
  if (!dosSession) incrementAnonQuota(ip)
  return NextResponse.json({
    entityType,
    entityId,
    riskScore: cached.risk_score,
    riskLevel: cached.risk_level,
    riskSignals: cached.signals,
    summary: cached.llm_summary,
    onChain: cached.external_results.onChain ?? null,
    threatIntel: null, // not stored in cache, would need separate lookup
    trustedMember: cached.external_results.dosMe ?? null,
    webSearchResults: cached.web_analysis?.webResults ?? null,
    llmAnalysis: cached.web_analysis?.llmAnalysis ?? null,
    cached: true,
    cachedAt: cached.computed_at,
  })
}
```

**Step 3: Save result to cache after full assessment**

After `assessEntity()` returns and before sending response, add:

```typescript
// Save to cache (fire-and-forget)
const threatIntelHash = computeThreatIntelHash(threatIntel?.entries ?? [])
const now = new Date().toISOString()
const externalResults: ExternalResults = {
  onChain: onChain ? { attestationCount: onChain.attestations.length, latestRiskScore: onChain.latestRiskScore, checkedAt: now } : undefined,
  dosMe: trustedMember ? { found: trustedMember.found, trustScore: trustedMember.member?.trustScore ?? null, isFlagged: trustedMember.member?.isFlagged ?? false, checkedAt: now } : undefined,
  webSearch: webAnalysis?.webResults ? { resultCount: webAnalysis.webResults.length, checkedAt: now } : undefined,
  llmAnalysis: webAnalysis?.llmAnalysis ? { checkedAt: now } : undefined,
}

saveCachedScore(
  entityType, entityId,
  { score: riskScore, riskLevel, confidence, signals, uniqueRiskSources: 0 },
  llmSummary,
  webAnalysis as Record<string, unknown> | null,
  externalResults,
  threatIntelHash,
).catch(() => {}) // fire-and-forget
```

**Step 4: Verify build**

Run: `cd apps/web && npm run build` (or `npx next build`)

**Step 5: Commit**

```bash
git add apps/web/src/app/api/entity-check/route.ts
git commit -m "feat(entity-check): integrate entity_scores cache layer"
```

---

### Task 4: Integrate Cache into `url-check` Route

**Files:**
- Modify: `apps/web/src/app/api/url-check/route.ts`

**Step 1: Add cache imports**

```typescript
import {
  getCachedScore, saveCachedScore, isScoreFresh,
  computeThreatIntelHash, type ExternalResults
} from '@/lib/entity-score-cache'
```

**Step 2: Extension fast path — cache-first lookup**

Right after URL normalization and before DB lookup, add early return for extension:

```typescript
// Extension ultra-fast path: check entity_scores cache first
if (isExtensionProtection) {
  const cached = await getCachedScore('url', normalized.normalized).catch(() => null)
    ?? await getCachedScore('domain', domain).catch(() => null)

  if (cached && isScoreFresh(cached)) {
    return NextResponse.json({
      url, domain,
      normalized: normalized.normalized,
      entityId: normalized.entityId,
      riskScore: cached.risk_score,
      riskLevel: cached.risk_level,
      riskSignals: cached.signals,
      confidence: cached.confidence,
      llmSummary: cached.llm_summary,
      cached: true,
      cachedAt: cached.computed_at,
      // Minimal response for extension — skip full checks/threatIntel/webAnalysis
      checks: null,
      onChain: null,
      threatIntel: null,
      webAnalysis: null,
      typosquatting: null,
    })
  }
  // If no cache, fall through to existing DB-only fast path
}
```

**Step 3: Save to cache after full-path assessment**

After `assessRisk()` and before the response, add (inside the `!fastPath` block):

```typescript
// Save computed score to cache (fire-and-forget, full path only)
if (!fastPath) {
  const threatIntelHash = computeThreatIntelHash(threatIntel?.entries ?? [])
  const now = new Date().toISOString()
  const cachedExternal: ExternalResults = {
    safeBrowsing: { threats: safeBrowsing.threats, checkedAt: now },
    whois: { ageDays: whois.ageDays, registrar: whois.registrar, checkedAt: now },
    onChain: onChain ? { attestationCount: onChain.attestations.length, latestRiskScore: onChain.latestRiskScore, checkedAt: now } : undefined,
    webSearch: webResults.length > 0 ? { resultCount: webResults.length, checkedAt: now } : undefined,
    llmAnalysis: webAnalysis?.llmAnalysis ? { checkedAt: now } : undefined,
  }

  saveCachedScore(
    'url', normalized.normalized,
    { score: riskScore, riskLevel: level, confidence, signals, uniqueRiskSources: 0 },
    webAnalysis?.llmAnalysis?.summary ?? null,
    webAnalysis as Record<string, unknown> | null,
    cachedExternal,
    threatIntelHash,
  ).catch(() => {})
}
```

**Step 4: Verify build**

Run: `cd apps/web && npm run build`

**Step 5: Commit**

```bash
git add apps/web/src/app/api/url-check/route.ts
git commit -m "feat(url-check): integrate entity_scores cache + extension ultra-fast path"
```

---

### Task 5: Integrate Cache into Partner API Routes

**Files:**
- Modify: `apps/web/src/app/api/check/route.ts` (single check)
- Modify: `apps/web/src/app/api/check/bulk/route.ts` (bulk check)

**Step 1: Add cache to single check route**

Same pattern as entity-check: cache lookup before `assessEntity()`, save after.

**Step 2: Add batch cache lookup to bulk route**

For bulk, query all 50 entity_scores in one batch, return cached hits, only run `assessEntity()` for misses.

```typescript
// Batch lookup cached scores
const cacheResults = await Promise.all(
  entities.map(e => getCachedScore(e.entityType, e.entityId).catch(() => null))
)

// Separate hits vs misses
const results = []
const misses = []
for (let i = 0; i < entities.length; i++) {
  const cached = cacheResults[i]
  if (cached && isScoreFresh(cached)) {
    results[i] = { /* build from cached */ cached: true }
  } else {
    misses.push({ index: i, ...entities[i] })
  }
}

// Only compute misses
const missResults = await Promise.all(
  misses.map(m => assessEntity(m.entityType, m.entityId, true).catch(() => null))
)
```

**Step 3: Commit**

```bash
git add apps/web/src/app/api/check/route.ts apps/web/src/app/api/check/bulk/route.ts
git commit -m "feat(partner-api): integrate entity_scores cache for check + bulk endpoints"
```

---

### Task 6: Deploy & Test

**Step 1: Push all commits**

```bash
git push origin main
```

**Step 2: Apply migration to Supabase**

Run migration via Supabase dashboard or `supabase db push`.

**Step 3: Test — first request (cache MISS, full compute)**

```bash
curl -X POST https://dosafe.io/api/entity-check \
  -H "Content-Type: application/json" \
  -d '{"entityType": "domain", "entityId": "example.com"}'
```

Verify: response has NO `cached: true` field. Check `entity_scores` table has a new row.

**Step 4: Test — second request (cache HIT)**

Same curl. Verify: response has `cached: true` and `cachedAt` timestamp. Response time < 100ms.

**Step 5: Test — extension fast path**

```bash
curl -X POST https://dosafe.io/api/url-check \
  -H "Content-Type: application/json" \
  -H "X-Client-Type: extension" \
  -d '{"url": "https://example.com"}'
```

Verify: if entity was previously checked via full path, returns cached score instantly.

**Step 6: Commit any fixes**

```bash
git add -A && git commit -m "fix: post-deploy adjustments for entity_scores cache"
```
