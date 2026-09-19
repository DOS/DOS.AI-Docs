# Messaging Channels

DOSClaw connects your autonomous AI agents directly to customer inboxes, team communication workspaces, and social messaging channels. Agents receive inbound messages in real time, apply relevant tools and knowledge bases, and send back rich interactive responses.

## Supported Channels

| Channel | Protocol | Message Types | Key Capabilities |
| --- | --- | --- | --- |
| [Zalo OA](zalo.md) | Webhook / Official Account API | Text, Images, Interactive Cards | Follower sync, human takeover, 2-way chat |
| [Telegram](telegram.md) | Bot API / Webhook | MarkdownV2, Media, Commands | Group chats, inline queries, instant pairing |
| [Facebook Messenger](messenger.md) | Meta Graph API / Webhook | Text, Quick Replies, Media | Page inbox, comment-to-inbox auto-reply |
| [Discord](discord.md) | Discord Gateway / REST | Markdown, Embeds, Slash Commands | Server moderation, threads, role-based access |
| [Slack](slack.md) | Slack Events API / Block Kit | Blocks, Modals, Mentions | Team assistant, channel monitoring, workflows |
| [WhatsApp](whatsapp.md) | WhatsApp Cloud API | Text, Media, Templates | Global reach, end-to-end encrypted, concierge |
| [Lark / Feishu](lark.md) | Open Platform Bot API | Interactive Cards, Rich Text | Enterprise chat, group mentions, card actions |

---

## How Channels Work

```
User Message
     │
     ▼
[Messaging Provider] (Zalo, Telegram, Meta, Slack, etc.)
     │  (Inbound Webhook)
     ▼
[DOSClaw Gateway / Event Router]
     │  (Context + Identity + Scope Guard)
     ▼
[Agent Container / Engine]
     │  (Knowledge Base + MCP Tools + Reasoning)
     ▼
[Outbound Dispatcher]
     │  (Formatted response, cards, media)
     ▼
User In-App Reply
```

### Key Architectural Concepts

1. **Independent Channel Bindings**:
   A single AI agent can listen and respond across multiple channels simultaneously (e.g. Zalo OA for Vietnam customers, Telegram for global community, and Slack for internal staff) while sharing the same underlying memory, personality, and knowledge base.

2. **Cross-Channel Identity**:
   DOSClaw associates customer identity across different channels when an identifying attribute (phone number, email, or verified account ID) is provided.

3. **Human Takeover & Handoff**:
   Every channel supports human agent takeover. When an agent detects customer frustration, an explicit request for human support, or when a human agent sends a message from the native channel console, the AI automatically pauses for that conversation session.

4. **Rate Limiting & Safety**:
   Inbound webhooks are signature-verified and rate-limited. Egress responses adhere to platform quotas and prevent spam loops.

---

## Getting Started

To connect a channel to an agent:

1. Open your agent in the **DOS.AI Dashboard** (`https://app.dos.ai/agents`).
2. Navigate to the **Channels** tab.
3. Select your desired platform and follow the step-by-step connection wizard.
4. Verify the connection by sending a test message to your bot.
