# NueLink Social Scheduler

Connect NueLink—a multi-platform social media management and automation tool—to DOSClaw AI agents. Schedule, auto-publish, and queue viral marketing content across Facebook, Instagram, Twitter/X, LinkedIn, TikTok, YouTube Shorts, and Pinterest.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `list_collections` | Read | Fetch active NueLink collections (content queues, topical categories, and campaigns). |
| `list_channels` | Read | Retrieve connected social profiles and platform destinations. |
| `create_post` | Write | Draft or schedule a social media post with captions, hashtags, links, and image/video URLs. |
| `get_post_status` | Read | Check publishing status, scheduled time, and distribution progress. |

---

## Prerequisites

1. An active account on **NueLink** (`https://nuelink.com/`).
2. A generated **NueLink API Key** from your NueLink account settings.

---

## Setup Guide

### Step 1: Obtain Your NueLink API Key

1. Log in to your NueLink dashboard.
2. Go to **Settings** $\rightarrow$ **API & Integrations**.
3. Click **Generate New API Key**.
4. Copy the API Key securely.

### Step 2: Connect in DOS.AI Dashboard

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **NueLink Social Scheduler**.
2. Click **Connect NueLink**.
3. Paste your **API Key** into the credential field.
4. Click **Verify & Connect**. DOS.AI verifies your key by querying `GET /api/v1/user`.

### Step 3: Bind to Your Marketing Agent

1. In Agent Settings $\rightarrow$ **Integrations**, toggle **NueLink**.
2. Select default destination collections (e.g. "Weekly Tips", "Product Launches").
3. Set your approval governance (`approval_required` or `autonomous`).
4. Click **Save**.

---

## Example Marketing Workflow

Agents create cross-platform queues without manual formatting:
> **User**: "Tạo 3 bài đăng về xu hướng AI trong thương mại điện tử và lên lịch đăng vào 9h sáng thứ Hai, thứ Tư, thứ Sáu tuần tới trên Fanpage và LinkedIn."  
> **Agent**: Prepares 3 tailored variations matching LinkedIn and Facebook formatting, calls `create_post` with the target timestamps, and returns confirmation receipts.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| 401 Unauthorized | API Key revoked or invalid | Regenerate your API key in NueLink Settings and update in DOS.AI. |
| Collection not found | Collection ID deleted or archived | Refresh collections list in the agent channel settings. |
| Media upload failed | Remote URL not publicly accessible | Ensure media URLs are public HTTPS links with valid Content-Type headers. |
