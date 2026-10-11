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
│ Docker Host Fleet (GCP Singapore VM dos / Beta VM dosclaw-beta)                                    │
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
   - The API Gateway transparently routes operations to the target host based on the agent's assigned backend (`gcp-sea` for the primary Singapore production fleet, `local` for the beta fleet on `dosclaw-beta`, with `mumbai` retired).

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

### Pre-Stop Safety & Fail-Closed Volume Cleanup

- **Preflight Image Pull & Backup**: When recreating or upgrading an agent container, the gateway pulls the target Docker image and completes volume backups before stopping the existing running container (`recreateContainerReleasing`). If image resolution, denylist validation, or backup fails, the existing container remains running without disruption.
- **Fallback Backup Re-evaluation**: If target image pull fails and falls back to a non-denied default image, the gateway re-evaluates whether a backup is required against the final fallback image before container stop.
- **Fail-Closed Volume Cleanup**: If container volume creation succeeds but subsequent template staging, image resolution, or image pull fails, the gateway immediately removes the allocated Docker volume (`RemoveVolume`), eliminating orphaned volume leaks across Docker hosts.

---

## Health Monitoring & Lifecycle Automation

An autonomous background monitor in the API Gateway (`api-monitor` service) continuously tracks the fleet:

- **Health Probing**: Every 60 seconds, pings all active Status API endpoints and probes container health.
- **Crash-Loop Detection**: Identifies containers whose process restarts repeatedly without reaching `healthy` state, alerting operations via Telegram.
- **Idle Sleep (Opt-in)**: Suspends containers idle for >24 hours to conserve server resources, waking them instantaneously upon inbound webhook traffic.
- **Self-Healing**: Automatically restarts containers that encounter transient Docker socket glitches.

---

## Runtime Lifecycle, Version Floors & Denylist Governance

DOSClaw maintains strict version governance to prevent agents from running on broken or vulnerable upstream runtime releases:

1. **Version Floor Enforcement**:
   - The API Gateway checks the agent's target image against an active minimum version floor. Requests to provision or upgrade below the floor are rejected fail-closed.
2. **Emergency Version Denylist (`OPENCLAW_DENIED_VERSIONS`)**:
   - Environment-level denylist parsed at gateway startup and hot-reload.
   - Any runtime image matching a denied version pattern is rejected at admission time.
   - **Mutable `:latest` Invariant**: When any version is actively denylisted (`hasDeniedVersions()`), the mutable upstream tag `ghcr.io/openclaw/openclaw:latest` is strictly rejected and excluded from automatic fallbacks to ensure unverified upstream rebuilds cannot bypass the denylist.
3. **OpenClaw 2026.9+ Trusted-Proxy & Device Pairing**:
   - Container entrypoint dynamically probes runtime capabilities using `openclaw config validate`.
   - On 2026.9+, configures `gateway.auth.mode = "trusted-proxy"` with `deviceAutoApprove.enabled = true` and `operator.*` scopes, omitting `auth.token` to satisfy mutual exclusivity rules.
   - Binds RFC1918 CIDRs (`127.0.0.1`, `::1`, `172.16.0.0/12`, `10.0.0.0/8`, `192.168.0.0/16`) for seamless console-proxy and Status API connectivity across Docker bridge network reassignment.
   - Enables loopback CLI password authentication (`OPENCLAW_GATEWAY_PASSWORD`) for in-container channel watchdog and status inspection.
