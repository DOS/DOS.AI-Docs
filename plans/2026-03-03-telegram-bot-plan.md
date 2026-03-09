# DOSafe Telegram Bot Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Telegram bot for DOSafe that detects AI-generated text and stubs scam detection, deployed as a Supabase Edge Function.

**Architecture:** Telegram sends webhook POSTs to a Supabase Edge Function (`dosafe-telegram`). The function routes commands, calls `dosafe.io/api/detect` for text analysis, and tracks per-user quota in Supabase DB. Shared utilities live in `supabase/functions/_shared/` for reuse by future bots.

**Tech Stack:** Deno (Supabase Edge Functions runtime), TypeScript, Telegram Bot API, Supabase JS client (native in Deno), dosafe.io REST API

---

## Pre-requisites (Manual steps — do before coding)

1. Create a Telegram bot via [@BotFather](https://t.me/BotFather) → get `TELEGRAM_BOT_TOKEN`
2. Add secret to Supabase: `supabase secrets set TELEGRAM_BOT_TOKEN=<token>`
3. Add secret: `supabase secrets set DOSAFE_API_URL=https://dosafe.io`
4. Install Supabase CLI if not present: `npm install -g supabase`
5. Link project: `cd d:/Projects/DOSafe && supabase link --project-ref gulptwduchsjcsbndmua`

---

## Task 1: Create Supabase DB migration for bot_quota table

**Files:**
- Create: `supabase/migrations/<timestamp>_bot_quota.sql`

**Step 1: Create migration file**

```bash
cd d:/Projects/DOSafe
supabase migration new bot_quota
```

**Step 2: Write the SQL**

Open the generated file and replace contents with:

```sql
create table if not exists public.bot_quota (
  chat_id bigint primary key,
  used int not null default 0,
  reset_at timestamptz not null default (now() + interval '1 day')
);

-- Reset quota daily
create or replace function reset_expired_bot_quota()
returns void language plpgsql as $$
begin
  update public.bot_quota
  set used = 0, reset_at = now() + interval '1 day'
  where reset_at < now();
end;
$$;
```

**Step 3: Apply migration locally (or push to production)**

```bash
supabase db push
```

Expected: migration applied, `bot_quota` table created.

**Step 4: Commit**

```bash
git add supabase/migrations/
git commit -m "feat: add bot_quota table migration for Telegram bot"
```

---

## Task 2: Create shared utilities

**Files:**
- Create: `supabase/functions/_shared/telegram.ts`
- Create: `supabase/functions/_shared/detect.ts`
- Create: `supabase/functions/_shared/quota.ts`
- Create: `supabase/functions/_shared/language.ts`

### Step 1: Create `_shared/telegram.ts`

Telegram Bot API helper — sends messages and shows typing indicator.

```typescript
// supabase/functions/_shared/telegram.ts

const TELEGRAM_API = 'https://api.telegram.org'

export async function sendMessage(
  token: string,
  chatId: number,
  text: string,
  parseMode: 'HTML' | 'Markdown' = 'HTML'
): Promise<void> {
  await fetch(`${TELEGRAM_API}/bot${token}/sendMessage`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: chatId, text, parse_mode: parseMode }),
  })
}

export async function sendTyping(token: string, chatId: number): Promise<void> {
  await fetch(`${TELEGRAM_API}/bot${token}/sendChatAction`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: chatId, action: 'typing' }),
  })
}

export async function registerWebhook(
  token: string,
  webhookUrl: string,
  secretToken: string
): Promise<void> {
  const res = await fetch(`${TELEGRAM_API}/bot${token}/setWebhook`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url: webhookUrl, secret_token: secretToken }),
  })
  const data = await res.json()
  console.log('Webhook registered:', data)
}
```

### Step 2: Create `_shared/detect.ts`

Calls `dosafe.io/api/detect` and returns structured result.

```typescript
// supabase/functions/_shared/detect.ts

export interface DetectResult {
  ai_probability: number
  human_probability: number
  verdict: 'AI' | 'Human' | 'Mixed'
  confidence: 'low' | 'medium' | 'high'
  reasoning: string
  source_matches: Array<{ title: string; url: string; trusted: boolean }>
}

export async function detectText(
  text: string,
  dosafApiUrl: string
): Promise<DetectResult> {
  const res = await fetch(`${dosafApiUrl}/api/detect`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text }),
  })

  if (!res.ok) {
    throw new Error(`Detection API error: ${res.status}`)
  }

  return res.json()
}
```

### Step 3: Create `_shared/quota.ts`

Per-chat-id quota tracking using Supabase DB directly.

```typescript
// supabase/functions/_shared/quota.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const DAILY_LIMIT = 20 // anonymous bot users

export interface QuotaStatus {
  allowed: boolean
  used: number
  limit: number
  resetAt: Date
}

export async function checkAndIncrementQuota(
  chatId: number,
  supabaseUrl: string,
  supabaseServiceKey: string
): Promise<QuotaStatus> {
  const supabase = createClient(supabaseUrl, supabaseServiceKey)

  // Reset expired quotas first
  await supabase.rpc('reset_expired_bot_quota')

  // Get or create quota row
  const { data, error } = await supabase
    .from('bot_quota')
    .upsert({ chat_id: chatId }, { onConflict: 'chat_id', ignoreDuplicates: false })
    .select()
    .single()

  if (error) throw error

  const used = data.used ?? 0
  const resetAt = new Date(data.reset_at)

  if (used >= DAILY_LIMIT) {
    return { allowed: false, used, limit: DAILY_LIMIT, resetAt }
  }

  // Increment
  await supabase
    .from('bot_quota')
    .update({ used: used + 1 })
    .eq('chat_id', chatId)

  return { allowed: true, used: used + 1, limit: DAILY_LIMIT, resetAt }
}

export async function getQuota(
  chatId: number,
  supabaseUrl: string,
  supabaseServiceKey: string
): Promise<QuotaStatus> {
  const supabase = createClient(supabaseUrl, supabaseServiceKey)
  await supabase.rpc('reset_expired_bot_quota')

  const { data } = await supabase
    .from('bot_quota')
    .select('used, reset_at')
    .eq('chat_id', chatId)
    .maybeSingle()

  return {
    allowed: true,
    used: data?.used ?? 0,
    limit: DAILY_LIMIT,
    resetAt: new Date(data?.reset_at ?? Date.now() + 86400000),
  }
}
```

### Step 4: Create `_shared/language.ts`

Detect language via regex (fast, no LLM needed for this).

```typescript
// supabase/functions/_shared/language.ts

export type Lang = 'vi' | 'en'

// Vietnamese diacritics are a reliable signal
const VN_PATTERN = /[àáâãèéêìíòóôõùúýăđơưạảấầẩẫậắằẳẵặẹẻẽếềểễệỉịọỏốồổỗộớờởỡợụủứừửữựỳỵỷỹ]/i

export function detectLanguage(text: string): Lang {
  const vnCount = (text.match(new RegExp(VN_PATTERN.source, 'gi')) ?? []).length
  return vnCount / text.length > 0.015 ? 'vi' : 'en'
}
```

### Step 5: Commit

```bash
git add supabase/functions/_shared/
git commit -m "feat: add shared utilities for Telegram bot (telegram, detect, quota, language)"
```

---

## Task 3: Create main bot handler

**Files:**
- Create: `supabase/functions/dosafe-telegram/index.ts`

**Step 1: Write the main handler**

```typescript
// supabase/functions/dosafe-telegram/index.ts
import { sendMessage, sendTyping } from '../_shared/telegram.ts'
import { detectText } from '../_shared/detect.ts'
import { checkAndIncrementQuota, getQuota } from '../_shared/quota.ts'
import { detectLanguage } from '../_shared/language.ts'

const TOKEN = Deno.env.get('TELEGRAM_BOT_TOKEN')!
const DOSAFE_API_URL = Deno.env.get('DOSAFE_API_URL') ?? 'https://dosafe.io'
const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!

// UI strings per language
const i18n = {
  vi: {
    welcome: `👋 Chào mừng đến với <b>DOSafe Bot</b>!\n\nTao giúp bạn phát hiện nội dung AI-generated.\n\n<b>Cách dùng:</b>\n• Gửi text trực tiếp (≥50 ký tự) để kiểm tra\n• /detect [text] — kiểm tra AI\n• /scam [text] — kiểm tra scam\n• /quota — xem quota còn lại\n• /help — danh sách lệnh`,
    analyzing: '🔍 Đang phân tích...',
    tooShort: '⚠️ Văn bản quá ngắn. Vui lòng nhập ít nhất 50 ký tự.',
    quotaExceeded: (reset: string) => `⚠️ Bạn đã dùng hết quota hôm nay.\nReset lúc: ${reset}\n\nLink tài khoản DOSafe để có nhiều lần hơn: /link`,
    quotaInfo: (used: number, limit: number, reset: string) => `📈 <b>Quota của bạn</b>\nĐã dùng: ${used}/${limit} hôm nay\nReset lúc: ${reset}`,
    scamPlaceholder: '🚧 <b>Tính năng đang phát triển</b>\nKiểm tra scam sẽ sớm ra mắt!',
    linkPlaceholder: '🔗 <b>Link tài khoản</b>\nTính năng này đang phát triển. Vui lòng đăng ký tại https://dosafe.io',
    help: `<b>Danh sách lệnh:</b>\n/detect [text] — Kiểm tra AI\n/scam [text] — Kiểm tra scam (sắp ra)\n/quota — Xem quota\n/link — Link tài khoản DOSafe\n/help — Trợ giúp`,
    errorDetect: '❌ Lỗi khi phân tích. Vui lòng thử lại sau.',
  },
  en: {
    welcome: `👋 Welcome to <b>DOSafe Bot</b>!\n\nI help you detect AI-generated content.\n\n<b>How to use:</b>\n• Send text directly (≥50 chars) to check\n• /detect [text] — check for AI\n• /scam [text] — check for scams\n• /quota — view remaining quota\n• /help — command list`,
    analyzing: '🔍 Analyzing...',
    tooShort: '⚠️ Text too short. Please enter at least 50 characters.',
    quotaExceeded: (reset: string) => `⚠️ You've used your quota for today.\nResets at: ${reset}\n\nLink your DOSafe account for more: /link`,
    quotaInfo: (used: number, limit: number, reset: string) => `📈 <b>Your Quota</b>\nUsed: ${used}/${limit} today\nResets at: ${reset}`,
    scamPlaceholder: '🚧 <b>Coming Soon</b>\nScam detection will be available soon!',
    linkPlaceholder: '🔗 <b>Link Account</b>\nThis feature is coming soon. Register at https://dosafe.io',
    help: `<b>Commands:</b>\n/detect [text] — Check for AI\n/scam [text] — Check for scams (coming soon)\n/quota — View quota\n/link — Link DOSafe account\n/help — Help`,
    errorDetect: '❌ Detection failed. Please try again later.',
  },
}

function formatVerdict(result: Awaited<ReturnType<typeof detectText>>, lang: 'vi' | 'en', quota: { used: number; limit: number; resetAt: Date }): string {
  const emoji = result.verdict === 'AI' ? '🤖' : result.verdict === 'Human' ? '👤' : '❓'
  const verdictLabel = {
    vi: { AI: 'Nội dung AI', Human: 'Nội dung người', Mixed: 'Hỗn hợp' },
    en: { AI: 'AI-generated', Human: 'Human-written', Mixed: 'Mixed' },
  }[lang][result.verdict]

  const confidenceLabel = {
    vi: { low: 'Thấp', medium: 'Trung bình', high: 'Cao' },
    en: { low: 'Low', medium: 'Medium', high: 'High' },
  }[lang][result.confidence]

  const resetStr = quota.resetAt.toLocaleTimeString(lang === 'vi' ? 'vi-VN' : 'en-US', { hour: '2-digit', minute: '2-digit' })

  let msg = `${emoji} <b>${verdictLabel} (${result.ai_probability}%)</b>\n`
  msg += `━━━━━━━━━━━━━━━━\n`
  msg += `📊 ${lang === 'vi' ? 'Độ tin cậy' : 'Confidence'}: ${confidenceLabel}\n`
  msg += `📝 ${result.reasoning}\n`

  if (result.source_matches?.length > 0) {
    msg += `━━━━━━━━━━━━━━━━\n`
    msg += `🔗 ${lang === 'vi' ? 'Nguồn khớp' : 'Matching sources'}:\n`
    result.source_matches.slice(0, 2).forEach(s => {
      msg += `• <a href="${s.url}">${s.title}</a>\n`
    })
  }

  msg += `━━━━━━━━━━━━━━━━\n`
  msg += `📈 Quota: ${quota.used}/${quota.limit} | Reset: ${resetStr}`

  return msg
}

async function handleDetect(chatId: number, text: string): Promise<void> {
  const lang = detectLanguage(text)
  const t = i18n[lang]

  if (text.length < 50) {
    await sendMessage(TOKEN, chatId, t.tooShort)
    return
  }

  // Check quota
  const quota = await checkAndIncrementQuota(chatId, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)
  if (!quota.allowed) {
    const resetStr = quota.resetAt.toLocaleTimeString()
    await sendMessage(TOKEN, chatId, t.quotaExceeded(resetStr))
    return
  }

  // Show typing indicator + analyzing message
  await sendTyping(TOKEN, chatId)
  await sendMessage(TOKEN, chatId, t.analyzing)

  try {
    const result = await detectText(text, DOSAFE_API_URL)
    const reply = formatVerdict(result, lang, quota)
    await sendMessage(TOKEN, chatId, reply)
  } catch (err) {
    console.error('Detection error:', err)
    await sendMessage(TOKEN, chatId, t.errorDetect)
  }
}

Deno.serve(async (req: Request) => {
  // Only accept POST
  if (req.method !== 'POST') {
    return new Response('OK', { status: 200 })
  }

  let update: any
  try {
    update = await req.json()
  } catch {
    return new Response('Bad request', { status: 400 })
  }

  const message = update?.message
  if (!message) return new Response('OK', { status: 200 })

  const chatId: number = message.chat.id
  const text: string = message.text ?? ''
  const lang = detectLanguage(text)
  const t = i18n[lang]

  // Route commands
  if (text.startsWith('/start')) {
    await sendMessage(TOKEN, chatId, t.welcome)
  } else if (text.startsWith('/help')) {
    await sendMessage(TOKEN, chatId, t.help)
  } else if (text.startsWith('/detect ')) {
    await handleDetect(chatId, text.slice('/detect '.length).trim())
  } else if (text.startsWith('/scam')) {
    await sendMessage(TOKEN, chatId, t.scamPlaceholder)
  } else if (text.startsWith('/quota')) {
    const quota = await getQuota(chatId, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)
    const resetStr = quota.resetAt.toLocaleTimeString()
    await sendMessage(TOKEN, chatId, t.quotaInfo(quota.used, quota.limit, resetStr))
  } else if (text.startsWith('/link')) {
    await sendMessage(TOKEN, chatId, t.linkPlaceholder)
  } else if (text && !text.startsWith('/')) {
    // Plain text — treat as detect
    await handleDetect(chatId, text)
  }

  return new Response('OK', { status: 200 })
})
```

**Step 2: Commit**

```bash
git add supabase/functions/dosafe-telegram/
git commit -m "feat: add dosafe-telegram Supabase Edge Function"
```

---

## Task 4: Deploy and register webhook

**Step 1: Deploy function**

```bash
cd d:/Projects/DOSafe
supabase functions deploy dosafe-telegram --no-verify-jwt
```

Expected output: function URL printed, e.g.:
`https://gulptwduchsjcsbndmua.supabase.co/functions/v1/dosafe-telegram`

**Step 2: Set secrets**

```bash
supabase secrets set TELEGRAM_BOT_TOKEN=<your_token>
supabase secrets set DOSAFE_API_URL=https://dosafe.io
```

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are auto-injected by Supabase.

**Step 3: Register webhook with Telegram**

```bash
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://gulptwduchsjcsbndmua.supabase.co/functions/v1/dosafe-telegram"}'
```

Expected response:
```json
{"ok": true, "result": true, "description": "Webhook was set"}
```

**Step 4: Verify webhook**

```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

Expected: `"url"` matches your function URL, `"pending_update_count": 0`

**Step 5: Test manually**

Open Telegram, find your bot, send:
- `/start` → welcome message
- `This is a test message to check if the AI detection is working correctly in the bot` → detection result
- `/scam` → placeholder message
- `/quota` → quota info

**Step 6: Commit**

```bash
git commit -m "chore: deploy dosafe-telegram bot and register webhook" --allow-empty
```

---

## Task 5: Set BotFather commands menu

**Step 1: Set command list via BotFather**

Send to @BotFather:
```
/setcommands
```
Then select your bot and paste:
```
detect - Check if text is AI-generated
scam - Check for scams (coming soon)
quota - View remaining quota
link - Link your DOSafe account
help - Show help
```

This makes commands appear as suggestions in Telegram UI.

---

## Done ✅

Bot is live. Test with:
1. Plain text ≥50 chars → AI detection result
2. `/scam anything` → placeholder
3. `/quota` → usage stats
4. Send 20+ messages → quota exceeded message
