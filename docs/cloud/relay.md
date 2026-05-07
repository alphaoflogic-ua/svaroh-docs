---
title: Relay (WSS)
sidebar_position: 4
---

# 🔁 Cloud Relay

Cloud doesn't store device data — it proxies device requests from Mobile to the relevant Station via JSON-RPC over WSS.

## Topology {#topology}

```mermaid
graph LR
    M["📱 Mobile"]
    H["🌐 HTTPS<br/>/api/devices/*"]
    R["🚛 Route handler"]
    P["📞 peer.call()"]
    S["🏠 Station"]

    M -->|HTTP| H
    H --> R
    R --> P
    P -->|WSS request| S
    S -->|WSS response| P
    P --> R
    R -->|HTTP| M
```

## Lifecycle {#lifecycle}

```mermaid
sequenceDiagram
    autonumber
    participant M as 📱 Mobile
    participant C as ☁️ Cloud
    participant S as 🏠 Station

    Note over S,C: WSS already established (station_auth)

    M->>C: HTTP POST /api/devices/:id/command
    C->>C: lookup station for device
    C->>S: WS { id: 42, method: "device.command", params: {...} }
    S->>S: execute locally (MQTT → ESP32)
    S-->>C: WS { id: 42, result: { ok: true } }
    C-->>M: HTTP 200 { ok: true }
```

## Primitives

| Method | Returns | Use case |
|---|---|---|
| `peer.call(method, params)` | `Promise<result>` | Request/response (waits for matching `id`) |
| `peer.notify(method, params)` | `void` | Fire-and-forget (no `id`) |

If the Station is offline, `peer.call()` rejects after timeout — Cloud returns 503 to Mobile.

## Reference

- [jsonrpc protocol ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/jsonrpc)
- [devices module (proxy only) ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/modules/devices)
- [ws/ ↗](https://github.com/alphaoflogic-ua/smart-home-cloud/tree/develop/src/ws)
