---
title: Station Claiming
sidebar_position: 3
---

# 🎯 Station Claiming

A fresh Raspberry Pi connects to Cloud as **unclaimed** and waits for an owner.

## Bootstrap Sequence {#sequence}

```mermaid
sequenceDiagram
    autonumber
    participant P as 🏠 RPi (fresh)
    participant C as ☁️ Cloud
    participant M as 📱 Mobile
    participant U as 👤 Owner

    Note over P: agent install generates<br/>bootstrap credentials
    P->>C: WSS connect (unclaimed)<br/>claim_handshake
    C->>C: store pending station

    Note over P: smartstation.local shows QR

    U->>M: scan QR
    M->>C: POST /api/stations/claim
    C->>C: validate match
    C->>C: INSERT station + station_member (owner)
    C->>C: issue station token
    C-->>M: 200 { stationId }
    C->>P: WSS message: claimed
    P->>P: persist token locally

    P->>C: WSS reconnect (claimed)<br/>station_auth
    C-->>P: ack
```

## Rules

- **Only owner** uses the claim flow. Other members are added via [invite](#invites).
- Claim token is one-time — invalidated after successful claim.
- Stations stay on passwords (no OAuth on Station side); the `auth_providers` table exists on Station but stays empty.

## Invite Flow {#invites}

```mermaid
sequenceDiagram
    autonumber
    participant O as 👤 Owner
    participant P as 🏠 Station
    participant C as ☁️ Cloud
    participant E as 📧 Email
    participant N as 👤 Newcomer

    O->>P: invite user@example.com (role)
    P->>P: store in outbox (works offline)
    P->>C: WS forward invite (when online)
    C->>E: send invitation email
    N->>C: clicks link → register (OAuth or password)
    C->>C: INSERT station_member (assigned role)
    C->>P: identity_sync (next reconnect)
    P->>P: cache new user identity
```

## Reference

- [stations module ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/modules/stations)
- [cloud-sync (Station side) ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/backend/src/modules/cloud-sync)
