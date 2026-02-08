# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### 2026-02-07 - Schema Reorganization, Models & User Settings

#### Added
- **`dosai.models` table**: Merged `model_pricing` into full model info table with provider, description, context_length, capabilities (JSONB), plus pricing columns
- **`dosai.user_settings` table**: Per-user DOS.AI preferences (default_model, billing_alerts JSONB, api_preferences JSONB) with RLS policies

#### Changed
- **Supabase schema reorganization**: Moved DOS.AI-specific tables to `dosai` schema
  - `usage_transactions` → `dosai.usage_transactions`
  - `model_pricing` → `dosai.models` (merged with full model info)
  - Shared tables (`billing_accounts`, `credit_transactions`) remain in `public`
- **PostgREST config**: Exposed `dosai` schema (`ALTER ROLE authenticator SET pgrst.db_schemas`)
- **Worker Supabase client**: Added schema support via `Content-Profile`/`Accept-Profile` headers
- **Updated docs**: `INFERENCE-API-ARCHITECTURE.md` rewritten to reflect current state

#### Fixed
- `credit_transactions.amount_cents` changed from INTEGER to NUMERIC(20,4)

#### dosai Schema (Final State)
| Table | Description |
|-------|-------------|
| `dosai.models` | Model info + pricing (7 models) |
| `dosai.usage_transactions` | Per-request usage logs |
| `dosai.user_settings` | User preferences |

#### Files Modified
- `packages/api-gateway/src/worker.ts` - Schema-aware Supabase REST client
- `docs/INFERENCE-API-ARCHITECTURE.md` - Full rewrite
- `CLAUDE.md` - Added Supabase Schema Layout section

---

### 2026-01-31 - UI Improvements & Settings Reorganization

#### Added
- **Theme and Language settings** moved to Settings page (`/settings`)
  - New `ThemeSelect` component
  - New `LanguageSelect` component in Settings form
- **Organization logo upload** on Settings page

#### Changed
- **Account dropdown menu** simplified: removed theme/language selectors, kept Account, Privacy, Feedback, Logout

#### Fixed
- **Favicon margins**: Deleted `icon.png` so Next.js uses `icon.svg` (full SVG without margins)
- **Select dropdown dark mode flash**: Changed from `bg-transparent` to explicit `bg-white dark:bg-zinc-800`

#### Commits
- `8ef990d` - fix: Remove icon.png to use full SVG favicon without margins
- `8c8d8cf` - feat: Move theme and language settings to Settings page
- `d1e3b7e` - fix: Select dropdown dark mode flash

---

### 2026-01-30 - Session Persistence & Billing Precision

#### Fixed
- **Session logout after ~1 hour**: Supabase access_token expires in ~1h but cookie TTL was 14 days
  - Implemented automatic token refresh in `/api/auth/verify-session` using `refresh_token`
  - Removed `expires_at` check from `auth-server.ts`
  - Cookie updated with refreshed tokens when needed
- **Billing 100x overcharge**: `Math.ceil` with `*100/100` rounded 0.00009 up to 0.01 cents per component
  - Removed `Math.ceil`, using exact calculation with 4 decimal precision
  - Removed minimum 1 cent charge
- **Differentiated input/output pricing**: Output tokens now priced higher than input (industry standard)

#### Model Pricing Update (D1)
| Model | Input ¢/1M | Output ¢/1M |
|-------|-----------|------------|
| Llama 3.1 8B | 10 | 20 |
| Llama 3.1 70B | 50 | 150 |
| Llama 3.3 70B | 60 | 180 |
| Qwen 2.5 72B | 50 | 150 |
| DeepSeek V3 | 27 | 110 |
| Qwen3 VL 30B | 10 | 80 |

#### Commits
- `a32a30d` - fix: Token refresh and billing precision fixes
- `6bf6531` - docs: Add MCP Playwright testing reminder to CLAUDE.md

---

### 2026-01-29 - Billing Migration to Supabase

#### Added
- **Supabase billing tables**: `billing_accounts`, `usage_transactions`, `credit_transactions`, `model_pricing`
- **Dual-write**: Worker writes billing data to both D1 and Supabase
- **Migration complete**: Supabase is now single source of truth for billing

#### Commits
- `f567699` - feat: Add billing dual-write to Supabase

---

### 2026-01-16 - Supabase Auth Migration & Account Features

#### Added
- **Avatar upload** with Supabase Storage (Assets bucket)
- **Organization update API** (`PATCH /api/orgs/:id`)
- **Account page** with profile editing and identity mapping

#### Changed
- **Auth migrated from Firebase to Supabase**: password, logout, identity APIs
- **Organization creation** now uses Supabase Auth
- **Server-side RLS** with JWT Authorization header

#### Fixed
- OAuth callback `/login` flash: use server-side redirect
- RLS permission issues: multiple iterations to get correct auth pattern
- DOS-Me API sync: removed incorrect `provider` field from `checkLogin`

#### Commits
- `9221f91` - feat: Add avatar upload and fix profile updates
- `56ddfed` - fix: Complete account page functionality and identity mapping
- `cec717d` - fix: Complete Firebase → Supabase migration for auth flows
- `ef854bd` - feat: Add organization update API endpoint
- `a08a10a` - fix: Migrate auth APIs to Supabase (password, logout, identity)
- `e7af5a9` - fix: Complete organization creation with Supabase Auth
- `40f608d` - fix: Use server-side redirect in OAuth callback to avoid /login flash

---

### 2024-12-20 - Server-Side OAuth Implementation

#### Added
- **Server-side OAuth flow** replacing Firebase client SDK
  - New API routes: `/api/auth/google/start`, `/api/auth/google/callback`
  - New session management routes: `/api/auth/verify-session`, `/api/auth/logout`
  - httpOnly session cookies for enhanced security
  - Performance measurement logging

#### Changed
- **3x faster session verification** (~600ms vs ~1700ms)
- Session cookies now httpOnly, preventing XSS attacks
- OAuth flow no longer depends on Firebase client SDK in production
- Localhost still uses Firebase client SDK for simplicity

#### Fixed
- **Critical**: httpOnly cookies cannot be read by JavaScript
  - Client now sends `credentials: 'include'` for automatic cookie handling
  - Server reads cookies from request headers instead of client sending in body
- **Critical**: Browsers don't store cookies from 307 redirect responses
  - Changed to 200 OK with intermediate HTML page
  - Client-side redirect after 100ms delay ensures cookie is stored
- **Critical**: Turbo.json missing server-side environment variables
  - Added `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `FIREBASE_SERVICE_ACCOUNT`, etc.
  - Without this, Vercel builds couldn't access env vars at runtime

#### Performance Metrics
| Metric | Before (Client SDK) | After (Server-side) | Improvement |
|--------|-------------------|-------------------|-------------|
| Session verification | ~1.7s (`getRedirectResult`) | ~0.6s (cookie verify) | **3x faster** |
| Total OAuth processing | N/A | ~2.3s (callback → dashboard) | Measured |
| Security | localStorage vulnerable | httpOnly cookies | **Much more secure** |

#### Technical Implementation

**OAuth Flow**:
```
User clicks Google
  → /api/auth/google/start
  → Google OAuth UI (user interaction, not measured)
  → /api/auth/google/callback
    - Exchange code for tokens (~800ms)
    - Create Firebase session cookie (~200ms)
    - Return intermediate HTML page
  → Client-side redirect (100ms delay)
  → Dashboard loads
  → /api/auth/verify-session (~600ms)
  → User authenticated ✓
```

**Security**:
- httpOnly cookies (XSS-proof)
- Secure flag (HTTPS-only)
- SameSite=lax (CSRF protection + OAuth compatibility)
- Domain=.dos.ai (cross-subdomain SSO)
- 5-day expiration

#### Files Modified
- `apps/app/src/app/api/auth/google/start/route.ts` - NEW: OAuth initiation
- `apps/app/src/app/api/auth/google/callback/route.ts` - NEW: OAuth callback handler
- `apps/app/src/app/api/auth/verify-session/route.ts` - NEW: Session verification
- `apps/app/src/app/api/auth/logout/route.ts` - NEW: Logout with session revocation
- `apps/app/src/components/auth/AuthContext.tsx` - Server-side OAuth integration
- `apps/app/src/lib/session.ts` - httpOnly cookie handling
- `turbo.json` - Added server-side env vars to build config

#### Documentation
- Created `docs/AUTHENTICATION-ARCHITECTURE.md` - Complete technical documentation

#### Commits
- `034d95b` - perf(auth): Measure OAuth processing time excluding Google UI
- `d956c8c` - perf(auth): Fix OAuth timing measurement using sessionStorage
- `1fdf8d2` - perf(auth): Add performance logging to measure OAuth flow time
- `05a557d` - fix(auth): Fix httpOnly cookie handling in session verification
- `370265a` - fix(auth): Use intermediate page instead of 307 redirect for OAuth callback
- `a0e4d14` - fix(turbo): Add server-side env vars to turbo.json
- `fea495f` - feat(auth): Implement server-side OAuth with NextJS API routes

#### References
- [Next.js Cookie Issue #48434](https://github.com/vercel/next.js/discussions/48434)
- [Better Auth Cookie Issue #2962](https://github.com/better-auth/better-auth/issues/2962)
- [Chromium Cookie Bug #696204](https://bugs.chromium.org/p/chromium/issues/detail?id=696204)
- [Firebase Session Cookies Docs](https://firebase.google.com/docs/auth/admin/manage-cookies)

---

### 2024-12-20 - OAuth Login Flow Optimization

#### Changed
- **Optimized OAuth redirect flow** to reduce perceived loading time
  - Before: User waits 4-5s on `/login` page after OAuth (sessionLogin API call)
  - After: User redirects to dashboard immediately, sees dashboard loading instead

#### Technical Details
- Split session creation into two phases:
  1. `/login`: Get ID token (~1.7s) → redirect to dashboard
  2. Dashboard: Create session cookie in background (4-5s)
- Added performance logging to track OAuth timing:
  - `getRedirectResult()`: ~1700ms (Firebase SDK overhead)
  - `getIdToken()`: ~0ms
- Store ID token in `sessionStorage` for dashboard to process
- Dashboard creates session cookie while showing loading UI

#### Performance Metrics
| Metric | Before | After |
|--------|--------|-------|
| Time on `/login` after OAuth | 4-5s | 1.7s |
| User perception | Stuck on login page | Dashboard loading |
| Total time to dashboard | ~5s | ~6s (but better UX) |

#### Files Modified
- `apps/app/src/components/auth/AuthContext.tsx`
  - Added timing instrumentation
  - Split session creation logic
  - Added `pending_id_token` sessionStorage flag

#### Known Limitations
- `getRedirectResult()` 1.7s overhead cannot be optimized with client-side Firebase SDK
- To eliminate this delay entirely, would need server-side OAuth callback implementation

#### Commits
- `58549b4` - debug: Save timing data to sessionStorage
- `003afb0` - debug: Add performance logging to OAuth redirect flow
- `2b9cbda` - perf(auth): Redirect to dashboard immediately, create session in background
- `dc828de` - revert: Remove /oauth-start page, use simple redirect flow
- `fcb63a5` - fix(auth): Redirect to login on OAuth failure or cancellation
- `c5bad0c` - fix(auth): OAuth redirect to dashboard directly instead of login page
- `406f2ba` - fix(auth): Use sessionStorage flag to track OAuth redirect
- `76e89c6` - fix(auth): Prevent redirect loop by only calling getRedirectResult on login page
- `89a2ae5` - perf(auth): Use lightweight /oauth-start page instead of dashboard

---

## Previous Changes

(To be documented from git history)
