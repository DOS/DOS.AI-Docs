# DOSClaw Agent Platform

**Autonomous multi-channel AI agent runtime powered by OpenClaw with isolated container execution, tool bindings, and omnichannel messaging.**

DOSClaw is DOS.AI's enterprise-grade autonomous agent platform. It provisions, orchestrates, and monitors containerized AI agents that operate 24/7 across multiple communication channels—including Telegram, Zalo Official Account (OA), Facebook Messenger, Discord, Slack, WhatsApp, and Lark / Feishu.

---

## High-Level Architecture

Each DOSClaw agent runs inside a dedicated, isolated Docker container powered by the **OpenClaw** multi-agent operating system. Containers are managed across a distributed Docker host fleet via a lightweight, authenticated Status API.

```
                    ┌────────────────────────────────────────────────────────┐
                    │                  Go API Gateway (Cloud Run)            │
                    │      Routing · Metering · Billing · Lifecycle CRUD     │
                    └───────────────────────────┬────────────────────────────┘
                                                │
                 Cloudflare Zero Trust Tunnel ──┼── Bearer-Authed Control Plane
                                                ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Docker Host Fleet (Mumbai Lambda / GCP Singapore / Local Canary)                                  │
│                                                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────┐  │
│   │ Node.js Status API (:18090)                                                                │  │
│   │ Container CRUD · Volume Uploads · Metrics · Exec Allowlist                                 │  │
│   └───────────────────────────────────────────┬────────────────────────────────────────────────┘  │
│                                               │                                                   │
│                 ┌─────────────────────────────┴─────────────────────────────┐                     │
│                 ▼                                                           ▼                     │
│   ┌───────────────────────────┐                               ┌───────────────────────────┐       │
│   │ Agent Container (Em Huyền)│                               │ Agent Container (Shop A)  │       │
│   │ ├── OpenClaw Runtime      │                               │ ├── OpenClaw Runtime      │       │
│   │ ├── /workspace (Volume)   │                               │ ├── /workspace (Volume)   │       │
│   │ ├── Local loopback ws/http│                               │ ├── Local loopback ws/http│       │
│   │ └── dos-grounding plugin  │                               │ └── dos-grounding plugin  │       │
│   └───────────────────────────┘                               └───────────────────────────┘       │
│                                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Core System Documentation

| Section | Description |
| --- | --- |
| [Architecture & Runtime](architecture.md) | Container fleet architecture, volume lifecycle, health probing, and auto-restart |
| [Workspace Files & Prompts](workspace-files.md) | Standard workspace files (`SOUL.md`, `IDENTITY.md`, `MEMORY.md`), template contracts, and dynamic compilation |
| [Tools & Action Invariants](tools-and-actions.md) | Action invariants, tool grounding, safe recovery loops, and MCP schema standards |

---

## Messaging Channels

DOSClaw connects your AI agents directly to customer inboxes, team workspaces, and social platforms:

| Channel | Key Capabilities | Guide |
| --- | --- | --- |
| **Zalo OA** | Follower sync, human takeover, interactive cards | [Zalo OA Guide](channels/zalo.md) |
| **Telegram** | Group chats, MarkdownV2 formatting, instant pairing | [Telegram Guide](channels/telegram.md) |
| **Facebook Messenger** | Page inbox, comment-to-inbox auto-reply | [Messenger Guide](channels/messenger.md) |
| **Discord** | Server moderation, threads, role-based access | [Discord Guide](channels/discord.md) |
| **Slack** | Team assistant, Block Kit messages, mentions | [Slack Guide](channels/slack.md) |
| **WhatsApp** | Global reach, end-to-end encrypted concierge | [WhatsApp Guide](channels/whatsapp.md) |
| **Lark / Feishu** | Enterprise chat, interactive card actions | [Lark Guide](channels/lark.md) |

For setup instructions and event flow architecture, see the [Messaging Channels Overview](channels/overview.md).

---

## Plugins & MCP Connectors

Equip your AI agents with real-time business actions, database queries, and third-party integrations:

* **E-Commerce**: [Haravan](connectors/haravan.md), [Shopee](connectors/shopee.md), [TikTok Shop](connectors/tiktokshop.md), [Shopify](connectors/shopify.md), [KiotViet](connectors/kiotviet.md), [WooCommerce](connectors/woocommerce.md).
* **CRM & Marketing**: [HubSpot](connectors/hubspot.md), [Salesforce](connectors/salesforce.md), [Crove Post](connectors/crove-post.md), [NueLink](connectors/nuelink.md), [WordPress](connectors/wordpress.md), [Meta Ads](connectors/meta.md).
* **Finance & Invoicing**: [MISA Enterprise Suite](connectors/misa.md), [FinOne / Vbill](connectors/finone-vbill.md), [Stripe Payments](connectors/stripe.md).
* **Productivity & Developer**: [Google Workspace](connectors/google-workspace.md), [GitHub](connectors/github.md), [Notion](connectors/notion.md), [Brave Search Engine](tools/brave-search.md).

For authentication, connection binding, and caching details, see the [Connectors Directory Overview](connectors/overview.md).

---

## Quickstart: Deploy Your First Agent

1. **Create an Agent**: Open the [DOS.AI Dashboard](https://app.dos.ai/agents) and click **Create Agent**.
2. **Configure Persona & System Instructions**: Provide your bot's identity, target audience, and guidelines. DOSClaw automatically compiles these into standardized workspace files.
3. **Connect a Channel**: Navigate to the **Channels** tab and connect Telegram, Zalo OA, Messenger, or your preferred messaging app.
4. **Attach Connectors & Skills**: Enable web search (Brave Search), CRM, or e-commerce tools from the **Tools & Skills** catalog.
5. **Start Chatting**: Send a message to your bot on your connected channel or test live inside the dashboard playground!
