# Meta Ads, Lead Ads & Conversions (CAPI)

Connect Meta Marketing API to DOSClaw AI agents. Capture real-time lead submissions from Facebook & Instagram Lead Ads to trigger instant follow-ups within seconds, monitor campaign ROAS, and report server-side conversion events via Meta Conversions API (CAPI).

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `list_meta_accounts` | Read | List connected Meta Ad Accounts, Pages, and Pixel Datasets. |
| `get_meta_lead` | Read | Fetch detailed lead data (name, email, phone number, form questions) by Lead ID. |
| `list_form_leads` | Read | Retrieve recent lead submissions from specific Instant Forms. |
| `list_meta_campaigns` | Read | Query active and paused ad campaigns, budgets, and objectives. |
| `get_campaign_insights` | Read | Retrieve performance metrics (Spend, Impressions, Clicks, CPC, CTR, Leads, ROAS). |
| `update_campaign_status` | Write | Pause or resume ad campaigns based on target CPA or performance thresholds. |
| `send_capi_event` | Write | Dispatch server-side conversion events (`Lead`, `Purchase`, `Contact`) with automatic PII SHA-256 hashing. |

---

## Prerequisites

1. A **Meta Business Manager** (Meta Business Suite) account.
2. A **Facebook Page** with active or scheduled Lead Generation Ad campaigns.
3. Administrator or Lead Access Manager permissions on the target Page.

---

## Setup Guide

### Step 1: Connect via Meta OAuth

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **Meta Ads & Lead Ads Automation**.
2. Click **Connect with Meta**.
3. Log in with your Facebook credentials.
4. Select the **Ad Accounts** and **Facebook Pages** you want to automate.
5. Review and approve the permissions:
   - `leads_retrieval`: Ingest real-time leads from instant forms.
   - `pages_read_engagement`: Access form definitions and metadata.
   - `ads_read`: Query campaign insights and ROAS metrics.
   - `ads_management`: (Optional) Allow agent to pause underperforming ad sets.
6. Confirm authorization. Your long-lived access token is encrypted and stored in the DOS-Me Vault.

### Step 2: Configure Inbound Lead Webhook

DOSClaw provides an inbound real-time webhook endpoint:
```
https://api.dos.ai/v1/webhooks/meta-leads
```
When a user submits a Lead Form on Facebook or Instagram, Meta sends a webhook event to DOSClaw within ~2 seconds.

### Step 3: Bind to an Agent

1. Open your Sales, Real Estate, or Customer Service agent $\rightarrow$ **Integrations**.
2. Enable **Meta Ads & Leads**.
3. Under **Lead Form Trigger Action**, specify what the agent should do when a new lead arrives:
   - **Send Zalo ZNS / WhatsApp Message**: Trigger an immediate greeting message to the lead's phone number.
   - **Notify Sales Team**: Dispatch a lead card to your team's Telegram or Slack channel with form responses.
   - **Create CRM Contact**: Sync the lead into HubSpot, KiotViet, or Google Sheets.

---

## Conversions API (CAPI)

When an AI agent successfully closes a sale or qualifies a prospect during chat:

1. The agent calls `send_capi_event({ "event_name": "Purchase", "value": 450000, "currency": "VND" })`.
2. DOSClaw automatically normalizes and applies **SHA-256 hashing** to personal identifiers (email, phone).
3. The event is delivered directly to Meta's server-side Conversions API, improving ad attribution and lowering customer acquisition costs (CAC).

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "100 Unsupported get request" | Meta user lacking Lead Access permissions | In Meta Business Suite $\rightarrow$ Settings $\rightarrow$ Integrations $\rightarrow$ **Leads Access**, grant your user account access to the Page's leads. |
| Inbound leads delayed | Webhook subscription dropped | Re-save the integration in DOS.AI to refresh the Page webhook subscription. |
| CAPI event deduplication warning | Event ID missing | DOSClaw generates unique event IDs; verify your browser pixel isn't sending mismatched IDs. |
