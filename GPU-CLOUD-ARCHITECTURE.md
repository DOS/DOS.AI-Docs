# GPU Cloud Rental Architecture

## Overview

This document describes the architecture for DOS.ai GPU Cloud rental service, allowing users to rent GPU instances from physical workstations.

## Current Infrastructure

### Hardware
- **GPU 1:** NVIDIA GeForce RTX 5090
- **GPU 2:** NVIDIA RTX PRO 6000 Blackwell Workstation Edition
- **OS:** Windows 11 + WSL2 + Docker Desktop

### Software Stack
- Docker Desktop with WSL2 integration
- NVIDIA Container Toolkit (GPU passthrough to containers)
- Existing vLLM container running

---

## Architecture Design

### High-Level Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   DOS.ai Web    │────▶│   Backend API   │────▶│  Docker Host    │
│   Dashboard     │     │   (Next.js)     │     │  (Workstation)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │                       │                       ▼
        │                       │               ┌───────────────┐
        │                       │               │  Container 1  │──▶ GPU 0 (5090)
        │                       │               │  (User A)     │
        │                       │               └───────────────┘
        │                       │               ┌───────────────┐
        │                       │               │  Container 2  │──▶ GPU 1 (PRO 6000)
        │                       │               │  (User B)     │
        │                       │               └───────────────┘
        │                       │
        ▼                       ▼
┌─────────────────────────────────────────┐
│         Cloudflare Tunnel               │
│   (Expose containers to internet)       │
└─────────────────────────────────────────┘
```

### User Flow

1. User logs into DOS.ai Dashboard
2. User selects GPU type and template (PyTorch, TensorFlow, ComfyUI, etc.)
3. User clicks "Launch Instance"
4. Backend API creates Docker container with specified GPU
5. Cloudflare Tunnel exposes container
6. User receives access URL (SSH/Jupyter/VS Code)
7. User works on their instance
8. User stops instance or billing ends → Container destroyed

---

## Components

### 1. Templates (Docker Images)

Pre-built images for common use cases:

| Template | Base Image | Included | Use Case |
|----------|------------|----------|----------|
| PyTorch | `pytorch/pytorch:2.x-cuda12` | PyTorch, Jupyter | ML Training |
| TensorFlow | `tensorflow/tensorflow:latest-gpu` | TensorFlow, Jupyter | ML Training |
| ComfyUI | Custom | ComfyUI, models | Image Generation |
| vLLM | `vllm/vllm-openai` | vLLM server | LLM Inference |
| Stable Diffusion | Custom | A1111/Forge | Image Generation |
| Base CUDA | `nvidia/cuda:12.x-devel` | CUDA toolkit | Custom Development |

**Template Dockerfile Example (PyTorch + Jupyter + SSH):**

```dockerfile
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-devel

# Install SSH, Jupyter, common tools
RUN apt-get update && apt-get install -y \
    openssh-server \
    git \
    wget \
    vim \
    htop \
    && rm -rf /var/lib/apt/lists/*

# Install JupyterLab
RUN pip install jupyterlab

# Setup SSH
RUN mkdir /var/run/sshd
RUN echo 'root:${SSH_PASSWORD}' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# Expose ports
EXPOSE 22 8888

# Start script
COPY start.sh /start.sh
RUN chmod +x /start.sh
CMD ["/start.sh"]
```

### 2. Backend API

**Endpoints:**

```
POST   /api/gpu/instances          # Create new instance
GET    /api/gpu/instances          # List user's instances
GET    /api/gpu/instances/:id      # Get instance details
DELETE /api/gpu/instances/:id      # Destroy instance
POST   /api/gpu/instances/:id/stop # Stop instance (keep data)
POST   /api/gpu/instances/:id/start # Start stopped instance
GET    /api/gpu/templates          # List available templates
GET    /api/gpu/availability       # Check GPU availability
```

**Create Instance Request:**

```json
{
  "template": "pytorch",
  "gpu": "rtx-5090",
  "storage": 50,
  "access": ["ssh", "jupyter"]
}
```

**Create Instance Response:**

```json
{
  "id": "inst_abc123",
  "status": "running",
  "gpu": "RTX 5090",
  "access": {
    "ssh": "ssh root@inst-abc123.dos.ai -p 22",
    "jupyter": "https://inst-abc123.dos.ai:8888",
    "password": "generated-password"
  },
  "created_at": "2025-01-15T10:00:00Z",
  "billing": {
    "rate": 1.99,
    "unit": "hour"
  }
}
```

### 3. Container Management Service

**Docker commands executed by backend:**

```bash
# Create container with GPU
docker run -d \
  --name inst_abc123 \
  --gpus '"device=0"' \
  -p 10022:22 \
  -p 10088:8888 \
  -v /data/users/abc123:/workspace \
  -e SSH_PASSWORD=generated-password \
  -e JUPYTER_TOKEN=generated-token \
  dos-pytorch:latest

# Stop container
docker stop inst_abc123

# Remove container
docker rm inst_abc123

# Check container status
docker inspect inst_abc123
```

**GPU Assignment Logic:**

```javascript
// Simple assignment - 1 GPU per user
const assignGPU = async () => {
  const gpu0InUse = await checkContainerUsingGPU(0);
  const gpu1InUse = await checkContainerUsingGPU(1);

  if (!gpu0InUse) return { device: 0, name: "RTX 5090" };
  if (!gpu1InUse) return { device: 1, name: "RTX PRO 6000" };

  throw new Error("No GPU available");
};
```

### 4. Network / Tunnel

**Option A: Cloudflare Tunnel (Recommended)**

```bash
# Install cloudflared
# Create tunnel for each instance dynamically

cloudflared tunnel --url http://localhost:10088 --name inst-abc123
```

- Free
- Secure (no open ports)
- Auto HTTPS
- Custom subdomains: `inst-abc123.dos.ai`

**Option B: Direct Port Forwarding**

If you have public IP:
- Map ports directly: `publicIP:10022 → container:22`
- Requires firewall management

**Option C: Tailscale/Zerotier**

- VPN-based access
- More secure
- Requires user to install client

### 5. User Access Methods

| Method | Port | URL Format | Use Case |
|--------|------|------------|----------|
| SSH | 22 | `ssh root@inst-xxx.dos.ai` | Power users, CLI |
| JupyterLab | 8888 | `https://inst-xxx.dos.ai/jupyter` | Data science, ML |
| code-server | 8080 | `https://inst-xxx.dos.ai/vscode` | Development |
| Web Terminal | 7681 | `https://inst-xxx.dos.ai/terminal` | Simple access |

### 6. Storage

**Per-user persistent storage:**

```
/data/users/
  └── user_abc/
      └── workspace/    # Mounted to /workspace in container
```

- Survives container restarts
- Can be snapshotted/backed up
- Charged per GB

---

## Database Schema

```sql
-- Instances table
CREATE TABLE gpu_instances (
  id VARCHAR(50) PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  template VARCHAR(50) NOT NULL,
  gpu_device INT NOT NULL,
  gpu_name VARCHAR(100),
  container_id VARCHAR(100),
  status ENUM('pending', 'running', 'stopped', 'terminated'),
  ssh_port INT,
  jupyter_port INT,
  ssh_password VARCHAR(100),
  jupyter_token VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW(),
  stopped_at TIMESTAMP,
  terminated_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Usage/Billing table
CREATE TABLE gpu_usage (
  id INT AUTO_INCREMENT PRIMARY KEY,
  instance_id VARCHAR(50) NOT NULL,
  user_id VARCHAR(50) NOT NULL,
  gpu_name VARCHAR(100),
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  duration_seconds INT,
  cost_usd DECIMAL(10,4),
  FOREIGN KEY (instance_id) REFERENCES gpu_instances(id)
);

-- Templates table
CREATE TABLE gpu_templates (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(100),
  description TEXT,
  docker_image VARCHAR(200),
  default_ports JSON,
  category VARCHAR(50)
);
```

---

## Pricing Model

| GPU | Hourly Rate | Target Users |
|-----|-------------|--------------|
| RTX 5090 | $1.49/hr | Gaming, inference, light training |
| RTX PRO 6000 | $2.99/hr | Heavy training, professional work |

**Billing Logic:**
- Bill per second, display per hour
- Minimum charge: 1 minute
- Stopped instances: storage only ($0.05/GB/month)

---

## Security Considerations

1. **Container Isolation**
   - Each user gets isolated container
   - No host filesystem access (except /workspace)
   - Resource limits (CPU, RAM)

2. **Network Security**
   - Cloudflare Tunnel (no exposed ports)
   - Generated passwords/tokens
   - HTTPS only

3. **Authentication**
   - DOS.ai login required
   - Instance passwords auto-generated
   - Optional SSH keys

4. **Resource Limits**

```bash
docker run \
  --memory=64g \
  --cpus=16 \
  --gpus '"device=0"' \
  ...
```

---

## Implementation Phases

### Phase 1: MVP
- [ ] Single template (PyTorch + Jupyter)
- [ ] Manual GPU assignment
- [ ] Basic dashboard UI (start/stop)
- [ ] Cloudflare Tunnel setup
- [ ] Simple hourly billing

### Phase 2: Multi-template
- [ ] Multiple templates (TensorFlow, ComfyUI, vLLM)
- [ ] Template selection UI
- [ ] Auto GPU assignment
- [ ] Usage tracking

### Phase 3: Production
- [ ] SSH key support
- [ ] Persistent storage management
- [ ] Snapshot/restore
- [ ] Auto-scaling (multiple workstations)
- [ ] Stripe integration for billing

---

## Tech Stack Summary

| Component | Technology |
|-----------|------------|
| Frontend | Next.js (existing DOS.ai app) |
| Backend API | Next.js API routes or separate Node.js service |
| Container Runtime | Docker Desktop + WSL2 |
| GPU Passthrough | NVIDIA Container Toolkit |
| Tunnel | Cloudflare Tunnel |
| Database | PostgreSQL / Supabase |
| Auth | Existing DOS.ai auth |
| Payments | Stripe |

---

## References

- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Vast.ai](https://vast.ai/) - P2P GPU marketplace
- [RunPod](https://runpod.io/) - GPU cloud
- [Lambda Labs](https://lambdalabs.com/) - GPU cloud

---

## Notes

- Current setup: Windows 11 + Docker Desktop + WSL2
- For production 24/7 operation, consider dual-boot Ubuntu Server
- GPU time-slicing possible for multiple users sharing 1 GPU (reduces performance)
