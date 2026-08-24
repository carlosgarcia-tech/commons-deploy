# commons-deploy

Infraestructura y despliegue del ecosistema Commons Marketplace, ejecutado con
GitHub Actions sobre Tailscale hacia el servidor propio.

Repositorios hermanos:

- `commons-marketplace/` — API backend (Node.js/Express)
- `automation-deployments/` — automatización legacy (GitLab + Ansible)

## Workflows

| Workflow | Qué hace | Disparo |
|---|---|---|
| `ping.yml` | Verifica que el host sea alcanzable por la tailnet (acepta rutas DERP) | Manual |
| `provision.yml` | Instala Docker Engine + compose, grupo `docker`, login GHCR opcional y descarga la imagen base de Mongo | Manual |

Ambos usan el mismo mecanismo que el deploy del repo API:
`tailscale/github-action@v3` + `appleboy/ssh-action@v1.0.3`.

## Secrets requeridos (Settings → Secrets → Actions)

| Secret | Uso | Requerido en |
|---|---|---|
| `TS_AUTHKEY` | Auth key de Tailscale para el runner | ping, provision |
| `SSH_HOST` | IP/MagicDNS del servidor en el tailnet | ping, provision |
| `SSH_USERNAME` | Usuario SSH del servidor | provision |
| `SSH_PRIVATE_KEY` | Llave privada para SSH | provision |
| `GHCR_USERNAME` | Usuario de GitHub (para `docker login ghcr.io`) | provision (opcional) |
| `GHCR_TOKEN` | PAT con scope `read:packages` | provision (opcional) |

Sin `GHCR_USERNAME/GHCR_TOKEN` el provision sigue y solo avisa; pero el deploy
de imágenes privadas fallará hasta configurarlo.

## Notas

- El ping acepta conectividad por relay DERP: entre runners de GitHub
  (datacenter) y servidores domésticos tras NAT normalmente no hay P2P directo.
- Tras añadir el usuario al grupo `docker` hace falta re-login para usar docker
  sin `sudo`; el workflow usa sudo automáticamente cuando es necesario.
- La imagen `mongo:7` se descarga pero no se ejecuta todavía (reservada).
