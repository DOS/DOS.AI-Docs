# Connectors & Integrations

DOSClaw Connectors allow your AI agents to query live data, perform actions, and integrate business workflows with external platforms—including e-commerce stores, CRMs, social scheduling tools, accounting software, and developer services.

## How Connectors Work

```
User in Chat
     │
     ▼
[AI Agent] ─── Decides to call tool (e.g. search_products)
     │
     ▼
[DOSClaw Model Context Protocol (MCP) Runtime]
     │
     ▼
[Connector Cache Shield] (Singleflight + In-memory TTL Cache)
     │
     ▼
[DOS-Me Vault] (Secure OAuth Tokens & API Credentials)
     │
     ▼
[Third-Party Service API] (Haravan, Shopee, HubSpot, etc.)
```

### Two-Layer Security & Connection Architecture

Following the DOS Platform security model:

1. **Authentication & Token Storage (`DOS-Me`)**:
   OAuth connections, API tokens, and secret credentials are encrypted and stored in `public.provider_connections` and secured in the DOS-Me Vault. Raw secrets are never exposed to browser clients, agent containers, or LLM reasoning prompts.

2. **Agent Binding & Governance (`DOS.AI`)**:
   Individual AI agents are bound to provider connections via `dosai.agent_connection_bindings`. Workspace admins specify:
   - **Enabled Tools**: Selectively turn on or off specific capabilities (e.g. allow `search_products` and `check_order_status`, but disable `create_order`).
   - **Governance Policy**:
     - `autonomous`: Agent executes allowed read and write tools automatically.
     - `approval_required`: Agent prepares action parameters and requests human confirmation before mutating data.
     - `draft_only`: Agent can only read and prepare drafts; never commits live records.

---

## Connector Directory

### E-Commerce & Retail
- [Haravan Omnichannel](haravan.md) — Vietnam retail, inventory, orders, customer sync.
- [Shopee Open Platform](shopee.md) — SEA marketplace product lookup, live shipping, stock.
- [TikTok Shop](tiktokshop.md) — Video commerce, live stream product specs, order status.
- [Shopify Storefront & Admin](shopify.md) — Global e-commerce catalog, cart assistance, order lookup.
- [KiotViet Retail POS](kiotviet.md) — Vietnam retail POS, branch stock, offline-to-online sync.
- [WooCommerce](woocommerce.md) — Open-source WordPress store integration.

### CRM & Marketing
- [HubSpot CRM](hubspot.md) — Inbound contacts, companies, deals, pipelines.
- [Salesforce Enterprise CRM](salesforce.md) — Enterprise accounts, leads, opportunities, cases.
- [Crove Post](crove-post.md) — Multi-platform social scheduler and campaign analytics.
- [NueLink](nuelink.md) — Cross-platform social publishing automation.
- [WordPress Direct CMS](wordpress.md) — Blog post drafting, SEO metadata, media uploads.
- [Meta Ads & Lead Ads](meta.md) — Real-time lead form webhook, instant follow-up, CAPI.

### Finance & Invoicing
- [MISA Enterprise Suite](misa.md) — meInvoice, Inbound E-Invoice, ASP, AMIS Accounting, WeSign, eSign.
- [FinOne / Vbill](finone-vbill.md) — Revenue statistics, reconciliation, QR payment generation.
- [Stripe Payments](stripe.md) — Instant checkout links, subscription verification.

### Productivity & Developer
- [Google Workspace](google-workspace.md) — Google Sheets, Calendar & Meet, Drive documents.
- [GitHub](github.md) — Issue triage, PR summaries, commit history.
- [Notion](notion.md) — Company wiki search, database queries, meeting notes.

---

## Connector Cache Shield

High-traffic bots serving peak concurrent customer queries benefit from the built-in **Connector Cache Shield**:
- **Singleflight Requests**: Prevents thundering herds by de-duplicating simultaneous requests for identical resources (e.g. 50 customers asking about the same flash-sale product).
- **Fast Read Cache**: Safe read operations (product searches, category lists) are cached in memory for sub-10ms response times.
- **Fail-Closed Write Safety**: Write and mutation operations (`create_order`, `refund`) always bypass the cache with strict idempotency keys.
