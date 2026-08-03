# 🐳 Homelab Docker Boilerplates

![Docker Compose](https://img.shields.io/badge/Docker%20Compose-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)
![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS%20%7C%20Windows-blue)
![Tested on Ubuntu](https://img.shields.io/badge/Tested%20on-Ubuntu%2024.04-E95420?logo=ubuntu&logoColor=white)
![GitHub release](https://img.shields.io/github/v/release/100dollarguy/homelab-docker-boilerplates)
![GitHub last commit](https://img.shields.io/github/last-commit/100dollarguy/homelab-docker-boilerplates)

Production-ready, self-hosted Docker Compose stacks. Each stack is fully isolated, environment-variable driven, and deployable with a single command.

Tested on **Ubuntu 24.04**. Compatible with any Linux distro, macOS, and Windows running Docker Desktop.

---

## 📦 Stacks

| Stack | Category | Host Port(s) |
|---|---|---|
| [Nginx Proxy Manager](#-nginx-proxy-manager) | Reverse Proxy | 80, 81 (admin), 443 |
| [Authentik](#-authentik) | Identity / SSO | 9100 |
| [Pi-hole + Unbound](#-pi-hole--unbound) | DNS | 8080 (UI), 53 (DNS) |
| [Prometheus Stack](#-prometheus-stack) | Monitoring | 9090, 3000, 9093, 9115 |
| [Uptime Kuma](#-uptime-kuma) | Uptime Monitoring | 3001 |
| [n8n](#-n8n) | Automation | 5678 |
| [Homepage](#-homepage) | Dashboard | 3005 |
| [Portainer](#-portainer) | Container Management | 9000 |

---

## ⚙️ Prerequisites

### 1. Install Docker & Docker Compose

**Linux:**
```bash
curl -fsSL https://get.docker.com | sh

# Allow running Docker without sudo (log out and back in after this)
sudo usermod -aG docker $USER
```

**macOS:** Download [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)

**Windows:** Download [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)

> Docker Compose v2 is bundled with Docker Desktop and Docker Engine 20.10+. Use `docker compose` (not `docker-compose`).

---

### 2. Create the shared proxy network

Run this **once** on your host before deploying any stack. All stacks (except Pi-hole) join this network to be proxied through Nginx Proxy Manager.

```bash
docker network create proxy_net
```

> This command is identical on Linux, macOS, and Windows.

---

## 🚀 How to Deploy Any Stack

**Linux / macOS:**
```bash
# 1. Go to the stack folder
cd <category>/<stack>

# 2. Copy the example env file
cp .env.example .env

# 3. Edit and fill in your values (replace all "changeme" entries)
nano .env        # or: vim .env / code .env / any text editor

# 4. Start the stack
docker compose up -d

# 5. Verify
docker compose ps
docker compose logs -f
```

**Windows (PowerShell):**
```powershell
# 1. Go to the stack folder
cd <category>\<stack>

# 2. Copy the example env file
Copy-Item .env.example .env

# 3. Edit and fill in your values
notepad .env     # or: code .env / any text editor

# 4. Start the stack
docker compose up -d

# 5. Verify
docker compose ps
docker compose logs -f
```

---

## 📋 Recommended Deploy Order

Deploy in this order to avoid dependency issues:

```
1. proxy/nginx-proxy-manager     ← must be first, creates proxy_net
2. identity/authentik            ← SSO (optional, deploy only if you need it)
3. dns/pihole-unbound            ← DNS resolver
4. monitoring/prometheus-stack   ← metrics + dashboards
5. monitoring/uptime-kuma        ← uptime checks
6. automation/n8n                ← workflows
7. dashboard/homepage            ← start page
8. management/portainer          ← container manager
```

---

## 🔐 Secrets Reference

All values marked `changeme` **must** be changed before deploying. Never use the defaults in production.

| Stack | Variable | Notes |
|---|---|---|
| Authentik | `PG_PASS` | Use a strong password |
| Authentik | `AUTHENTIK_SECRET_KEY` | Generate — see below |
| Pi-hole | `WEBPASSWORD` | Use a strong password |
| Prometheus Stack | `GRAFANA_PASS` | Use a strong password |
| n8n | `N8N_ENCRYPTION_KEY` | Generate — see below |

**Generating secrets:**

Linux / macOS:
```bash
# Authentik secret key
openssl rand -base64 36

# n8n encryption key
openssl rand -hex 32
```

Windows (PowerShell):
```powershell
# Authentik secret key
[Convert]::ToBase64String((1..36 | ForEach-Object { [byte](Get-Random -Max 256) }))

# n8n encryption key
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Max 256) })
```

> Alternatively use any password manager's random generator or an offline secret generator tool.

---

## 📁 Stack Details

### 🔀 Nginx Proxy Manager

**Path:** `proxy/nginx-proxy-manager/`

Reverse proxy with a web UI for managing proxy hosts and SSL certificates (Let's Encrypt). This stack **creates** the `proxy_net` Docker network that all other stacks join.

**Default credentials on first login:**
- Email: `admin@example.com`
- Password: `changeme`

Change these immediately after first login.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `IMAGE` | `jc21/nginx-proxy-manager:latest` | Docker image |
| `CONTAINER_NAME` | `nginx-proxy-manager` | Container name |
| `HTTP_PORT` | `80` | HTTP port |
| `ADMIN_PORT` | `81` | Admin UI port |
| `HTTPS_PORT` | `443` | HTTPS port |
| `NETWORK` | `proxy_net` | Network name — must match all other stacks |

```bash
# Linux / macOS
cd proxy/nginx-proxy-manager
cp .env.example .env
docker compose up -d
```
```powershell
# Windows
cd proxy\nginx-proxy-manager
Copy-Item .env.example .env
docker compose up -d
```

---

### 🔒 Authentik

**Path:** `identity/authentik/`

SSO and identity provider. Runs 4 containers: PostgreSQL, Redis, server, and worker. The server and worker wait for PostgreSQL to be fully healthy before starting — no manual restarts needed on cold deploy.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `POSTGRES_IMAGE` | `postgres:16-alpine` | PostgreSQL image |
| `REDIS_IMAGE` | `redis:7-alpine` | Redis image |
| `AUTHENTIK_IMAGE` | `ghcr.io/goauthentik/server:2024.12.2` | Authentik image |
| `PG_DB` | `authentik` | Database name |
| `PG_USER` | `authentik` | Database user |
| `PG_PASS` | **changeme** | Database password |
| `AUTHENTIK_SECRET_KEY` | **changeme** | Secret key — generate with command below |
| `PORT` | `9100` | Authentik UI host port |
| `PROXY_NETWORK` | `proxy_net` | Must match NPM's network |

```bash
# Linux / macOS
cd identity/authentik
cp .env.example .env
openssl rand -base64 36    # copy output → paste as AUTHENTIK_SECRET_KEY in .env
nano .env
docker compose up -d
```
```powershell
# Windows
cd identity\authentik
Copy-Item .env.example .env
# Generate secret (see Secrets Reference above) → paste as AUTHENTIK_SECRET_KEY
notepad .env
docker compose up -d
```

> First boot takes ~60 seconds while PostgreSQL initialises. Monitor with `docker compose logs -f server`.

---

### 🌐 Pi-hole + Unbound

**Path:** `dns/pihole-unbound/`

Network-wide ad blocker (Pi-hole) with a recursive DNS resolver (Unbound). Pi-hole forwards all DNS queries to Unbound on port `5335`. Do not change `UNBOUND_PORT` unless you supply a custom Unbound config.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `PIHOLE_IMAGE` | `pihole/pihole:latest` | Pi-hole image |
| `UNBOUND_IMAGE` | `mvance/unbound:latest` | Unbound image |
| `PIHOLE_CONTAINER` | `pihole` | Pi-hole container name |
| `UNBOUND_CONTAINER` | `unbound` | Unbound container name |
| `UNBOUND_PORT` | `5335` | Unbound listen port — do not change |
| `HTTP_PORT` | `8080` | Pi-hole admin UI host port |
| `DNS_PORT` | `53` | DNS port exposed on host |
| `TIMEZONE` | `Asia/Kolkata` | Your timezone (e.g. `Europe/London`, `America/New_York`) |
| `WEBPASSWORD` | **changeme** | Pi-hole admin password |
| `NETWORK` | `dns_net` | Isolated internal network |

```bash
# Linux / macOS
cd dns/pihole-unbound
cp .env.example .env
nano .env    # set WEBPASSWORD and TIMEZONE
docker compose up -d
```
```powershell
# Windows
cd dns\pihole-unbound
Copy-Item .env.example .env
notepad .env    # set WEBPASSWORD and TIMEZONE
docker compose up -d
```

> After deploying, point your router's DNS server to your host IP to enable network-wide ad blocking.

> Pi-hole runs on its own isolated `dns_net` network. It does not need to be behind NPM.

---

### 📊 Prometheus Stack

**Path:** `monitoring/prometheus-stack/`

Full metrics stack: Prometheus (scraping), Grafana (dashboards), Alertmanager (alerts), Node Exporter (host metrics), cAdvisor (container metrics), Blackbox Exporter (endpoint probing).

Prometheus data is persisted in a named Docker volume (`prometheus_data`) and survives `docker compose down`.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `PROM_IMAGE` | `prom/prometheus:latest` | Prometheus image |
| `GRAFANA_IMAGE` | `grafana/grafana:latest` | Grafana image |
| `ALERT_IMAGE` | `prom/alertmanager:latest` | Alertmanager image |
| `NODE_EXPORTER_IMAGE` | `prom/node-exporter:latest` | Node Exporter image |
| `CADVISOR_IMAGE` | `gcr.io/cadvisor/cadvisor:latest` | cAdvisor image |
| `BLACKBOX_IMAGE` | `prom/blackbox-exporter:latest` | Blackbox Exporter image |
| `PROM_PORT` | `9090` | Prometheus host port |
| `GRAFANA_PORT` | `3000` | Grafana host port |
| `ALERT_PORT` | `9093` | Alertmanager host port |
| `BLACKBOX_PORT` | `9115` | Blackbox Exporter host port |
| `GRAFANA_USER` | `admin` | Grafana admin username |
| `GRAFANA_PASS` | **changeme** | Grafana admin password |
| `PROXY_NETWORK` | `proxy_net` | Must match NPM's network |

```bash
# Linux / macOS
cd monitoring/prometheus-stack
cp .env.example .env
nano .env    # set GRAFANA_PASS
docker compose up -d
```
```powershell
# Windows
cd monitoring\prometheus-stack
Copy-Item .env.example .env
notepad .env    # set GRAFANA_PASS
docker compose up -d
```

**To add endpoints for Blackbox Exporter to probe**, edit `prometheus/prometheus.yml` under the `blackbox` job:

```yaml
static_configs:
  - targets:
      - https://yourdomain.com
      - https://another.com
```

Then hot-reload Prometheus (no restart needed):

```bash
docker compose exec prometheus kill -HUP 1
```

---

### ✅ Uptime Kuma

**Path:** `monitoring/uptime-kuma/`

Self-hosted uptime monitor with a clean UI and notifications. Supports HTTP, TCP, DNS, Docker container monitoring, and more.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `IMAGE` | `louislam/uptime-kuma:1` | Docker image |
| `CONTAINER_NAME` | `uptime-kuma` | Container name |
| `PORT` | `3001` | Host port |
| `TIMEZONE` | `Asia/Kolkata` | Your timezone |
| `NETWORK` | `proxy_net` | Proxy network |

```bash
# Linux / macOS
cd monitoring/uptime-kuma
cp .env.example .env
docker compose up -d
```
```powershell
# Windows
cd monitoring\uptime-kuma
Copy-Item .env.example .env
docker compose up -d
```

> Create your admin account on the first visit to `http://your-server-ip:3001`. There is no default account.

---

### ⚡ n8n

**Path:** `automation/n8n/`

Workflow automation platform. The `N8N_ENCRYPTION_KEY` encrypts all credentials stored in n8n. Generate it once, back it up, and never change it — losing it (or the volume) means all stored credentials are permanently unrecoverable.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `IMAGE` | `n8nio/n8n:latest` | Docker image |
| `CONTAINER_NAME` | `n8n` | Container name |
| `PORT` | `5678` | Host port |
| `TIMEZONE` | `Asia/Kolkata` | Your timezone |
| `N8N_PROTOCOL` | `https` | Protocol used by your domain (`http` or `https`) |
| `N8N_HOST` | `n8n.yourdomain.com` | Your n8n domain |
| `WEBHOOK_URL` | `https://n8n.yourdomain.com/` | Full base URL for webhooks |
| `N8N_ENCRYPTION_KEY` | **changeme** | Credential encryption key — generate below |
| `NETWORK` | `proxy_net` | Proxy network |

```bash
# Linux / macOS
cd automation/n8n
cp .env.example .env
openssl rand -hex 32    # copy output → paste as N8N_ENCRYPTION_KEY in .env
nano .env               # also set N8N_HOST and WEBHOOK_URL
docker compose up -d
```
```powershell
# Windows
cd automation\n8n
Copy-Item .env.example .env
# Generate encryption key (see Secrets Reference above) → paste as N8N_ENCRYPTION_KEY
notepad .env    # also set N8N_HOST and WEBHOOK_URL
docker compose up -d
```

---

### 🏠 Homepage

**Path:** `dashboard/homepage/`

A highly customisable start page and dashboard. Configuration lives in the `config/` directory — add your services, bookmarks, and widgets there after deploying.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `IMAGE` | `ghcr.io/gethomepage/homepage:latest` | Docker image |
| `CONTAINER_NAME` | `homepage` | Container name |
| `PORT` | `3005` | Host port |
| `NETWORK` | `proxy_net` | Proxy network |

```bash
# Linux / macOS
cd dashboard/homepage
cp .env.example .env
docker compose up -d
```
```powershell
# Windows
cd dashboard\homepage
Copy-Item .env.example .env
docker compose up -d
```

> Configure your dashboard by editing the YAML files inside `config/`. See the [Homepage docs](https://gethomepage.dev/configs/) for full configuration options.

---

### 🐳 Portainer

**Path:** `management/portainer/`

Web UI for managing Docker containers, images, volumes, and networks across your host.

**`.env` variables:**

| Variable | Default | Description |
|---|---|---|
| `IMAGE` | `portainer/portainer-ce:latest` | Docker image (Community Edition) |
| `CONTAINER_NAME` | `portainer` | Container name |
| `PORT` | `9000` | Host port |
| `NETWORK` | `proxy_net` | Proxy network |

```bash
# Linux / macOS
cd management/portainer
cp .env.example .env
docker compose up -d
```
```powershell
# Windows
cd management\portainer
Copy-Item .env.example .env
docker compose up -d
```

> Create your admin account on first visit to `http://your-server-ip:9000`. **Do this within 5 minutes** — Portainer locks itself out if no account is created promptly.

---

## 🗂️ Repository Structure

```
docker-boilerplates/
├── automation/
│   └── n8n/
├── dashboard/
│   └── homepage/
├── dns/
│   └── pihole-unbound/
├── identity/
│   └── authentik/
├── management/
│   └── portainer/
├── monitoring/
│   ├── prometheus-stack/
│   │   ├── alertmanager/alertmanager.yml
│   │   └── prometheus/prometheus.yml
│   └── uptime-kuma/
└── proxy/
    └── nginx-proxy-manager/
```

Each stack contains:
- `docker-compose.yml` — service definition
- `.env.example` — all variables with safe defaults and instructions
- `data/` or similar — runtime data directories (gitignored, auto-created on first run)

---

## 🔧 Common Commands

All `docker compose` commands work identically on Linux, macOS, and Windows.

```bash
# Start a stack
docker compose up -d

# Stop a stack
docker compose down

# Stop and remove volumes (⚠️ destroys all persistent data for that stack)
docker compose down -v

# View logs (all services)
docker compose logs -f

# View logs for a specific service
docker compose logs -f <service-name>

# Restart a single service
docker compose restart <service-name>

# Pull latest images and redeploy
docker compose pull && docker compose up -d

# Check live resource usage
docker stats
```

---

## ❓ Troubleshooting

**Container exits immediately on start**
```bash
docker compose logs <service-name>
```
Usually a missing or wrong `.env` variable. Compare your `.env` against `.env.example`.

---

**Port already in use**

Linux / macOS:
```bash
ss -tlnp | grep <port>
# or
lsof -i :<port>
```

Windows (PowerShell):
```powershell
netstat -ano | findstr :<port>
```

Change the conflicting host port in your `.env` file.

---

**`proxy_net` network not found**
```bash
docker network create proxy_net
```
You skipped the prerequisite step. Run it once and retry.

---

**Authentik server keeps restarting on first deploy**

This is normal — it waits for PostgreSQL to finish initialising. Wait ~60 seconds then check:
```bash
docker compose logs server
```

---

**n8n webhooks not working behind NPM**

Make sure `N8N_HOST`, `N8N_PROTOCOL`, and `WEBHOOK_URL` in your `.env` exactly match the domain configured in Nginx Proxy Manager for n8n.

---

**Pi-hole not resolving DNS**

Check that your router or client is pointed at the host IP where Pi-hole is running on port `53`. Verify Unbound is running:
```bash
docker compose logs unbound
```

---

## 📝 Notes

- All stacks use `restart: unless-stopped` — containers come back automatically after a host reboot.
- `.env` files are gitignored and never committed. Only `.env.example` is tracked.
- Runtime data directories (`data/`, `postgres/`, `redis/`, `grafana/`, `letsencrypt/`) are gitignored and created automatically on first `docker compose up`.
- The `proxy_net` network is **created** by the NPM stack and **joined** as external by all other stacks. Always deploy NPM first.

---

## 📄 License

MIT — see [LICENSE](./LICENSE) for details.
