# WhatsApp Business Cloud API

Deploy your DOSClaw AI agents to WhatsApp to reach global customers with official Meta Cloud API infrastructure. Ideal for cross-border e-commerce, VIP hospitality, appointment scheduling, and customer concierge.

## Capabilities

- **Official Business Verification**: Operates via Meta's WhatsApp Business Cloud API using verified phone numbers.
- **24/7 AI Concierge**: Automated text and audio message processing.
- **Interactive Buttons & Lists**: Send structured list options and call-to-action buttons.
- **Template Messages**: Proactive shipping notifications, appointment reminders, and confirmation messages.
- **End-to-End Encryption**: Compliant with WhatsApp privacy and encryption standards.

---

## Prerequisites

1. A **Meta Business Manager** account.
2. A **Meta Developer Account** with a registered Meta App.
3. A clean phone number that is not currently registered on a consumer WhatsApp account (or migrated via Meta Cloud API).

---

## Setup Guide

### Step 1: Set Up Meta WhatsApp Cloud API

1. In the [Meta for Developers Portal](https://developers.facebook.com/), create an app with the **Business** type.
2. Add the **WhatsApp** product to your application.
3. In WhatsApp $\rightarrow$ **API Setup**, retrieve your:
   - **Phone Number ID**
   - **WhatsApp Business Account ID (WABA ID)**
   - **Permanent System User Access Token** (generated with `whatsapp_business_messaging` permissions in Business Manager).

### Step 2: Configure Inbound Webhooks

1. In your Meta App under **WhatsApp** $\rightarrow$ **Configuration**, click **Edit** on Webhook.
2. Set the Callback URL to:
   ```
   https://api.dos.ai/v1/channels/whatsapp/webhook
   ```
3. Enter your verification token (provided in the DOS.AI Dashboard when you start the setup wizard).
4. Under Webhook fields, click **Manage** and subscribe to **messages**.

### Step 3: Connect in DOS.AI Dashboard

1. In the DOS.AI Dashboard, go to your Agent's **Channels** tab.
2. Select **WhatsApp Business** and enter:
   - **Phone Number ID**
   - **WABA ID**
   - **System User Access Token**
3. Click **Save & Verify**.
4. Send a test message from a personal WhatsApp phone to your business number to verify instant reply.

---

## 24-Hour Customer Service Window

Meta enforces a strict 24-hour customer service window:

- **Customer-Initiated Conversations**: Once a customer messages your bot, the bot can respond freely with session messages for up to 24 hours.
- **Business-Initiated / Re-engagement**: After 24 hours of inactivity, proactive outbound messages must use pre-approved **WhatsApp Message Templates** (Utility, Authentication, or Marketing categories).

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Error code 131030 | Recipient phone number not on sandbox whitelist | In test mode, add recipient numbers to the allowed list in Meta portal, or register a live phone number. |
| Inbound message not triggering reply | Webhook field `messages` not subscribed | In Meta App $\rightarrow$ WhatsApp $\rightarrow$ Configuration, ensure `messages` is checked. |
| Expired Access Token | Temporary token used | Create a permanent System User in Meta Business Manager $\rightarrow$ System Users. |
