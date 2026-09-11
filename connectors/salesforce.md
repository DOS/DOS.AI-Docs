# Salesforce Enterprise CRM

Connect Salesforce Enterprise CRM to DOSClaw AI agents. Empower sales, service, and operations teams with automated Lead ingestion, Account and Contact lookups, Opportunity tracking, and Service Cloud case creation.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_leads` | Read | Search Salesforce Leads by email, company name, status, or lead source. |
| `get_account_detail` | Read | Retrieve complete Salesforce Account records, billing addresses, and parent accounts. |
| `search_opportunities` | Read | Query active Opportunities, stage names, probability percentages, and close dates. |
| `get_case_status` | Read | Look up customer support cases in Service Cloud by Case Number or subject. |
| `create_case` | Write | Open a new support ticket in Service Cloud when an issue requires escalation. |

---

## Prerequisites

1. A **Salesforce Enterprise, Unlimited, or Developer Edition** org.
2. An administrative account with permissions to install Connected Apps (`Manage Connected Apps`).

---

## Setup Guide

### Step 1: Authorize via Salesforce OAuth

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **Salesforce Enterprise CRM**.
2. Click **Connect Salesforce**.
3. Choose your instance type:
   - **Production / Developer Edition**: `login.salesforce.com`
   - **Sandbox / Test Environment**: `test.salesforce.com`
4. Log in with your Salesforce credentials and click **Allow** to grant access scopes:
   - `api` (Access and manage your data)
   - `refresh_token, offline_access` (Perform requests at any time)
5. Upon callback, DOS-Me securely stores your org URL and vaulted OAuth refresh credentials.

### Step 2: Bind to Agent & Select Governance

1. In Agent Settings $\rightarrow$ **Integrations**, toggle **Salesforce**.
2. Set the governance mode:
   - `read_only`: Agent can search records and cases but cannot create or modify data.
   - `approval_required`: Creating cases or updating lead statuses requires human staff approval.
   - `autonomous`: Agent creates triage cases directly in Service Cloud.

---

## Example Enterprise Scenario

### VIP Support Escalation
> **Customer**: "Our production server is throwing timeout errors on our API integration. Case #002910."  
> **AI Agent**: Calls `get_case_status({ "case_number": "002910" })`  
> **Reply**: "I've pulled up Case #002910 (Priority: High). Technical Account Manager David Chen is currently assigned. I have escalated this directly to the Tier 3 on-call queue and attached your latest error logs."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "OAUTH_APPROVAL_ERROR_GENERIC" | Connected app blocked in Salesforce Org | In Salesforce Setup $\rightarrow$ Connected Apps OAuth Usage, ensure DOSClaw app is set to **Admin approved users are pre-authorized** or unblocked. |
| Sandbox vs Production domain mismatch | Selected wrong login endpoint | Re-connect and choose Sandbox if testing on a `.sandbox.my.salesforce.com` domain. |
| API Limit Exceeded | Daily Salesforce REST API calls exhausted | DOSClaw implements smart caching; review API allocations in Salesforce System Overview. |
