# E-Commerce & Retail Connectors

Connect your online storefronts, marketplaces, and physical retail POS systems to DOSClaw AI agents. Enable agents to search product catalogs, check live multi-branch inventory, track orders, and assist customers with purchases across all chat channels.

## Supported Platforms

| Platform | Channel / Type | Key Capabilities |
| --- | --- | --- |
| [TikTok Shop](tiktokshop.md) | Video & Live Commerce | Live stream inquiries, variant lookup, order status |
| [Shopee Open Platform](shopee.md) | Marketplace (SEA) | Product search, variant stock, live shipping tracking |
| [Haravan Omnichannel](haravan.md) | Omnichannel Retail (Vietnam) | Real-time catalog, multi-warehouse stock, order fulfillment |
| [Shopify](shopify.md) | Global Storefront | Product search, multi-currency pricing, cart assistance |
| [KiotViet Retail POS](kiotviet.md) | Retail POS (Vietnam) | O2O offline-online sync, branch stock, barcode lookup |
| [WooCommerce](woocommerce.md) | Self-Hosted WordPress | Storefront products, order management, coupons |

---

## Multi-Platform Commerce Suite

Many merchants sell across multiple platforms simultaneously (e.g. Shopee + TikTok Shop + physical store using KiotViet). 

DOSClaw provides native **Multi-Platform Commerce Aggregation**:
- **Unified Product Search**: When a customer asks about a product, the agent can query multiple connected stores at once.
- **Platform-Qualified Identifiers**: Items are tagged with platform prefixes (e.g. `shopee:12048512`, `kiotviet:SP00142`) so stock checks and orders route to the correct platform automatically.
- **Connector Cache Shield**: High-concurrency read operations are cached with singleflight de-duplication to prevent rate limit exhaustion during flash sales.
