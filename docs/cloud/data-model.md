---
title: Data Model
sidebar_position: 1
---

# 🗄️ Cloud Data Model

```mermaid
erDiagram
    USER ||--o{ AUTH_PROVIDER : "linked via"
    USER ||--o{ STATION_MEMBER : "has membership"
    USER ||--o{ PUSH_DEVICE : "has device"
    STATION ||--o{ STATION_MEMBER : "has members"
    STATION ||--o| LICENSE : "has subscription"
    STATION ||--o{ CLAIM_TOKEN : "issued"

    USER {
        uuid id
        string email
        string password_hash
    }
    AUTH_PROVIDER {
        uuid user_id
        string provider "google|apple"
        string provider_user_id
    }
    STATION {
        uuid id
        string chip_id
        string station_token
    }
    STATION_MEMBER {
        uuid user_id
        uuid station_id
        string role "owner|admin|member"
    }
    LICENSE {
        uuid station_id
        string state "trial|active|cancelled"
    }
    CLAIM_TOKEN {
        uuid station_id
        string token
        timestamp expires_at
        timestamp used_at
    }
    PUSH_DEVICE {
        uuid user_id
        string token "FCM|APNs"
    }
```

## Entities {#entities}

| Entity | Description |
|---|---|
| **User** | Cloud account. One person = one account. Auth via OAuth (Google, Apple) or email/password. `password_hash` is **always** set — OAuth users must set a password (required for Station LAN access offline). |
| **AuthProvider** | OAuth link. Auto-linked by email when a password user later signs in via OAuth. |
| **Station** | One Raspberry Pi. Identified by `station_id` generated on first boot. |
| **StationMember** | User-station link. Roles: `owner` (full), `admin` (devices), `member` (view). |
| **License** | Subscription tied to Station. States: `trial → active → cancelled`. Without active license, cloud features stop; local features keep working. |
| **ClaimToken** | One-time code for claiming. Stored in `claim_tokens`, invalidated after use. |
| **PushDevice** | FCM/APNs token tied to a user. One user = many devices. |

## Reference

- [Cloud spec ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/blob/develop/docs/cloud-spec.md)
- [Migrations ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/db/migrations)
