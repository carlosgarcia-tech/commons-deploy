# commons-deploy

Infraestructura y despliegue del ecosistema Commons Marketplace.
**Ansible puro** (playbooks + roles) ejecutado desde GitHub Actions sobre
Tailscale hacia el servidor propio.

Repositorios hermanos:

- `commons-marketplace/` — API backend (Node.js/Express)
- `automation-deployments/` — automatización legacy (GitLab, referencia)

## Estructura

```
├── .github/
│   ├── actions/
│   │   ├── tailscale-gate/     # conectar a tailnet + ping al host (acepta DERP)
│   │   └── ansible-setup/      # venv con ansible-core, collections, deploy_key
│   └── workflows/              # disparadores delgados
│       ├── ping.yml            # solo verificación de conectividad
│       ├── provision.yml       # Docker + grupo docker + login GHCR + mongo:7
│       └── sonarqube.yml       # SonarQube + PostgreSQL
├── ansible.cfg
├── inventory/hosts.yml         # host "target" (SSH_HOST / SSH_USERNAME)
├── group_vars/all.yml          # toda la configuración (lookups a env)
├── collections/requirements.yml
├── playbooks/
│   ├── provision.yml           # roles: docker → ghcr_login → mongo_image
│   └── sonarqube.yml           # role: sonarqube (Postgres + Sonar + health gate)
└── roles/
    ├── docker/tasks/main.yml   # apt repo deb822 + docker-ce + compose plugin
    ├── ghcr_login/tasks/main.yml
    ├── mongo_image/tasks/main.yml
    └── sonarqube/tasks/main.yml
```

## Workflows

| Workflow | Qué hace |
|---|---|
| **Ping** | Gate de conectividad contra `SSH_HOST` por la tailnet |
| **Provision** | Instala Docker Engine + compose plugin (idempotente), añade el usuario SSH al grupo `docker`, login en ghcr.io (si hay secrets) y descarga `mongo:7` sin arrancarla |
| **SonarQube** | Red Docker compartida + PostgreSQL 15 (solo interno) + SonarQube community con volúmenes persistentes; espera a `/api/system/status = UP` |

Ambos workflows de despliegue pasan primero por el gate de ping: si el host
no responde no se toca nada.

## Secrets requeridos

| Secret | Uso | Workflows |
|---|---|---|
| `TS_AUTHKEY` | Auth key de Tailscale para el runner | todos |
| `SSH_HOST` | IP/MagicDNS del servidor en la tailnet | todos |
| `SSH_USERNAME` | Usuario SSH (también entra al grupo docker) | provision |
| `SSH_PRIVATE_KEY` | Llave privada de acceso | provision, sonarqube |
| `GHCR_USERNAME` / `GHCR_TOKEN` | PAT `read:packages` para ghcr.io (opcional) | provision |
| `SONAR_DB_PASSWORD` | Password del Postgres de Sonar | sonarqube |

## Variables opcionales (env, ver `group_vars/all.yml`)

`MONGO_IMAGE` (mongo:7) · `DOCKER_NETWORK_NAME` (commons-net) ·
`SONARQUBE_PORT` (9000) · `SONARQUBE_IMAGE` · `POSTGRES_*` · rutas de
volúmenes bajo `/opt`.

## Notas

- El ping acepta conectividad vía relay DERP: entre runners de GitHub y
  servidores domésticos tras NAT no suele haber P2P directo.
- Tras el provision hace falta re-login en el servidor para usar docker
  sin sudo; los playbooks usan `become` automáticamente donde toca.
- Credenciales iniciales de SonarQube: `admin/admin` (cambiarlas en el
  primer arranque desde la UI).
- La imagen Mongo se descarga pero no corre todavía: reservada para uso
  posterior como BD local.
