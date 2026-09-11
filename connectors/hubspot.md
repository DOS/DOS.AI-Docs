# HubSpot CRM

Connect your HubSpot CRM account to DOSClaw AI agents. Enable customer service, sales development (SDR), and account management agents to search contacts, look up company profiles, check active deal stages, and inspect pipeline health in real time.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_contacts` | Read | Search CRM contacts by email, phone, or name. Returns contact properties, lifecycle stage, and lead status. |
| `get_contact_detail` | Read | Retrieve complete contact record including owner ID, associated company, and custom properties. |
| `search_companies` | Read | Query companies by domain or name. Returns company revenue, industry, and phone. |
| `search_deals` | Read | Search active and closed deals by name, pipeline, or associated contact/company. |
| `get_pipeline_stages` | Read | Fetch deal pipelines, stages, probability percentages, and stage IDs. |

---

## Prerequisites

1. An active **HubSpot account** (Free, Starter, Professional, or Enterprise).
2. Super Admin or App Marketplace installation permissions in your HubSpot portal.

---

## Setup Guide

### Step 1: Connect via HubSpot OAuth

1. In the **DOS.AI Dashboard**, navigate to **Integrations** $\rightarrow$ **HubSpot CRM**.
2. Click **Connect with HubSpot**.
3. You will be redirected to HubSpot's official authorization screen (`app.hubspot.com`).
4. Select your target HubSpot Account / Portal.
5. Review the requested CRM scopes:
   - `crm.objects.contacts.read`: Search and read contacts.
   - `crm.objects.companies.read`: Search companies.
   - `crm.objects.deals.read`: Look up deals and pipelines.
   - `crm.schemas.contacts.read`: Retrieve custom property mappings.
6. Click **Connect app**.
7. DOS-Me securely encrypts and vaults the generated OAuth tokens.

### Step 2: Bind to Your Agent

1. Open your agent $\rightarrow$ **Integrations** $\rightarrow$ enable **HubSpot CRM**.
2. Configure permissions:
   - **Allowed Objects**: Toggle Contacts, Companies, and Deals.
3. Click **Save Changes**.

---

## Example Conversational Workflows

### Contact & Lead Verification
When a customer interacts with your agent on WhatsApp, Telegram, or Web Chat:
> **Customer**: "Chào bạn, mình là Hoàng từ công ty ABC Logistics (email: hoang@abclogistics.vn). Dự án trước báo giá đến đâu rồi?"  
> **AI Agent**: Calls `search_contacts({ "email": "hoang@abclogistics.vn" })` followed by `search_deals({ "associated_contact_id": "189421" })`  
> **Response**: "Dạ chào anh Hoàng! Em thấy trên hệ thống hợp đồng triển khai giải pháp kho vận của ABC Logistics đang ở giai đoạn **Thương thảo hợp đồng** do chuyên viên Nguyễn Văn Nam phụ trách. Anh Nam đã gửi bản phụ lục hôm qua qua email của anh. Anh có cần em kết nối trực tiếp với anh Nam không ạ?"

---

## Security & Architecture

- **No Third-Party Brokers**: Unlike generic integration wrappers, DOS.AI connects directly to HubSpot's official API (`api.hubapi.com`). No third-party data broker touches your customer records.
- **Zero-Secret Containers**: Agents execute tools through the DOSClaw Gateway runtime. Access tokens are dynamically injected per request and never stored in agent disk memory or LLM contexts.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| 403 Forbidden on Deals | Missing deal scopes | In HubSpot, verify your user account has access to the Sales Hub and Deals object. |
| Contact search returns no records | Strict query formatting | Search by exact email or international phone format (e.g. `+84901234567`). |
| Token refresh failure | Portal uninstalled app | Re-click **Connect with HubSpot** in the dashboard to re-authorize. |
