# Shopify Storefront & Admin

Connect your Shopify store to DOSClaw AI agents. Provide high-converting product recommendations, live inventory checks, cart assistance, and order status lookups across global markets.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_products` | Read | Search Shopify store collections, products, tags, and vendors by natural language. |
| `get_product_detail` | Read | Fetch variant options, inventory levels across multi-location warehouses, and currency pricing. |
| `check_order_status` | Read | Look up customer orders by order number (e.g. `#1001`) or customer email. Returns line items and tracking URLs. |
| `create_draft_order` | Write | Create draft orders with customized discounts, shipping addresses, and send checkout invoices. |

---

## Prerequisites

1. An active **Shopify Store** (Basic, Shopify, Advanced, or Shopify Plus).
2. Store owner or staff account with permissions to install and configure apps.

---

## Setup Guide

### Method 1: Shopify App OAuth (Recommended)

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **Shopify**.
2. Click **Connect Store**.
3. Enter your `.myshopify.com` store domain (e.g. `your-brand.myshopify.com`).
4. You will be redirected to the Shopify App installation screen.
5. Review the requested access scopes:
   - `read_products`, `read_inventory`: Real-time stock and catalog lookup.
   - `read_orders`, `read_fulfillments`: Order tracking and delivery milestones.
   - `write_draft_orders`: (Optional) Allow agent to assemble customer checkout links.
6. Click **Install App**.

### Method 2: Custom App (Private Access Token)

If your enterprise uses a custom private app:

1. In Shopify Admin, go to **Settings** $\rightarrow$ **Apps and sales channels** $\rightarrow$ **Develop apps**.
2. Create a custom app named "DOSClaw Assistant".
3. Configure the Admin API access scopes listed above.
4. Click **Install app** and reveal the **Admin API access token** (`shpat_...`).
5. In DOS.AI Dashboard, select **Custom App Credentials** and paste your Shopify Domain and Access Token.

---

## Example AI Agent Scenarios

### Multilingual Global Sales Assistant
> **Customer (US)**: "Do you have waterproof hiking boots in men's size 10?"  
> **AI Agent**: Queries `search_products({ "query": "waterproof hiking boots", "tag": "men" })`  
> **Reply**: "Yes! We have the **Apex Trail Pro Waterproof Boots** in Men's Size 10 (US), priced at \$149.00 USD. We currently have 7 pairs remaining at our California fulfillment center. Would you like me to send you the direct checkout link?"

### Self-Serve Order Tracking
> **Customer**: "Where is my order #1084?"  
> **AI Agent**: Calls `check_order_status({ "order_number": "1084" })`  
> **Reply**: "Your order #1084 was shipped via DHL Express on August 14. Here is your tracking number: `940011189956` ([Track on DHL](https://dhl.com)). Current status: **Out for delivery**."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Invalid myshopify domain" | Custom domain used instead of primary store domain | Enter your original `.myshopify.com` URL (found in Shopify Settings $\rightarrow$ Domains). |
| Rate limit 429 error | High API call volume | DOSClaw implements leaky-bucket throttling and singleflight cache to stay within Shopify API quotas. |
| Missing line item images | Product media not published to sales channel | In Shopify Admin, ensure product media is enabled for the Storefront sales channel. |
