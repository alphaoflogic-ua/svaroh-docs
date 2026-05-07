---
title: Cloud Deployment
sidebar_position: 2
---

# ☁️ Cloud Staging Deployment

[Link to source repo ↗](https://github.com/alphaoflogic-ua/smart-home-cloud)

Goal: deploy `smart-home-cloud` to a public VPS with HTTPS/WSS and auto-deploy from GitHub, ready for stations and the iOS app to connect.

**Time estimate:** 3–4 hours of active work + up to 1 hour waiting for DNS/TLS.
**Starting budget:** ~$15 (domain $10 + first month of VPS $5).

---

## Stage 0. Prerequisites

- [ ] 0.1. GitHub account with the private `smart-home-cloud` repo pushed to the `develop` branch (or `main`).
- [ ] 0.2. Local SSH key. Check: `ls ~/.ssh/id_ed25519.pub`. If missing — `ssh-keygen -t ed25519 -C "alphaoflogic@gmail.com"` (Enter through all prompts).
- [ ] 0.3. International payment card (Visa/MC, Wise/Monobank FX). Check that the limit is at least $20.
- [ ] 0.4. `docker` and `docker compose` installed locally (for the first smoke build): `docker --version`, `docker compose version`.
- [ ] 0.5. `curl` and `dig`/`nslookup` installed: `which curl dig`.

---

## Stage 1. Domain (15 min + 5–60 min of DNS waiting)

- [ ] 1.1. Open https://www.cloudflare.com/products/registrar/ (or https://www.namecheap.com if Cloudflare doesn't accept the card).
- [ ] 1.2. Pick a name: `<slug>.com` (for example `smartstation-lab.com`). Check availability.
- [ ] 1.3. Pay for 1 year of registration. **Disable auto-renew right away** so you don't get charged automatically for a second year.
- [ ] 1.4. In Cloudflare: Domain → DNS → empty for now. Enable **Full (strict)** SSL mode in _SSL/TLS → Overview_, but **disable Proxy (grey cloud)** for `api.staging.*` — Cloudflare Proxy breaks WebSocket connections from stations.
- [ ] 1.5. Save the domain in your notes: `DOMAIN=<chosen domain>`. From here on I'll write `<DOMAIN>`.

**Verification:** `dig <DOMAIN> NS +short` returns Cloudflare (or Namecheap) NS servers.

---

## Stage 2. VPS on Hetzner (20 min)

- [ ] 2.1. Sign up: https://accounts.hetzner.com/signUp. Need an email + card. Initial validation may require ID (passport photo).
- [ ] 2.2. Once the account is approved: Hetzner Cloud → https://console.hetzner.cloud → _New Project_ → "smart-home-staging".
- [ ] 2.3. In the project → _Security_ → _SSH Keys_ → _Add SSH Key_ → paste the contents of `~/.ssh/id_ed25519.pub` (locally `cat ~/.ssh/id_ed25519.pub`).
- [ ] 2.4. _Servers_ → _Add Server_:
  - Location: **Helsinki (hel1)** or **Nuremberg (nbg1)** — closer to Ukraine in terms of latency.
  - Image: **Ubuntu 24.04**.
  - Type: **CX22** (x86, 2 vCPU, 4 GB RAM, 40 GB disk) — €4.51/mo.
  - Networking: leave both IPv4 and IPv6 enabled.
  - SSH Key: pick the key you just added.
  - Name: `cloud-staging`.
  - _Create & Buy Now_.
- [ ] 2.5. Wait for status _Running_ (~30 sec). Copy the IPv4 address into your notes: `SERVER_IP=<x.x.x.x>`.

**Verification:** `ssh root@<SERVER_IP>` connects without a password. Exit: `exit`.

---

## Stage 3. DNS records (5 min + up to 60 min propagation)

- [ ] 3.1. Cloudflare → DNS → _Add record_:
  - Type: **A**, Name: `api.staging`, Content: `<SERVER_IP>`, Proxy: **DNS only (grey cloud)**, TTL: Auto.
- [ ] 3.2. Another record:
  - Type: **A**, Name: `wss.staging`, Content: `<SERVER_IP>`, Proxy: **DNS only**, TTL: Auto.
- [ ] 3.3. (Alternative) If you decide to use a single host for HTTP+WS — one `api.staging` record is enough; the WS upgrade will happen on the same domain at the `/ws` path.

**Verification:** `dig api.staging.<DOMAIN> +short` returns `<SERVER_IP>` (may take 1–5 minutes).

---

## Stage 4. VPS hardening (15 min)

All commands run on the VPS over SSH.

- [ ] 4.1. `ssh root@<SERVER_IP>`
- [ ] 4.2. System update:
  ```bash
  apt update && apt -y full-upgrade
  apt -y install ufw fail2ban unattended-upgrades curl gnupg ca-certificates
  ```
- [ ] 4.3. Automatic security updates:
  ```bash
  dpkg-reconfigure --priority=low unattended-upgrades
  ```
- [ ] 4.4. Firewall:
  ```bash
  ufw default deny incoming
  ufw default allow outgoing
  ufw allow 22/tcp
  ufw allow 80/tcp
  ufw allow 443/tcp
  ufw --force enable
  ufw status verbose
  ```
- [ ] 4.5. Disable password SSH (the key is already in place):
  ```bash
  sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
  sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
  systemctl restart ssh
  ```
- [ ] 4.6. Create the deploy user:
  ```bash
  adduser --disabled-password --gecos "" deploy
  usermod -aG sudo deploy
  mkdir -p /home/deploy/.ssh
  cp /root/.ssh/authorized_keys /home/deploy/.ssh/
  chown -R deploy:deploy /home/deploy/.ssh
  chmod 700 /home/deploy/.ssh
  chmod 600 /home/deploy/.ssh/authorized_keys
  echo 'deploy ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/deploy
  ```

**Verification:** `ssh deploy@<SERVER_IP>` works, `sudo whoami` returns `root`.

---

## Stage 5. Docker + Docker Compose (5 min)

As the `deploy` user:

- [ ] 5.1. Install Docker:
  ```bash
  curl -fsSL https://get.docker.com | sudo sh
  sudo usermod -aG docker deploy
  ```
- [ ] 5.2. Re-login so the group takes effect:
  ```bash
  exit
  ssh deploy@<SERVER_IP>
  docker run --rm hello-world
  ```

**Verification:** `docker compose version` returns `Docker Compose version v2.x`.

---

## Stage 6. Staging config (locally, 20 min)

All actions happen in your local `smart-home-cloud` repo.

- [ ] 6.1. Create `docker-compose.staging.yml` (see template at the end of the file).
- [ ] 6.2. Create `Caddyfile` at the repo root:
  ```
  api.staging.<DOMAIN> {
    reverse_proxy cloud:4000
  }
  ```
  (Caddy will fetch a Let's Encrypt TLS cert for this domain on its own.)
- [ ] 6.3. Create `.env.staging` locally (do NOT commit). Contents:
  ```
  NODE_ENV=production
  PORT=4000
  DB_HOST=postgres
  DB_PORT=5432
  DB_USER=smarthome
  DB_PASSWORD=<generate: openssl rand -hex 24>
  DB_NAME=smart_home_cloud
  JWT_SECRET=<openssl rand -hex 32>
  JWT_REFRESH_SECRET=<openssl rand -hex 32>
  JWT_EXPIRES_IN=15m
  JWT_REFRESH_EXPIRES_IN=30d
  GOOGLE_CLIENT_ID=placeholder-not-used-yet
  APPLE_CLIENT_ID=com.andriiprudnikov.smarthomemobile
  ```
- [ ] 6.4. Add `.env.staging` to `.gitignore` (verify it's already covered by `.env.*`).
- [ ] 6.5. Commit Dockerfile + .dockerignore + docker-compose.staging.yml + Caddyfile:
  ```bash
  git add Dockerfile .dockerignore docker-compose.staging.yml Caddyfile
  git commit -m "SHC-XX staging deployment: docker + caddy"
  git push
  ```

**Verification:** `docker build -t cloud-test .` builds locally without errors.

---

## Stage 7. First deploy (manual, 15 min)

- [ ] 7.1. SSH to the server: `ssh deploy@<SERVER_IP>`
- [ ] 7.2. Create the directory:
  ```bash
  mkdir -p ~/smart-home-cloud
  cd ~/smart-home-cloud
  ```
- [ ] 7.3. Clone the repo (via deploy key or PAT):
  ```bash
  # Option with HTTPS + Personal Access Token:
  git clone https://<GITHUB_USER>:<PAT>@github.com/<GITHUB_USER>/smart-home-cloud.git .
  # Or set up a deploy key beforehand.
  ```
- [ ] 7.4. Copy `.env.staging` from your local machine to the server:
  ```bash
  # On the local machine:
  scp .env.staging deploy@<SERVER_IP>:~/smart-home-cloud/.env
  ```
- [ ] 7.5. Edit `Caddyfile` on the server — substitute the actual `<DOMAIN>`:
  ```bash
  nano Caddyfile
  ```
- [ ] 7.6. Bring it up:
  ```bash
  docker compose -f docker-compose.staging.yml --env-file .env up -d --build
  docker compose -f docker-compose.staging.yml logs -f
  ```
- [ ] 7.7. The logs should show `Server listening at http://0.0.0.0:4000` and `Successfully connected to PostgreSQL`. Migrations apply automatically (there's a `runMigrations()` call in server.ts).

**Verification:**

```bash
curl -I https://api.staging.<DOMAIN>/health
# HTTP/2 200
# content-type: application/json

curl https://api.staging.<DOMAIN>/health
# {"status":"ok"}
```

If TLS didn't come up — check the Caddy logs: `docker compose logs caddy`. Common reasons: DNS hasn't propagated yet, or Cloudflare Proxy is enabled (grey/orange cloud — should be **grey**).

---

## Stage 8. GitHub Actions auto-deploy (20 min)

- [ ] 8.1. On the server, generate a separate SSH key for CI:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N ""
  cat ~/.ssh/github_deploy.pub >> ~/.ssh/authorized_keys
  cat ~/.ssh/github_deploy  # ← private key, copy in full
  ```
- [ ] 8.2. In GitHub → `smart-home-cloud` repo → _Settings_ → _Secrets and variables_ → _Actions_ → add:
  - `SSH_HOST` = `<SERVER_IP>`
  - `SSH_USER` = `deploy`
  - `SSH_KEY` = the private key contents (including `-----BEGIN OPENSSH...-----`)
- [ ] 8.3. Create `.github/workflows/deploy-staging.yml` (see template at the end of the file).
- [ ] 8.4. Push: `git push origin develop` → _Actions_ → see the run start → success in ~2–3 minutes.

**Verification:** make a test commit (e.g. tweak `docs/cloud-spec.md`), push, confirm the workflow goes green and `docker compose logs` on the server shows the container restarting.

---

## Stage 9. Monitoring and verification (15 min)

- [ ] 9.1. **UptimeRobot** (free): https://uptimerobot.com → _Add Monitor_ → HTTPS → `https://api.staging.<DOMAIN>/health`, interval 5 min, email alert.
- [ ] 9.2. Check WS:
  ```bash
  # locally:
  npm i -g wscat
  wscat -c wss://api.staging.<DOMAIN>/ws/station
  # Should connect. Close with Ctrl+C.
  ```
- [ ] 9.3. Connect a real station to staging: on the Raspberry Pi set `CLOUD_WSS_URL=wss://api.staging.<DOMAIN>/ws/station` in its `.env`, then restart with `docker compose restart`.
- [ ] 9.4. In the cloud logs `docker compose logs -f cloud` you should see `claim_handshake` from the station.
- [ ] 9.5. (optional) Set up a daily Postgres backup on the server — a cron script with `pg_dump` writing into `~/backups/`. Retention 7 days.

---

## Template: `docker-compose.staging.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${DB_USER}']
      interval: 10s
      timeout: 5s
      retries: 5

  cloud:
    build: .
    restart: unless-stopped
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
    expose:
      - '4000'

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - cloud

volumes:
  pgdata:
  caddy_data:
  caddy_config:
```

## Template: `.github/workflows/deploy-staging.yml`

```yaml
name: Deploy staging

on:
  push:
    branches: [develop]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            set -e
            cd ~/smart-home-cloud
            git fetch origin develop
            git reset --hard origin/develop
            docker compose -f docker-compose.staging.yml --env-file .env up -d --build
            docker image prune -f
```

---

## Final verification (green checks)

- [ ] `https://api.staging.<DOMAIN>/health` → 200 OK
- [ ] TLS certificate is valid (lock icon in the browser)
- [ ] `wscat` connects to WSS
- [ ] Push to `develop` → GitHub Actions green → container rebuilt
- [ ] UptimeRobot reports status "up"
- [ ] (once mobile is ready) Claim QR code works → identity sync arrives at the station
