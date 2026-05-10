---
title: Cloud Deployment
sidebar_position: 2
---

# ☁️ Cloud — Staging Environment

[Source repo ↗](https://github.com/alphaoflogic-ua/smart-home-cloud)

Staging is live at **svaroh.com**. Auto-deploys on every `v*` tag via GitHub Actions.

---

## Endpoints

| Purpose | URL |
|---------|-----|
| REST API | `https://api.staging.svaroh.com` |
| WebSocket (stations + mobile relay) | `wss://api.staging.svaroh.com/<ws-path>` |
| Health check | `https://api.staging.svaroh.com/health` |
| Docs site | `https://docs.svaroh.com` |
| Universal Links / AASA | `https://app.staging.svaroh.com` |
| API reference (Scalar UI) | `https://api.staging.svaroh.com/api/reference` |
| OpenAPI JSON | `https://api.staging.svaroh.com/documentation/json` |

---

## Infrastructure

**Hosting:** Hetzner Cloud, Ubuntu 24.04. DNS via Cloudflare (Proxy disabled — WebSocket requires DNS-only).

**Services (docker-compose.staging.yml):**

| Service | Image | Role |
|---------|-------|------|
| `postgres` | `postgres:16-alpine` | Primary database |
| `cloud` | built from repo | Fastify API on `:4000` |
| `caddy` | `caddy:2-alpine` | Reverse proxy + TLS (Let's Encrypt) |

Caddy serves:
- `api.staging.svaroh.com` → `cloud:4000` (HTTP + WebSocket)
- `docs.svaroh.com` → static build from `~/svaroh-docs/build`
- `app.staging.svaroh.com` → static files (AASA, Universal Links landing)

---

## Deploy flow

Triggered by:
- Push of a `v*` tag (e.g. `v0.2.1`) — standard release
- Manual `workflow_dispatch` with a tag or branch ref

Steps performed by CI (`.github/workflows/deploy-staging.yml`):
1. Record version/commit/timestamp into `VERSION` file
2. Rsync source to `~/smart-home-cloud/` on the server (excludes `.env`, `node_modules`, `dist`)
3. Pull base images (`postgres`, `caddy`), build `cloud` image on the server
4. `docker compose up -d`, prune dangling images
5. Health check `https://api.staging.svaroh.com/health` — 5 attempts, 10 s apart

**GitHub secrets required:** `SSH_KEY`, `SSH_HOST`, `SSH_USER`  
**GitHub variable (optional):** `HEALTH_URL` (defaults to the health endpoint above)

To deploy manually:

```bash
git release v0.2.1   # uses scripts/release.sh — never tag manually
```

Or trigger via GitHub Actions UI with a specific ref.

---

## Configuration

`.env` lives on the server at `~/smart-home-cloud/.env` (never committed). Template: `.env.example` in the repo.

Key variables:

| Variable | Description |
|----------|-------------|
| `NODE_ENV` | `production` |
| `PORT` | `4000` |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | PostgreSQL connection |
| `JWT_SECRET` / `JWT_REFRESH_SECRET` | Token signing |
| `JWT_EXPIRES_IN` / `JWT_REFRESH_EXPIRES_IN` | `15m` / `30d` |
| `APPLE_CLIENT_ID` | `<APPLE_CLIENT_ID>` (Sign in with Apple bundle ID) |

---

## Connecting a station to staging

On the Raspberry Pi, set `CLOUD_WSS_URL` in `.env` (exact path is in the deployment runbook) and restart. Then watch cloud logs:

```bash
docker compose -f docker-compose.staging.yml logs -f cloud
```