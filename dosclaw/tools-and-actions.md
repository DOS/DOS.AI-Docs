# Tools, MCP Connectors & Action Invariants

In an autonomous agent system, language models do not merely generate text—they interact with external systems, query live databases, and execute real-world business actions.

---

## The 3-Tier Tool Architecture

While the documentation file `TOOLS.md` has been retired, **Tools (Function Calling)** remain the essential core of DOSClaw. An agent's available tools stem from three distinct layers:

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                 Autonomous AI Agent                                      │
└────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                             │
      ┌──────────────────────────────────────┼──────────────────────────────────────┐
      ▼                                      ▼                                      ▼
┌───────────────────────────┐  ┌───────────────────────────┐  ┌───────────────────────────┐
│ 1. Business MCP Tools     │  │ 2. OpenClaw Core Tools    │  │ 3. Bundled Skills         │
│ Model Context Protocol    │  │ Sandboxed System Tools    │  │ Packaged Extensions       │
├───────────────────────────┤  ├───────────────────────────┤  ├───────────────────────────┤
│ • search_knowledge (RAG)  │  │ • read_file               │  │ • summarize               │
│ • search_memory / remember│  │ • write_file              │  │ • weather                 │
│ • Commerce: create_order  │  │ • exec (allowlist only)   │  │ • tmux                    │
│ • CRM: search_contacts    │  │ • web_search              │  │ • session-logs            │
│ • IoT: reset_speaker      │  │ • image                   │  │                           │
└───────────────────────────┘  └───────────────────────────┘  └───────────────────────────┘
```

### 1. Business MCP (Model Context Protocol) Tools
Injected dynamically via `openclaw.json`. These connect the agent to DOS.AI's managed infrastructure:
- **`search_knowledge`**: Hybrid vector + keyword search over the merchant's live knowledge base (Supabase pgvector), synced automatically with Google Sheets or CSV uploads.
- **Platform Connectors**: E-commerce tools (Shopee, TikTok Shop, Haravan), CRM tools (HubSpot, Salesforce), and Invoicing tools (MISA, FinOne).
- **Custom Hardware/IoT Actions**: Dedicated tools for device control (e.g. `reset_speaker` / `reset_device`).

### 2. OpenClaw Core Runtime Tools
Built into the OpenClaw container runtime to enable general-purpose task automation. DOS.AI enforces strict security policies on these tools:
- **`fs.workspaceOnly: true`**: Filesystem read and write tools are confined strictly to `/workspace` to prevent container escapes.
- **`exec.safeBins` Allowlist**: The shell `exec` tool can only invoke explicitly allowlisted binaries:
  ```json
  ["ls", "cat", "grep", "find", "head", "tail", "wc", "sort", "uniq", "diff", "python3", "node", "npm", "npx", "git", "curl", "jq", "rg", "clawhub", "openclaw"]
  ```
- **Tool Deny List (`tools.deny`)**: Dangerous or redundant capabilities are blocked entirely:
  ```json
  ["gateway", "cron", "sessions_spawn", "sessions_send", "message", "memory_search", "memory_get"]
  ```

### 3. Bundled Skills
Pre-packaged capabilities bundled within the OpenClaw container image (e.g. `summarize`, `weather`) that can be activated selectively.

---

## Action Execution Invariants

A common failure mode in production customer-service agents is **premature claim of success**—where an LLM's RLHF alignment makes it overly eager to please, causing it to state *"I have reset your speaker"* or *"I have placed your order"* without actually invoking the underlying tool.

DOS.AI enforces strict **Action Execution Invariants** across all agent prompts:

### 1. Tool-First Mandate & Wait-For-Result
- **Action Invocations Precede Confirmations**: For any request requiring a state mutation (resetting a device, updating an order, booking an appointment), the agent **must emit the tool call first**.
- **No Premature Confirmation**: The agent is strictly prohibited from claiming an action has succeeded until the tool has finished executing and returned a valid `tool_result`.
- **Direct Negative Constraint**:
  > *"Never report success or acknowledge execution based on assumptions. Only confirm after parsing the actual tool_result."*

### 2. Know vs. Act Demarcation
Agents maintain a clear separation between informational inquiries and state-changing actions:
- **Informational Inquiries (Know)**: Product specifications, store policies, pricing, FAQs → routed strictly through `search_knowledge` (RAG). The agent must never guess or read stale files.
- **Actionable Execution (Act)**: State mutations (resetting hardware, placing orders, transferring calls) → routed directly to the specific action tool. The agent must never invoke `search_knowledge` as an evasive excuse when an action tool exists.

### 3. Anti-Echo Directives
- When a user repeats an identical instruction, quotes an older message, or re-prompts after a delay, language models often fall into an "echo loop"—repeating instructions or quoting previous chat claims.
- The `AGENTS.md` operating manual directs the model:
  - Treat quoted text as raw user intent, not as instructions to be echoed or critiqued.
  - Every action request requires an immediate, fresh tool invocation regardless of conversational history.

### 4. Grounding Plugin Recovery (`dos-grounding`)
The `dos-grounding` OpenClaw plugin operates as an in-process safety interceptor:
- Inspects turn responses before delivery to messaging channels.
- If a deceptive echo pattern is detected (e.g. the model generated success text without emitting the required tool call), the plugin strips the deceptive text and injects a recovery directive forcing the model to issue the actual tool call.

---

## Inspecting Container Tools via Status API

Developers and operators can inspect an agent's active tool policies and runtime health using the Status API `/containers/:name/exec` endpoint:

```bash
# 1. Check system overview and channel connections
curl -s -X POST -H "Authorization: Bearer $STATUS_KEY" -H "Content-Type: application/json" \
  -d '{"command":["openclaw","status"]}' \
  "https://claw-status-lambda.dos.ai/containers/{agent-container-name}/exec"

# 2. Inspect active tool authorization policy (deny list, safeBins)
curl -s -X POST -H "Authorization: Bearer $STATUS_KEY" -H "Content-Type: application/json" \
  -d '{"command":["openclaw","config","get","tools"]}' \
  "https://claw-status-lambda.dos.ai/containers/{agent-container-name}/exec"

# 3. List installed and bundled skills
curl -s -X POST -H "Authorization: Bearer $STATUS_KEY" -H "Content-Type: application/json" \
  -d '{"command":["openclaw","skills","list"]}' \
  "https://claw-status-lambda.dos.ai/containers/{agent-container-name}/exec"
```
