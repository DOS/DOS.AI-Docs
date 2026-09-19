# Lark / Feishu

Connect your DOSClaw agents to Lark (and Feishu) to provide enterprise collaboration, project triage, employee assistance, and automated card interactions in enterprise groups.

## Capabilities

- **Direct Messages & Group Chats**: Respond in private 1-on-1 chats and group mentions (`@AgentName`).
- **Interactive Message Cards**: Send structured interactive cards with buttons, date pickers, and status indicators.
- **Thread Context**: Respond within the thread to maintain topic organization.
- **Enterprise Identity**: Sync employee user IDs and organization departments for permissioned responses.

---

## Prerequisites

1. An enterprise account on **Lark** (`https://www.larksuite.com/`) or **Feishu** (`https://open.feishu.cn/`).
2. Administrator access to create enterprise custom applications in the Lark Developer Console.

---

## Setup Guide

### Step 1: Create an App in Lark Open Platform

1. Go to the [Lark Developer Console](https://open.larksuite.com/app).
2. Click **Create Custom App**, enter an app name, and upload an icon.
3. Under **Credentials & Basic Info**, copy the **App ID** and **App Secret**.

### Step 2: Enable Bot Feature & Permissions

1. In the app navigation, go to **Add Features** $\rightarrow$ select **Bot**.
2. Go to **Permissions & Scopes**:
   - Add `im:message` (Send and receive messages).
   - Add `im:message.group_at_msg:readonly` (Receive messages in groups where bot is mentioned).
   - Add `im:message.p2p_msg:readonly` (Receive private direct messages).
   - Add `im:chat:readonly` (Access group chat information).

### Step 3: Configure Event Subscriptions

1. Go to **Event Subscriptions**:
2. Set the Request URL to:
   ```
   https://api.dos.ai/v1/channels/lark/events
   ```
3. Subscribe to the event: `im.message.receive_v1`.
4. Copy the **Verification Token** and **Encrypt Key** (if enabled).

### Step 4: Connect in DOS.AI Dashboard

1. In the DOS.AI Dashboard, go to your Agent's **Channels** tab.
2. Select **Lark / Feishu**.
3. Enter your **App ID**, **App Secret**, **Verification Token**, and **Encrypt Key** (optional).
4. Publish a version of your custom app in the Lark Developer Console to make it available to your organization.

---

## Usage in Lark

1. Open any chat or group in Lark.
2. Search for the bot by name or add it via group settings.
3. Send `@YourBot <message>` in a group or message the bot directly in a private DM.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Verification URL failed" | Endpoint unreachable or token mismatch | Check that the verification token in DOS.AI matches the token in Lark developer console. |
| Bot not replying to group mentions | App version not published | Ensure you submitted a release version in Lark Developer Console and an org admin approved it. |
| Message encryption error | Encrypt Key mismatch | If Encrypt Key is enabled in Lark, provide the exact string in the DOS.AI configuration. |
