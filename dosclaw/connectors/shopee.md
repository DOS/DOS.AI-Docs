# Shopee Open Platform

Connect your Shopee merchant store to DOSClaw AI agents. Enable automated product recommendations, instant variant availability checks, live shipping status lookups, and customer chat assistance across Southeast Asia.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Search store catalog across Shopee items by keyword, item ID, or category. |
| `get_product_detail` | Read | Retrieve variant specifications, promotional prices, and real-time stock levels. |
| `check_order_status` | Read | Query order status using Shopee Order SN (serial number) to report package tracking and delivery stage. |
| `get_shipping_info` | Read | Retrieve tracking numbers, logistics partner info (Shopee Xpress, SPX, J&T), and estimated arrival. |

---

## Prerequisites

1. An active **Shopee Seller Center** account in Vietnam, Singapore, Malaysia, Philippines, Thailand, or Indonesia.
2. Store login credentials with permissions to authorize third-party applications.

---

## Setup Guide

### Step 1: Authorize Shopee Store

1. In the **DOS.AI Dashboard**, navigate to **Integrations** $\rightarrow$ **Shopee Open Platform**.
2. Click **Connect Shop**.
3. Select your store region (e.g. `Vietnam - VN`, `Singapore - SG`).
4. You will be redirected to the official Shopee Open Platform authorization portal (`partner.shopeemobile.com`).
5. Log in with your Shopee Seller account and review the requested permissions:
   - Item Management & Catalog Read
   - Order Management & Logistics Tracking
6. Click **Confirm Authorization**.
7. Once confirmed, you will be redirected back to DOS.AI, where your `shop_id` is registered and verified.

### Step 2: Bind Shopee to Your AI Agent

1. Navigate to **Agents** $\rightarrow$ select your agent $\rightarrow$ **Integrations**.
2. Enable the **Shopee** connector for this agent.
3. If you run multiple stores (e.g. Shopee + TikTok Shop or Haravan), DOSClaw's **Multi-Platform Commerce Router** allows the agent to search across all stores simultaneously, qualifying item codes as `shopee:<item_id>`.
4. Click **Save**.

---

## Example Agent Interactions

### Product & Stock Inquiries
> **Customer**: "Shop ơi son Black Rouge A12 còn hàng không, có ship hỏa tốc không?"  
> **AI Agent**: Calls `get_product_detail({ "query": "Black Rouge A12" })`  
> **Response**: "Dạ chào bạn, Son kem Black Rouge Air Fit Velvet Tint màu A12 bên shop hiện còn sẵn 45 cây tại kho ạ. Shop có hỗ trợ Shopee Hỏa Tốc (giao trong 2h) bạn nhé! Bạn đặt hàng qua link Shopee của shop tại đây nha: [Link]"

### Order Tracking
> **Customer**: "Kiểm tra đơn Shopee 260815AB1234 giúp mình với"  
> **AI Agent**: Calls `check_order_status({ "order_sn": "260815AB1234" })`  
> **Response**: "Đơn hàng 260815AB1234 của bạn đã được bàn giao cho đơn vị vận chuyển SPX Express vào sáng nay, dự kiến giao vào ngày mai 17/08 ạ!"

---

## Security & Reliability

- **Token Refresh**: Shopee OAuth tokens expire every 4 hours and are automatically refreshed by DOS-Me background workers using the encrypted refresh token.
- **Push Notifications**: Inbound order status changes are synced in real time via Shopee Push API.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Authorization timeout | Browser session expired | Ensure you complete the Shopee Seller login within 3 minutes of opening the OAuth link. |
| Shop ID disconnected | Store permissions changed | Click **Reconnect** in DOS.AI dashboard to issue fresh tokens. |
| Order SN not found | Order belongs to a different shop | Verify that the order was placed on the authorized Shopee store. |
