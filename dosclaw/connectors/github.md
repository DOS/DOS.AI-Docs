# GitHub Developer Connector

Connect GitHub repositories to DOSClaw AI agents. Empower engineering assistants and DevOps bots to triage issues, summarize pull requests, inspect commit history, and answer code repository questions.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `list_issues` | Read | Search and list repository issues by state (`open`, `closed`), label, or assignee. |
| `get_issue_detail` | Read | Fetch issue descriptions, comments, author info, and discussion threads. |
| `create_issue` | Write | Open a new GitHub issue with title, Markdown body, labels, and assignees. |
| `get_pull_request` | Read | Retrieve PR diffs, changed files, review comments, and mergeability status. |
| `list_commits` | Read | Inspect recent commit history on a specific branch. |
| `get_release_info` | Read | Fetch latest release tags, changelogs, and release assets. |

---

## Prerequisites

1. A GitHub account with access to the target repository (public or private).
2. A **GitHub Personal Access Token (Classic or Fine-Grained)** or GitHub App installation.

---

## Setup Guide

### Step 1: Generate a GitHub Access Token

1. Go to your GitHub profile $\rightarrow$ **Settings** $\rightarrow$ **Developer settings** $\rightarrow$ **Personal access tokens**.
2. Generate a fine-grained token with access to your desired repository:
   - **Repository permissions**:
     - `Issues`: Read & Write
     - `Pull requests`: Read-only
     - `Contents`: Read-only
3. Copy the token (`github_pat_...` or `ghp_...`).

### Step 2: Connect in DOS.AI Dashboard

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **GitHub Developer Connector**.
2. Click **Connect GitHub**.
3. Paste your **Personal Access Token** and default repository (e.g. `owner/repo-name`).
4. Click **Verify & Save**. DOSClaw validates token validity against `https://api.github.com/user`.

### Step 3: Bind to Your Agent

1. Open your developer or tech lead agent $\rightarrow$ **Integrations** $\rightarrow$ enable **GitHub**.
2. Select your governance mode (`approval_required` for creating issues or `autonomous` for triaging).
3. Save changes.

---

## Example DevOps Scenarios

### Automated Bug Triage in Slack/Discord
When a user reports a bug in a developer community chat:
> **Developer**: "@DevBot issue #142 đang bị lỗi gì vậy?"  
> **Agent**: Calls `get_issue_detail({ "repo": "DOS/DOS.AI", "issue_number": 142 })`  
> **Reply**: "Issue #142 báo lỗi `TypeError: Cannot read properties of undefined (reading 'session')` tại file `auth-redirect.ts`. Lỗi này đã được đánh nhãn `bug` và assign cho @lead-dev. Hiện PR #145 đang fix và chờ review ạ."

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| 404 Not Found | Token lacks access to private repo | Ensure your fine-grained token includes repository permissions for that specific private repo. |
| 403 Rate limit exceeded | Unauthenticated rate limit or token exhausted | Verify token is active; authenticated requests have a 5,000 req/hour limit on GitHub API. |
| Cannot create issue | Token missing `issues:write` scope | Regenerate token with Read and Write permissions for Issues. |
