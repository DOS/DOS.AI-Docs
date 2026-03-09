# DOSafe Extension Architecture

Last updated: 2026-03-07

## 1) Scope

This document defines the architecture for the DOSafe Chrome Extension located at:

- `apps/extension`

Current goals:

- Read content from the active page (selected text or full visible page text).
- Run DOSafe text detection via `/api/detect`.
- Crawl structured page data and send it to a configurable ingestion server.

## 2) Current Implementation (MVP)

Extension type:

- Chrome Extension Manifest V3 popup-based extension.

Main files:

- `apps/extension/manifest.json`
- `apps/extension/popup.html`
- `apps/extension/popup.css`
- `apps/extension/popup.js`

Key permissions:

- `activeTab`
- `scripting`
- `storage`
- `host_permissions`: `https://*/*`, `http://*/*` (plus DOSafe and localhost explicit entries)

Current UI actions:

- `Scan`: Send crawled text to DOSafe detector endpoint.
- `Crawl to server`: Collect page data and POST JSON payload to user-defined crawl endpoint.
- `Save`: Persist settings in `chrome.storage.local`.

## 3) Data Flows

### 3.1 Scan flow (AI text detection)

1. User opens popup and clicks `Scan`.
2. Extension executes script in active tab:
   - `selectedText = window.getSelection()`
   - fallback: `document.body.innerText`
3. Truncate text by `maxChars` (500-5000).
4. POST to detector endpoint (default: `https://dosafe.io/api/detect`) with:
   - `{ text, lang }`
5. Render response fields:
   - `ai_probability`, `human_probability`, `verdict`, `confidence`, `signals`, `sentence_scores`

### 3.2 Crawl flow (page ingestion)

1. User sets `crawlEndpoint` and clicks `Crawl to server`.
2. Extension executes script in active tab and extracts:
   - URL metadata: `url`, `title`, `lang`, `capturedAt`
   - Content: `selectedText`, `text`, `fullTextLength`
   - SEO/content hints: `metaDescription`, top `h1` headings
   - Commerce hints: `priceCandidates` (regex-based candidate extraction)
3. POST to crawl endpoint with payload:

```json
{
  "source": "dosafe-extension",
  "page": {
    "url": "https://example.com/...",
    "title": "...",
    "lang": "vi",
    "selectedText": "...",
    "text": "...",
    "fullTextLength": 12345,
    "metaDescription": "...",
    "headings": ["..."],
    "priceCandidates": ["153.513₫"],
    "capturedAt": "2026-03-03T..."
  },
  "detector_hint": {
    "lang": "vi"
  }
}
```

4. Popup shows server response JSON.

## 4) Capability Mapping vs Browser-Agent Tooling

Some AI agents expose tools named `find`, `read_page`, `get_page_text`, `computer`, `javascript_tool`, etc.
The extension can implement equivalent capabilities as follows:

- `get_page_text` equivalent:
  - `document.body.innerText` (already implemented)
- `find` equivalent:
  - DOM scanning and keyword/regex matching (`querySelectorAll`, `textContent`, price regex)
- `read_page` equivalent:
  - Full DOM traversal (can be added with `TreeWalker`) and ARIA/role extraction
- `javascript_tool` equivalent:
  - Executing page-context JS via `chrome.scripting.executeScript`
- `computer` (screenshot) equivalent:
  - `chrome.tabs.captureVisibleTab` (not yet implemented)
- `read_network_requests` equivalent:
  - Requires debugger/webRequest strategy (not yet implemented)

Important distinction:

- Browser agents can run broad automation sessions.
- Product extension should keep deterministic, user-triggered, auditable behavior.

## 5) Security and Privacy Model

User-trigger model:

- Page reading happens when user explicitly clicks `Scan` or `Crawl to server`.

Stored data:

- Stored locally in `chrome.storage.local`:
  - detector endpoint, crawl endpoint, language, max chars

Sensitive data handling:

- The extension currently sends only extracted page data the user triggers.
- No background continuous crawling or passive page exfiltration.

Risk notes:

- `https://*/*` and `http://*/*` host permissions are broad for ingestion flexibility.
- For production hardening, consider allowlist-based domains and signed request auth.

## 6) Known Constraints

- Cannot access restricted Chrome pages (`chrome://`, extensions store internals, etc.).
- Cross-origin iframes may not be fully readable unless frame execution is explicitly handled.
- Dynamic/canvas-heavy pages may expose limited text through DOM extraction.
- Regex price extraction is heuristic, not schema-guaranteed.

## 7) Recommended Server Contract (Crawl Endpoint)

Endpoint:

- `POST /api/crawl` (example)

Required behaviors:

- Validate payload size and schema.
- Attach server-side timestamps and request metadata.
- Deduplicate by `(url, capturedAt window, content hash)`.
- Return JSON `{ ok: true, id: "...", receivedAt: "..." }`.

Recommended auth:

- `Authorization: Bearer <extension-token>` or HMAC signature.
- Rotate tokens and enforce rate limits per installation/device.

## 8) Available Backend API Endpoints

The DOSafe backend (`https://dosafe.io`) exposes these APIs that the extension should integrate:

### 8.1 `/api/detect` (AI + Scam Text Detection) — ALREADY INTEGRATED

```
POST /api/detect
Body: { "text": "...", "lang": "vi", "task": "ai_detection" | "scam_detection" }
Response: { ai_probability, human_probability, verdict, confidence, signals, sentence_scores, source_matches }
```

### 8.2 `/api/url-check` (URL/Domain Risk Assessment) — NOT YET INTEGRATED

Check any URL for phishing/malware/scam risks. Combines DB lookup + Google Safe Browsing + WHOIS + on-chain flags.

```
POST /api/url-check
Body: { "url": "https://suspicious-site.com" }
Response: {
  riskLevel: "safe" | "low" | "medium" | "high" | "critical",
  riskSignals: ["Domain registered 3 days ago", "Found in MetaMask blacklist"],
  checks: {
    trustedDomain: { isTrusted, domain },
    safeBrowsing: { isSafe, threats },
    whois: { domainAge, createdDate, registrar },
    onChain: { flags },
    threatIntel: { entries, maxRiskScore, sources, categories, cluster }
  }
}
```

**Extension integration idea:** Auto-check the current tab's URL in background when user navigates. Show badge icon (green/yellow/red) based on riskLevel.

### 8.3 `/api/entity-check` (Phone/Email/Wallet/Domain/Bank Account Check) — NOT YET INTEGRATED

Check individual entities across threat DB + on-chain.

```
POST /api/entity-check
Body: { "entityType": "phone" | "email" | "wallet" | "domain" | "bank_account" | ..., "entityId": "0912345678" }
Response: {
  entityType, entityId,
  riskLevel: "safe" | "low" | "medium" | "high" | "critical",
  riskSignals: [...],
  threatIntel: { entries, maxRiskScore, sources, categories, cluster },
  onChain: { flags }
}
```

**Supported entity types:** phone, email, wallet, url, domain, bank_account, national_id, facebook, telegram, organization

**Extension integration idea:** Detect phone numbers, bank accounts, wallet addresses in page content. Highlight or annotate them with risk indicators.

### 8.4 `/api/detect-image` (Image AI Detection) — NOT YET INTEGRATED

```
POST /api/detect-image
Body: FormData with image file
Response: { ai_probability, verdict, confidence, signals }
```

## 9) Roadmap

Phase 1 (current — COMPLETE):

- Text extraction + detector scan + server crawl POST from popup.
- Facebook content script: capture post text + author profile.
- Side panel UI with AI mode / Scam mode tabs.

Phase 2 (NEXT — real-time protection):

- **URL auto-check:** Call `/api/url-check` on navigation. Show risk badge on extension icon (green/yellow/red).
- **Entity detection:** Scan page content for phone numbers, bank accounts, wallet addresses. Check via `/api/entity-check`.
- **Inline annotations:** Highlight detected entities with risk indicators (tooltip with source count, risk level).
- Add `contextMenus` action: "Check this with DOSafe" for selected text/links.

Phase 3:

- Add structured field extraction profiles (e-commerce/article/forum).
- Add optional screenshot capture for evidence snapshots.
- Add network-aware mode (capture API responses used by page).
- Add frame-aware crawling and pagination helpers.
- Add retry queue/offline sync (background service worker + local queue).

## 9) Testing Checklist

Manual:

- Load unpacked extension from `apps/extension`.
- Scan selected text (`>= 50 chars`) on `https://dosafe.io` and a third-party page.
- Crawl to test ingestion endpoint and verify payload integrity.

Technical:

- `manifest.json` parses correctly.
- `popup.js` passes syntax check.
- CORS and endpoint auth validated on ingestion server.

## 10) Claude-Style Tactics and Tools (Captured Notes)

The following reflects the workflow style described by Claude-like browser agents and how to apply it in DOSafe extension development:

- `find` tactic:
  - First locate target elements semantically (price, title, seller, rating) instead of fixed selectors.
  - In extension implementation, emulate with keyword scoring over DOM text blocks and selector fallbacks.
- `read_page` tactic:
  - Read broad page structure first (headings, regions, forms, interactive elements), then extract targeted fields.
  - In extension implementation, add structured DOM traversal mode (TreeWalker + role/aria map).
- `get_page_text` tactic:
  - Extract full text body for broad understanding; then do targeted extraction.
  - Already partially implemented via `document.body.innerText`.
- `javascript_tool` tactic:
  - Execute in-page JS for high-fidelity extraction from dynamic apps (React/Vue state, globals, inline JSON).
  - In extension implementation, use `chrome.scripting.executeScript`.
- `computer` interaction tactic:
  - Use click/scroll/type only when DOM extraction is insufficient.
  - For extension product, keep interaction optional and user-triggered for predictability.
- `read_network_requests` tactic:
  - Observe XHR/fetch responses to get clean structured data directly from site APIs.
  - Recommended for Phase 3 with debugger/webRequest-backed mode.
- `tabs` tactic:
  - Run extraction on multiple tabs in parallel for throughput.
  - Extension roadmap: queue jobs per tab and bounded concurrency.
- `screenshot` tactic:
  - Capture evidence when text/DOM is incomplete or disputed.
  - Extension roadmap: `chrome.tabs.captureVisibleTab`.

Execution heuristics from Claude-style operation:

- Read first, act second: snapshot structure before interacting.
- Prefer stable references (DOM path/semantic ref) over pixel coordinates.
- Combine DOM signals + network signals for highest accuracy.
- Run tasks in parallel where independent (multiple tabs/pages).
- Keep extraction deterministic; use UI automation only as fallback.

## 11) Additional Research-Backed Techniques for DOSafe Crawler

These are practical techniques researched and recommended for robust production crawling via extension:

- Multi-layer extraction pipeline:
  - Layer 1: selected text.
  - Layer 2: visible body text.
  - Layer 3: structured fields (title, meta, headings, price candidates).
  - Layer 4: API/network payload capture (optional advanced mode).
- E-commerce specific parsing:
  - Normalize localized prices (`153.513₫`, `153,513 VND`, `$12.99`).
  - Extract currency, numeric value, and confidence per candidate.
- Content deduplication:
  - Hash normalized text and key fields before upload.
  - Skip duplicate sends within a short time window per URL.
- Quality scoring:
  - Attach `extraction_confidence` from field completeness + text length + source type.
- Anti-fragile parsing:
  - Use multiple selectors and regex routes; avoid single brittle selectors.
  - Gracefully degrade to text-only payload when structure fails.
- Privacy-safe default:
  - Redact obvious PII patterns before upload (email, phone, card-like numbers).
  - Add optional domain allowlist mode for enterprise deployments.
- Reliability:
  - Add retry with exponential backoff for crawl uploads.
  - Add local queue when offline, flush later from background worker.
- Observability:
  - Include `capture_id`, timing metrics, and parser version in payload.
  - Keep server-side audit trail for each crawl event.

Recommended next implementation slice:

- Add parser profiles: `generic`, `ecommerce`, `article`, `social`.
- Add payload schema versioning (`schema_version: 1`).
- Add authenticated ingestion (Bearer/HMAC) and replay protection.

## 12) Facebook Profile Crawl Limits and Browser-Agent Direction

Current verified behavior in pps/extension:

- content-facebook.js binds actions only to visible Facebook rticle nodes on the current page.
- Profile Check sends uthorName, profileUrl, and postText to the background worker.
- ackground.js opens the main profileUrl in a new tab and extracts only:
  - document.title
  - first h1
  - meta[name="description"]
  - current document.body.innerText
- It does not automatically click into About, Friends, Photos, or Posts tabs.
- It does not run a multi-step observe -> decide -> act loop.

Implication:

- Current profile scraping is a single-page capture, not a full browser agent.
- Data that is hidden behind profile tabs, lazy-loading, or interaction gates is not collected.
- This is acceptable for lightweight context capture, but not sufficient for robust scam-profile investigation.

Recommended browser-agent pattern for DOSafe:

- Use the model as planner only. In current stack, this means Qwen3.5 emits structured actions.
- Use the extension runtime as executor via:
  - chrome.tabs
  - chrome.scripting.executeScript
  - chrome.storage
  - 	abs.onUpdated / wait logic
- Keep a tight loop:
  1. Observe current page state
  2. Model returns next allowed action
  3. Runtime executes action
  4. Runtime returns compact observation
- Restrict actions by whitelist and block destructive actions by default.

Suggested next extension scope:

- Add Facebook profile deep crawl sequence:
  - main profile
  - About
  - Posts
  - Photos
- Add bounded step execution and timeout guards.
- Return structured observations with:
  - current URL
  - visible tabs
  - extracted text summary
  - entities found (phones, links, ank_accounts, domains)
