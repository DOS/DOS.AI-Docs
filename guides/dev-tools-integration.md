# Developer Tools & IDE Integration

DOS AI is designed for seamless compatibility with popular developer tools, coding agent IDE extensions, and UI chat clients. Thanks to support for **Anthropic Messages API**, **OpenAI-compatible endpoints**, **OpenRouter-compatible `/v1/key`**, and **real-time per-completion token cost reporting (`usage.cost`)**, tools can easily track balances, render token pricing, and optimize prompt context.

---

## 1. Claude Code (Anthropic CLI Agent)

[Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview) is Anthropic's agentic command-line tool. DOS AI natively supports the Anthropic Messages API (`/v1/messages`), including **Anthropic Tool Search (Deferred Tools)**.

### Configuration Steps

Set the following environment variables in your terminal:

**macOS / Linux:**
```bash
export ANTHROPIC_BASE_URL="https://api.dos.ai"
export ANTHROPIC_API_KEY="dos_sk_your_api_key_here"
export ENABLE_TOOL_SEARCH="true"
```

**Windows (PowerShell):**
```powershell
$env:ANTHROPIC_BASE_URL = "https://api.dos.ai"
$env:ANTHROPIC_API_KEY = "dos_sk_your_api_key_here"
$env:ENABLE_TOOL_SEARCH = "true"
```

### ⚡ Context Optimization with Tool Search (~83% Token Reduction)

When running Claude Code with multiple MCP (Model Context Protocol) servers (e.g. GitHub, Supabase, Postgres, filesystem tools), tool schemas can easily explode to **300K – 450K tokens**, causing massive context bloat and high API bills.

When `ENABLE_TOOL_SEARCH="true"` is enabled:
1. **Client-side Deferred Tools**: Claude Code withholds heavy MCP tool schemas locally (350K+ tokens) and only sends a compact `search_tools` function.
2. **Gateway Server-Side Loop**: DOS AI Gateway intercepts search requests, performs local regex ranking, dynamically expands only matching schemas (`expandDiscoveredTools`), and executes the continuation turn in a single consolidated billing call.
3. **Measured Outcome**: Context consumption drops from ~425K tokens down to **68.4K tokens** (**~83% token reduction** on initial turns).

---

## 2. CC Switch

[CC Switch](https://github.com/nicejoy/cc-switch) is a model switcher and proxy manager for Claude Code and developer AI agents.

### Configuration Steps

In CC Switch, navigate to **Providers** -> Add or Edit **DOS.AI**:

1. **Base URL:** `https://api.dos.ai` (or `https://api.dos.ai/v1`)
2. **API Key:** `dos_sk_...`
3. **Tool Search:** Toggle **Enable Tool Search** ON to activate the 83% context reduction feature.
4. **Usage Query Configuration**:
   * **Preset:** Select `General` (or `DeepSeek`)
   * **Endpoint:** `{{baseUrl}}/v1/user/balance` (or `{{baseUrl}}/user/balance`)
   * **Extractor Script**:
     ```javascript
     ({
       request: {
         url: "{{baseUrl}}/user/balance",
         method: "GET",
         headers: {
           "Authorization": "Bearer {{apiKey}}",
           "User-Agent": "cc-switch/1.0"
         }
       },
       extractor: function(response) {
         return {
           isValid: response.is_active,
           invalidMessage: response.invalid_message,
           remaining: response.balance,
           total: response.total_deposited,
           used: response.total_spent,
           unit: response.currency || "USD"
         };
       }
     })
     ```
5. Click **Test Query** to view your real-time balance.

---

## 3. Cline (VS Code Extension)

[Cline](https://github.com/cline/cline) (formerly Claude Dev) is an autonomous coding agent extension for VS Code.

### Configuration Steps

1. In VS Code, open the **Cline** sidebar and click the **Settings (gear)** icon.
2. Under **API Provider**, select **OpenRouter** or **OpenAI Compatible**.
3. Configure the following fields:
   * **Base URL:** `https://api.dos.ai/v1`
   * **API Key:** `dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
   * **Model ID:** Enter your preferred model ID (e.g. `deepseek/deepseek-chat`, `qwen/qwen-2.5-coder-32b-instruct`, or `dos-ai`).
4. Click **Done**.

### Supported Features in Cline
* **Automatic Key Validation & Balance:** Cline calls `GET /v1/key` to verify your key and displays your remaining credit balance at the bottom of the chat panel.
* **Per-Turn Cost Tracking:** Cline reads the `cost` field in `usage` to calculate and display the exact cost of each tool call and coding iteration.

---

## 4. Roo Code (VS Code Extension)

[Roo Code](https://github.com/RooVetGit/Roo-Code) is an agentic coding assistant fork of Cline with advanced multi-mode support.

### Configuration Steps

1. Open Roo Code settings in VS Code.
2. Set **Provider** to **OpenRouter** or **OpenAI Compatible**.
3. Enter settings:
   * **Base URL:** `https://api.dos.ai/v1`
   * **API Key:** `dos_sk_...`
   * **Model:** Choose your desired model ID (e.g., `deepseek/deepseek-chat`).
4. Save settings. Roo Code will automatically test connection using `/v1/models` and `/v1/key`.

---

## 5. OpenWebUI

[OpenWebUI](https://openwebui.com/) is a self-hosted AI interface for web and mobile.

### Configuration Steps

1. Go to **Admin Panel** -> **Settings** -> **Connections**.
2. Under **OpenAI API**, enable the connection:
   * **API Base URL:** `https://api.dos.ai/v1`
   * **API Key:** `dos_sk_...`
3. Click the **Verify Connection (reload)** icon. OpenWebUI will fetch the full model list from `GET /v1/models`.
4. Click **Save**.

---

## 6. Cursor & Aider

For AI-first code editors like **Cursor** and CLI tools like **Aider**:

* **Cursor:** Under Settings -> Models -> OpenAI API Key, enter your `dos_sk_...` key and set Override OpenAI Base URL to `https://api.dos.ai/v1`.
* **Aider:** Run with:
  ```bash
  export OPENAI_API_BASE="https://api.dos.ai/v1"
  export OPENAI_API_KEY="dos_sk_..."
  aider --model deepseek/deepseek-chat
  ```

---

## 7. Context Safety & Error Recovery

DOS AI protects developer coding agents with built-in safeguards:

* **Preflight Guard (1.05x Ceiling)**: If an agent builds up an excessive context payload exceeding the target model's maximum context window, DOS AI intercepts the request before calling upstream providers and immediately returns HTTP 413 `context_length_exceeded`. This prevents upstream provider timeouts and eliminates charges for doomed requests.
* **Normalized Error Reporting**: Error codes are cleanly normalized to `context_length_exceeded`, allowing agents like Cline, Roo Code, and Claude Code to trigger active history compaction or prune prior tool outputs instead of entering endless retry loops.
