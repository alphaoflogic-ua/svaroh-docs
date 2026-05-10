---
title: Mobile Deployment (TestFlight)
sidebar_position: 3
---

# 📱 Mobile — iOS Distribution

[Source repo ↗](https://github.com/alphaoflogic-ua/smart-home-mobile)

App is live in TestFlight. Builds automatically on every `v*.*.*` tag via GitHub Actions + EAS.

---

## App identifiers

| Field | Value |
|-------|-------|
| App name | Svaroh |
| Bundle ID (iOS + Android) | `com.andriiprudnikov.svaroh` |
| URL scheme | `svaroh://` |
| Apple Team ID | `<APPLE_TEAM_ID>` |
| Apple ID (submit account) | `<APPLE_SUBMIT_EMAIL>` |
| ASC App ID | `<ASC_APP_ID>` |
| EAS project ID | `<EAS_PROJECT_ID>` |
| EAS owner | `<EAS_OWNER>` |

:::note
Real values live in the team password manager / EAS dashboard, not in this repo.
:::

---

## Cloud endpoints (staging)

| Purpose | URL |
|---------|-----|
| REST API | `https://api.staging.svaroh.com` |
| WebSocket | `wss://api.staging.svaroh.com/ws` |
| Universal Links host | `https://app.staging.svaroh.com` |

---

## Deploy flow

Triggered by pushing a `v*.*.*` tag (via `git release v1.3.1`).

Steps performed by CI (`.github/workflows/release-ios.yml`):
1. Extract version from tag, write into `app.json` (`expo.version`)
2. `eas build --platform ios --profile production --non-interactive --wait`
3. `eas submit --platform ios --profile production --latest --non-interactive` → TestFlight

Build is available in TestFlight in ~30–40 min after the tag.

**GitHub secret required:** `EXPO_TOKEN`

To release:

```bash
git release v1.3.2   # uses scripts/release.sh — never tag manually
```

---

## EAS build profiles

| Profile | Distribution | Use |
|---------|-------------|-----|
| `development` | internal (simulator) | local dev with dev client |
| `preview` | internal (device) | ad-hoc testing on a real device |
| `production` | App Store | TestFlight + App Store submission |

Production profile uses `autoIncrement: true` (build number managed remotely by EAS).

---

## Universal Links / Deep Links

iOS Associated Domains: `applinks:app.staging.svaroh.com`

| Path | Purpose |
|------|---------|
| `/reset-password/*` | Password reset link from email |
| `/invite/*` | Station invite link |

Both paths served by Caddy at `app.staging.svaroh.com` → redirects to `landing/install.html`.  
AASA file: `https://app.staging.svaroh.com/.well-known/apple-app-site-association`

---

## OTA updates (JS-only changes)

For UI fixes and logic changes that don't require a native rebuild:

```bash
eas update --branch production --message "fix: описание изменения"
```

Or trigger manually via GitHub Actions → _OTA Update_ workflow.

**OTA vs full build:**
- OTA: UI changes, bug fixes, new screens, translations
- Full build: new native modules, Expo SDK upgrade, `app.json` changes, new permissions

---

## Day-to-day workflow

```
git release v1.3.2
  → GitHub Actions
  → eas build (ios, production)
  → eas submit → TestFlight
  → ready in ~30–40 min
```