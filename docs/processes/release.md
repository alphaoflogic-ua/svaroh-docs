---
title: Release Flow
sidebar_position: 1
---

# 🚀 Release Flow

All repos use a unified release script: `git release <tag>` (alias for `scripts/release.sh`). **Never edit `package.json` versions or create tags manually.**

## Pipeline

```mermaid
flowchart TD
    Dev["👨‍💻 git release v0.2.77"]
    Script["📜 scripts/release.sh"]
    Bump["📝 bump package.json versions"]
    Commit["💾 git commit"]
    Tag["🏷️ git tag"]
    Push["⬆️ git push --tags"]
    GHA["⚙️ GitHub Actions"]

    Dev --> Script
    Script --> Bump
    Bump --> Commit
    Commit --> Tag
    Tag --> Push
    Push --> GHA

    GHA --> TagType{Tag pattern?}
    TagType -->|"v*"| App["🐳 Docker Hub<br/>(backend + frontend)"]
    TagType -->|"backend-v*"| Backend["🐳 Docker Hub<br/>(backend only)"]
    TagType -->|"frontend-v*"| Frontend["🐳 Docker Hub<br/>(frontend only)"]
    TagType -->|"firmware-*"| FW["📦 Public registry<br/>(.bin files)"]

    App --> Pull["🤖 station-agent pulls"]
    Backend --> Pull
    Frontend --> Pull
    FW --> OTA["📡 OTA via MQTT"]
```

## Tag Patterns by Repo

| Repo | Tag examples | What gets built |
|---|---|---|
| [`smart-home`](https://github.com/alphaoflogic-ua/smart-home) | `v0.2.77`, `backend-v...`, `frontend-v...`, `firmware-esp32-climate-v...` | Docker Hub (app) / public registry (firmware) |
| [`smart-home-cloud`](https://github.com/alphaoflogic-ua/smart-home-cloud) | `v0.2.1` | Cloud deploy |
| [`smart-home-mobile`](https://github.com/alphaoflogic-ua/smart-home-mobile) | `v1.3.1` | EAS build (TestFlight) |

## Default Branches & Jira Prefixes

| Repo | Branch | Jira |
|---|---|---|
| `smart-home` | `develop` | `SHS-` |
| `smart-home-cloud` | `develop` | `SHC-` |
| `smart-home-mobile` | `main` | `SHM-` |

The Jira prefix is **mandatory** in commit messages: `SHC-63 add password reset`.

## Mobile Caveat

For `smart-home-mobile`, `release.sh` syncs `package.json` ↔ `app.json` (`expo.version`) before tagging — they must match for EAS Build to accept the version.
