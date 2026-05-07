---
title: Authentication
sidebar_position: 2
---

# 🔐 Authentication

Cloud handles all user authentication. Stations cache identity locally for LAN-offline operation.

## Registration

### Email + Password {#email-register}

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User
    participant M as 📱 Mobile
    participant C as ☁️ Cloud
    participant DB as 🗄️ Postgres

    U->>M: signup form
    M->>C: POST /api/auth/register<br/>{ email, password }
    C->>C: hash password (bcrypt)
    C->>DB: INSERT user
    C-->>M: { accessToken, refreshToken }
```

### OAuth (Google / Apple) {#oauth-register}

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User
    participant M as 📱 Mobile
    participant OP as 🔑 OAuth Provider
    participant C as ☁️ Cloud
    participant DB as 🗄️ Postgres

    U->>M: tap "Sign in with Google"
    M->>OP: OAuth flow
    OP-->>M: idToken
    M->>C: POST /api/auth/oauth/google { idToken }
    C->>OP: verify token
    OP-->>C: { email, providerUserId }

    alt user exists by email
        C->>DB: INSERT auth_providers (auto-link)
    else new user
        C->>DB: INSERT user (no password yet)
        C->>DB: INSERT auth_providers
        Note over C,M: response flag: "set password required"
    end
    C-->>M: { accessToken, refreshToken }
```

:::warning OAuth users must set a password
A password is **required** for Station LAN access without internet (Pi can't reach OAuth providers offline). Mobile prompts the user to set one after OAuth registration.
:::

## Login {#login}

- **Password:** standard email/password — works for all users regardless of registration method.
- **OAuth:** match by `(provider, provider_user_id)` in `auth_providers`.
- **Auto-linking:** if a user registered with password and later signs in with Google, the email match adds an `auth_providers` row automatically.

## Identity Cache on Station {#cache}

```mermaid
sequenceDiagram
    autonumber
    participant C as ☁️ Cloud
    participant S as 🏠 Station
    participant U as 👤 User (LAN)

    Note over C: member added/removed/updated
    C->>S: WSS notify identity_sync<br/>{ users: [...] }
    S->>S: upsert / delete cached users
    Note over S: TTL 14 days

    U->>S: LAN login attempt
    S->>S: verify against cache
    S-->>U: ✅ session (works offline)
```

First-time login on Station requires internet (redirect to Cloud OAuth). Subsequent LAN logins work offline up to TTL.

## Password Reset {#reset}

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User
    participant M as 📱 Mobile
    participant C as ☁️ Cloud
    participant E as 📧 Email

    U->>M: "forgot password"
    M->>C: POST /api/auth/password/reset/request { email }
    C->>C: generate reset_token (TTL ~1h)
    C->>E: send link with token
    U->>M: opens link
    M->>C: POST /api/auth/password/reset/confirm<br/>{ token, newPassword }
    C->>C: validate, update password_hash, invalidate token
    C-->>M: 200
```

## JWT Lifecycle

- `accessToken` — short-lived (~15 min), Bearer in `Authorization` header
- `refreshToken` — long-lived (~14 days), refreshed via `POST /api/auth/refresh`
- Refresh token rotation: each refresh issues a new pair, old refresh is invalidated

## Reference

- [auth module ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/modules/auth)
- [authHooks ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/blob/develop/src/hooks/authHooks.ts)
