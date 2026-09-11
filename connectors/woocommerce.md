# WooCommerce

Connect self-hosted WordPress WooCommerce stores to DOSClaw AI agents. Provide 24/7 store support, inventory lookups, coupon code verification, and order tracking directly via WooCommerce REST API.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Query store products by name, category slug, SKU, or attribute (color, size). |
| `get_product_detail` | Read | Retrieve product prices, description, stock status, gallery images, and variation IDs. |
| `check_order_status` | Read | Look up WooCommerce order status, line items, customer notes, and payment status. |
| `verify_coupon` | Read | Validate promotion coupons, expiration dates, and discount calculations. |

---

## Prerequisites

1. A self-hosted WordPress site with the **WooCommerce** plugin installed and active.
2. An HTTPS domain with a valid SSL certificate (required by WooCommerce REST API).
3. Administrator access to generate WooCommerce REST API keys.

---

## Setup Guide

### Step 1: Generate WooCommerce REST API Keys

1. Log in to your WordPress Admin Dashboard.
2. Go to **WooCommerce** $\rightarrow$ **Settings** $\rightarrow$ **Advanced** $\rightarrow$ **REST API**.
3. Click **Add key**.
4. Set:
   - **Description**: `DOSClaw AI Agent`
   - **User**: Select an admin user account.
   - **Permissions**: `Read` (or `Read/Write` if using order creation tools).
5. Click **Generate API key**.
6. Copy the **Consumer Key** (`ck_...`) and **Consumer Secret** (`cs_...`). Keep this tab open—WooCommerce only displays the secret once.

### Step 2: Connect in DOS.AI Dashboard

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **WooCommerce**.
2. Click **Connect Store**.
3. Enter:
   - **Store URL**: Your website's root URL (e.g. `https://my-store.com`).
   - **Consumer Key**: `ck_...`
   - **Consumer Secret**: `cs_...`
4. Click **Verify & Connect**. DOSClaw makes a lightweight test call to `GET /wp-json/wc/v3/system_status`.

### Step 3: Bind to Your Agent

1. Open your agent $\rightarrow$ **Integrations** $\rightarrow$ enable **WooCommerce**.
2. Save changes.

---

## Multi-Platform Commerce Aggregation

If your business sells on both WooCommerce (website) and an offline POS (like KiotViet):

- DOSClaw supports **Multi-Platform Commerce**: Your agent can query both stores at once.
- Products are qualified with platform prefixes (e.g. `woocommerce:11`, `kiotviet:DH000002`).
- The agent automatically routes customer questions to the appropriate store.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| 401 Unauthorized error | WordPress permalinks set to Plain | WooCommerce REST API requires "Pretty Permalinks". Go to WP Admin $\rightarrow$ Settings $\rightarrow$ Permalinks $\rightarrow$ select **Post name**. |
| SSL / HTTPS error | Self-signed or expired SSL | WooCommerce API requires a valid HTTPS certificate. |
| Cloudflare / WAF blocking API | Cloudflare Bot Fight Mode blocking requests | In Cloudflare WAF, add a custom firewall rule to allow requests with User-Agent `DOSClaw/1.0`. |
