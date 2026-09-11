# TikTok Shop

Connect your TikTok Shop Seller Center to DOSClaw AI agents. Empower your AI assistants during live streams, direct messages, and social campaigns with live product specs, inventory counts, and order fulfillment status.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Search products in your TikTok Shop catalog by title, SKU, or category. |
| `get_product_detail` | Read | Query variant details, live pricing, stock availability, and promotional links. |
| `check_order_status` | Read | Look up customer TikTok Shop orders by Order ID. |
| `get_fulfillment_tracking` | Read | Retrieve logistics provider details, tracking numbers, and delivery package status. |

---

## Prerequisites

1. An active **TikTok Shop Seller** account (Vietnam, US, UK, SEA).
2. Store owner credentials to authorize third-party partner applications.

---

## Setup Guide

### Step 1: Connect via TikTok Shop OAuth

1. Open the **DOS.AI Dashboard** (`https://app.dos.ai/`).
2. Go to **Integrations** $\rightarrow$ select **TikTok Shop**.
3. Click **Connect TikTok Shop**.
4. Select your target market region (e.g. `Vietnam`, `United States`, `Southeast Asia`).
5. Log in to TikTok Shop Seller Center and review required permissions:
   - Read product catalog and inventory levels.
   - Read orders and shipping fulfillment events.
6. Click **Authorize**. Once approved, your `seller_id` and cipher tokens are saved to the secure vault.

### Step 2: Bind to Your AI Agent

1. In the Agent settings, navigate to **Integrations**.
2. Select **TikTok Shop** from your list of connected stores.
3. Configure the **Reply Context**:
   - Enable "Attach TikTok Product Showcase Links" so the agent formats answers with clickable purchase buttons.
4. Click **Save**.

---

## Example Live Commerce Use Cases

### Real-Time Live Stream FAQ Bot
While streaming on TikTok Live, buyers frequently ask about sizing, material, and bundle discounts:
> **Viewer**: "Mẫu đầm hoa nhí còn size S không shop, cao 1m55 mặc vừa không?"  
> **AI Agent**: Queries `get_product_detail({ "query": "đầm hoa nhí" })` and cross-references your agent's knowledge base sizing chart:  
> **Reply**: "Dạ đầm hoa nhí bên em còn sẵn 8 cái size S ạ! Chiều cao 1m55 nặng 45-50kg mặc size S vừa xinh chuẩn dáng nha. Bạn bấm vào giỏ hàng góc trái màn hình để nhận voucher giảm 20k ạ!"

### Order Tracking in Direct Messages
> **Customer**: "Shop kiểm tra đơn TikTok 57891234 giúp mình"  
> **AI Agent**: Calls `check_order_status({ "order_id": "57891234" })`  
> **Reply**: "Đơn hàng 57891234 của bạn đang được đơn vị vận chuyển J&T Express giao đến khu vực Quận 1, dự kiến phát trong chiều nay ạ!"

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Authorization failed | Seller region mismatch | Ensure you selected the exact market region where your TikTok Shop is registered. |
| Products show 0 stock | Inactive catalog listing | In TikTok Shop Seller Center, verify the product has passed product review and is live. |
| Token expired | Token rotation desync | Click **Reconnect** in DOS.AI dashboard to re-authenticate with TikTok Shop. |
