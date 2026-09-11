# Haravan Omnichannel

Connect your Haravan store to DOSClaw AI agents to automate product inquiries, inventory lookups, order status tracking, and fulfillment updates across Zalo OA, Messenger, Telegram, and your website.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Search product catalog by keywords, title, SKU, or tags. Returns pricing, description, and images. |
| `get_product_detail` | Read | Retrieve complete product specifications, variants, and real-time inventory count. |
| `check_order_status` | Read | Look up customer orders by order ID or phone number. Returns fulfillment and shipping status. |
| `create_order` | Write | Generate a draft or confirmed order with selected line items and customer delivery info. |

---

## Prerequisites

1. An active **Haravan store** (`your-shop.myharavan.com`).
2. An account with **Owner** or **Administrator** privileges in your Haravan admin dashboard.

---

## Setup Guide

### Step 1: Connect via Haravan OAuth

1. Open the **DOS.AI Dashboard** (`https://app.dos.ai/`).
2. Navigate to **Integrations** $\rightarrow$ find **Haravan Omnichannel**.
3. Click **Connect Store**.
4. Enter your Haravan shop domain (e.g. `your-store-name.myharavan.com`).
5. You will be redirected to Haravan's secure authorization screen (`accounts.haravan.com`).
6. Log in with your Haravan credentials and approve access permissions:
   - Read Products & Variants (`com.read_products`)
   - Read Orders & Customers (`com.read_orders`)
   - Offline Access for automatic token refresh (`offline_access`)
7. Upon successful authorization, you are redirected back to DOS.AI.

### Step 2: Bind to Your AI Agent

1. Open your agent in **Agents** $\rightarrow$ **Select Agent** $\rightarrow$ **Integrations / Connectors**.
2. Under **Connected Stores**, check **Haravan Omnichannel**.
3. Select your desired governance policy:
   - **Autonomous**: Agent answers stock inquiries and looks up orders automatically.
   - **Approval Required**: For order creation, the agent asks the customer or operator to confirm details before submission.
4. Click **Save Changes**.

---

## How AI Agents Use Haravan

### 1. Natural Language Product Search
Customers ask:
> "Bên bạn có áo sơ mi trắng size L không, giá bao nhiêu?"

The agent automatically calls:
```json
search_products({ "query": "áo sơ mi trắng", "limit": 5 })
```
The agent inspects the inventory across branches, extracts the price and product URL, and answers:
> "Dạ bên em còn mẫu Áo Sơ Mi Trắng Oxford size L, giá 350.000đ, hiện còn 12 cái trong kho ạ! Bạn có muốn đặt hàng luôn không?"

### 2. Live Order Tracking
Customers provide an order code or phone number:
> "Đơn hàng #HD1042 của mình giao đến đâu rồi?"

The agent calls:
```json
check_order_status({ "order_id": "HD1042" })
```
Returns carrier information (e.g. GHN, Viettel Post) and current delivery stage without requiring human intervention.

---

## Security & Architecture

- **Token Security**: OAuth access tokens and rotating refresh tokens are stored in the DOS-Me Vault (`public.provider_connections`). Secret credentials never enter LLM prompts or client-side storage.
- **Single-Flight Cache Shield**: Frequent catalog queries are cached to protect your Haravan API rate limits during peak sale events.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Token expired" or 401 error | Haravan refresh token invalid | Click **Reconnect** in DOS.AI Dashboard to re-authorize the OAuth connection. |
| Product search returns no results | Inactive status or missing SKU | Ensure products in Haravan admin are marked as **Active** and visible on the Web channel. |
| Order lookup denied | Phone number mismatch | For customer privacy, verify that the order's phone matches the verified caller ID. |
