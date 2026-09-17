# Developer Tools & IDE Integration

DOS AI is designed for seamless compatibility with popular developer tools, coding agent IDE extensions, and UI chat clients. Thanks to support for **OpenRouter-compatible `/v1/key`**, **`usage.cost` per completion**, and **OpenAI-compatible endpoints**, these tools can automatically track balances, render token pricing, and display request costs in real time.

---

## 1. Cline (VS Code Extension)

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

## 2. Roo Code (VS Code Extension)

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

## 3. OpenWebUI

[OpenWebUI](https://openwebui.com/) is a feature-rich, self-hosted AI interface.

### Configuration Steps

1. Go to **Admin Panel** -> **Settings** -> **Connections**.
2. Under **OpenAI API**, enable the connection:
   * **API Base URL:** `https://api.dos.ai/v1`
   * **API Key:** `dos_sk_...`
3. Click the **Verify Connection (reload)** icon. OpenWebUI will fetch the full model list from `GET /v1/models`.
4. Click **Save**.

### Supported Features
* All available models on DOS AI immediately appear in your model dropdown.
* Context window lengths and token limits are automatically configured from `context_length`.
* Token pricing (`pricing.prompt`, `pricing.completion`) is displayed inside model info.

---

## 4. Cherry Studio & Chatbox

Desktop GUI clients for interacting with LLM models.

### Configuration Steps

1. Open **Settings** -> **Providers** -> Add **OpenAI** (or **Custom**).
2. Set:
   * **API Host / Base URL:** `https://api.dos.ai` (or `https://api.dos.ai/v1`)
   * **API Key:** `dos_sk_...`
3. Click **Fetch Models** to automatically populate models.
4. Test with a quick greeting prompt.

---

## 5. CC Switch

[CC Switch](https://github.com/nicejoy/cc-switch) is a model switcher and proxy manager for Claude Code and AI tools.

### Configuration Steps

In CC Switch, open **Providers** -> Add or Edit **DOS.AI**:

1. **Base URL:** `https://api.dos.ai` (or `https://api.dos.ai/v1`)
2. **API Key:** `dos_sk_...`
3. **Usage Query Configuration**:
   * **Preset:** Select `General` (or `DeepSeek`)
   * **Endpoint:** `{{baseUrl}}/v1/user/balance` (or `{{baseUrl}}/user/balance`)
   * **Extractor Script** (if using General preset):
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
4. Click **Test Query**. CC Switch will display your live balance in USD.
