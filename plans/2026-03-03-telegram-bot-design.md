# DOSafe Telegram Bot — Design Document

## Overview

A Telegram bot for DOSafe that lets users check AI-generated text and (in future) detect scams, directly from Telegram without visiting the web app.

## Architecture

```
Telegram → webhook POST → Supabase Edge Function (dosafe-telegram)
                                    ↓
                         dosafe.io/api/detect   (text detection)
                                    ↓
                         Supabase DB (bot_quota table, per chat_id)
```

## Platform Decision

**Supabase Edge Function** (not Vercel) because:
- 150s timeout vs Vercel's 30s — AI detection can take 10-30s
- Native Supabase DB access for quota tracking
- 500k free invocations/month
- Shared infrastructure with DOSafe backend

## File Structure

```
supabase/functions/
├── _shared/
│   ├── detect.ts       # Call dosafe.io/api/detect
│   ├── quota.ts        # Quota DB read/write
│   ├── language.ts     # Language detection (LLM + regex fallback)
│   └── telegram.ts     # Telegram Bot API helpers (sendMessage, etc.)
└── dosafe-telegram/
    └── index.ts        # Main webhook handler + command routing
```

Future bots follow same pattern: `dosafe-discord/`, `dosai-telegram/`, etc.

## Commands

| Command | Behavior |
|---------|----------|
| `/start` | Welcome message + usage instructions |
| `/detect [text]` | AI detection (also triggered by plain text ≥50 chars) |
| `/scam [text]` | Scam detection — placeholder for now |
| `/quota` | Show remaining quota |
| `/link` | Link DOSafe account (magic link via bot) |
| `/help` | Command list |

## Detection Response Format

```
🔍 Analyzing...

[verdict emoji] Result: AI-generated (87%)
━━━━━━━━━━━━━━━━
📊 Confidence: High
📝 [reasoning in same language as input]

🔗 Matching sources: [if any]
━━━━━━━━━━━━━━━━
📈 Quota: 12/100 uses today
```

Verdict emojis: 🤖 AI | 👤 Human | ❓ Mixed

## Scam Placeholder Response

```
🚧 Feature coming soon
We'll notify you when scam detection is available!
```

## Quota System

- **Unlinked**: tracked by `telegram_chat_id` in `bot_quota` table
  - Same limit as anonymous web users
- **Linked DOSafe account**: uses that account's quota plan
- DB table: `bot_quota(chat_id bigint primary key, used int, reset_at timestamptz)`

## Language Detection

1. **Primary**: LLM detect via `inference-ref.dos.ai` (same Qwen3-8B observer)
2. **Fallback**: regex — Vietnamese diacritics present → VI, else EN
3. Bot UI strings (commands, labels) match detected language

## Webhook Setup

- URL: `https://gulptwduchsjcsbndmua.supabase.co/functions/v1/dosafe-telegram`
- Secret: `TELEGRAM_BOT_TOKEN` validated via `X-Telegram-Bot-Api-Secret-Token` header
- Register via: `setWebhook` Telegram Bot API call

## Secrets Required

```
TELEGRAM_BOT_TOKEN=        # From @BotFather
DOSAFE_API_URL=https://dosafe.io
DOSAFE_INTERNAL_KEY=       # Optional: bypass public quota for bot requests
```

## Not In Scope (v1)

- Image detection (future)
- Group chat support (future — private chat only for now)
- Discord bot (separate function, same _shared logic)
- Account linking flow (stub /link command only)
