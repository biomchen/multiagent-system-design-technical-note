# Multi-agent System Design

## Table of Contents

1. [Introduction](#1-what-it-is)
2. [Pipeline Flow](#2-pipeline-execution-flow)
3. [Agent Tiers & Model Config](#3-agent-tiers--model-config)
4. [Hardware](#4-hardware)
5. [Network](#5-network)
6. [Services on Machine D](#6-services-on-machine-d)
7. [Deployment — Step by Step](#7-deployment--step-by-step)
8. [OpenClaw Workstation CLI](#8-openclaw-workstation-cli)
9. [Observability](#9-observability)

# Introduction

A self-hosted multi-agent LLM inference system. A commercial API (Claude Sonnet 4.6) acts as the primary orchestrator; locally hosted open-weight models on three GPU servers handle task execution and critic validation. All coordination, routing, logging, monitoring, and proxying runs as Docker containers on a single EPYC-based control-plane server.

The system exposes a single HTTP API. Clients POST a query; the system decomposes it, executes subtasks in parallel across GPU servers, and returns a validated response.

# Pipeline Execution Flow

```
POST /run {"query": "..."}
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 1 — Planning (~2s)                                │
│ Orchestrator: Claude Sonnet 4.6 (Anthropic API)         │
│ → Outputs a JSON execution plan: list of tasks with     │
│   IDs, descriptions, dependencies, and agent type       │
│ Fallback: if API unreachable → Qwen3.5 27B (Machine B)  │
└───────────────────────────┬─────────────────────────────┘
                            │ task list
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 2 — Execution (~3.5s, parallel via asyncio.gather)│
│ Sub-agent A: Qwen3.5 4B, port 8003                      │
│ Sub-agent B: Qwen3.5 4B, port 8004                      │
│ → Each task polls Redis (0.5s) for dependency results   │
│ → Results written back to Redis keyed by task ID        │
└───────────────────────────┬─────────────────────────────┘
                            │ all sub-agent outputs
                            ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 3 — Synthesis (~11s)                              │
│ Critic: Llama 3.3 70B AWQ port 8001                     │
│ → Receives all outputs, validates, synthesises          │
│ → Returns final structured response                     │
└───────────────────────────┬─────────────────────────────┘
                            │
                     Final response + pipeline_id
```

# Agent Tiers & Model Config

| Agent | Model | Quant | Hardware | Port | Context | Temp |
|---|---|---|---|---|---|---|
| Orchestrator | Claude Sonnet 4.6 | — | Anthropic API | — | 1M | 0.1 |
| Fallback Orch | Qwen3.5 27B | AWQ INT4 | Dell R740xd | 8002 | 32,768 | 0.1 |
| Critic | Llama 3.3 70B | AWQ INT4 | Dell R740 | 8001 | 65,536 | 0.1 |
| Sub-agent A | Qwen3.5 4B | FP16 | Dell R7920 | 8003 | 16,384 | 0.1 |
| Sub-agent B | Qwen3.5 4B | FP16 | Dell R7920 | 8004 | 16,384 | 0.1 |


# Hardware

| Machine | Server | CPU | RAM | GPU | Data IP |
|---|---|---|---|---|---|
| A | Dell R740 | 2× Xeon Platinum 8168 | 256 GB | 2× NVIDIA A6000 48 GB | 192.168.10.10 |
| B | Dell R740xd | 2× Xeon Platinum 8168 | 256 GB | 2× NVIDIA A4000 16 GB | 192.168.10.20 |
| C | Dell R7920 | 2× Xeon Silver 4110 | 256 GB | 2× NVIDIA V100 16 GB PCIe | 192.168.10.30 |
| D | Dell R6515 | 1× AMD EPYC 7713P | 256 GB | — | 192.168.10.40 |


# Network

| Device | Model | Role |
|---|---|---|
| Router / Firewall | Ubiquiti UDM Pro | WAN, inter-VLAN routing, UniFi controller, ACL enforcement |
| Data-plane switch | MikroTik CRS510-8XS-2XQ-IN | 8× SFP28 25 GbE (servers) + 2× QSFP28 100 GbE (future) |
| Management switch | Ubiquiti USW-48-Pro | 48× 1 GbE (management NICs, iDRAC), UniFi managed |

### VLANs

| VLAN | ID | Subnet | Purpose |
|---|---|---|---|
| AI Data Plane | 10 | 192.168.10.0/24 | Inference calls (LiteLLM → vLLM), Prometheus DCGM scrape |
| Management | 20 | 192.168.1.0/24 | SSH, Docker, Grafana, admin workstations |
| iDRAC / IPMI | 30 | 192.168.30.0/24 | Hardware out-of-band management — fully isolated |

### Key firewall rules (UDM Pro)

1. Allow `192.168.10.40` → any port 8001–8004 (Machine D only reaches vLLM)
2. Allow `192.168.10.40` → any port 9400 (Prometheus DCGM scrape)
3. Allow `192.168.1.0/24` → `192.168.10.40` ports 80, 443, 3000, 9090 (admins → services)
4. Deny VLAN 10 → VLAN 10 (servers cannot reach each other directly)
5. Deny VLAN 30 → all (iDRAC fully isolated)

Per-server UFW adds a second enforcement layer: vLLM ports only accept connections from `192.168.10.40`.

# Services on Machine D

All services run via a single `docker compose up -d` on Machine D at `/opt/<your-repo>`.

```
docker-compose.yml
├── redis          :6379   — pipeline state (TTL 1h), task dependency coordination
├── litellm        :4000   — model router: primary=Claude Sonnet, fallback=Qwen3.5 27B
├── pipeline       :8080   — FastAPI: /run, /pipeline/{id}, /logs, /health
├── prometheus     :9090   — scrapes vLLM metrics, DCGM GPU stats, node-exporter
├── grafana        :3000   — dashboards (import IDs: 21374, 1860, 12239)
├── node-exporter  :9100   — host CPU/RAM/disk metrics for Prometheus
├── nginx          :80/:443 — reverse proxy: / → pipeline:8080, /grafana/ → grafana:3000
└── watchdog       (cron)  — Alpine container, checks all 7 endpoints every 2 min,
                             sends email via msmtp on first failure and on recovery
```

# Deployment — Step by Step

### Step 1 — Configure network (do this first)

**On UDM Pro (UI):**
- Settings → Networks → create `AI-Data` (VLAN 10, 192.168.10.0/24, DHCP off), `Management` (VLAN 20, DHCP on), `iDRAC` (VLAN 30, DHCP on, Isolate Network ✓)
- Settings → Firewall → add LAN rules per the table in [Section 5](#5-network)
- Devices → USW-48-Pro → assign port profiles (ports 1–4: VLAN 20, ports 5–8: VLAN 30)

**On MikroTik CRS510 (RouterOS CLI via Winbox or SSH):**
```bash
/interface bridge add name=br0 vlan-filtering=yes
/interface bridge port
  add bridge=br0 interface=sfp28-1 pvid=10   # Machine A
  add bridge=br0 interface=sfp28-2 pvid=10   # Machine B
  add bridge=br0 interface=sfp28-3 pvid=10   # Machine C
  add bridge=br0 interface=sfp28-4 pvid=10   # Machine D
  add bridge=br0 interface=sfp28-5 pvid=1    # UDM Pro uplink
  add bridge=br0 interface=sfp28-6 pvid=1    # USW-48-Pro trunk
/interface bridge vlan
  add bridge=br0 vlan-ids=10 tagged=sfp28-5,sfp28-6 \
      untagged=sfp28-1,sfp28-2,sfp28-3,sfp28-4
/interface vlan add interface=sfp28-6 name=mgmt vlan-id=20
/ip address add address=192.168.1.2/24 interface=mgmt
/ip route add gateway=192.168.1.1
```

**On each server (Netplan)** — set static IPs for both NICs. Example for Machine A:
```yaml
# /etc/netplan/01-static.yaml
network:
  version: 2
  ethernets:
    enp4s0f0:                        # 25G SFP28 → MikroTik
      addresses: [192.168.10.10/24]
      routes:
        - to: 192.168.10.0/24
          via: 192.168.10.1
    eno1:                            # 1G onboard → USW-48-Pro
      addresses: [192.168.1.10/24]
      gateway4: 192.168.1.1
      nameservers: {addresses: [1.1.1.1, 8.8.8.8]}
```
Substitute `.20`/`.30`/`.40` for Machines B/C/D. Then: `sudo netplan apply`

### Step 2 — Base OS setup (all four servers)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim htop tmux build-essential ca-certificates ufw fail2ban

# Default-deny firewall, allow SSH
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable

sudo timedatectl set-timezone UTC
```
### Step 3 — GPU drivers + CUDA (Machines A, B, C only)

```bash
# Add NVIDIA CUDA repo
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb \
  -o /tmp/cuda-keyring.deb
sudo dpkg -i /tmp/cuda-keyring.deb
sudo apt update
sudo apt install -y cuda-drivers nvidia-cuda-toolkit
sudo reboot

Install DCGM Exporter (GPU metrics for Prometheus, port 9400):
```bash
curl -fsSL https://nvidia.github.io/dcgm-exporter/gpg.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/dcgm-exporter.gpg
echo "deb [signed-by=/usr/share/keyrings/dcgm-exporter.gpg] \
  https://nvidia.github.io/dcgm-exporter/ubuntu2204 /" \
  | sudo tee /etc/apt/sources.list.d/dcgm-exporter.list
sudo apt update && sudo apt install -y datacenter-gpu-manager
sudo systemctl enable --now nvidia-dcgm dcgm-exporter
```

Create Python 3.11 venv and install vLLM:
```bash
sudo apt install -y python3.11 python3.11-venv
python3.11 -m venv /opt/vllm/venv
source /opt/vllm/venv/bin/activate
pip install vllm
```

### Step 4 — Download models

Run on each GPU server after activating the vLLM venv:
```bash
hf download model/repo --local-dir model/folder
```

### Step 5 — Deploy vLLM systemd services (Machines A, B, C)

Copy and enable the service files from the repo. Example for Machine A:
```bash
scp systemd/<service-tier>.service user@192.168.1.x:/tmp/
ssh user@192.168.1.x "
  sudo mv /tmp/<service-tier>.service /etc/systemd/system/
  sudo systemctl daemon-reload
  sudo systemctl enable --now <service-tier>
"
```

Configure per-server UFW (Machine A shown — adjust port for B and C):
```bash
sudo ufw allow from 192.168.10.40 to any port 8001 comment 'vLLM — Machine D only'
sudo ufw allow from 192.168.10.40 to any port 9400 comment 'DCGM — Prometheus'
sudo ufw reload
```

### Step 6 — Start control-plane stack (Machine D)

Install Docker:
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu && newgrp docker
```

Clone and configure:
```bash
sudo mkdir -p /opt/<your-repo> && sudo chown ubuntu:ubuntu /opt/<your-repo>
git clone https://github.com/your-org/<your-repo> /opt/<your-repo>
cd /opt/<your-repo>
cp .env.example .env
vim .env   # fill all values (Anthropic key, vLLM URLs, SMTP creds, Grafana password)
sudo mkdir -p /data && sudo chown ubuntu:ubuntu /data   # SQLite + model storage
```

Start all containers:
```bash
docker compose up -d
docker compose logs -f   # watch startup
```

Configure UFW on Machine D:
```bash
sudo ufw allow from 192.168.1.0/24 to any port 80  comment 'Nginx HTTP'
sudo ufw allow from 192.168.1.0/24 to any port 443 comment 'Nginx HTTPS'
sudo ufw allow from 192.168.1.0/24 to any port 3000 comment 'Grafana'
sudo ufw allow from 192.168.1.0/24 to any port 9090 comment 'Prometheus'
sudo ufw reload
```

Enable auto-start on boot:
```bash
sudo tee /etc/systemd/system/<your-repo>.service <<'EOF'
[Unit]
Description=<your-repo>
After=docker.service
Requires=docker.service
[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/repo
ExecStart=docker compose up -d
ExecStop=docker compose down
User=ubuntu
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable <your-repo>
```

### Step 7 — Frontend portal (Machine D)

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

cd /opt/<your-repo>/frontend
cp .env.local.example .env.local
vim .env.local   # confirm vLLM and API IPs match .env
npm install && npm run build

sudo tee /etc/systemd/system/<your-repo>-portal.service <<'EOF'
[Unit]
Description=<your-repo>
After=network.target <your-repo>.service
[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/<your-repo>/frontend
ExecStart=/usr/bin/npm start
Restart=always
RestartSec=5
Environment=NODE_ENV=production
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now <your-repo>-portal
```

Portal runs on port **3001**. Access at `http://192.168.1.x/portal/` (via Nginx) or `http://192.168.1.x:3001` directly.

Add to Nginx by appending to the `server` block in `nginx/nginx.conf`:
```nginx
location /portal/ {
    rewrite ^/portal/(.*) /$1 break;
    proxy_pass http://host-gateway:3001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
}
```
Then `docker compose restart nginx`.

# OpenClaw Workstation CLI

OpenClaw is the workstation-side CLI used to interact with the system from a developer's Mac. It connects to the LiteLLM gateway on Machine D as an OpenAI-compatible client, exposing all model tiers through two shell aliases.

### How it fits in the architecture

```
Workstation (macOS)
  oc-chat  →  http://192.168.1.x:4000/v1   (LiteLLM — direct single-model call)
  oc-run   →  http://192.168.1.x:8080      (Pipeline API — full 3-phase execution)
```

`oc-chat` bypasses the pipeline and talks directly to any model tier through LiteLLM.
`oc-run` sends a query through the full orchestrator → sub-agents → critic flow.

### Model aliases (config.json)

| Alias | LiteLLM model | Actual model | Hardware |
|---|---|---|---|
| `default` | `orchestrator` | Claude Sonnet 4.6 | Anthropic API |
| `local-orch` | `local-orchestrator` | Qwen3.5 27B AWQ | Machine B |
| `powerful` | `critic` | Llama 3.3 70B AWQ | Machine A |
| `fast` | `sub-agent` | Qwen3.5 4B FP16 | Machine C |

### Installation (run once on workstation)

Machine D must be running before setup:

```bash
# From the repo root, with MACHINE_D_IP set if not 192.168.1.x
MACHINE_D_IP=192.168.1.x ./openclaw/setup.sh
```
### Shell aliases added after openclaw installed

```bash
alias oc-chat="openclaw --base-url http://192.168.1.x:4000/v1"
alias oc-run="openclaw --base-url http://192.168.1.x:8080"
export OPENCLAW_MODEL="orchestrator"
export OPENCLAW_TIMEOUT=60
```

### Usage examples

```bash
# Run a query through the full 3-phase pipeline
oc-run "Summarise the key risks of AWQ quantisation for LLM inference"

# Chat directly with a specific model tier
oc-chat --model critic "Review this code for bugs: ..."
oc-chat --model fast "What is the capital of France?"
oc-chat --model local-orch "Break this task into subtasks: ..."
oc-chat --model default "Plan a data pipeline for streaming logs"

# Stream output (default — set in config.json)
oc-chat "Explain tensor parallelism"

# Use powerful critic model by name
openclaw --base-url http://192.168.1.x:4000/v1 --model critic "Validate this reasoning: ..."
```

### Config file reference

```json
{
  "provider": "openai-compatible",
  "baseUrl": "http://192.168.1.x:4000/v1",
  "apiKey": "none",
  "models": {
    "default":    "orchestrator",
    "fast":       "sub-agent",
    "powerful":   "critic",
    "local-orch": "local-orchestrator"
  },
  "timeout": 60,
  "stream": true
}
```

# Observability

**Prometheus scrape targets** (all configured in `prometheus/prometheus.yml`):

| Target | Address | What it exposes |
|---|---|---|
| vLLM Critic | 192.168.10.10:8001 | Request latency, throughput, queue depth |
| vLLM Fallback Orch | 192.168.10.20:8002 | Same |
| vLLM Sub-agent A | 192.168.10.30:8003 | Same |
| vLLM Sub-agent B | 192.168.10.30:8004 | Same |
| DCGM Machine A | 192.168.10.10:9400 | GPU VRAM used, utilisation %, temperature |
| DCGM Machine B | 192.168.10.20:9400 | Same |
| DCGM Machine C | 192.168.10.30:9400 | Same |
| LiteLLM | litellm:4000 | Routing decisions, fallback events |
| Pipeline API | pipeline:8080 | Pipeline duration, error rate |
| Node Exporter | node-exporter:9100 | Machine D CPU, RAM, disk |

**Grafana dashboards** — import by ID at `http://192.168.1.x:3000`:
- `21374` — vLLM performance (TTFT, throughput, KV cache hit rate)
- `12239` — DCGM GPU utilisation, VRAM, temperature per card
- `1860` — Node Exporter (Machine D host resources)

**SQLite inference log** at `/data/inference_logs.db`:
- Stores every model call: model, pipeline_id, prompt tokens, completion tokens, prefill speed (tok/s), generation speed (tok/s), duration, error
- Queryable via `GET /logs?model=critic&limit=100&pipeline_id=<uuid>`
- Visualised in the Next.js portal (Logs page with sortable table and speed chart)

**Watchdog** checks these endpoints every 2 minutes and sends an email on first failure and on recovery:
- `192.168.10.10:8001/health` — Machine A vLLM
- `192.168.10.20:8002/health` — Machine B vLLM
- `192.168.10.30:8003/health` — Machine C Sub-agent A
- `192.168.10.30:8004/health` — Machine C Sub-agent B
- `pipeline:8080/health` — Pipeline API
- `prometheus:9090/-/healthy` — Prometheus
- `grafana:3000/api/health` — Grafana