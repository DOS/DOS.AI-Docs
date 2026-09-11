# Slack Workspace

Connect DOSClaw AI agents to your enterprise Slack workspaces. Deploy internal knowledge base assistants, automated triage bots, incident response monitors, and productivity tools.

## Capabilities

- **Channel Mentions**: Mention `@AgentName` in any public or private channel to trigger an answer.
- **Direct Messages**: 1-on-1 conversations with employees for private tasks, HR inquiries, or IT helpdesk.
- **Threaded Replies**: Keeps channel conversations neat by replying directly in the message thread.
- **Block Kit UI**: Structured visual messages, action buttons, dropdown menus, and modal dialogs.
- **Multi-Tenant OAuth**: Single-click authorization for your entire workspace or enterprise grid.

---

## Prerequisites

1. Administrative or App Installation permissions in your target Slack workspace.
2. A Slack account associated with your organization.

---

## Connection Guide

### Option 1: 1-Click Slack App Installation (Recommended)

1. In the **DOS.AI Dashboard**, open your Agent $\rightarrow$ **Channels** $\rightarrow$ **Slack**.
2. Click **Add to Slack**.
3. You will be redirected to Slack's official authorization screen.
4. Select your target Slack Workspace from the top-right corner.
5. Review the requested permissions:
   - `app_mentions:read`: Listen to `@agent` mentions in channels.
   - `chat:write`: Send messages and replies.
   - `im:history` & `im:write`: Send and receive direct messages.
   - `channels:history` & `groups:history`: Contextual thread continuity.
6. Click **Allow**. The app will automatically register webhooks and secure credentials in the DOS-Me Vault.

### Option 2: Custom Slack App (Enterprise / Internal App)

For enterprises requiring custom internal Slack App credentials:

1. Create an app on [api.slack.com/apps](https://api.slack.com/apps).
2. Set the Request URL for Event Subscriptions to:
   ```
   https://api.dos.ai/v1/channels/slack/events
   ```
3. Subscribe to bot events: `app_mention`, `message.im`.
4. Install the app to your workspace and copy the **Bot User OAuth Token** (`xoxb-...`) and **Signing Secret**.
5. Paste these credentials into the custom fields in the DOS.AI Dashboard.

---

## Channel Setup in Slack

Once installed:

1. In any Slack channel, type `/invite @YourAgentName` to add the bot to the channel.
2. Tag the bot in a message:
   ```
   @MyAgent How do I request approval for new equipment?
   ```
3. The bot will automatically respond in a threaded reply, maintaining thread context for follow-up questions.

---

## Security & Access Control

- **Internal Data Protection**: Bot interactions adhere to organization-level tenant boundaries.
- **Channel Whitelist**: Restrict bot operation to specific Slack channels (e.g. `#help-it`, `#ask-marketing`).
- **Prompt Injection Defense**: Default Scope Guard filters prevent external manipulation when bots interact with mixed channel audiences.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "dispatch_failed" on mention | Bot not invited to channel | Run `/invite @AgentName` in the channel. |
| URL Verification failed | Webhook endpoint mismatch | Ensure you used `https://api.dos.ai/v1/channels/slack/events`. |
| Bot doesn't reply in DMs | `message.im` event missing | Verify `message.im` is checked under Event Subscriptions in Slack App settings. |
