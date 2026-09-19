# Crove Post

Connect **Crove Post** (`post.crove.com`)—DOS's first-party social publishing and marketing platform—to your AI agents. Empower marketing agents to discover social channels, upload media assets, draft campaign posts, schedule publications across platforms, and inspect engagement analytics with review-gated approval workflows.

## Capabilities & MCP Tools

DOSClaw exposes normalized, English `snake_case` MCP tools for Crove Post:

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `list_channels` | Read | List active social channels (Facebook, X/Twitter, LinkedIn, Instagram, TikTok, YouTube, Pinterest). |
| `list_channel_groups` | Read | Discover organization channel groups and brand profiles. |
| `list_social_posts` | Read | Query drafts, scheduled posts, and historical published content with status filters. |
| `get_social_post_analytics` | Read | Retrieve engagement statistics (impressions, clicks, likes, shares, comments) for published posts. |
| `get_channel_metrics` | Read | Aggregated audience growth and channel performance metrics. |
| `create_social_post_draft` | Write | Draft social posts with text copy, media attachments, first comments, and thread replies. |
| `schedule_social_posts` | Write | Schedule approved posts to specific date/time slots across multiple channels. |
| `upload_marketing_media_from_url` | Write | Ingest remote image or video URLs into Crove Post's CDN for social attachments. |

---

## Prerequisites

1. An account on **Crove Post** (`https://post.crove.com`).
2. Connected social profiles on Crove Post (e.g. your Facebook Page, LinkedIn Company, X handle, or Instagram Business account).

---

## Setup Guide

### Step 1: Connect via 1-Click OAuth

1. Open the **DOS.AI Dashboard** (`https://app.dos.ai/`).
2. Navigate to **Integrations** $\rightarrow$ **Crove Post**.
3. Click **Connect Crove Post**.
4. Authorize access. DOSClaw pairs your DOS organization directly with your Crove Post workspace.
5. All credentials and `pos_` access tokens are securely vaulted in DOS-Me.

### Step 2: Bind to Marketing Agents

1. In Agent Settings, select your Marketing or Content Creator agent.
2. Under **Integrations**, toggle **Crove Post**.
3. Set your **Governance Policy**:
   - **Review Gated (Default & Recommended)**: The agent creates drafts and submits them for human review. Posts are not published live without explicit approval.
   - **Autonomous**: The agent is authorized to schedule and publish approved content directly to connected channels.
4. Click **Save**.

---

## Human Approval & Review Workflow

To protect brand reputation, DOSClaw enforces a strict approval gate for social media:

1. **Content Generation**: The AI agent drafts a series of posts based on a marketing prompt or calendar topic.
2. **Draft Submission**: The agent calls `create_social_post_draft` and provides a review preview card in chat.
3. **Operator Verification**: The business owner or marketing lead reviews the copy, hashtags, and media preview.
4. **Approval & Dispatch**: Once approved (via the dashboard button or chat confirmation), Crove Post queues and schedules the post to the designated social networks.

---

## Analytics & Performance Monitoring

Marketing agents can also inspect post performance:
> **User**: "Tuần trước bài viết về AI Agent trên LinkedIn và Facebook tiếp cận được bao nhiêu người?"  
> **Agent**: Calls `get_channel_metrics` and `get_social_post_analytics`  
> **Response**: "Tuần trước, 3 bài đăng trên LinkedIn và Facebook Page đã đạt tổng cộng 14.850 lượt hiển thị (impressions), 842 lượt tương tác (engagements) và 126 lượt click vào link landing page ạ!"

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "No active channels found" | Channels not connected in Crove Post | Open `post.crove.com`, navigate to Channels, and connect your social accounts first. |
| Post rejected by social network | Media format or aspect ratio invalid | Verify video length and image dimensions meet individual platform rules (e.g. Instagram Reels vs X images). |
| Approval link expired | One-time action ticket timeout | Generate a new draft by asking the agent to refresh the campaign. |
