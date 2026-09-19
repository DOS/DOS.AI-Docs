# Zalo Official Account (OA)

Deploy your DOSClaw AI agents to Vietnam's leading messaging platform. Connect your verified Zalo OA to automate 24/7 customer service, product inquiries, interactive menus, and order updates.

## Capabilities

- **2-Way Customer Chat**: Natural conversation with customers who message your Official Account.
- **Interactive Action Cards**: Rich product carousels, action buttons, and quick-reply options.
- **Follower Synchronization**: Track active followers and store user profiles.
- **Human Agent Takeover**: Automatic AI pause when a human support staff member joins the conversation.
- **Media Support**: Send and receive photos, PDFs, catalog images, and audio voice messages.

---

## Prerequisites

Before connecting, ensure you have:

1. A verified **Zalo Official Account** (Enterprise or Service type).
2. A **Zalo for Developers** account (`https://developers.zalo.me/`).
3. An active Zalo Developer App linked to your Official Account.

---

## Connection Methods

DOS.AI supports two setup options:

### Option 1: 1-Click OAuth (Recommended)

1. Go to **Dashboard** $\rightarrow$ **Integrations** or open your Agent's **Channels** tab.
2. Select **Zalo Official Account (OA)** and click **Connect via Zalo OAuth**.
3. Log in with your Zalo account that has Admin permissions on the target Official Account.
4. Select the Official Account to grant permissions (`oa.message`, `oa.profile`, `oa.media`).
5. Complete authorization. The access token and refresh token are securely vaulted in DOS-Me.

### Option 2: Manual Credentials / Webhook Pairing

If configuring custom webhooks or enterprise apps:

1. In **Zalo Developer Portal**, retrieve your **App ID** and **App Secret**.
2. Set the Webhook URL to:
   ```
   https://api.dos.ai/v1/channels/zalo/webhook
   ```
3. Set your verification token and subscribe to the `user_send_text`, `user_send_image`, and `follow` events.
4. In the DOS.AI Dashboard, enter your **OA ID**, **App ID**, and **App Secret**.

---

## Agent Configuration & Behavior

### Scope Guard & Business Focus

Zalo OA bots usually serve customer queries. By default, DOSClaw activates **Scope Guard** for business templates (e-commerce, real estate, customer service), ensuring the bot politely redirects off-topic requests (e.g. general coding or school homework) back to your business products and services.

### Human Agent Handoff

When a customer asks for human support ("cho gặp nhân viên", "gặp tư vấn viên"):

1. The agent transitions the conversation state to `human_takeover`.
2. The agent sends a polite confirmation to the customer.
3. Notification alerts can be dispatched to your team's Telegram or Slack group.
4. The AI stays muted until the customer issue is closed or the handover timeout expires.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Bot not replying to user messages | Webhook event not subscribed | Ensure `user_send_text` event is enabled in Zalo Developer portal. |
| Token expired error | Refresh token rotation failed | Re-authenticate via the **Reconnect** button on the dashboard. |
| Image messages not sending | Unsupported format | Ensure images are PNG/JPEG and under 5MB. |
| Messages duplicated | Multiple webhook endpoints | Ensure only DOSClaw webhook URL is configured. |
