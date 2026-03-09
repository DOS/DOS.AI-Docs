# DOSafe Mobile — Architecture

**Created:** 2026-03-07
**Status:** Planning. Not yet implemented.
**Market:** Vietnam-first (Android 66%, iOS 34%)
**Monetization:** Freemium + ads. Premium tier for AI features.

---

## What is DOSafe Mobile?

A mobile safety app for the Vietnamese market. Core use case: protect users from scam/spam calls, phishing SMS, and AI-generated deception (voice cloning, deepfake images, fake messages). Backed by DOSafe threat intelligence (1.2M+ entries) and AI inference pipeline.

Differentiates from TrueCaller by:
- AI voice analysis (post-call deepfake detection)
- Deepfake image detection in messages
- Crypto/wallet scam lookup
- Zalo message scan (VN-specific)
- DOSafe threat_intel depth (1.2M+ entries, 63k+ scammer clusters)

---

## Platform Decision: Flutter

**Single codebase for Android + iOS.**

Android 10+ blocks real-time audio access for third-party apps (`CallScreeningService` = metadata only, no audio stream). iOS CallKit = static block list only. These limitations are OS-level — native does not unlock anything Flutter cannot. Flutter with platform channels is sufficient for all call screening needs.

```
Flutter (Dart)
├── Android: platform channel → CallScreeningService, CallKit-equivalent overlay
└── iOS: platform channel → CallKit CallDirectory (block list) + notification overlay
```

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DOSafe Mobile (Flutter)                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Call Screen │  │  SMS / Chat  │  │  Manual Scanner  │   │
│  │  (Android)   │  │  Monitor     │  │  (link, QR,      │   │
│  │  CallScreen  │  │  Notification│  │   image, text)   │   │
│  │  ingService  │  │  listener    │  │                  │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
│         └─────────────────┴──────────────────┘              │
│                           ▼                                  │
│                   DOSafe Mobile SDK                          │
│              (reputation lookup, AI submit)                  │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS
        ┌───────────────────┼──────────────────┐
        ▼                   ▼                  ▼
┌──────────────┐   ┌────────────────┐  ┌──────────────────┐
│ DOSafe API   │   │ DOSafe         │  │ DOSafe Inference │
│ (Supabase)   │   │ Threat Intel   │  │ (vLLM on Pro     │
│ phone rep,   │   │ 1.2M+ entries  │  │  6000)           │
│ clusters,    │   │ 63k+ clusters  │  │ - AI voice anal. │
│ user reports │   │ phone, url,    │  │ - Deepfake image │
│              │   │ wallet, domain │  │ - Scam text anal │
└──────────────┘   └────────────────┘  └──────────────────┘
```

---

## Feature Clusters & Roadmap

### Phase 1 — Android MVP (launch)

**Goal:** Get to market fast with core call protection.

| Feature | Implementation |
|---|---|
| Caller ID + Spam Score | Phone lookup → DOSafe threat_intel API |
| Auto call blocking | Android `CallScreeningService` — block known scam numbers |
| Incoming call overlay | Android notification overlay showing risk score + label |
| SMS link scanner | Intercept SMS notifications, check URLs vs threat_intel |
| 1-tap report | Submit phone number → DOSafe DB (with category: scam/telemarketing/etc.) |
| nospam.vncert.vn integration | POST /complain-dnc for official VN DNC reporting |
| Scam feed | Push notifications for new scam patterns (from DOSafe pipeline) |

**iOS Phase 1:** Caller ID display via CallKit CallDirectory (static block list sync). No dynamic screening (iOS limitation).

---

### Phase 2 — AI Detection Features (differentiation)

**Goal:** Launch features TrueCaller cannot match.

| Feature | Implementation | Notes |
|---|---|---|
| AI Voice Analysis | Post-call: user submits recording → DOSafe inference (vLLM audio model) | Real-time blocked on Android 10+. Post-call UX: "Nghi ngờ cuộc gọi vừa rồi? Phân tích ngay." |
| Zalo message scan | User shares message screenshot or text → AI scam analysis | Zalo doesn't expose API; user-initiated paste/share |
| Link safety check | In-app URL scanner + QR code camera scan | Check vs DOSafe URL threat_intel (phishing.database, scamsniffer, etc.) |
| Screenshot analysis | User shares screenshot → AI detects scam patterns in text | OCR + scam classifier via DOSafe inference |
| Scam pattern explainer | After detection: "Đây là dạng lừa đảo X, cách nhận biết: ..." | LLM explanation via DOS-AI |

---

### Phase 3 — Premium & B2B

| Feature | Tier | Implementation |
|---|---|---|
| Deepfake image detection | Premium | Share image → DOSafe /api/detect-image (C2PA + EXIF + Binoculars) |
| AI-generated text detection | Premium | Paste text → Binoculars pipeline (perplexity + observer model) |
| Wallet/crypto address lookup | Premium | Check vs DOSafe crypto threat clusters |
| Identity cross-reference | Premium | Phone ↔ social profile ↔ threat_intel correlation |
| Full scam report + evidence | Premium | Detailed cluster report with evidence links |
| DOSafe Reputation API | B2B | Banks, telcos, fintech — pay-per-lookup phone reputation |
| Telco partnership | B2B | Network-level audio → real-time AI voice detection |

---

## Monetization

```
FREE (with ads)
├── Caller ID + spam score (unlimited lookups)
├── Auto call blocking (community-reported numbers)
├── SMS link scanner (basic)
├── 1-tap report
└── Scam feed (daily digest)

PREMIUM (~29,000–49,000 VND/month)
├── No ads
├── AI voice analysis (post-call)
├── Zalo / chat message scan
├── Deepfake image detection
├── AI text detection
├── Wallet address check
├── Full evidence reports
└── Priority threat alerts

B2B API
└── Reputation API for banks, telcos, fintech apps
```

---

## AI Voice Detection — Architecture Detail

Real-time audio access is blocked on Android 10+ by Google policy. DOSafe Mobile uses a post-call flow:

```
Call ends
    ↓
Notification: "Cuộc gọi từ 0xxx xxx xxx có vẻ đáng ngờ. Phân tích AI?"
    ↓ (user taps "Phân tích")
Prompt user to submit recording (manual record during call, or system recording if available)
    ↓
Upload audio chunk → DOSafe inference API
    ↓
AI Voice Analysis:
  - Spectral analysis: detect TTS artifacts, pitch uniformity
  - Temporal patterns: unnatural pauses, rhythm
  - Model: fine-tuned audio classifier (server-side)
    ↓
Result: "Giọng thật" / "Có dấu hiệu giọng AI (xx%)" + explanation
```

**Future (B2B):** Partner with telcos (Viettel, Vinaphone, Mobifone) who have network-level audio → real-time detection without Android restrictions.

---

## Phone Reputation Pipeline

Backed by DOSafe threat_intel DB and VN phone reputation plan:

```
Sources:
├── DOSafe community reports (in-app 1-tap)
├── nospam.vncert.vn (official VN DNC)
├── Scraped sources: checkscam.vn, scam.vn, admin.vn
├── FTC Do Not Call data (international numbers)
└── News crawl (extract VN phone patterns from scam news)

Processing:
├── Normalize to E.164 (+84...)
├── Weighted score: source trust × recency × frequency × cross-source confirmation
├── Hard-block threshold: high confidence only (reduce false positives)
└── Appeal/unlist process

API response:
{
  "phone": "+84xxxxxxxxx",
  "risk_score": 0.92,
  "label": "scam",
  "reason_codes": ["community_reports", "checkscam_vn", "news_mention"],
  "report_count": 47,
  "first_seen": "2025-11-03",
  "evidence_urls": [...]
}
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Mobile framework | Flutter (Dart) |
| Call screening (Android) | `CallScreeningService` via platform channel |
| Caller ID (iOS) | `CallKit CallDirectory` via platform channel |
| State management | Riverpod |
| Local storage | Hive (block list cache, settings) |
| Backend API | DOSafe Supabase (phone reputation, user reports) |
| AI inference | DOSafe vLLM (`inference-ref.dos.ai`) — text, image, audio |
| Push notifications | Firebase Cloud Messaging |
| Analytics | PostHog (privacy-first) |
| Auth | Firebase Auth (phone number + Google) |

---

## Data Flow — Incoming Call

```
Phone rings
    ↓
Android CallScreeningService.onScreenCall()
    ↓
Lookup phone number → DOSafe API (cached locally, TTL 24h)
    ↓
risk_score < 0.3  → Allow call, show caller label
risk_score 0.3–0.7 → Allow + overlay warning banner
risk_score > 0.7  → Auto-block OR show "Block?" prompt (user setting)
    ↓ (if allowed)
Call connects → overlay persists with risk info
Call ends → if risk > 0.5: prompt for AI voice analysis
```

---

## Privacy & Compliance

- Phone numbers are hashed before sending to DOSafe API (SHA-256 prefix query — only full hash sent if match found)
- Audio recordings for AI analysis: encrypted upload, deleted from server after analysis (30s max retention)
- VN PDPD (Personal Data Protection Decree 13/2023) compliance: consent screen on first run, data deletion on request
- No contact list upload (unlike TrueCaller's controversial model)
- Local caching of block list — most lookups never leave device

---

## Implementation Notes

### CallScreeningService setup (Android)

```kotlin
// AndroidManifest.xml
<service android:name=".CallScreeningService"
    android:permission="android.permission.BIND_SCREENING_SERVICE">
    <intent-filter>
        <action android:name="android.telecom.CallScreeningService"/>
    </intent-filter>
</service>
```

Requires user to set DOSafe as default phone app screening provider (RoleManager, Android 10+). Prompt shown on first launch.

### iOS CallDirectory setup

CallDirectory Extension syncs DOSafe block list to iOS system. Updates via background fetch (max 24h). iOS does NOT support dynamic per-call screening — only static block list + caller ID label.

### Flutter platform channel

```dart
// lib/services/call_screening_channel.dart
const _channel = MethodChannel('dosafe/call_screening');

Future<void> setBlockList(List<String> numbers) async {
  await _channel.invokeMethod('setBlockList', {'numbers': numbers});
}
```
