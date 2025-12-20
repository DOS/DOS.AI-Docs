# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

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
