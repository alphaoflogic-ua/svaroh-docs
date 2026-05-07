---
title: MQTT
sidebar_position: 1
---

# 📡 MQTT Protocol

The Station's backend runs an MQTT broker. ESP32 devices connect to it for telemetry and commands.

## Topic Map {#topics}

```mermaid
graph LR
    E["💡 ESP32"]
    B["🖥️ Backend"]

    E -->|".../state"| B
    E -->|".../event"| B
    E -->|".../heartbeat"| B
    E -->|".../handshake"| B
    B -->|".../handshake/ack"| E
    B -->|".../command"| E

    classDef d2b fill:#dff,stroke:#08a
    classDef b2d fill:#ffd,stroke:#a80
```

## Topic Reference

All topics are scoped: `station/{stationId}/device/{deviceId}/<suffix>`.

| Direction | Topic | Payload |
|---|---|---|
| Device → Backend | `.../state` | `{"state": {...}}` |
| Device → Backend | `.../event` | `{"event": "device_event"\|"user_event", "data": {"action":"..."}}` |
| Device → Backend | `.../heartbeat` | `{"status": "online"}` (auto by SmartHomeCore) |
| Device → Backend | `.../handshake` | `{"type":"...","firmwareVersion":"...","chip":"..."}` |
| Backend → Device | `.../handshake/ack` | `{"status":"ok"}` or `{"status":"unknown_device"}` |
| Backend → Device | `.../command` | `{"action":"...", ...}` |

## Payloads {#payloads}

### State

`{"state": { ... }}` — JSON object with sensor/actuator values. Keys match capability names from the [Device Type Registry](/firmware/device-types).

### Event

```json
{"event": "device_event", "data": {"action": "motion_detected"}}
{"event": "user_event",   "data": {"action": "button_press"}}
```

### Command

```json
{"action": "toggle"}
{"action": "set_brightness", "brightness": 75}
```

Built-in commands (handled by SmartHomeCore): `reboot`, `factory_reset`, `ota_update`.

## Lifecycle {#lifecycle}

```mermaid
sequenceDiagram
    autonumber
    participant E as 💡 ESP32
    participant B as 🖥️ Backend

    Note over E: boot
    E->>B: connect (with LWT preset)
    E->>B: .../handshake
    B->>B: lookup device_type
    alt device known
        B-->>E: .../handshake/ack { status: "ok" }
        loop telemetry
            E->>B: .../state or .../event
            E->>B: .../heartbeat (periodic)
        end
        B->>E: .../command (when user acts)
    else device unknown
        B-->>E: .../handshake/ack { status: "unknown_device" }
    end
    Note over E,B: on disconnect → LWT publishes<br/>{"status": "offline"} to heartbeat
```

## LWT (Last Will and Testament) {#lwt}

`SmartHomeCore` sets LWT on MQTT connect — publishes `{"status":"offline"}` to the heartbeat topic on unexpected disconnect, so Backend can mark the device offline immediately.

## Reference

- [Source: backend MQTT module](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/backend/src/modules/device-core/adapters)
- [Source: SmartHomeCore](https://github.com/alphaoflogic-ua/smart-home/tree/develop/firmware/lib/smart-home-core)
