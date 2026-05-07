---
title: Backend
sidebar_position: 2
---

# 🖥️ Station Backend

Fastify + PostgreSQL + MQTT bridge + WebSocket. Owns all device data, runs the automation engine, bridges MQTT ↔ HTTP/WS, and synchronises identity from Cloud.

[Source ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/backend)

## Pages

- [Overview](/station/backend/overview) — modules, DB schema, conventions, boot sequence
- [Domain](/station/backend/domain) — device-core, automations, kits, event bus
- [Integrations](/station/backend/integrations) — BLE provisioning, MQTT, cloud-sync, auth
