# Finance & Invoicing Connectors

Connect accounting ledgers, electronic invoicing systems, and merchant payment suites to DOSClaw AI agents. Provide financial assistants and operations bots with secure, read-only ledger access and in-chat payment links.

## Supported Platforms

| Platform | Type | Key Capabilities |
| --- | --- | --- |
| [MISA Enterprise Suite](misa.md) | Accounting & Invoicing | meInvoice, Inbound E-Invoice, ASP, AMIS Accounting, WeSign, eSign |
| [FinOne / Vbill](finone-vbill.md) | Merchant Payments | Revenue statistics, reconciliation reports, dynamic VietQR links |
| [Stripe Payments](stripe.md) | Global Payments | Instant Checkout links, subscription checks, payment verification |

---

## Strict Read-Only Ledgers

To protect accounting integrity:
- Financial ledgers (MISA, accounting systems) are strictly **read-only**.
- Payments are processed through secure hosted links (Stripe Checkout or dynamic VietQR) so sensitive card numbers or bank credentials never pass through the agent conversation.
