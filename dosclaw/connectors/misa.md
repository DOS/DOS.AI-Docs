# MISA Enterprise Suite

Connect MISA—Vietnam's leading accounting, taxation, and enterprise software ecosystem—to DOSClaw AI agents. Provide financial assistants, operations bots, and account managers with secure, read-only access to electronic invoices, inbound invoices, accounting dictionaries, debt balances, electronic signatures, and document templates.

## Supported MISA Products

DOSClaw provides specialized, read-only connectors for six MISA products:

| Product | Connector Slug | Auth Method | Capabilities |
| --- | --- | --- | --- |
| **MISA meInvoice** | `misa-meinvoice` | App Credentials / Vault | Template lookup, existing invoice status verification |
| **Inbound E-Invoice** | `misa-inbound-einvoice` | Service Credentials / Vault | Inbound invoice search, organization discovery, item details |
| **MISA ASP** | `misa-asp` | Access Code / Vault | Read-only accounting dictionaries (accounts, tax rates) |
| **AMIS Accounting** | `misa-amis-accounting` | Tenant Access Code / Vault | Company information, inventory balances, debt balances |
| **MISA WeSign** | `misa-wesign` | OAuth2 / Vault | Signing templates, document signing status |
| **MISA eSign** | `misa-esign` | Credentials & OTP | Digital certificate metadata, signing transaction status |

---

## Security Model: Read-Only by Design

> [!IMPORTANT]
> To comply with Vietnamese financial regulations and protect enterprise ledgers, all MISA connectors in DOSClaw are **strictly read-only**. AI agents **cannot** modify financial ledgers, issue invoices, sign contracts, or alter tax filings.

- **Encrypted Token Vault**: Credentials, client tokens, and access codes are stored in the DOS-Me Vault (`public.provider_connections`).
- **Fail-Closed Boundary**: If a tenant access code expires or is revoked in MISA, all agent queries fail closed immediately without exposing underlying system details.

---

## Setup Guide

### 1. MISA meInvoice (`misa-meinvoice`)
1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **MISA meInvoice**.
2. Enter your MISA App ID, Tax Code (Mã số thuế), and Service Account credentials.
3. Click **Connect & Verify**.
4. Agent gains tool: `get_invoice_status({ "invoice_number": "...", "symbol": "..." })`.

### 2. Inbound E-Invoice (`misa-inbound-einvoice`)
1. Open **Integrations** $\rightarrow$ **Inbound E-Invoice** (`Hóa đơn điện tử đầu vào`).
2. Provide your MISA MEINVOICEBOT service token.
3. Agent gains tools: `search_inbound_invoices`, `get_inbound_invoice_detail`.

### 3. AMIS Accounting (`misa-amis-accounting`)
1. Open **Integrations** $\rightarrow$ **AMIS Accounting** (`AMIS Kế toán DN`).
2. Enter your company's registered AMIS access code.
3. Agent gains tools:
   - `get_company_info`: Verify company registration and active branches.
   - `get_inventory_balance`: Real-time warehouse balance from accounting books.
   - `get_debt_balance`: Outstanding accounts receivable (AR) and payable (AP) balances.

### 4. MISA WeSign & eSign (`misa-wesign`, `misa-esign`)
1. Connect via MISA WeSign OAuth or provide verified eSign certificate identifiers.
2. Agents gain tools to track whether a contract has been signed by all parties (`get_document_status`).

---

## Example Financial Queries

### Checking Supplier Invoices
> **Accountant**: "Kiểm tra xem hóa đơn đầu vào số 000123 của nhà cung cấp FPT Telecom đã về hệ thống chưa?"  
> **Agent**: Calls `search_inbound_invoices({ "supplier_tax_code": "0101248141", "invoice_number": "000123" })`  
> **Reply**: "Hóa đơn số 000123 ngày 10/08/2026 từ Cty Cổ phần FPT Telecom (Tổng tiền: 2.200.000đ, VAT: 200.000đ) đã được đồng bộ vào hệ thống Inbound Invoice, trạng thái hợp lệ ạ."

### Checking Customer Debt Balance
> **Sales Rep**: "Khách hàng Công ty TNHH Hoàng Gia còn dư nợ bao nhiêu?"  
> **Agent**: Calls `get_debt_balance({ "customer_tax_code": "0312345678" })`  
> **Reply**: "Theo dữ liệu MISA AMIS Kế toán, Công ty TNHH Hoàng Gia hiện có số dư nợ phải thu là 45.000.000đ, trong đó có 1 hóa đơn quá hạn 5 ngày ạ."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| 401 Unauthorized | MISA Access Code revoked | Regenerate the access code in your MISA product management portal and update in DOS.AI. |
| Inbound invoice missing | Tax code desync | Verify that the supplier issued the invoice to the exact Tax Code registered in MISA. |
| Read operation blocked | Permission restriction | Verify that your MISA API account has Read permissions on the requested data dictionary. |
