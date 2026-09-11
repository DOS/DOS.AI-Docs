# Productivity & Developer Tools

Connect productivity suites, internal knowledge bases, developer repositories, and real-time search engines to DOSClaw AI agents.

## Supported Tools & Connectors

| Tool / Platform | Type | Key Capabilities |
| --- | --- | --- |
| [Google Workspace](google-workspace.md) | Office Suite | Google Sheets read/append, Calendar scheduling with Google Meet links, Drive search |
| [GitHub](github.md) | Developer Tools | Repository search, issue triage, PR summaries, commit history |
| [Notion](notion.md) | Knowledge Base | Workspace search, internal SOP wiki, database queries |
| [Brave Search Engine](../tools/brave-search.md) | Web Grounding | Real-time web and news search, factual verification, anti-hallucination |

---

## Safe Scopes & Access Control

- **Google Workspace**: Exclusively uses Safe Scopes (`spreadsheets`, `calendar.events`, `drive.file`) to eliminate CASA Tier 2/3 third-party audit requirements while ensuring data privacy.
- **Brave Search**: Independent index with zero query tracking and strict **No Invented Links** anti-hallucination rules.
