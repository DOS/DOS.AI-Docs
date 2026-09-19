# Discord Bot & Server

Integrate DOSClaw AI agents into your Discord servers. Provide automated community management, answer technical questions, run slash commands, and support thread-based discussions.

## Capabilities

- **Server & Channel Chat**: Mention the bot (`@AgentName <question>`) in public channels.
- **Direct Messages (DMs)**: Private support and 1-on-1 assistance for community members.
- **Slash Commands**: Register custom slash commands (e.g. `/ask`, `/help`, `/lookup`).
- **Thread Continuity**: Auto-create or participate in threads to keep main channels organized.
- **Role-Based Access**: Restrict bot commands or sensitive knowledge base content to specific Discord server roles.

---

## Prerequisites

1. A Discord account with administrative permissions on your target server.
2. A registered Discord Application created in the [Discord Developer Portal](https://discord.com/developers/applications).

---

## Setup Guide

### Step 1: Create a Discord Application

1. Open the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**.
2. Give your bot a name and accept the terms.
3. Go to the **Bot** tab on the left sidebar:
   - Click **Reset Token** to generate a new **Bot Token**. Copy this value securely.
   - Under **Privileged Gateway Intents**, enable **Message Content Intent** (required for the bot to read user messages).
   - Enable **Server Members Intent** if your agent uses role or member lookups.

### Step 2: Invite the Bot to Your Server

1. In the Developer Portal, go to **OAuth2** $\rightarrow$ **URL Generator**.
2. Under **Scopes**, check:
   - `bot`
   - `applications.commands`
3. Under **Bot Permissions**, select:
   - `Send Messages`, `Read Message History`, `Send Messages in Threads`, `Embed Links`, `Attach Files`.
4. Copy the generated URL, open it in your browser, and select your server to authorize the bot.

### Step 3: Connect in DOS.AI Dashboard

1. In the DOS.AI Dashboard, go to your Agent's **Channels** tab.
2. Click **Add Channel** $\rightarrow$ **Discord**.
3. Paste the **Bot Token** and your **Application ID**.
4. Click **Save & Connect**.

---

## Channel Configuration

- **Allowed Channels**: You can specify a list of channel IDs where the bot is permitted to respond. In all other channels, the bot will remain silent even if mentioned.
- **Thread Participation**: Enable "Respond in Threads" to ensure long multi-turn answers stay contained inside threads.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Bot appears offline | Token invalid or regenerated | Re-copy the bot token from Developer Portal and update in DOS.AI. |
| Bot does not read message text | Message Content Intent disabled | In Developer Portal $\rightarrow$ Bot tab, enable **Message Content Intent**. |
| Missing permissions | Discord role below bot role | Ensure the bot's role in server settings has permission to send messages in the target channel. |
