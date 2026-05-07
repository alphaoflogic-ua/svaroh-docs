---
title: OTA Updates
sidebar_position: 3
---

# 📡 OTA Firmware Updates

Over-the-air firmware updates triggered via MQTT command.

## End-to-End Flow {#flow}

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 👨‍💻 Developer
    participant GH as 🐙 GitHub
    participant CI as ⚙️ Actions
    participant FE as 🌐 Frontend
    participant B as 🖥️ Backend
    participant E as 💡 ESP32

    Dev->>GH: git release firmware-esp32-light-v0.1.5
    GH->>CI: tag trigger
    CI->>CI: PlatformIO build
    CI->>GH: publish .bin to public registry

    Note over FE: admin sees new firmware available
    FE->>B: POST /api/firmware/update<br/>{ deviceId, version }
    B->>E: MQTT .../command<br/>{ "action": "ota_update", "url": "https://.../firmware.bin" }
    E->>GH: HTTP GET firmware.bin
    GH-->>E: binary
    E->>E: write to OTA partition
    E->>E: switch boot partition + reboot

    Note over E: boot with new firmware
    E->>B: MQTT .../handshake<br/>{ firmwareVersion: "0.1.5" }
    B-->>E: MQTT .../handshake/ack
```

## Triggering OTA

The backend exposes admin endpoints to push OTA. Built-in `ota_update` command is implemented inside SmartHomeCore — no per-device code needed.

## Reference

- [SmartHomeCore OTA ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/firmware/lib/smart-home-core)
- [Firmware build script ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/scripts/build-firmware.js)
