---
title: Cloud Deployment
sidebar_position: 2
---

# ☁️ Cloud Staging Deployment

[Link to source repo ↗](https://github.com/alphaoflogic-ua/smart-home-cloud)

Цель: развернуть `smart-home-cloud` на публичном VPS с HTTPS/WSS, авто-деплоем из GitHub, готовый к подключению станций и iOS-приложения.

**Ориентир по времени:** 3–4 часа активной работы + до 1 часа ожидания DNS/TLS.
**Стартовый бюджет:** ~$15 (домен $10 + первый месяц VPS $5).

---

## Stage 0. Предусловия

- [ ] 0.1. GitHub аккаунт с приватным репо `smart-home-cloud` запушен в ветку `develop` (или `main`).
- [ ] 0.2. Локальный SSH-ключ. Проверить: `ls ~/.ssh/id_ed25519.pub`. Если нет — `ssh-keygen -t ed25519 -C "alphaoflogic@gmail.com"` (Enter на все вопросы).
- [ ] 0.3. Международная карта для оплаты (Visa/MC, Wise/Monobank FX). Проверить лимит минимум $20.
- [ ] 0.4. Установлен `docker` и `docker compose` локально (для первого smoke-билда): `docker --version`, `docker compose version`.
- [ ] 0.5. Установлен `curl` и `dig`/`nslookup`: `which curl dig`.

---

## Stage 1. Домен (15 мин + 5–60 мин ожидания DNS)

- [ ] 1.1. Открыть https://www.cloudflare.com/products/registrar/ (или https://www.namecheap.com если Cloudflare не принимает карту).
- [ ] 1.2. Придумать имя: `<slug>.com` (например `smartstation-lab.com`). Проверить доступность.
- [ ] 1.3. Оплатить регистрацию на 1 год. **Сразу включить auto-renew отключи** — чтобы не списали второй год автоматически.
- [ ] 1.4. В Cloudflare: Домен → DNS → пока пусто. Включить **Full (strict)** SSL mode в _SSL/TLS → Overview_, но **выключить Proxy (серое облако)** для `api.staging.*` — Cloudflare Proxy ломает WebSocket от станций.
- [ ] 1.5. Записать домен в блокнот: `DOMAIN=<выбранный домен>`. Дальше везде буду писать `<DOMAIN>`.

**Проверка:** `dig <DOMAIN> NS +short` показывает NS-сервера Cloudflare (или Namecheap).

---

## Stage 2. VPS на Hetzner (20 мин)

- [ ] 2.1. Регистрация: https://accounts.hetzner.com/signUp. Нужен email + карта. Первая валидация может потребовать ID (паспорт фото).
- [ ] 2.2. После одобрения аккаунта: Hetzner Cloud → https://console.hetzner.cloud → _New Project_ → "smart-home-staging".
- [ ] 2.3. В проекте → _Security_ → _SSH Keys_ → _Add SSH Key_ → скопируй содержимое `~/.ssh/id_ed25519.pub` (локально `cat ~/.ssh/id_ed25519.pub`).
- [ ] 2.4. _Servers_ → _Add Server_:
  - Location: **Helsinki (hel1)** или **Nuremberg (nbg1)** — ближе к Украине по латентности.
  - Image: **Ubuntu 24.04**.
  - Type: **CX22** (x86, 2 vCPU, 4 GB RAM, 40 GB disk) — €4.51/мес.
  - Networking: IPv4 + IPv6 оставить включёнными.
  - SSH Key: выбрать добавленный ключ.
  - Name: `cloud-staging`.
  - _Create & Buy Now_.
- [ ] 2.5. Дождаться статуса _Running_ (~30 сек). Скопировать IPv4 адрес в блокнот: `SERVER_IP=<x.x.x.x>`.

**Проверка:** `ssh root@<SERVER_IP>` подключается без пароля. Выйти: `exit`.

---

## Stage 3. DNS записи (5 мин + до 60 мин пропагация)

- [ ] 3.1. Cloudflare → DNS → _Add record_:
  - Type: **A**, Name: `api.staging`, Content: `<SERVER_IP>`, Proxy: **DNS only (серое облако)**, TTL: Auto.
- [ ] 3.2. Еще одна запись:
  - Type: **A**, Name: `wss.staging`, Content: `<SERVER_IP>`, Proxy: **DNS only**, TTL: Auto.
- [ ] 3.3. (Альтернатива) Если решишь использовать один хост для HTTP+WS — достаточно одной записи `api.staging`, WS upgrade будет на том же домене по пути `/ws`.

**Проверка:** `dig api.staging.<DOMAIN> +short` возвращает `<SERVER_IP>` (может потребоваться 1–5 минут).

---

## Stage 4. Hardening VPS (15 мин)

Все команды — на VPS через SSH.

- [ ] 4.1. `ssh root@<SERVER_IP>`
- [ ] 4.2. Обновление системы:
  ```bash
  apt update && apt -y full-upgrade
  apt -y install ufw fail2ban unattended-upgrades curl gnupg ca-certificates
  ```
- [ ] 4.3. Автообновления безопасности:
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
- [ ] 4.5. Отключить парольный SSH (ключ уже есть):
  ```bash
  sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
  sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
  systemctl restart ssh
  ```
- [ ] 4.6. Создать деплой-юзера:
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

**Проверка:** `ssh deploy@<SERVER_IP>` работает, `sudo whoami` возвращает `root`.

---

## Stage 5. Docker + Docker Compose (5 мин)

Под `deploy` юзером:

- [ ] 5.1. Установить Docker:
  ```bash
  curl -fsSL https://get.docker.com | sudo sh
  sudo usermod -aG docker deploy
  ```
- [ ] 5.2. Перезайти чтобы группа применилась:
  ```bash
  exit
  ssh deploy@<SERVER_IP>
  docker run --rm hello-world
  ```

**Проверка:** `docker compose version` возвращает `Docker Compose version v2.x`.

---

## Stage 6. Конфиг staging (локально, 20 мин)

Все действия — в твоём локальном репо `smart-home-cloud`.

- [ ] 6.1. Создать `docker-compose.staging.yml` (см. шаблон в конце файла).
- [ ] 6.2. Создать `Caddyfile` в корне репо:
  ```
  api.staging.<DOMAIN> {
    reverse_proxy cloud:4000
  }
  ```
  (Caddy сам возьмёт Let's Encrypt TLS для этого домена.)
- [ ] 6.3. Создать `.env.staging` локально (НЕ коммитить). Содержимое:
  ```
  NODE_ENV=production
  PORT=4000
  DB_HOST=postgres
  DB_PORT=5432
  DB_USER=smarthome
  DB_PASSWORD=<сгенерировать: openssl rand -hex 24>
  DB_NAME=smart_home_cloud
  JWT_SECRET=<openssl rand -hex 32>
  JWT_REFRESH_SECRET=<openssl rand -hex 32>
  JWT_EXPIRES_IN=15m
  JWT_REFRESH_EXPIRES_IN=30d
  GOOGLE_CLIENT_ID=placeholder-not-used-yet
  APPLE_CLIENT_ID=com.andriiprudnikov.smarthomemobile
  ```
- [ ] 6.4. Добавить `.env.staging` в `.gitignore` (проверить что уже матчится `.env.*`).
- [ ] 6.5. Закоммитить Dockerfile + .dockerignore + docker-compose.staging.yml + Caddyfile:
  ```bash
  git add Dockerfile .dockerignore docker-compose.staging.yml Caddyfile
  git commit -m "SHC-XX staging deployment: docker + caddy"
  git push
  ```

**Проверка:** `docker build -t cloud-test .` локально собирается без ошибок.

---

## Stage 7. Первый деплой (ручной, 15 мин)

- [ ] 7.1. SSH на сервер: `ssh deploy@<SERVER_IP>`
- [ ] 7.2. Создать директорию:
  ```bash
  mkdir -p ~/smart-home-cloud
  cd ~/smart-home-cloud
  ```
- [ ] 7.3. Клонировать репо (через deploy key или PAT):
  ```bash
  # Вариант с HTTPS + Personal Access Token:
  git clone https://<GITHUB_USER>:<PAT>@github.com/<GITHUB_USER>/smart-home-cloud.git .
  # Или настроить deploy key заранее.
  ```
- [ ] 7.4. Скопировать `.env.staging` с локальной машины на сервер:
  ```bash
  # На локальной машине:
  scp .env.staging deploy@<SERVER_IP>:~/smart-home-cloud/.env
  ```
- [ ] 7.5. Отредактировать `Caddyfile` на сервере — подставить реальный `<DOMAIN>`:
  ```bash
  nano Caddyfile
  ```
- [ ] 7.6. Поднять:
  ```bash
  docker compose -f docker-compose.staging.yml --env-file .env up -d --build
  docker compose -f docker-compose.staging.yml logs -f
  ```
- [ ] 7.7. В логах должно появиться `Server listening at http://0.0.0.0:4000` и `Successfully connected to PostgreSQL`. Миграции применятся автоматически (есть `runMigrations()` в server.ts).

**Проверка:**

```bash
curl -I https://api.staging.<DOMAIN>/health
# HTTP/2 200
# content-type: application/json

curl https://api.staging.<DOMAIN>/health
# {"status":"ok"}
```

Если TLS не поднялся — Caddy логи: `docker compose logs caddy`. Частые причины: DNS ещё не пропагирован, или включён Cloudflare Proxy (серое/оранжевое облако — должно быть **серое**).

---

## Stage 8. GitHub Actions auto-deploy (20 мин)

- [ ] 8.1. На сервере сгенерировать отдельный SSH-ключ для CI:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N ""
  cat ~/.ssh/github_deploy.pub >> ~/.ssh/authorized_keys
  cat ~/.ssh/github_deploy  # ← приватный ключ, скопируй целиком
  ```
- [ ] 8.2. В GitHub → репо `smart-home-cloud` → _Settings_ → _Secrets and variables_ → _Actions_ → добавить:
  - `SSH_HOST` = `<SERVER_IP>`
  - `SSH_USER` = `deploy`
  - `SSH_KEY` = содержимое приватного ключа (включая `-----BEGIN OPENSSH...-----`)
- [ ] 8.3. Создать `.github/workflows/deploy-staging.yml` (см. шаблон в конце файла).
- [ ] 8.4. Пуш: `git push origin develop` → _Actions_ → видно запуск → успех за ~2–3 минуты.

**Проверка:** сделать тестовый коммит (напр. правка `docs/cloud-spec.md`), запушить, убедиться что workflow зелёный и на сервере `docker compose logs` показывает рестарт контейнера.

---

## Stage 9. Мониторинг и verification (15 мин)

- [ ] 9.1. **UptimeRobot** (free): https://uptimerobot.com → _Add Monitor_ → HTTPS → `https://api.staging.<DOMAIN>/health`, interval 5 min, email алерт.
- [ ] 9.2. Проверить WS:
  ```bash
  # локально:
  npm i -g wscat
  wscat -c wss://api.staging.<DOMAIN>/ws/station
  # Должно подключиться. Закрыть Ctrl+C.
  ```
- [ ] 9.3. Подключить к staging реальную станцию: на Raspberry Pi установить `CLOUD_WSS_URL=wss://api.staging.<DOMAIN>/ws/station` в её `.env`, перезапустить `docker compose restart`.
- [ ] 9.4. В логах cloud `docker compose logs -f cloud` увидеть `claim_handshake` от станции.
- [ ] 9.5. (опц) Настроить ежедневный бэкап Postgres на сервере — cron-скрипт с `pg_dump` в `~/backups/`. Retention 7 дней.

---

## Шаблон: `docker-compose.staging.yml`

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

## Шаблон: `.github/workflows/deploy-staging.yml`

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

## Итоговая проверка (green checks)

- [ ] `https://api.staging.<DOMAIN>/health` → 200 OK
- [ ] TLS сертификат валидный (замочек в браузере)
- [ ] `wscat` подключается к WSS
- [ ] Push в `develop` → GitHub Actions зелёный → контейнер пересобран
- [ ] UptimeRobot шлёт статус "up"
- [ ] (когда будет mobile) Claim QR-код работает → identity sync приходит на станцию
