# Stripe Payments

Connect Stripe to DOSClaw AI agents. Generate dynamic checkout links directly in conversation threads, check customer subscription statuses, verify payment receipts, and automate customer billing inquiries.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `create_payment_link` | Write | Generate an instant Stripe Checkout URL with customized line items, currency, and tax calculation. |
| `check_subscription_status` | Read | Look up customer subscription state (`active`, `past_due`, `canceled`) by email or customer ID. |
| `get_invoice_detail` | Read | Retrieve payment invoices, line items, and PDF download links. |
| `verify_payment_intent` | Read | Verify successful payment webhook execution before granting digital access. |

---

## Prerequisites

1. An active **Stripe account** (`https://stripe.com`).
2. Administrator access to retrieve your Stripe Restricted API Key or connect via Stripe Connect.

---

## Setup Guide

### Step 1: Connect via Stripe API Key

1. In the **Stripe Dashboard**, go to **Developers** $\rightarrow$ **API keys**.
2. Click **Create restricted key**:
   - `Checkout Sessions`: Write
   - `Payment Intents`: Read
   - `Customers`: Read
   - `Subscriptions`: Read
   - `Invoices`: Read
3. Copy the restricted key (`rk_live_...` or `rk_test_...` for testing).
4. In **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **Stripe Payments**.
5. Paste your restricted key and click **Save & Connect**.
6. DOS-Me validates the key by retrieving account information and stores the credential in encrypted Vault storage.

### Step 2: Configure Webhook (Optional but Recommended)

To enable automatic fulfillment when a customer pays:
1. In Stripe Dashboard, add a webhook destination:
   ```
   https://api.dos.ai/v1/webhooks/stripe
   ```
2. Select events: `checkout.session.completed`, `customer.subscription.updated`.
3. Copy the **Signing secret** (`whsec_...`) and paste it into DOS.AI.

---

## Example Sales Concierge Flow

> **Customer**: "I'd like to sign up for your quarterly consulting package for \$450."  
> **Agent**: Calls `create_payment_link({ "amount": 45000, "currency": "usd", "description": "Quarterly Consulting Package" })`  
> **Reply**: "Great! You can complete your secure checkout here: [Pay \$450 with Stripe Checkout](https://buy.stripe.com/demo123). Once finished, I will automatically send you the calendar invite for our kickoff call!"

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Key does not have required permissions" | Restricted key missing write scope | In Stripe Dashboard, ensure `Checkout Sessions` has **Write** permission. |
| Currency not supported | Currency code not enabled on merchant account | Check Stripe Dashboard $\rightarrow$ Settings $\rightarrow$ Currencies. |
| Test mode transactions in production | Using `rk_test_...` key | Replace test key with live restricted key (`rk_live_...`) when going to production. |
