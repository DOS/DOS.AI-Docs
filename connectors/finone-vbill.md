# FinOne / Vbill Merchant Suite

Connect FinOne and Vbill merchant accounts to DOSClaw AI agents. Enable automated revenue reporting, payment reconciliation, and instant VietQR dynamic payment link generation during customer chat conversations.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `get_revenue_stats` | Read | Fetch daily, weekly, or monthly merchant transaction revenue and order count. |
| `reconcile_orders` | Read | Reconcile bank transfer records and POS transactions against pending merchant orders. |
| `generate_payment_qr` | Write | Generate dynamic VietQR code images and payment links with exact order amount and transfer description. |
| `check_transaction_status` | Read | Verify if a specific payment transfer has been credited to the merchant wallet. |

---

## Prerequisites

1. An active **FinOne** or **Vbill** merchant account.
2. Merchant ID and API Secret Key provided by your FinOne account manager.

---

## Setup Guide

### Step 1: Connect in DOS.AI Dashboard

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **FinOne / Vbill Merchant Suite**.
2. Click **Connect Merchant**.
3. Enter your:
   - **Merchant ID**
   - **Partner Secret Key**
   - **Bank Account / QR Preset**
4. Click **Verify & Connect**.

### Step 2: Bind to Your AI Agent

1. Open your Sales or E-Commerce agent $\rightarrow$ **Integrations**.
2. Enable **FinOne / Vbill**.
3. Select whether the agent can automatically generate payment QR codes (`autonomous`) when customers confirm an order.

---

## Example In-Chat Payment Flow

1. Customer agrees to purchase products totaling 450,000 VND.
2. Agent calls:
   ```json
   generate_payment_qr({
     "amount": 450000,
     "order_ref": "ORDER-9821",
     "description": "DH 9821"
   })
   ```
3. Agent sends back a formatted VietQR image card with bank transfer details.
4. Customer scans QR code via banking app (Vietcombank, MB, Techcombank, etc.).
5. FinOne webhook alerts DOSClaw, and the agent confirms payment receipt to the customer within seconds.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Invalid Merchant Signature" | Partner Secret Key mismatch | Ensure the secret key in DOS.AI matches the key in FinOne merchant portal. |
| Payment confirmation delayed | Bank NAPAS delay | Check FinOne transaction dashboard or allow up to 30 seconds for bank webhook callback. |
