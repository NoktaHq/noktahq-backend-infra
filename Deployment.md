# NoktaHQ API Gateway – Deployment Guide

Single VPC (**api.noktahq.in**), one **Traefik** API gateway, multiple backends (SkillFort, Stitchfolio, future services). Path-based routing and TLS termination are handled at the infrastructure level.

---

## 1. Architecture

```
                    ┌─────────────────────────────────────────┐
                    │  api.noktahq.in (80 → 443, 443)         │
                    │  Traefik (TLS, path routing)            │
                    └─────────────────┬───────────────────────┘
                                      │ network: noktahq_web
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
         ▼                            ▼                            ▼
  /api/sf/v1                   /api/skillfort/v1            (future backends)
  stitchfolio-prod             skillfort-prod
  /api/sf/dev/v1               /api/skillfort/dev/v1
  stitchfolio-dev               skillfort-dev
```

- **One domain**: `api.noktahq.in`
- **One gateway**: Traefik (this repo) – TLS, HTTP→HTTPS, path routing
- **Backends**: No nginx; each app runs as a single backend container and joins Docker network `noktahq_web`. Traefik discovers them via Docker labels.

### URL layout

| Backend     | Prod (main)                    | Dev (dev)                           |
|------------|----------------------------------|-------------------------------------|
| Stitchfolio | `https://api.noktahq.in/api/sf/v1` | `https://api.noktahq.in/api/sf/dev/v1` |
| SkillFort   | `https://api.noktahq.in/api/skillfort/v1` | `https://api.noktahq.in/api/skillfort/dev/v1` |

---

## 2. Server layout (VPC)

Target layout (clone **noktahq-backend-infra** to `/infra` so the gateway is at `/infra/traefik/`):

```
/infra/                    # clone of noktahq-backend-infra
├── traefik/
│   ├── docker-compose.yml
│   ├── traefik.yml
│   └── dynamic/
│       └── tls.yml
├── skillfort/
│   ├── skillfort-prod/    # main branch deploy (CI clones repo here)
│   │   ├── .env           # written by CI
│   │   ├── docker-compose.yml
│   └── skillfort-dev/     # dev branch deploy
│       ├── .env
│       ├── docker-compose.yml
│       └── docker-compose.dev.yml
├── stitchfolio/
│   ├── backend-prod/
│   └── backend-dev/
└── /root/
    ├── skillfort_env/
    │   ├── prod.env
    │   └── dev.env
    └── stitchfolio_env/
        ├── prod.env
        └── dev.env
```

- **Infra** runs once and stays up; **SkillFort** and **Stitchfolio** CIs deploy their own compose (no nginx, no SSL in app repos).
- Env files stay in `/root/*_env/` and are referenced by path in CI (e.g. `ENV_FILE_PATH=/root/skillfort_env/prod.env`).

---

## 3. What to do on the VPC (one-time)

### 3.1 Install Docker and Docker Compose

```bash
# Install Docker (e.g. Ubuntu)
curl -fsSL https://get.docker.com | sh
usermod -aG docker $USER
# Install Compose v2 plugin (usually bundled with Docker)
docker compose version
```

### 3.2 SSL certificate (already present)

You said the certificate is already in the default path. Ensure:

- Domain: `api.noktahq.in`
- Path: `/etc/letsencrypt/live/api.noktahq.in/`
- Files: `fullchain.pem`, `privkey.pem`

Traefik will mount this directory read-only. If you use a different path, change the volume in `traefik/docker-compose.yml`:

```yaml
volumes:
  - /etc/letsencrypt/live/api.noktahq.in:/etc/traefik/certs:ro
```

### 3.3 Clone infra and start Traefik

```bash
cd /infra
git clone https://github.com/NoktaHq/noktahq-backend-infra.git
cd noktahq-backend-infra/traefik
docker compose up -d
```

This:

- Creates network `noktahq_web`
- Listens on 80 (redirect to HTTPS) and 443
- Loads TLS from `dynamic/tls.yml` (certs from `/etc/traefik/certs`)

Check:

```bash
docker compose ps
docker network ls | grep noktahq_web
```

### 3.4 (Optional) Portainer

To manage containers and see deployments/ports/status:

```bash
docker volume create portainer_data
docker run -d -p 9000:9000 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Then open `http://<VPS-IP>:9000`, create admin user, and use the UI to see stacks (Traefik, SkillFort, Stitchfolio) and containers.

---

## 4. How deployments work

### 4.1 Infra (Traefik)

- **Who**: You (or a separate “infra” CI).
- **When**: Once, or when you change Traefik config/certs.
- **How**:
  ```bash
  cd /infra/noktahq-backend-infra/traefik
  docker compose up -d
  ```
- **Reload certs**: After renewing Let’s Encrypt, restart Traefik:
  ```bash
  docker compose restart
  ```

### 4.2 SkillFort

- **Main** → deploy to `/skillfort/skillfort-prod`, router `skillfort-prod`, path `/api/skillfort/v1`.
- **Dev** → deploy to `/skillfort/skillfort-dev`, router `skillfort-dev`, path `/api/skillfort/dev/v1` (Traefik strips `/dev` before forwarding).

CI runs:

- Prod: `docker compose -f /skillfort/skillfort-prod/docker-compose.yml -p skillfort-backend-prod up -d --build`
- Dev: `docker compose -f /skillfort/skillfort-dev/docker-compose.yml -f /skillfort/skillfort-dev/docker-compose.dev.yml -p skillfort-backend-dev up -d --build`

No nginx, no SSL in repo; `.env` is written by CI (env file path, config file, Traefik router name and rule).

### 4.3 Stitchfolio

- **Main** → `/stitchfolio/backend-prod`, path `/api/sf/v1`.
- **Dev** → `/stitchfolio/backend-dev`, path `/api/sf/dev/v1` (strip `/dev`).

Same idea: backend-only compose, join `noktahq_web`, labels for Traefik; CI writes `.env` and runs compose.

---

## 5. Add a new backend later

1. New repo (or new service in same VPC): backend container exposes one port (e.g. 9000), no nginx.
2. In its `docker-compose.yml`:
   - Use network `noktahq_web` (external).
   - Add Traefik labels: router name, rule (e.g. `PathPrefix(\`/api/newservice/v1\`)`), entrypoint `websecure`, service port.
3. If you need a “dev” path (e.g. `/api/newservice/dev/v1`), add an override compose with a replace-path middleware (like `docker-compose.dev.yml` in SkillFort/Stitchfolio).
4. Deploy that compose to a directory under `/infra` or a dedicated path (e.g. `/newservice/newservice-prod`).
5. No change to Traefik static config unless you add a new entrypoint or TLS option; routing is dynamic via Docker labels.

---

## 6. Traefik dashboard (optional)

Traefik’s API/dashboard can be enabled in `traefik.yml` (already set in this repo). To expose it only locally or via a separate port, use a file provider router and restrict access (e.g. IP or auth). For production, many teams leave the dashboard off or only on localhost.

---

## 7. Migrating from the old setup (nginx per backend)

If you currently have **nginx + backend** containers (each backend with its own nginx on different ports):

1. **Stop all existing backend stacks** so nothing is using 80/443 or the old ports. From the server, go to each deploy directory and run `docker compose down` (use the same project name as in your current workflows, e.g. `docker compose -p stitchfolio-prod down` in `/stitchfolio/backend-prod`, etc.).
2. **Start Traefik** (section 3.3).
3. **Redeploy backends** by pushing to main/dev so CI runs the new workflows, or manually run the same `docker compose` commands the CI uses. The new compose files have no nginx and use Traefik labels.

---

## 8. Summary checklist

| Step | Action |
|------|--------|
| 1 | Docker + Compose on VPC |
| 2 | SSL cert at `/etc/letsencrypt/live/api.noktahq.in` (or adjust volume in compose) |
| 3 | Clone noktahq-backend-infra, run `docker compose` in `traefik/` |
| 4 | Env files in `/root/skillfort_env/` and `/root/stitchfolio_env/` |
| 5 | Push to main/dev on SkillFort and Stitchfolio so CI deploys backends |
| 6 | (Optional) Install Portainer for UI |

After that, all traffic goes to **api.noktahq.in**; Traefik terminates TLS and routes by path; backends only run their app container and join `noktahq_web`.
