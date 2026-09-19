# Facebook Messenger & Pages

Connect your Facebook Pages to DOSClaw to automate customer support, answer product questions directly in Messenger, and turn post comments into private conversation threads.

## Capabilities

- **Page Direct Messages**: 24/7 automated responses to inbound customer inquiries in Messenger.
- **Comment-to-Inbox (Auto-DM)**: Automatically send a private message to users who comment on your Facebook posts.
- **Interactive Quick Replies**: Action chips and buttons that let users choose options with one tap.
- **Product & Order Inquiries**: Connect with your store catalog to recommend products and check delivery status.
- **Human Takeover**: Live agent status sync with Meta Business Suite Inbox.

---

## Prerequisites

1. An active **Facebook Page** where you have **Admin** or **Task Access: Manage Messages** permissions.
2. A **Meta Business Account** linked to the Page.

---

## Step-by-Step Setup

### Step 1: Connect via Meta OAuth

1. In the DOS.AI Dashboard, navigate to **Integrations** or your Agent's **Channels** tab.
2. Click **Connect Facebook Messenger**.
3. Log in with your Facebook account and select the **Facebook Pages** you want the AI agent to manage.
4. Grant the required permissions:
   - `pages_messaging`: Send and receive messages as the Page.
   - `pages_manage_metadata`: Subscribe webhooks to page events.
   - `pages_read_engagement`: Read comments and post reactions for auto-reply.
5. Click **Confirm Authorization**.

### Step 2: Select Managed Pages

Once authorized, choose which specific Page(s) to bind to your agent. You can bind multiple Pages to the same agent or assign different agents to different Pages.

### Step 3: Configure Comment-to-Inbox (Optional)

1. Under the Page settings in DOS.AI, toggle **Comment Auto-Reply**.
2. Set trigger keywords or enable for all public comments.
3. Define whether the bot should:
   - Reply to the public comment on the post.
   - Send a private Messenger message with detailed information or a product link.

---

## Human Agent Handoff

DOSClaw integrates with Meta Business Suite Inbox:

- When an agent detects that a customer needs human assistance, it tags the conversation as `human_takeover`.
- If a human operator types a reply directly inside Meta Business Suite or the Facebook Pages Manager app, the AI agent automatically enters standby mode for 24 hours (or until manually released).

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Bot not receiving messages | Webhook subscription dropped | In DOS.AI dashboard, click **Re-subscribe Webhook** next to your Page. |
| Permission denied error | Admin rights revoked | Verify your Facebook user account still has full task permissions on the Page. |
| Standard Messaging Window | 24-hour policy restriction | Meta requires messages outside 24h to use approved Message Tags (e.g. `CONFIRMED_EVENT_UPDATE`). |
