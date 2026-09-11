# Brave Search Engine

Equip your AI agents with real-time web grounding using the **Brave Search API**. Enables bots to retrieve current news, market data, sports scores, weather updates, and fact-check information that occurred beyond the LLM's training cutoff date.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `web_search` | Read | Perform privacy-first web searches. Returns ranked page titles, snippets, URLs, and publication dates. |
| `news_search` | Read | Search recent news articles with country, freshness, and language filters. |
| `fact_check` | Read | Cross-reference factual claims against authoritative public web sources. |

---

## Why Brave Search?

1. **Independent Index**: Powered by Brave's own web index, not resold from Google or Bing, avoiding vendor lock-in.
2. **Privacy First**: Zero tracking of customer search queries or personal identifiers.
3. **High Speed & Low Latency**: Fast API response times (<250ms) prevent lag during streaming conversational chat.

---

## Setup Guide

### Method 1: Managed by DOS Platform (Default)

For agents on Plus and Pro plans, web grounding is pre-configured and managed by the platform. You do not need to provide an external API key.

1. Open your agent $\rightarrow$ **Tools / Skills**.
2. Toggle **Web Search (Brave Search)** to `ON`.
3. Save changes. Your agent can now search the web autonomously when answering questions about current events.

### Method 2: Custom Brave Search API Key (BYOK)

If you have your own Brave Search developer plan with custom rate limits:

1. Register at [brave.com/search/api/](https://brave.com/search/api/) and copy your **Brave Search API Key**.
2. In the **DOS.AI Dashboard**, go to **Integrations** $\rightarrow$ **Brave Search Engine**.
3. Select **Use Custom API Key** and paste your key.
4. Click **Verify & Save**.

---

## Agent Usage & Grounding Rules

### When Does the Agent Search?

Agents evaluate user prompts using reasoning models:
- **Searches Triggered**: "Thời tiết Hà Nội hôm nay thế nào?", "Tỷ giá USD/VND hiện tại", "Tin tức mới nhất về iPhone 18", "Giá cổ phiếu VinFast hôm nay".
- **Searches Avoided**: General coding, math calculations, roleplay, or questions answered fully by the agent's uploaded knowledge base.

### Anti-Hallucination & Link Verification

DOSClaw enforces strict **No Invented Links** grounding policies:
- The agent is forbidden from fabricating URLs.
- All outbound links included in answers must be derived directly from the `url` returned in `web_search` results.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| Search returns 429 Too Many Requests | Rate limit on custom key reached | Upgrade your Brave Search plan tier or switch to the DOS Platform managed search pool. |
| Outdated search results | Query lacks freshness parameter | Specify time constraints in prompt (e.g. "tin tức trong 24h qua") so the agent passes `freshness: "pd"`. |
