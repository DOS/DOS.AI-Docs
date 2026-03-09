# R&D Findings - AI Safety, ENS Agent Identity, and Phone Spam Data

Date: 2026-03-06
Audience: DOS.AI / DOSafe team (Claude reference)

## 1) ENS + AI Agent (ERC-8004) relevance to DOS.AI/DOSafe

### What was checked
- ENS blog post on AI agents and ERC-8004
- EIP-8004 status
- ENSIP-25 follow-up
- Current DOS-AI repo signals for ENS/ERC-8004 integration

### Finding
- Relevance is strategic, not implemented yet.
- ENS positioning: human-readable identity + trust rails for AI agents.
- ERC-8004 is still Draft (not final standard).
- ENSIP-25 introduces a verification pattern for binding ENS names to agent registries.
- In current DOS-AI codebase, no direct ENS/ERC-8004 integration was found.

### Practical implication
- If DOS.AI roadmap includes autonomous agents, marketplace agents, or portable onchain trust/reputation:
  - ENS + registry linkage is relevant.
- For current DOSafe anti-scam/browser protection scope:
  - No immediate dependency.

Sources:
- https://ens.domains/blog/post/ens-ai-agent-erc8004
- https://eips.ethereum.org/EIPS/eip-8004
- https://ens.domains/blog/post/ensip-25

## 2) Phone spam/scam data sources (global)

### Core conclusion
- "Truecaller/Google-grade" phone reputation data is mostly proprietary.
- Public open data exists but is fragmented and often country-specific.

### High-value sources
1. FTC Do Not Call Reported Calls Data (US, public CSV/API)
- https://www.ftc.gov/policy-notices/open-government/data-sets/do-not-call-data
- https://www.ftc.gov/developer/api/v0/endpoints/do-not-call-dnc-reported-calls-data-api

2. FCC Consumer Complaints Data (US, public open data catalog)
- https://catalog.data.gov/dataset/cgb-consumer-complaints-data

3. Australia DNCR industry wash-list model (not open dump)
- https://www.donotcall.gov.au/industry/subscription-overview
- https://www.donotcall.gov.au/industry/washing-process-overview/washing-steps/

4. Canada National DNCL access model (regulated, subscription/list-format based)
- https://web.crtc.gc.ca/eng/phone/telemarketing/format.htm

5. Croatia "Do Not Call" e-registry (query/registry model, no public bulk dump observed)
- https://www.hakom.hr/en/e-registry-do-not-call/224

6. UK nuisance-calls open statistics dataset (useful trend signal, not raw number blacklist)
- https://www.data.gov.uk/dataset/16dfac72-414e-4be9-98b7-585eabf7480d/consumer-concerns-tracking-survey

## 3) Vietnam anti-spam official portal (nospam.vncert.vn)

### What was verified
- Public site behavior and network calls.

### Finding
- No public bulk export endpoint observed.
- Exposed workflow endpoints are transactional:
  - `POST /search-dnc` (single-number DNC status lookup)
  - `POST /register-dnc`, `POST /unregister-dnc`
  - `POST /confirm-register-dnc`, `POST /confirm-unregister-dnc` (OTP flow)
  - `POST /complain-dnc`, `POST /confirm-complain-dnc`
  - `GET /complain-types`

### Practical implication
- Treat this as a validation/reporting channel, not a bulk data feed.
- Do not rely on brute-force enumeration for production data strategy.

Source:
- https://nospam.vncert.vn/

## 4) Substack article: "AI safety has 12 months left"

Article:
- https://mhdempsey.substack.com/p/ai-safety-has-12-months-left

### Finding
- The article argues that autonomous agent capability is scaling fast enough that safety governance, evals, and misuse controls are lagging.
- For DOS.AI, this is relevant as a strategic risk signal, not as direct product requirement text.

### Relevance to DOS.AI roadmap
- If DOS.AI expands toward higher-autonomy workflows (multi-step agents, external tool use), this increases:
  - misuse surface,
  - policy/compliance burden,
  - need for runtime safeguards and eval gates.

### Recommended actions for DOS.AI (near-term)
1. Define safety levels by feature class
- Chat completion, tool-calling, autonomous execution should have different guardrails.

2. Add eval gates before feature release
- Prompt-injection tests, harmful-task refusal tests, abuse simulation benchmarks.

3. Strengthen abuse monitoring
- Per-key risk scoring, anomaly detection, automated throttling for suspicious patterns.

4. Add user/report feedback loop
- Especially for DOSafe threat signals and phone reputation false-positive handling.

5. Document incident response
- Detection -> triage -> mitigation -> communication -> postmortem.

## 6) Mobile Platform Research — Flutter vs Native (2026-03-07)

### Context
DOSafe Mobile targets VN-first with call protection (TrueCaller-like) + AI detection features.

### VN Market Share (2025 data)
- Android: ~66% | iOS: ~34%
- iOS share is significant (urban iPhone adoption) — cannot ignore

### Critical finding: Real-time audio access blocked on Android 10+
Google blocked third-party audio access during calls since Android 10. No workaround via standard APIs.
- `CallScreeningService`: can screen calls (allow/block) + get metadata — **NO audio**
- `InCallService`: manages call UI — **NO audio access**
- Implication: **AI Voice Detection cannot run real-time during a live call**

Revised AI voice detection approach:
- During call: metadata-based warning (unknown number + pattern signals)
- Post-call: user optionally submits recording → server-side AI analysis (DOSafe inference)
- Future: telco B2B partnership (network-level audio access)

### Flutter vs Native verdict: **Flutter**

| Criterion | Flutter | Native Android | Native iOS |
|---|---|---|---|
| Dev effort | 1 codebase | Android only | iOS only |
| CallScreeningService | Plugin exists (`call_screen_service`) | Cleanest | N/A — iOS has CallKit (block list only) |
| Real-time audio | ❌ Android 10+ blocked | ❌ Same limitation | ❌ Fully blocked |
| iOS CallKit | Via platform channel | N/A | Full access |
| Performance overhead | ~5% vs native | Baseline | Baseline |
| Market coverage | 66% + 34% = 100% | 66% | 34% |

Decision rationale:
- iOS 34% is too large to skip
- OS-level limitations (audio block, iOS CallKit restrictions) affect native equally — Flutter adds nothing worse
- Platform channels handle call screening adequately
- Single codebase = faster iteration for early-stage product

### Android-first for call screening features
iOS CallKit CallDirectory = static block list only, no dynamic screening. Android-first for call screening depth; iOS gets caller ID overlay via CallKit.

## 5) VN phone reputation DB plan (crawl + scoring)

### Phase 1 - source onboarding
- Official advisories
- Major news outlets
- Community report sources (lower trust weight)

### Phase 2 - extraction + normalization
- Extract VN phone patterns from text.
- Normalize to E.164 (+84...).
- Keep raw evidence, URL, timestamp.

### Phase 3 - scoring
- Weighted score by source trust, recency, frequency, and cross-source confirmation.
- Hard-block only at high confidence threshold.

### Phase 4 - product integration
- Reputation API response with `risk_score`, `reason_codes`, and evidence links.
- Add appeal/unlist process to reduce false-positive harm.
