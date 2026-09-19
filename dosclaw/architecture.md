# DOSClaw Agent Architecture

DOSClaw is DOS.AI's autonomous agent runtime platform. It provisions, orchestrates, and monitors containerized AI agents that operate 24/7 across multiple communication channels—including Telegram, Zalo Official Account (OA), Facebook Messenger, Discord, Slack, WhatsApp, and Lark.

---

## High-Level Architecture

DOSClaw adopts a **standalone container per user agent** model. Unlike monolithic or shared-process agent systems, each agent runs inside an isolated Docker container powered by the **OpenClaw** multi-agent OS.

```
                    ┌────────────────────────────────────────────────────────┐
                    │                  Go API Gateway (Cloud Run)            │
                    │      Routing · Metering · Billing · Lifecycle CRUD     │
                    └───────────────────────────┬────────────────────────────┘
                                                │
                 Cloudflare Zero Trust Tunnel ──┼── Bearer-Authed Control Plane
                                                ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Docker Host Fleet (Mumbai Lambda / GCP Singapore / Local Canary)                                  │
│                                                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────┐  │
│   │ Node.js Status API (:18090)                                                                │  │
│   │ Container CRUD · Volume Uploads · Metrics · Exec Allowlist                                 │  │
│   └───────────────────────────────────────────┬────────────────────────────────────────────────┘  │
│                                               │                                                   │
│                 ┌─────────────────────────────┴─────────────────────────────┐                     │
│                 ▼                                                           ▼                     │
│   ┌───────────────────────────┐                               ┌───────────────────────────┐       │
│   │ Agent Container (Em Huyền)│                               │ Agent Container (Shop A)  │       │
│   │ ├── OpenClaw Runtime      │                               │ ├── OpenClaw Runtime      │       │
│   │ ├── /workspace (Volume)   │                               │ ├── /workspace (Volume)   │       │
│   │ ├── Local loopback ws/http│                               │ ├── Local loopback ws/http│       │
│   │ └── dos-grounding plugin  │                               │ └── dos-grounding plugin  │       │
│   └───────────────────────────┘                               └───────────────────────────┘       │
│                                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Pillars

1. **Strict Container Isolation**:
   - Every agent has dedicated memory, persistent storage, channel connections, and local tools.
   - Resource ceilings: standard agents run with 1 vCPU and 1.5 GB RAM, with Node heap locked to 3.5 GB (`NODE_OPTIONS="--max-old-space-size=3584"`).
2. **Direct Status API Management**:
   - The Go API Gateway communicates with remote Docker hosts through a dedicated, authenticated **Status API**.
   - No complex intermediary cluster managers (no HiClaw Manager, Matrix sync servers, or Higress dependencies).
3. **Multi-Backend Routing (`BackendRouter`)**:
   - The API Gateway transparently routes operations to the target host based on the agent's assigned backend (`mumbai` for production fleet, `gcp-sea` for Singapore capacity, `local` for beta/canary smoke tests).

---

## Container Storage & Volume Lifecycle

Each agent's state is preserved using Docker named volumes and a deterministic initialization script:

```
Go Gateway (Provisioning)
   │
   ├─► Staged Templates (/tmp/init read-only mount)
   │     ├── openclaw.json
   │     ├── SOUL.md
   │     ├── IDENTITY.md
   │     ├── AGENTS.md
   │     ├── entrypoint.sh
   │     └── patches/zalouser-vendor.mjs
   │
   └─► Agent Workspace Volume (agent-{id}-data mounted to /workspace)
         │
         ▼
     entrypoint.sh (On Boot)
         ├── Overwrites: openclaw.json, entrypoint.sh (enforces security & model updates)
         ├── Preserves: SOUL.md, IDENTITY.md (via cp -n, preserving user customizations)
         └── Manages: AGENTS.md (refreshes platform rules unless customized by owner)
```

### Storage Semantics

- **/workspace is Persistent**: The Docker volume persists across restarts, upgrades, and host maintenance. Any files created during runtime (session logs, downloaded media, scratch files) remain intact.
- **Offline Updates & Safe Rollout**: When the API Gateway stages configuration updates (e.g. model migrations or channel tokens), the next container restart applies platform configs while preserving owner modifications to personality files.

---

## Health Monitoring & Lifecycle Automation

An autonomous background monitor in the API Gateway (`api-monitor` service) continuously tracks the fleet:

- **Health Probing**: Every 60 seconds, pings all active Status API endpoints and probes container health.
- **Crash-Loop Detection**: Identifies containers whose process restarts repeatedly without reaching `healthy` state, alerting operations via Telegram.
- **Idle Sleep (Opt-in)**: Suspends containers idle for >24 hours to conserve server resources, waking them instantaneously upon inbound webhook traffic.
- **Self-Healing**: Automatically restarts containers that encounter transient Docker socket glitches.
