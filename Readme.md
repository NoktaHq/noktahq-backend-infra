# noktahq-backend-infra

API gateway (Traefik) for **api.noktahq.in**: TLS termination and path-based routing to multiple backends (SkillFort, Stitchfolio, etc.).

See **[Deployment.md](./Deployment.md)** for:

- VPC setup (Docker, SSL, Traefik, Portainer)
- Server layout and how CI deploys each backend
- Migrating from the previous nginx-per-backend setup

## Quick start on VPC

```bash
cd /infra
git clone https://github.com/NoktaHq/noktahq-backend-infra.git
cd noktahq-backend-infra/traefik
docker compose up -d
```

Ensure SSL certs exist at `/etc/letsencrypt/live/api.noktahq.in/` (or adjust the volume in `traefik/docker-compose.yml`).
