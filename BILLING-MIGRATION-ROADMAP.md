# Billing Migration Roadmap: D1 → Supabase

## Overview

Migrate billing/credits data from Cloudflare D1 to Supabase to consolidate all business data in one database.

## Current State (After Phase 5)

```
┌─────────────────────────────────────────────────────────────┐
│              CURRENT ARCHITECTURE (Jan 2026)                 │
│                  SUPABASE SINGLE SOURCE OF TRUTH             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Supabase (PostgreSQL)              Cloudflare D1 (Edge)    │
│  ┌─────────────────────┐            ┌─────────────────────┐ │
│  │ auth.users          │            │ api_keys       ←────┼─┼─ Edge validation
│  │ profiles            │            │ rate_limit_log ←────┼─┼─ Edge rate limit
│  │ organizations       │            │ model_pricing  ←────┼─┼─ Pricing cache
│  │ billing_accounts ←──┼── ALL ─────│                     │ │
│  │ usage_transactions  │  WRITES    │ (billing tables     │ │
│  │ credit_transactions │            │  now deprecated)    │ │
│  │ model_pricing       │            └─────────────────────┘ │
│  └─────────────────────┘                                    │
│          ↑                                                  │
│          │                                                  │
│    All billing reads/writes                                 │
│    (Dashboard + API Gateway)                                │
└─────────────────────────────────────────────────────────────┘

Write: Supabase only (single source of truth)
Read:  Supabase (Dashboard + API Gateway balance check)
D1:    api_keys, rate_limit_log, model_pricing only
```

### Current Architecture

1. **Supabase is single source of truth**: All billing data in Supabase
2. **D1 for edge-critical operations**: API key validation, rate limiting
3. **No more dual-write**: Simplified architecture, no sync issues
4. **Latency trade-off**: Balance check adds ~50-100ms (acceptable)

---

## Target State

```
┌─────────────────────────────────────────────────────────────┐
│                     TARGET ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Supabase (ap-southeast-1)          Cloudflare D1 (Edge)    │
│  ┌─────────────────────┐            ┌─────────────────────┐ │
│  │ auth.users          │            │ api_keys            │ │
│  │ organizations       │            │ rate_limit_log      │ │
│  │ org_members         │            └─────────────────────┘ │
│  │ profiles            │                    ↑               │
│  │ stripe_customers    │                    │               │
│  │ payment_transactions│              Edge-only data        │
│  │ billing_accounts ←──┼── NEW            (fast lookups)    │
│  │ usage_transactions  │                                    │
│  │ model_pricing       │                                    │
│  └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
```

### Benefits of Target State

1. **Single Source of Truth**: All business data in Supabase
2. **Better Queries**: Join billing with users, orgs, Stripe
3. **Simplified Ops**: One database to manage
4. **RLS Ready**: Row-level security if needed later

---

## Migration Phases

### Phase 1: Schema Setup (Day 1) ✅ COMPLETED

**Created Supabase tables** in `apps/app/supabase/migrations/20260129_billing_tables.sql`:

```sql
-- Billing accounts (replaces D1 billing table)
CREATE TABLE billing_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  balance_cents INTEGER NOT NULL DEFAULT 0,
  total_deposited_cents INTEGER NOT NULL DEFAULT 0,
  total_spent_cents INTEGER NOT NULL DEFAULT 0,
  lifetime_tokens_used BIGINT DEFAULT 0,
  stripe_customer_id TEXT,
  auto_recharge_enabled BOOLEAN DEFAULT FALSE,
  auto_recharge_threshold_cents INTEGER DEFAULT 500,
  auto_recharge_amount_cents INTEGER DEFAULT 1000,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id)
);

-- Usage transactions + Credit transactions + Model pricing
-- See full migration file for complete schema
```

**Deliverables:**
- [x] Create migration file
- [x] Apply to Supabase
- [x] Seed model_pricing data

---

### Phase 2: Dual-Write Implementation (Day 2-3) ✅ COMPLETED

**Modified API Gateway** in `packages/api-gateway/src/worker.ts`:

```typescript
// Custom Supabase REST client (no SDK needed in Workers)
class SupabaseClient {
  constructor(private url: string, private serviceKey: string) {}

  from(table: string) {
    return new SupabaseTable(this.url, this.serviceKey, table);
  }
}

async function deductBalance(
  db: D1Database,
  supabase: SupabaseClient | null,
  userId: string,
  apiKeyId: string,
  model: string,
  inputTokens: number,
  outputTokens: number
): Promise<void> {
  // D1 write (primary)
  await db.prepare(`UPDATE billing SET ...`).run();
  await db.prepare(`INSERT INTO usage_transactions ...`).run();

  // Supabase write (secondary, async, non-blocking)
  if (supabase) {
    try {
      await supabase.from('billing_accounts').eq('user_id', userId).update({...});
      await supabase.from('usage_transactions').insert({...});
    } catch (e) {
      console.error('Supabase dual-write failed:', e);
    }
  }
}
```

**Deliverables:**
- [x] Add Supabase client to Worker (custom REST client)
- [x] Implement dual-write for billing updates
- [x] Implement dual-write for usage logging
- [x] Add error handling (D1 is primary, Supabase is secondary)
- [x] Deploy worker with `SUPABASE_SERVICE_KEY` secret

---

### Phase 3: Data Migration (Day 4) ✅ COMPLETED

**Migrated existing data from D1 to Supabase** (Jan 29, 2026):

```sql
-- Exported from D1
SELECT * FROM billing;
-- Results:
-- joy@dos.ai: 0 cents (old Firebase UID → updated to Supabase UUID)
-- kaka007vn@gmail.com: 100000 cents ($1000)

-- Inserted into Supabase billing_accounts
INSERT INTO billing_accounts (user_id, balance_cents, total_deposited_cents, total_spent_cents)
VALUES
  ('48fc3631-ec8c-4e78-aa98-ec89c1c3624d', 0, 0, 0),           -- joy@dos.ai
  ('aa32cc23-c0ad-4167-aa9a-4833fb880ea8', 100000, 100000, 0); -- kaka007vn@gmail.com
```

**Note**: User ID mapping issue discovered and fixed:
- Old Firebase UIDs (e.g., `jqlu061LlNV7plecvutxs6ukgrr1`) → New Supabase UUIDs
- Updated `api_keys.user_id` in D1 to match Supabase UUIDs

**Deliverables:**
- [x] Create migration script (manual SQL)
- [x] Run migration in production
- [x] Verify data integrity
- [x] Fix user ID mapping (Firebase → Supabase)

---

### Phase 4: Read Migration (Day 5-6) ✅ COMPLETED

**Dashboard reads switched to Supabase:**

```typescript
// apps/app/src/app/api/billing/route.ts
const supabase = createClient(supabaseUrl, supabaseServiceKey);
const { data: billing } = await supabase
  .from('billing_accounts')
  .select('*')
  .eq('user_id', user.uid)
  .single();
```

**Implementation:**
1. ✅ Dashboard billing page reads from Supabase
2. ✅ Dashboard transactions page reads from Supabase
3. ⚠️ API Gateway balance check: **Intentionally kept on D1** for edge performance
   - D1 co-located with Worker = sub-ms latency
   - Supabase network call would add 50-100ms
   - Dual-write ensures both DBs stay in sync

**Deliverables:**
- [x] Update `/api/billing` to read from Supabase
- [x] Update `/api/billing/transactions` to read from Supabase
- [x] ~~Keep API Gateway balance check on D1~~ → Moved to Supabase in Phase 5
- [x] No data sync issues (single source of truth now)

---

### Phase 5: Remove D1 Writes (Day 7) ✅ COMPLETED

**Removed D1 billing writes, Supabase is now single source of truth:**

```typescript
// packages/api-gateway/src/worker.ts

// Balance check now reads from Supabase
const billingTable = await supabase.from('billing_accounts');
const { data: billingData } = await billingTable.eq('user_id', userId).select('balance_cents');
balanceCents = billingData?.[0]?.balance_cents || 0;

// deductBalance() now writes only to Supabase
async function deductBalance(...) {
  // Supabase is PRIMARY - single source of truth
  const billingTable = await supabase.from('billing_accounts');
  await billingTable.eq('user_id', userId).update({
    balance_cents: currentBalance - cost,
    total_spent_cents: currentSpent + cost,
  });
  await supabase.from('usage_transactions').insert({ ... });
}

// /billing/add-credits now writes only to Supabase
// /billing/payment-webhook now writes only to Supabase
```

**Deliverables:**
- [x] Remove D1 billing writes from `deductBalance()`
- [x] Remove D1 billing writes from `/billing/add-credits`
- [x] Remove D1 billing writes from `/billing/payment-webhook`
- [x] Update balance check to read from Supabase
- [x] Keep D1 for `api_keys`, `rate_limit_log`, and `model_pricing` only
- [ ] Cleanup D1 billing tables (after 30 days - optional)

---

## Architecture After Migration

### API Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│                        API Request                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ CF Worker (api.dos.ai)                                       │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 1. Validate API key (D1 - edge fast)                    │ │
│ │ 2. Check rate limit (D1 - edge fast)                    │ │
│ │ 3. Check balance (Supabase - cached or async)           │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ inference.dos.ai (Cloudflare Tunnel → Local vLLM)           │
│ Response includes: { usage: { prompt_tokens, ... } }        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ CF Worker (async, non-blocking)                              │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 4. Deduct balance (Supabase)                            │ │
│ │ 5. Log usage (Supabase)                                 │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Balance Check Optimization

Since inference takes 500-2000ms, we can optimize balance check:

**Option A: Check on every request (simple)**
```typescript
// ~50-100ms latency to Supabase
const { data } = await supabase
  .from('billing_accounts')
  .select('balance_cents')
  .eq('user_id', userId)
  .single();
```

**Option B: Cache in KV (faster)**
```typescript
// Check KV first (~5ms)
let balance = await env.KV.get(`balance:${userId}`);
if (!balance) {
  // Cache miss: fetch from Supabase
  const { data } = await supabase
    .from('billing_accounts')
    .select('balance_cents')
    .eq('user_id', userId)
    .single();
  balance = data.balance_cents;
  // Cache for 5 minutes
  await env.KV.put(`balance:${userId}`, balance, { expirationTtl: 300 });
}
```

**Option C: Include in JWT (no extra call)**
```typescript
// Balance embedded in API key JWT
// Refresh when balance changes significantly
const decoded = verifyApiKeyJWT(apiKey);
if (decoded.balance_cents <= 0) {
  return error(402, 'Insufficient balance');
}
```

---

## Rollback Plan

If migration fails:

1. **Phase 2 (Dual-Write)**: Remove Supabase writes, continue D1 only
2. **Phase 4 (Read Migration)**: Revert to D1 reads
3. **Phase 5 (D1 Removal)**: Re-enable D1 writes from backup

**Backup Strategy:**
- Export D1 billing data daily during migration
- Keep D1 tables for 30 days after migration
- Monitor Supabase error rates

---

## Timeline

| Phase | Duration | Risk Level | Status |
|-------|----------|------------|--------|
| Phase 1: Schema Setup | 1 day | Low | ✅ Done (Jan 29) |
| Phase 2: Dual-Write | 2 days | Medium | ✅ Done (Jan 29) |
| Phase 3: Data Migration | 1 day | Medium | ✅ Done (Jan 29) |
| Phase 4: Read Migration | 1 day | Low | ✅ Done (Jan 29) |
| Phase 5: D1 Removal | 1 day | Medium | ✅ Done (Jan 29) |

---

## Success Metrics

- [x] Zero data loss during migration (verified: 2 users migrated)
- [x] No latency increase on API requests (D1 kept for edge checks)
- [ ] No billing errors for 7 days post-migration (monitoring)
- [ ] All Stripe payments correctly linked (pending)
- [x] Dashboard reads from Supabase (single source of truth)

---

## Dependencies

- Supabase project: `gulptwduchsjcsbndmua` (DOS)
- Cloudflare D1: `dos-api` (9b92faaf-97f0-420d-9269-1dc4d1f45df0)
- API Gateway Worker: `dos-api-gateway`
- Dashboard: `apps/app`

---

## References

- [Supabase JavaScript Client](https://supabase.com/docs/reference/javascript)
- [Cloudflare Workers + Supabase](https://supabase.com/docs/guides/functions/quickstart)
- [D1 Export/Import](https://developers.cloudflare.com/d1/reference/migrations/)
