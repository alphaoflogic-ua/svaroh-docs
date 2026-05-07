---
title: Data Model
sidebar_position: 1
---

# 🗄️ Cloud Data Model

![Cloud DB Schema](/img/db-schema-cloud.png)

## Entities {#entities}

| Table | Description |
|---|---|
| **users** | Cloud account. One person = one account. Auth via OAuth (Google, Apple) or email/password. `password_hash` is set for both methods — OAuth users still need a password for Station LAN access offline. `station_password_hash` is a separate hash synced to stations for local LAN auth. |
| **auth_providers** | OAuth links. Stores `provider` (google/apple), `provider_user_id`, and the email returned by the provider. Auto-linked by email when a password user later signs in via OAuth. |
| **stations** | One Raspberry Pi. Identified by `hardware_id` generated on first boot, plus `station_token` issued on claim. `last_seen_at` updated on every WSS heartbeat. |
| **station_members** | User-station link. Roles: `owner` (full), `admin` (devices), `member` (view). PK = `(station_id, user_id)`. |
| **licenses** | Subscription tied to a station. States: `trial → active → cancelled`. Without active license, cloud features stop; local features keep working. |
| **refresh_tokens** | JWT refresh tokens. `family_id` groups tokens from a single login session for rotation tracking — reuse of a stale token invalidates the whole family. |
| **schema_migrations** | Migration tracking — `filename` + `applied_at`. |

## Relationships

- `users 1—N auth_providers` — one user, many OAuth links
- `users 1—N station_members` — user can be in many stations
- `stations 1—N station_members` — station has many members
- `stations 1—1 licenses` — license per station
- `users 1—N refresh_tokens` — multiple device sessions

## Reference

- [Migrations ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/db/migrations)
- [Cloud spec ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/blob/develop/docs/cloud-spec.md)
