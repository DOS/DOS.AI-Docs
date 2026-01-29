# Billing Migration Roadmap: D1 → Supabase

## Overview

Migrate billing/credits data from Cloudflare D1 to Supabase to consolidate all business data in one database.

## Current State

```
┌─────────────────────────────────────────────────────────────┐
│                    CURRENT ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Supabase (ap-southeast-1)          Cloudflare D1 (Edge)    │
│  ┌─────────────────────┐            ┌─────────────────────┐ │
│  │ auth.users          │            │ api_keys            │ │
│  │ organizations       │            │ billing             │ │
│  │ org_members         │            │ credits             │ │
│  │ stripe_customers  ←─┼── DUPLICATE ─→ payment_transactions│
│  │ payment_transactions│            │ usage_transactions  │ │
│  │ profiles            │            │ usage_logs          │ │
│  └─────────────────────┘            │ model_pricing       │ │
│                                     │ rate_limit_log      │ │
│                                     └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Problems with Current State

1. **Data Duplication**: `stripe_customers` and `payment_transactions` exist in both DBs
2. **Inconsistency Risk**: Billing updates in D1, Stripe data in Supabase
3. **Operational Overhead**: Managing 2 databases
4. **Query Complexity**: Can't join billing with user/org data

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

### Phase 1: Schema Setup (Day 1)

**Create Supabase tables:**

```sql
-- Billing accounts (replaces D1 billing table)
CREATE TABLE billing_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  balance_cents INTEGER NOT NULL DEFAULT 0,
  total_deposited_cents INTEGER NOT NULL DEFAULT 0,
  total_spent_cents INTEGER NOT NULL DEFAULT 0,
  stripe_customer_id TEXT REFERENCES stripe_customers(stripe_customer_id),
  auto_recharge_enabled BOOLEAN DEFAULT FALSE,
  auto_recharge_threshold_cents INTEGER DEFAULT 500,
  auto_recharge_amount_cents INTEGER DEFAULT 1000,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id)
);

-- Usage transactions (replaces D1 usage_transactions)
CREATE TABLE usage_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id),
  api_key_id TEXT NOT NULL,
  model TEXT NOT NULL,
  input_tokens INTEGER NOT NULL,
  output_tokens INTEGER NOT NULL,
  input_cost_cents INTEGER NOT NULL,
  output_cost_cents INTEGER NOT NULL,
  total_cost_cents INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Model pricing
CREATE TABLE model_pricing (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  model_id TEXT UNIQUE NOT NULL,
  display_name TEXT,
  input_price_per_million NUMERIC(10,4) NOT NULL,
  output_price_per_million NUMERIC(10,4) NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_billing_user ON billing_accounts(user_id);
CREATE INDEX idx_usage_user_created ON usage_transactions(user_id, created_at DESC);
CREATE INDEX idx_usage_api_key ON usage_transactions(api_key_id);
```

**Deliverables:**
- [ ] Create migration file
- [ ] Apply to Supabase
- [ ] Seed model_pricing data

---

### Phase 2: Dual-Write Implementation (Day 2-3)

**Modify API Gateway to write to both D1 and Supabase:**

```typescript
// packages/api-gateway/src/worker.ts

async function deductBalance(
  d1: D1Database,
  supabase: SupabaseClient,
  userId: string,
  ...
) {
  // Write to D1 (existing)
  await d1.prepare(`UPDATE billing SET ...`).run();

  // Write to Supabase (new)
  await supabase
    .from('billing_accounts')
    .update({ balance_cents: newBalance })
    .eq('user_id', userId);

  // Log to both
  await Promise.all([
    d1.prepare(`INSERT INTO usage_transactions ...`).run(),
    supabase.from('usage_transactions').insert({ ... })
  ]);
}
```

**Deliverables:**
- [ ] Add Supabase client to Worker
- [ ] Implement dual-write for billing updates
- [ ] Implement dual-write for usage logging
- [ ] Add error handling (D1 is primary, Supabase is secondary)

---

### Phase 3: Data Migration (Day 4)

**Migrate existing data from D1 to Supabase:**

```typescript
// scripts/migrate-billing-to-supabase.ts

async function migrateBilling() {
  // 1. Get all billing records from D1
  const d1Billing = await d1.prepare('SELECT * FROM billing').all();

  // 2. Insert into Supabase
  for (const record of d1Billing.results) {
    await supabase.from('billing_accounts').upsert({
      user_id: record.user_id,
      balance_cents: record.balance_cents,
      total_deposited_cents: record.total_deposited_cents,
      total_spent_cents: record.total_spent_cents,
      stripe_customer_id: record.stripe_customer_id,
      // ...
    });
  }

  // 3. Migrate usage_transactions (last 30 days)
  const d1Usage = await d1.prepare(`
    SELECT * FROM usage_transactions
    WHERE created_at > datetime('now', '-30 days')
  `).all();

  // Batch insert to Supabase
  await supabase.from('usage_transactions').insert(d1Usage.results);
}
```

**Deliverables:**
- [ ] Create migration script
- [ ] Run migration in staging
- [ ] Verify data integrity
- [ ] Run migration in production

---

### Phase 4: Read Migration (Day 5-6)

**Switch reads from D1 to Supabase:**

```typescript
// Before: Read from D1
const billing = await d1.prepare(
  'SELECT balance_cents FROM billing WHERE user_id = ?'
).bind(userId).first();

// After: Read from Supabase
const { data: billing } = await supabase
  .from('billing_accounts')
  .select('balance_cents')
  .eq('user_id', userId)
  .single();
```

**Implementation Order:**
1. Dashboard reads (apps/app) - low risk
2. API Gateway balance check - medium risk
3. Usage reporting - low risk

**Deliverables:**
- [ ] Update dashboard to read from Supabase
- [ ] Update API Gateway balance check
- [ ] Add caching layer (optional)
- [ ] Monitor latency

---

### Phase 5: Remove D1 Writes (Day 7)

**Stop writing billing data to D1:**

```typescript
async function deductBalance(...) {
  // Only write to Supabase now
  const { error } = await supabase
    .from('billing_accounts')
    .update({
      balance_cents: sql`balance_cents - ${cost}`,
      total_spent_cents: sql`total_spent_cents + ${cost}`
    })
    .eq('user_id', userId);

  if (error) throw error;

  await supabase.from('usage_transactions').insert({ ... });
}
```

**Deliverables:**
- [ ] Remove D1 billing writes
- [ ] Keep D1 for api_keys and rate_limit only
- [ ] Monitor for errors
- [ ] Cleanup D1 billing tables (after 30 days)

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

| Phase | Duration | Risk Level |
|-------|----------|------------|
| Phase 1: Schema Setup | 1 day | Low |
| Phase 2: Dual-Write | 2 days | Medium |
| Phase 3: Data Migration | 1 day | Medium |
| Phase 4: Read Migration | 2 days | Medium |
| Phase 5: D1 Removal | 1 day | Low |
| **Total** | **7 days** | |

---

## Success Metrics

- [ ] Zero data loss during migration
- [ ] Latency increase < 100ms per request
- [ ] No billing errors for 7 days post-migration
- [ ] All Stripe payments correctly linked
- [ ] Usage reports match D1 data

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
