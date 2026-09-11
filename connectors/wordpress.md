# WordPress Direct CMS

Connect your WordPress sites (both self-hosted WordPress 5.6+ and WordPress.com / Jetpack sites) directly to DOSClaw AI agents. Enable automated blogging, SEO content creation, article revision management, category tagging, and featured media uploads.

## Capabilities & MCP Tools

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `list_posts` | Read | Query published, draft, and scheduled WordPress articles by status, category, or author. |
| `get_post_detail` | Read | Fetch raw HTML/Gutenberg post content, excerpt, SEO metadata, and slug. |
| `create_post_draft` | Write | Create a new WordPress draft article with title, content, categories, tags, and excerpt. |
| `update_post` | Write | Update existing post content, title, status (`draft`, `pending`, `publish`), or featured image. |
| `upload_media` | Write | Upload image attachments directly to the WordPress Media Library (`wp-content/uploads`). |
| `list_categories` | Read | Retrieve taxonomy categories and tags to organize generated articles. |

---

## Prerequisites

### For Self-Hosted WordPress
- WordPress 5.6 or higher.
- HTTPS enabled on your website.
- Pretty Permalinks enabled (Settings $\rightarrow$ Permalinks $\rightarrow$ `Post name`).
- Application Passwords enabled (standard feature in WordPress 5.6+).

### For WordPress.com / Jetpack Sites
- A WordPress.com account with admin access to the target site.

---

## Setup Guide

### Method 1: Self-Hosted WordPress (Secure Application Password Flow)

1. In the **DOS.AI Dashboard**, open **Integrations** $\rightarrow$ **WordPress Direct CMS**.
2. Click **Connect WordPress Site**.
3. Select **Self-Hosted WordPress**.
4. Enter your site URL (e.g. `https://myblog.com`).
5. DOSClaw redirects you to your site's native authorization screen:
   ```
   https://myblog.com/wp-admin/authorize-application.php?app_name=DOSClaw+AI
   ```
6. Log in to your WordPress admin account and click **Yes, I approve this connection**.
7. WordPress mints a dedicated Application Password and redirects back to DOS-Me to encrypt and store it in Vault.
8. You never have to manually copy-paste passwords.

### Method 2: WordPress.com / Jetpack (OAuth)

1. Select **WordPress.com / Jetpack** in the connection dialog.
2. Click **Authorize with WordPress.com**.
3. Log in with your WordPress.com account and select your site from the list.
4. Approve the requested content creation scopes.

---

## Human Approval & Editorial Governance

Content creation workflows adhere to strict editorial policies:

- **Draft Only (Recommended)**: The agent creates posts strictly with `status: "draft"` or `status: "pending"`. Human editors review, polish, and publish from WordPress admin.
- **Direct Publish**: If authorized, the agent can publish immediately or schedule posts for specific publication times.

---

## Example Blogging Prompt
> **User**: "Viết một bài viết chuẩn SEO 1200 từ về 'Top 5 công cụ AI tự động hóa bán hàng năm 2026', chèn các thẻ H2/H3 hợp lý, chọn chuyên mục 'Kiến thức AI' và lưu vào bản nháp trên website WordPress."  
> **Agent**: 
> 1. Calls `list_categories` to find the ID for 'Kiến thức AI'.
> 2. Generates the structured HTML content with proper semantic tags.
> 3. Calls `create_post_draft({ "title": "Top 5 Công Cụ AI...", "content": "...", "categories": [14], "status": "draft" })`.
> 4. Returns the draft URL (`/wp-admin/post.php?post=452&action=edit`) for one-click editorial review.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "REST API disabled" | Security plugin blocking `/wp-json` | In Wordfence, iThemes, or Cloudflare, allow REST API requests to `/wp-json/wp/v2/*`. |
| Application Passwords not showing | SSL disabled or plugin conflict | Ensure your site uses `https://`. Some basic auth plugins may disable Application Passwords. |
| Media upload HTTP 413 error | Server `upload_max_filesize` exceeded | Increase `client_max_body_size` in Nginx or `upload_max_filesize` in `php.ini`. |
