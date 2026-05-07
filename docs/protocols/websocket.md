---
title: WebSocket
sidebar_position: 2
---

# 🔌 WebSocket Protocol

## Cloud ↔ Station

The Station maintains a persistent WSS connection to Cloud. Two modes:

```mermaid
stateDiagram-v2
    [*] --> Unclaimed: install-agent.sh<br/>chipId + claimSecret
    Unclaimed --> Claimed: mobile claim<br/>(stationToken issued)
    Claimed --> Unclaimed: factory reset

    state Unclaimed {
        [*] --> claim_handshake
        claim_handshake: claim_handshake<br/>{ chipId, claimSecret }
    }
    state Claimed {
        [*] --> station_auth
        station_auth: station_auth<br/>{ stationToken }
    }
```

## JSON-RPC over WS {#jsonrpc}

Cloud and Station communicate via JSON-RPC. Cloud uses `peer.call()` to proxy mobile requests:

```mermaid
sequenceDiagram
    autonumber
    participant Mobile
    participant Cloud
    participant Station

    Mobile->>Cloud: HTTP request
    Cloud->>Station: WS { id, method, params } (request)
    Station->>Station: handle locally
    Station-->>Cloud: WS { id, result } (response)
    Cloud-->>Mobile: HTTP 200
```

Two primitives:

- `peer.call(method, params)` → returns `Promise<result>` (waits for matching `id`)
- `peer.notify(method, params)` → fire-and-forget (no `id`)

## Backend ↔ Frontend (Local) {#frontend-ws}

The Station Backend exposes a local WebSocket for the SPA frontend. Used for:

- Device state push (telemetry → UI in real time)
- Provisioning candidate updates (BLE flow)
- Connection status

Frontend subscribes via `useWsSubscription` hook ([source](https://github.com/alphaoflogic-ua/smart-home/blob/develop/packages/frontend/src/shared/useWsSubscription.ts)).

## Reference

- [Source: cloud WS](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/ws)
- [Source: cloud-sync (Station side)](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/backend/src/modules/cloud-sync)
- [Source: jsonrpc protocol](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/jsonrpc)
