# Telegram Bot

Connect your AI agent to Telegram to serve private direct messages, community group chats, support desks, and broadcast notifications with ultra-low latency.

## Capabilities

- **Private 1-on-1 Chat**: Direct, conversational assistant for personal productivity or customer support.
- **Group Assistant**: Mention your bot in group chats (`@YourBot`) or allow it to participate in group discussions.
- **Rich Markdown Formatting**: Bold, italics, inline code, syntax-highlighted code blocks, and structured lists.
- **Inline Commands & Buttons**: Interactive inline keyboards, callback buttons, and quick actions.
- **Voice & Media Handling**: Process voice notes (with automatic speech-to-text), images, documents, and files.

---

## Connection Methods

### Method 1: Instant Pairing via @DOSClawBot (Fastest)

1. Open Telegram and search for `@DOSClawBot`.
2. Send `/start` to receive your pairing code.
3. In the DOS.AI Dashboard under **Channels** $\rightarrow$ **Telegram**, enter the pairing code.
4. Your agent is immediately connected to your Telegram user account for private testing.

### Method 2: Custom Bot with BotFather (For Production & Brand Bots)

To run under your own brand's bot username (e.g. `@MyStoreAssistantBot`):

1. Open Telegram and chat with [@BotFather](https://t.me/botfather).
2. Send `/newbot` and follow the prompts to choose a display name and username (must end in `bot`).
3. BotFather will provide an **HTTP API Bot Token** formatted like `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`.
4. (Optional) Configure group privacy if you want the bot to see all group messages:
   - In BotFather, send `/setprivacy` $\rightarrow$ select your bot $\rightarrow$ choose `Disable`.
5. In **DOS.AI Dashboard**, open your Agent $\rightarrow$ **Channels** $\rightarrow$ **Telegram**.
6. Paste your Bot Token and click **Save & Connect**.
7. DOSClaw automatically registers the high-performance webhook with Telegram API.

---

## Group Chat Usage

When added to a Telegram group:

1. **Mention-Only Mode (Default)**: The bot responds only when explicitly mentioned (`@YourBot <message>`) or when a user directly replies to one of its messages. This prevents noise in active groups.
2. **Always-Listen Mode**: If enabled in Agent Channel Settings and Group Privacy is disabled in BotFather, the bot evaluates every group message and responds when relevant.

---

## Security & Privacy

- **Token Safety**: Your bot token is stored in encrypted vault storage (`public.provider_connections`). It is never exposed to client browsers or LLM prompts.
- **Allowed Users / Chat IDs**: You can restrict bot access to specific Telegram User IDs or Group Chat IDs using the whitelist setting in the dashboard.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Bot does not reply in groups | Privacy mode enabled | Chat with `@BotFather`, send `/setprivacy`, choose `Disable`. |
| Conflict: terminated by other getUpdates | Another service is polling the bot | Ensure no local script or test server is running `getUpdates` with the same token. |
| Formatting error | Invalid Markdown characters | DOSClaw handles escaping automatically, but verify raw template outputs if using custom prompts. |
