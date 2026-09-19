# Notion Knowledge Base

Connect your Notion workspace to DOSClaw AI agents. Enable agents to search internal SOPs, product documentation, team wikis, and databases as a dynamic Retrieval-Augmented Generation (RAG) knowledge base, and capture meeting notes into Notion databases.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `search_notion_pages` | Read | Search Notion workspace pages and documents by semantic query or keywords. |
| `get_page_content` | Read | Fetch page blocks, Markdown content, nested toggles, and callouts. |
| `query_database` | Read | Query Notion databases with filters, sorting, and property values (select, tags, dates). |
| `create_database_page` | Write | Add a new record/row to a Notion database (e.g. log customer feedback or meeting minutes). |

---

## Prerequisites

1. An active **Notion workspace** (Free, Plus, or Enterprise).
2. Permission to add connections or install Notion Integrations in your workspace.

---

## Setup Guide

### Step 1: Connect via Notion OAuth

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **Notion Knowledge Base**.
2. Click **Connect with Notion**.
3. You will be redirected to Notion's authorization screen (`api.notion.com/v1/oauth/authorize`).
4. Select the specific Notion pages or workspace sections you want to share with the AI agent.
5. Click **Allow Access**.
6. The connection is registered with encrypted authorization tokens stored in the DOS-Me Vault.

### Step 2: Bind to Your Agent

1. Open your agent $\rightarrow$ **Integrations** $\rightarrow$ enable **Notion**.
2. Under **Knowledge Base Source**, you can choose to:
   - Use Notion as an **On-Demand MCP Tool** (agent searches Notion when prompted).
   - Ingest Notion pages into the agent's **Vector Knowledge Base** (pre-indexed for instant RAG responses).
3. Save changes.

---

## Example Knowledge Base Queries

### Querying Company SOPs
> **Employee in Slack**: "Chính sách nghỉ phép năm của công ty áp dụng như thế nào đối với nhân viên thử việc?"  
> **Agent**: Calls `search_notion_pages({ "query": "chính sách nghỉ phép nhân viên thử việc" })`  
> **Reply**: "Theo mục 3.2 trong tài liệu **Sổ tay Nhân sự 2026** trên Notion: Nhân viên trong thời gian thử việc chưa được hưởng ngày nghỉ phép năm có lương. Tuy nhiên, sau khi ký hợp đồng chính thức, thời gian thử việc sẽ được tính cộng dồn vào thâm niên để tính số ngày phép trong năm ạ."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Page not found in search | Page not shared with integration | In Notion, open the page $\rightarrow$ click `...` top right $\rightarrow$ **Connections** $\rightarrow$ add **DOSClaw AI**. |
| Database query returns empty | Database permissions missing | Ensure the parent page containing the database is shared with the integration. |
| Token expired error | Workspace re-authenticated | Click **Reconnect** in DOS.AI dashboard to update the integration token. |
