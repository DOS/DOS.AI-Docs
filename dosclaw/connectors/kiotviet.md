# KiotViet Retail POS

Connect KiotViet—Vietnam's leading retail point-of-sale (POS) and inventory management software—to DOSClaw AI agents. Enable unified online-to-offline (O2O) stock management, branch inventory inquiries, price list checks, and offline order tracking.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Search product catalog by barcode, SKU, or name across POS inventory. |
| `get_branch_stock` | Read | Query real-time inventory levels broken down by specific retail branches/stores. |
| `get_product_prices` | Read | Look up retail prices, wholesale tiers, and active promotional discounts. |
| `check_order_status` | Read | Look up customer invoices and delivery tickets by KiotViet invoice code (e.g. `HD000123`). |

---

## Prerequisites

1. An active **KiotViet** account (Retail, F&B, or Pharmacy edition).
2. KiotViet Open API access enabled for your retailer (`https://developer.kiotviet.vn/`).
3. **Retailer Name**, **Client ID**, and **Client Secret** issued by KiotViet.

---

## Setup Guide

### Step 1: Retrieve KiotViet API Credentials

1. Log in to your KiotViet management portal (`https://manage.kiotviet.vn/`).
2. Go to **Thiết lập** (Settings) $\rightarrow$ **Thiết lập cửa hàng** $\rightarrow$ **Thiết lập tính năng**.
3. Under **Ứng dụng & Tích hợp**, click **Tạo ứng dụng kết nối** (Create connection app) or register on KiotViet Developer Portal.
4. Note your:
   - **Retailer (Tên gian hàng)**: The identifier in your store URL (e.g. `cuahangthoitrang`).
   - **Client ID**: Public application client key.
   - **Client Secret**: Private secret key.

### Step 2: Connect in DOS.AI Dashboard

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **KiotViet Retail POS**.
2. Click **Connect KiotViet**.
3. Fill in:
   - **Retailer Name**
   - **Client ID**
   - **Client Secret**
4. Click **Test & Connect**. DOS.AI verifies connectivity with KiotViet's OAuth Token endpoint (`https://id.kiotviet.vn/connect/token`).

### Step 3: Bind to Your AI Agent

1. Open your agent $\rightarrow$ **Integrations** $\rightarrow$ enable **KiotViet**.
2. (Optional) Set the **Default Branch ID** if your bot represents a specific store location.
3. Save changes.

---

## Example O2O Inquiries

### Branch Stock Check
> **Customer**: "Shop xem giúp em chi nhánh Cầu Giấy còn mẫu giày sneaker trắng size 38 không ạ?"  
> **AI Agent**: Calls `get_branch_stock({ "sku": "GI-SNK-TR-38", "branch": "Cầu Giấy" })`  
> **Response**: "Dạ chào bạn, chi nhánh Cầu Giấy (125 Cầu Giấy) hiện còn 3 đôi size 38 ạ! Bạn có muốn shop giữ hàng trước để bạn qua thử không ạ?"

### Price & Barcode Search
> **Customer**: "Quét mã vạch 8935001234567 xem giá bao nhiêu?"  
> **AI Agent**: Calls `search_products({ "barcode": "8935001234567" })`  
> **Response**: "Mã vạch 8935001234567 là sản phẩm **Nước hoa vùng kín Foellie Eau de Bijou 5ml**, giá niêm yết là 210.000đ ạ."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Invalid retailer or credentials" | Typos in Client ID/Secret | Copy the exact credentials from KiotViet settings without leading/trailing spaces. |
| Stock numbers out of date | KiotViet inventory sync delay | KiotViet stock caches for up to 60 seconds; verify direct in KiotViet POS if recent transaction occurred. |
| Inactive branch error | Branch ID closed or archived | Verify in KiotViet admin that the branch is marked as active. |
