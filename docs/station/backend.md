---
title: Backend
sidebar_position: 1
---

# 🖥️ Station Backend

Fastify + PostgreSQL + MQTT bridge + WebSocket. Owns all device data and automation execution.

[Source ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/backend)

## Module Layout {#modules}

```mermaid
graph TD
    App["app.ts"]
    Auth["auth/"]
    DC["device-core/<br/>(DDD)"]
    DB["device-bootstrap/<br/>(BLE)"]
    Au["automations/<br/>(DDD)"]
    Kits["kits/"]
    Cloud["cloud/<br/>+ cloud-sync/"]
    Backup["backup/"]
    EB["event-bus.ts"]

    App --> Auth
    App --> DC
    App --> DB
    App --> Au
    App --> Kits
    App --> Cloud
    App --> Backup

    DC --> EB
    Au --> EB
    DB --> EB
    Kits --> EB
```

## Standard Module Pattern

Most modules follow Routes / Repository / Service:

```
modules/{module}/
  {module}Routes.ts       — Fastify routes (Zod validation)
  {module}Repository.ts   — SQL queries
  {module}Service.ts      — Business logic
```

## DDD Modules: device-core, automations

These two have a richer layout:

```
device-core/
  domain/          — aggregates, events
  application/     — service factory
  infrastructure/  — repository factory
  adapters/        — mqtt, ws, db-writer
  event-bus.ts     — typed EventEmitter
  index.ts         — wiring
```

## Auth Hooks {#auth}

```mermaid
flowchart TD
    R["incoming request"]
    R --> H{preHandler hook}
    H -->|"verifyToken"| JWT["JWT user"]
    H -->|"verifyDeviceToken"| Dev["ESP32 device"]
    H -->|"verifyAgentToken"| Ag["station-agent"]
    JWT --> AU{authorize roles?}
    AU -->|"['owner','admin']"| OK["✅"]
    AU -->|"requireStationRole"| OK
```

## Event Bus

Cross-module events use a typed EventEmitter ([`event-bus.ts`](https://github.com/alphaoflogic-ua/smart-home/blob/develop/packages/backend/src/modules/device-core/event-bus.ts)):

```typescript
eventBus.emit(DEVICE_EVENTS.DEVICE_DELETED, { deviceId, deviceName });
eventBus.on(DEVICE_EVENTS.DEVICE_STATE_CHANGED, (payload) => { /* typed */ });
```

## Cloud Integration {#cloud}

- `cloud/` — owns `cloud_config` (single-row table with stationToken)
- `cloud-sync/` — handles `identity_sync`, `member_added/removed/updated` notifications from Cloud
- `ws/cloudClient.ts` — WSS client to Cloud (claim_handshake or station_auth)

## Reference

- Conventions: see [Fastify backend rules ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/.claude/rules/svaroh/fastify-backend.md)
- Station-specific: [backend.md ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/.claude/rules/backend.md)
