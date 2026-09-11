# Google Workspace

Connect your Google Workspace account to DOSClaw AI agents. Automate spreadsheet record keeping with **Google Sheets**, schedule meetings with automatic **Google Meet** links via **Google Calendar**, and search and manage documents in **Google Drive**.

## Capabilities & MCP Tools

DOSClaw provides 12 atomic MCP tools adhering to the English `snake_case` naming standard:

| MCP Tool | Operation | Description |
| --- | --- | --- |
| `read_google_sheets` | Read | Read cell values and rows from a spreadsheet range (e.g. `Sheet1!A1:E50`). |
| `append_google_sheets` | Write | Append new rows to the end of a sheet (ideal for capturing leads and customer feedback). |
| `update_google_sheets` | Write | Overwrite or update specific cell ranges. |
| `get_google_sheet_info` | Read | Retrieve spreadsheet metadata, sheet titles, and column dimensions. |
| `list_calendar_events` | Read | Check user/company calendar availability for upcoming meetings. |
| `create_calendar_event` | Write | Schedule a calendar event with attendee emails and an automatic **Google Meet video link**. |
| `update_calendar_event` | Write | Reschedule or modify an existing meeting. |
| `delete_calendar_event` | Write | Cancel a scheduled calendar event. |
| `list_drive_files` | Read | Search files and documents in Drive created by or shared with the application. |
| `get_drive_file` | Read | Fetch document metadata and temporary download links. |

---

## Prerequisites

1. A **Google Account** (Google Workspace enterprise email or personal `@gmail.com`).
2. Permissions to grant OAuth access to third-party applications.

---

## Safe Scopes & Security Boundary

To protect enterprise privacy and ensure data isolation, DOSClaw connects strictly using **Safe Scopes**:
- `https://www.googleapis.com/auth/spreadsheets`: Full read and append capabilities on Google Sheets.
- `https://www.googleapis.com/auth/calendar.events`: Read and create calendar events with Meet video links.
- `https://www.googleapis.com/auth/drive.file`: Per-file access to files created or uploaded by DOSClaw, protecting private Drive folders.

Raw secrets and OAuth refresh tokens are stored exclusively in the DOS-Me Vault (`public.provider_connections`). The AI agent container receives temporary, scoped credentials only at tool execution time.

---

## Setup Guide

### Step 1: Connect via Google OAuth

1. In the **DOS.AI Dashboard**, navigate to **Integrations** $\rightarrow$ **Google Workspace**.
2. Click **Connect Google Account**.
3. Choose your Google Workspace or Gmail account.
4. Review the requested permissions (Google Sheets, Google Calendar Events, and Drive Files).
5. Click **Allow**.
6. The connection will appear as **Connected** in your dashboard.

### Step 2: Bind to Your Agent

1. Open your agent $\rightarrow$ **Integrations** $\rightarrow$ enable **Google Workspace**.
2. Under **Tool Permissions**, select the tools you want to activate (e.g. enable Sheets append and Calendar booking, but keep Drive disabled).
3. If using Google Sheets, provide the default `Spreadsheet ID` (the string between `/d/` and `/edit` in your Google Sheets URL).
4. Save your changes.

---

## Example Agent Automations

### 1. Booking a Discovery Call with Google Meet
> **Customer**: "Tôi muốn đặt lịch tư vấn vào 14h00 chiều thứ Tư tuần này qua Google Meet."  
> **Agent**:
> 1. Calls `list_calendar_events` to verify the team is free at 14:00.
> 2. Calls `create_calendar_event({ "summary": "Tư vấn giải pháp AI - Khách hàng", "start_time": "2026-08-26T14:00:00+07:00", "end_time": "2026-08-26T14:45:00+07:00", "create_meeting_link": true, "attendees": ["khachhang@gmail.com"] })`.
> 3. Replies: "Dạ em đã đặt lịch tư vấn cho anh vào lúc **14:00 - 14:45 thứ Tư ngày 26/08/2026**. Link Google Meet tham gia: `https://meet.google.com/abc-defg-hij`. Lịch hẹn cũng đã được gửi về email của anh ạ!"

### 2. Capturing Inbound Leads to Google Sheets
> When a customer provides their contact info in chat, the agent calls:
> ```json
> append_google_sheets({
>   "spreadsheet_id": "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms",
>   "range": "Leads!A:E",
>   "values": [["2026-08-26 10:15", "Trần Văn Minh", "0912345678", "Gói Doanh nghiệp", "Zalo OA"]]
> })
> ```
> The row is instantly saved into your master Google Sheet.

---

## Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| "Spreadsheet not found or 403 Forbidden" | Sheet not shared with the connected account | Ensure the Google account you authorized in DOS.AI has Editor access to the spreadsheet. |
| Timezone discrepancy in Calendar | Missing UTC offset | Always include timezone offset in ISO 8601 timestamps (e.g. `+07:00` for Indochina Time). |
| Meet link not generated | `create_meeting_link` parameter omitted | Ensure `create_meeting_link: true` is passed to the tool call. |
