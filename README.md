# Svaroh Docs

Сайт документации `docs.svaroh.com`, собирается [Docusaurus](https://docusaurus.io/).

## Локальная разработка

```bash
npm ci
npm start          # http://localhost:3333
npm run build      # → build/
npm run serve      # отдать build/ локально
```

## Деплой

CI: `.github/workflows/docs.yml` — на каждый push в `main` (или `workflow_dispatch`) собирает `build/` и rsync'ит на VPS, где Caddy из `smart-home-cloud` отдаёт `docs.svaroh.com` с автоматическим TLS.

### Архитектура

```
GitHub (main)
   │  push
   ▼
GH Actions: npm ci → npm run build → rsync build/ → ~/svaroh-docs/build/ на VPS
                                                        │
                                                        ▼
                                  ~/smart-home-cloud/  ─┤  docker-compose.staging.yml
                                  Caddy (тот же VPS)    │  mount: ../svaroh-docs/build → /srv/docs
                                                        │  Caddyfile: docs.svaroh.com → /srv/docs
                                                        ▼
                                                  https://docs.svaroh.com
```

### GitHub secrets (Settings → Secrets and variables → Actions)

| Имя | Значение |
|-----|----------|
| `SSH_KEY` | Приватный ed25519 ключ деплоя (тот же, что у `smart-home-cloud` — учётка имеет доступ к `~/svaroh-docs/`) |
| `SSH_HOST` | Хост VPS (тот же, что у `smart-home-cloud`) |
| `SSH_USER` | Юзер на VPS |

Опционально: переменная `DOCS_HEALTH_URL` (vars, не secret) — по умолчанию `https://docs.svaroh.com/`.

Environment: `production` — заведи его в Settings → Environments, можно повесить required reviewers, если хочется approve перед деплоем.

### Первичная настройка (один раз)

1. **Cloudflare DNS.** Добавь запись для `svaroh.com`:
   - Type: `A`, Name: `docs`, Content: `<IP того же VPS, что api.staging.svaroh.com>`, Proxy: **DNS only** (серое облако — иначе Caddy не сможет получить Let's Encrypt сертификат через TLS-ALPN/HTTP-01).
   - Или CNAME `docs → api.staging.svaroh.com` тоже DNS only.
2. **GitHub secrets** — добавь три значения выше в репу `svaroh-docs`.
3. **VPS** — папка создастся сама первым деплоем (`mkdir -p ~/svaroh-docs/build` есть в workflow).
4. **smart-home-cloud** — задеплой свежий тег: новый Caddyfile + bind mount `../svaroh-docs/build:/srv/docs:ro` приедут на сервер, Caddy подцепит docs.svaroh.com и закажет сертификат.

### Порядок первого деплоя

1. Сначала push в `svaroh-docs` (создаст `~/svaroh-docs/build/` на сервере).
2. Затем релиз `smart-home-cloud` (`git release vX.Y.Z`) — Caddy перечитает Caddyfile, увидит docs.svaroh.com, выпишет сертификат, начнёт отдавать.

Если сделать наоборот, Caddy упадёт на старте из-за отсутствующего bind-mount источника. На свежем VPS можно один раз руками: `ssh user@host 'mkdir -p ~/svaroh-docs/build'`.

### Откат

Откат деплоя = откат коммита в `main` + повторный run workflow. Билды не версионируются, поэтому "вернуть прошлую версию" = пересобрать из старого коммита.
