---
title: Data Model
sidebar_position: 1
---

# 🗄️ Cloud Data Model

## Entities {#entities}

| Entity | Description |
|---|---|
| **users** | Cloud account. One person = one account. Auth via OAuth (Google, Apple) or email/password. |
| **auth_providers** | OAuth provider links per user. |
| **stations** | One Raspberry Pi per row, identified on first boot and authorized after claim. |
| **station_members** | User–station link with role (owner / admin / member). |
| **licenses** | Subscription tied to a station. Without an active license, cloud features stop; local keeps working. |
| **refresh_tokens** | Session refresh tokens with rotation. |
| **schema_migrations** | Migration tracking. |

:::note
Column-level details, token TTLs and rotation mechanics are kept in the internal runbook, not here.
:::

## Relationships

- one user ↔ many OAuth provider links
- many users ↔ many stations (via `station_members`)
- one station ↔ one license
- one user ↔ many refresh tokens (multiple device sessions)
