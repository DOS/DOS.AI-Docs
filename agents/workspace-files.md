# Agent Workspace Files & Prompt Contract

In DOSClaw, each agent workspace operates under a strict **Separation of Concerns** across its Markdown prompt and configuration files. This ensures high predictability, eliminates contradictory instructions, and maintains reliable tool-calling behavior on large and small language models alike.

---

## Workspace Files Overview

```
/workspace/
├── openclaw.json       # Runtime configuration (models, channels, tool permissions)
├── SOUL.md             # Persona, voice, tone, and empathy boundaries ONLY (< 1KB)
├── IDENTITY.md         # Entity identity (name, role, avatar, business affiliation)
└── AGENTS.md           # Single rulebook for operating instructions and ## Tools
```

| File | Primary Responsibility | Critical Rule |
| :--- | :--- | :--- |
| **`SOUL.md`** | **Persona & Tone ONLY**<br>Defines who the agent is, how it speaks, formality, empathy, and pronouns. | **Keep under 1KB.** Never include operational procedures, tool lists, or RAG guidance here. Bloated personas suppress tool-calling instincts on smaller or quantized LLMs (e.g. Qwen 3.6). |
| **`IDENTITY.md`** | **Entity Identity**<br>Defines the agent's display name, role, creature type, and business branding. | Updated dynamically when an operator renames the bot or customizes identity in the dashboard. |
| **`AGENTS.md`** | **Single Rulebook & Operating Instructions**<br>Operating guidelines, safety boundaries, and local tool conventions under `## Tools`. | **Injected into 100% of sessions.** Even when a merchant configures a `custom_soul`, the platform's `AGENTS.md` remains active to enforce tool safety and RAG rules. |
| **`openclaw.json`** | **Runtime Configuration**<br>Declares LLM providers, model mappings, channel bindings, and tool authorization policies. | Managed and updated atomically by the DOS.AI Gateway. Overwritten safely on container restarts. |

---

## The Retirement of `TOOLS.md`

In earlier versions of the OpenClaw ecosystem, environment notes and tool usage guidelines were placed in a separate `/workspace/TOOLS.md` file.

### Upstream OpenClaw Evolution
- **Consolidation into `AGENTS.md`**: Upstream OpenClaw officially **retired `TOOLS.md`**. Separate prompt files were found to fragment model attention and cause conflicting behavior.
- **Single Source of Truth**: All local tool instructions, usage conventions, and environment variables are now consolidated into the **`## Tools` section of `AGENTS.md`**.
- **Automated Migration (`openclaw doctor --fix`)**:
  OpenClaw includes a repair utility that automatically archives legacy `TOOLS.md` files, merges custom tool notes into `AGENTS.md`, and removes the retired file:
  ```bash
  openclaw doctor --fix
  ```
- **DOS.AI Provisioning Policy**: The DOS.AI Go Gateway never provisions or generates `TOOLS.md`. All tool orchestration instructions are rendered directly into `AGENTS.md`.

> **Note on Existing Containers:**  
> Because an agent's `/workspace` is mounted from a persistent Docker volume (`agent-{id}-data`), containers provisioned under earlier versions of OpenClaw may still have a legacy `TOOLS.md` file on disk. The file is ignored by modern OpenClaw runtimes.

---

## Customizing Agent Personas (`custom_soul`)

Merchants and developers can customize their agent's personality through the DOS.AI Dashboard:

1. **Pure Voice Customization**:
   When you provide a custom personality, it replaces only `SOUL.md`. Your custom prompt dictates the tone (e.g. "You are an energetic TikTok live seller" or "You are a courteous financial assistant").
2. **Preserved Platform Guardrails**:
   Your custom agent still receives the full `AGENTS.md` rulebook automatically. This guarantees that:
   - The agent retains the ability to call `search_knowledge` for verified FAQ answers.
   - The agent retains connected e-commerce or CRM tools.
   - The agent adheres to anti-echo directives and tool-first action invariants.
