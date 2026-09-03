# commons-deploy

Infrastructure-as-code repository for the **Commons Marketplace** ecosystem. Pure Ansible (playbooks + roles) executed from GitHub Actions over a Tailscale tailnet to a self-hosted server.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      GitHub Actions Runner                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ tailscale   │  │ ansible     │  │ workflows   │              │
│  │ -gate       │──│ -setup      │──│ (dispatch)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Tailscale (DERP-relay friendly)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Target Server (SSH)                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Docker Network: commons-net                            │    │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐   │    │
│  │  │ PostgreSQL  │ │ SonarQube   │ │ MongoDB (image) │   │    │
│  │  │ :5432       │ │ :9000       │ │ (pre-pulled)    │   │    │
│  │  └─────────────┘ └─────────────┘ └─────────────────┘   │    │
│  │  ┌─────────────────────────────────────────────────────┐  │    │
│  │  │ API Blue-Green (commons-marketplace-blue/green)     │  │    │
│  │  │ ┌──────────────────┐  ┌──────────────────┐         │  │    │
│  │  │ │ blue:3000        │  │ green:3000       │         │  │    │
│  │  │ └────────┬─────────┘  └────────┬─────────┘         │  │    │
│  │  │          │                     │                    │  │    │
│  │  │          ▼                     ▼                    │  │    │
│  │  │  ┌─────────────────────────────────────┐           │  │    │
│  │  │  │ commons-proxy (nginx) :3000→80      │           │  │    │
│  │  │  │ upstream marketplace_active         │           │  │    │
│  │  │  └──────────────┬──────────────────────┘           │  │    │
│  │  │                 │                                  │  │    │
│  │  └─────────────────┼──────────────────────────────────┘  │    │
│  │                    │                                     │    │
│  │  ┌─────────────────┼──────────────────────────────────┐  │    │
│  │  │ Client Blue-Green (commons-client-blue/green)      │  │    │
│  │  │ ┌──────────────────┐  ┌──────────────────┐         │  │    │
│  │  │ │ blue:3000        │  │ green:3000       │         │  │    │
│  │  │ └────────┬─────────┘  └────────┬─────────┘         │  │    │
│  │  │          │                     │                    │  │    │
│  │  │          ▼                     ▼                    │  │    │
│  │  │  ┌─────────────────────────────────────┐           │  │    │
│  │  │  │ commons-client-proxy (nginx) :8080  │           │  │    │
│  │  │  │ • /*           → active client      │           │  │    │
│  │  │  │ • /api/        → commons-proxy      │           │  │    │
│  │  │  │ • /socket.io/  → commons-proxy      │           │  │    │
│  │  │  │ • /_next/static/ → client (cached)  │           │  │    │
│  │  │  └─────────────────────────────────────┘           │  │    │
│  │  └─────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
commons-deploy/
├── .github/
│   ├── actions/
│   │   ├── tailscale-gate/      # Join tailnet + verify host reachability (accepts DERP)
│   │   └── ansible-setup/       # Isolated venv: ansible-core + collections + deploy_key
│   └── workflows/
│       ├── ping.yml             # Connectivity gate only (runs on push to main + dispatch)
│       ├── provision.yml        # Docker Engine + compose plugin + GHCR login + mongo:8
│       ├── sonarqube.yml        # PostgreSQL 18 + SonarQube Community on commons-net
│       ├── marketplace.yml      # Deploy API (blue-green + health gate + auto-rollback)
│       ├── client.yml           # Deploy Client (blue-green + health gate + auto-rollback)
│       ├── marketplace-rollback.yml  # Instant traffic flip between API colors
│       └── client-rollback.yml       # Instant traffic flip between Client colors
├── ansible.cfg                  # SSH tuning (ControlPersist, pipelining, timeout=30s)
├── inventory/hosts.yml          # Single "target" host via SSH_HOST/SSH_USERNAME env
├── group_vars/all.yml           # All configuration (lookups → env/Secrets)
├── collections/requirements.yml # community.docker >= 5.2.2
├── playbooks/
│   ├── provision.yml            # roles: docker → ghcr_login → mongo_image
│   ├── sonarqube.yml            # role: sonarqube
│   ├── marketplace.yml          # roles: ghcr_login → marketplace
│   ├── marketplace_rollback.yml # role: marketplace (tasks_from: rollback.yml)
│   ├── client.yml               # roles: ghcr_login → client
│   └── client_rollback.yml      # role: client (tasks_from: rollback.yml)
└── roles/
    ├── docker/                  # Docker CE repo (Debian/Fedora) + service + group + user
    ├── ghcr_login/              # docker login as SSH user (writes ~/.docker/config.json)
    ├── mongo_image/             # Pre-pull mongo:8 (no container, reserved for later)
    ├── sonarqube/               # PG data dir → PG container → wait pg_isready → Sonar → wait /api/system/status=UP
    ├── marketplace/             # Blue-green API deployment with nginx proxy
    │   ├── tasks/main.yml       # Deploy + health gate + proxy switch + public verify + park old color
    │   ├── tasks/rollback.yml   # Instant rollback: start parked color → health gate → switch → verify
    │   └── templates/marketplace.conf.j2  # nginx upstream → active color
    └── client/                  # Blue-green Client deployment with nginx proxy (public entrypoint)
        ├── tasks/main.yml       # Deploy + health gate + proxy switch + public verify + park old color
        ├── tasks/rollback.yml   # Instant rollback: start parked color → health gate → switch → verify
        └── templates/marketplace-client.conf.j2  # nginx: /* → client, /api + /socket.io → API proxy
```

## Core Concepts

### Blue-Green Deployments (API & Client)

Both the **API** (`commons-marketplace`) and **Client** (`commons-marketplace-client`) use identical blue-green patterns:

| Component | Containers | Proxy | Published Port | Health Endpoint |
|-----------|------------|-------|----------------|-----------------|
| API | `commons-marketplace-blue`, `commons-marketplace-green` | `commons-proxy` (nginx) | `MARKETPLACE_PORT` (default 3000) | `/health` (container port 3000) |
| Client | `commons-client-blue`, `commons-client-green` | `commons-client-proxy` (nginx) | `CLIENT_PORT` (default 8080) | `/api/health` (container port 3000) |

**Deployment flow:**
1. Determine active color by reading current nginx proxy config (`slurp` + `b64decode`)
2. Deploy new image to **target color** (opposite of active)
3. **Internal health gate**: `docker exec` + polling loop (24×5s) against container localhost
4. Rewrite proxy config to point at target color (`template` → nginx reload)
5. **Public verification**: `uri` module against host port (12×5s retries)
6. On failure: **auto-revert** proxy to previous color, fail deployment
7. On success: **park** previous color container (stopped, not removed) for instant rollback

**Key design decisions:**
- **No `comparisons.image=ignore`**: Image tag changes MUST recreate containers (use immutable tags like `20260825-a1b2c3d`, not `latest`)
- **Root-context GHCR login**: Container pulls use `/root/.docker/config.json`; SSH user login is separate for manual pulls
- **Stopped containers retained**: Previous color stays with exact image+env for instant rollback without pulls

### Client Proxy: Single Public Entrypoint

The client proxy (`commons-client-proxy`) is the **only public-facing service** (target of `tailscale funnel <CLIENT_PORT>`). It routes by path:

```
/*                  → active client color (commons-client-{blue,green}:3000)
/api/               → API blue-green proxy (commons-proxy:80)
/socket.io/         → API blue-green proxy (WebSocket upgrade)
/_api/health        → active client color (liveness, NOT proxied to API)
/_next/static/      → client (aggressive caching: 1 year, immutable)
```

This keeps browser calls same-origin (no CORS) and hides the API behind the client proxy.

### Tailscale Gate

All workflows start with the **tailscale-gate** composite action:
- Joins runner to tailnet via `TS_AUTHKEY`
- Runs `tailscale ping --c 4 --timeout 5s <SSH_HOST>`
- Accepts **any pong** (direct P2P or DERP-relayed)
- Fails fast if host unreachable — no Ansible runs against dead targets

### Ansible Setup

The **ansible-setup** composite action:
- Creates isolated venv at `.venv/`
- Installs `ansible-core` + `community.docker` collection
- Writes `SSH_PRIVATE_KEY` secret to `deploy_key` (mode 600)
- Adds venv `bin/` to `GITHUB_PATH` for subsequent steps

### Provisioning Idempotency

The `provision.yml` playbook is safe to re-run:
- Docker CE repository + packages (Debian deb822 / Fedora)
- Docker service enabled/started
- `docker` group + SSH user membership (requires re-login for `docker` without sudo)
- GHCR login (skipped if secrets absent)
- `mongo:8` image pulled (no container started)

## Configuration (`group_vars/all.yml`)

All variables use `lookup('env', 'VAR')` with sensible defaults. **Required secrets** map 1:1 to workflow env:

| Variable | Source | Default | Description |
|----------|--------|---------|-------------|
| `docker_install_user` | `SSH_USERNAME` | — | User added to `docker` group |
| `ghcr_username` / `ghcr_token` | `GHCR_USERNAME` / `GHCR_TOKEN` | *empty* | GHCR credentials (optional; skips login if empty) |
| `mongo_image` | `MONGO_IMAGE` | `mongo:8` | MongoDB image to pre-pull |
| `docker_network_name` | `DOCKER_NETWORK_NAME` | `commons-net` | Shared Docker network for all stacks |
| `postgres_image` | `POSTGRES_IMAGE` | `postgres:18` | PostgreSQL for SonarQube |
| `postgres_host_data_dir` | `POSTGRES_HOST_DATA_DIR` | `/opt/sonar-postgres/data` | PG data volume (parent dir for v18+) |
| `postgres_user` / `postgres_password` / `postgres_db` | `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | `sonar` / **required** / `sonar` | PG credentials |
| `sonarqube_image` | `SONARQUBE_IMAGE` | `sonarqube:community` | SonarQube Community Edition |
| `sonarqube_host_data_dir` | `SONARQUBE_HOST_DATA_DIR` | `/opt/sonarqube/data` | Sonar data volume |
| `sonarqube_host_extensions_dir` | `SONARQUBE_HOST_EXTENSIONS_DIR` | `/opt/sonarqube/extensions` | Sonar extensions volume |
| `sonarqube_host_logs_dir` | `SONARQUBE_HOST_LOGS_DIR` | `/opt/sonarqube/logs` | Sonar logs volume |
| `sonarqube_port` | `SONARQUBE_PORT` | `9000` | Host port published for SonarQube |
| `sonarqube_startup_timeout` | `SONARQUBE_STARTUP_TIMEOUT` | `300` | Max seconds waiting for `/api/system/status=UP` |
| `marketplace_image` | `MARKETPLACE_IMAGE` | `ghcr.io/carlosgarcia-tech/commons-marketplace:latest` | API image |
| `marketplace_host_port` | `MARKETPLACE_PORT` | `3000` | Host port for API proxy |
| `marketplace_env.*` | Various secrets | — | `DB_URL`, `SUPABASE_*`, `ABLY_API_KEY`, `CORS_ORIGINS`, `TRUST_PROXY=1` |
| `marketplace_proxy_name` | — | `commons-proxy` | API proxy container name |
| `marketplace_proxy_image` | `MARKETPLACE_PROXY_IMAGE` | `nginx:1.29-alpine` | Proxy image |
| `marketplace_proxy_conf_dir` | `MARKETPLACE_PROXY_CONF_DIR` | `/opt/commons-proxy/conf.d` | Proxy config directory |
| `client_image` | `CLIENT_IMAGE` | `ghcr.io/carlosgarcia-tech/commons-marketplace-client:latest` | Client image |
| `client_host_port` | `CLIENT_PORT` | `8080` | Host port for Client proxy (tailscale funnel target) |
| `client_proxy_name` | — | `commons-client-proxy` | Client proxy container name |
| `client_proxy_image` | `CLIENT_PROXY_IMAGE` | `nginx:1.29-alpine` | Proxy image |
| `client_proxy_conf_dir` | `CLIENT_PROXY_CONF_DIR` | `/opt/commons-client-proxy/conf.d` | Proxy config directory |
| `client_api_upstream_host` | `CLIENT_API_UPSTREAM_HOST` | `commons-proxy` | API proxy hostname on `commons-net` |
| `client_api_upstream_port` | `CLIENT_API_UPSTREAM_PORT` | `80` | API proxy port on `commons-net` |

**Required GitHub Secrets:**

| Secret | Used By |
|--------|---------|
| `TS_AUTHKEY` | All workflows (tailscale-gate) |
| `SSH_HOST` | All workflows (inventory + tailscale-gate) |
| `SSH_USERNAME` | Provision, Marketplace, Client, SonarQube |
| `SSH_PRIVATE_KEY` | Provision, Marketplace, Client, SonarQube (ansible-setup) |
| `GHCR_USERNAME` / `GHCR_TOKEN` | Provision, Marketplace, Client (optional) |
| `SONAR_DB_PASSWORD` | SonarQube (`POSTGRES_PASSWORD`) |
| `DB_URL` | Marketplace (`marketplace_env.DB_URL`) |
| `SUPABASE_URL` / `SUPABASE_PUBLISHABLE_KEY` / `SUPABASE_SECRET_KEY` / `SUPABASE_STORAGE_BUCKET` | Marketplace |
| `ABLY_API_KEY` | Marketplace |
| `CORS_ORIGINS` | Marketplace |
| `TRUST_PROXY` | Marketplace (default `1`) |

## Workflows

### `ping.yml`
- **Trigger**: Push to `main` + manual dispatch
- **Purpose**: Continuous connectivity verification
- **Concurrency**: Cancels in-progress runs (`group: ping`)

### `provision.yml`
- **Trigger**: Manual dispatch
- **Steps**: tailscale-gate → ansible-setup → `ansible-playbook playbooks/provision.yml`
- **Run once** on new server; re-run for Docker updates or user changes

### `sonarqube.yml`
- **Trigger**: Manual dispatch
- **Steps**: tailscale-gate → ansible-setup → `ansible-playbook playbooks/sonarqube.yml`
- **Deploys**: PostgreSQL 18 + SonarQube Community on `commons-net`
- **Health gate**: Waits for TCP port + `/api/system/status = UP`
- **Initial credentials**: `admin/admin` (change on first UI login)

### `marketplace.yml` / `client.yml`
- **Trigger**: Manual dispatch with `image_tag` input (default `latest`)
- **Steps**: tailscale-gate → ansible-setup → respective playbook
- **Image tag**: Passed via `MARKETPLACE_IMAGE` / `CLIENT_IMAGE` env (e.g., `ghcr.io/.../commons-marketplace:20260825-a1b2c3d`)
- **⚠️ Warning**: Using `latest` reuses already-pulled image; pin tag for guaranteed freshness

### `marketplace-rollback.yml` / `client-rollback.yml`
- **Trigger**: Manual dispatch with `color` input (`blue` or `green`)
- **Purpose**: Instant traffic rollback to the **other** color (which previous deploys left stopped)
- **No image pulls**, no recreation — starts parked container, health-gates it, switches proxy, verifies
- **Fails fast** if target color never deployed or health gate fails (auto-reverts)

## SSH & Ansible Tuning (`ansible.cfg`)

```ini
[defaults]
timeout = 30                    # Slow tailnet + sudo prompts
host_key_checking = False       # Ephemeral runners
retry_files_enabled = False
interpreter_python = auto_silent
inject_facts_as_vars = False    # Cleaner variable namespace

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=accept-new
pipelining = True               # Reduces SSH round-trips
```

## Common Operations

### Initial Server Setup
```bash
# 1. Configure GitHub Secrets (TS_AUTHKEY, SSH_HOST, SSH_USERNAME, SSH_PRIVATE_KEY, GHCR_*)
# 2. Run Provision workflow from GitHub Actions UI
# 3. SSH to server and re-login (or `newgrp docker`) to use docker without sudo
```

### Deploy SonarQube
```bash
# Set SONAR_DB_PASSWORD secret, run SonarQube workflow
# Access at http://<server-tailscale-ip>:9000 (admin/admin)
```

### Deploy API (Marketplace)
```bash
# Set all MARKETPLACE_* secrets + DB_URL + Supabase + Ably + CORS
# Run Marketplace workflow with image_tag (e.g., 20260825-a1b2c3d)
```

### Deploy Client
```bash
# Set CLIENT_IMAGE (usually same tag as API)
# Run Client workflow with image_tag
# Client available at http://<server-tailscale-ip>:8080 (or tailscale funnel)
```

### Rollback API
```bash
# Run Marketplace Rollback workflow, select color (blue/green)
# Traffic flips in seconds; old color parked stopped
```

### Rollback Client
```bash
# Run Client Rollback workflow, select color (blue/green)
```

## Security Notes

- **No secrets in repo**: All sensitive values via GitHub Secrets → env → Ansible lookups
- **`no_log: true`** on all `docker_login` tasks
- **Root-context login** for container pulls (separate from SSH user login)
- **SSH private key** written to `deploy_key` (600) at runtime, not persisted
- **Tailscale auth key** (`TS_AUTHKEY`) should be reusable/ephemeral, not a user key
- **PostgreSQL password** required (entrypoint crashes without it)

## Supported Platforms

- **Debian/Ubuntu** (apt, deb822 repo format)
- **Fedora** (dnf, docker-ce repo)
- Other distributions: extend `roles/docker/tasks/main.yml` dispatch logic

## Extending

- **New service**: Add role under `roles/`, playbook under `playbooks/`, workflow under `.github/workflows/`
- **New OS**: Add `<Distro>.yml` under `roles/docker/tasks/` and extend `when` condition in `main.yml`
- **Additional proxy routes**: Modify `marketplace-client.conf.j2` (client proxy is the edge router)

## Related Repositories

- `commons-marketplace/` — API backend (Node.js/Express, publishes to `ghcr.io/carlosgarcia-tech/commons-marketplace`)
- `commons-marketplace-client/` — Next.js frontend (publishes to `ghcr.io/carlosgarcia-tech/commons-marketplace-client`)
- `automation-deployments/` — Legacy GitLab automation (reference only)