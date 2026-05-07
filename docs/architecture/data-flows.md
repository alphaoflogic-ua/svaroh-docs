---
title: Data Flows
sidebar_position: 2
---

# 🌊 Data Flows

Sequence diagrams for the system's major operations.

## 1. Mobile Controls a Device {#device-control}

The mobile app sends a command (e.g. toggle a light). Mobile never reaches the Station directly — Cloud relays via WSS.

```mermaid
sequenceDiagram
    autonumber
    participant M as 📱 Mobile
    participant C as ☁️ Cloud
    participant S as 🏠 Station Backend
    participant E as 💡 ESP32

    M->>C: POST /api/devices/:id/command<br/>{ "action": "toggle" }
    C->>S: WSS peer.call("device.command", payload)
    S->>E: MQTT station/{sid}/device/{did}/command<br/>{ "action": "toggle" }
    E->>E: actuate hardware
    E-->>S: MQTT station/{sid}/device/{did}/state<br/>{ "state": { "on": true } }
    S-->>C: WS push: state changed
    C-->>M: WS push: state changed
    S-->>C: peer.call() resolves
    C-->>M: HTTP 200
```

## 2. Device Telemetry {#telemetry}

ESP32 publishes sensor readings; the system fans them out to all connected clients.

```mermaid
sequenceDiagram
    autonumber
    participant E as 💡 ESP32
    participant S as 🏠 Station Backend
    participant F as 🌐 Frontend
    participant C as ☁️ Cloud
    participant M as 📱 Mobile

    E->>S: MQTT .../state<br/>{ "state": { "temperature": 23.5 } }
    S->>S: persist + dispatch event
    par fan-out
        S-->>F: WS broadcast (local UI)
    and
        S-->>C: WS device state update
        C-->>M: WS push (remote UI)
    end
```

## 3. Station Claiming {#claiming}

A fresh Raspberry Pi connects to Cloud as **unclaimed**. The owner scans a QR code in the mobile app to claim it.

```mermaid
sequenceDiagram
    autonumber
    participant P as 🏠 RPi (fresh boot)
    participant C as ☁️ Cloud
    participant M as 📱 Mobile

    Note over P: install-agent.sh ran<br/>chipId + claimSecret generated
    P->>C: WSS connect (unclaimed)<br/>claim_handshake { chipId, claimSecret }
    C->>C: register pending station
    Note over P: smartstation.local shows QR<br/>(chipId + claimSecret)

    M->>M: user scans QR
    M->>C: POST /api/stations/claim<br/>{ chipId, claimSecret }
    C->>C: validate → create Station record<br/>+ owner membership
    C-->>M: HTTP 200 { stationId }
    C->>P: WSS upgrade message<br/>{ stationToken }
    P->>P: persist token in cloud_config

    P->>C: WSS reconnect (claimed)<br/>station_auth { stationToken }
    C-->>P: ack
```

## 4. OTA Firmware Update {#ota}

Developer cuts a firmware release tag; CI publishes; Station instructs ESP32 to flash over MQTT.

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 👨‍💻 Developer
    participant GH as 🐙 GitHub
    participant CI as ⚙️ GitHub Actions
    participant S as 🏠 Station Backend
    participant E as 💡 ESP32

    Dev->>GH: git release firmware-esp32-climate-v0.1.5
    GH->>CI: tag trigger
    CI->>CI: build firmware (PlatformIO)
    CI->>GH: publish .bin to public registry

    Note over S: admin issues OTA from frontend
    S->>E: MQTT .../command<br/>{ "action": "ota_update", "url": "..." }
    E->>GH: HTTP GET firmware.bin
    E->>E: flash + reboot
    E->>S: MQTT .../handshake<br/>{ "firmwareVersion": "0.1.5" }
    S-->>E: MQTT .../handshake/ack
```

## 5. Authentication {#auth}

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User
    participant M as 📱 Mobile
    participant C as ☁️ Cloud
    participant DB as 🗄️ Postgres

    alt Email + Password Registration
        U->>M: signup form
        M->>C: POST /api/auth/register
        C->>DB: INSERT user (password_hash)
        C-->>M: { accessToken, refreshToken }
    else OAuth (Google / Apple)
        U->>M: tap "Sign in with Google"
        M->>C: POST /api/auth/oauth/google { idToken }
        C->>C: verify token
        alt user exists by email
            C->>DB: INSERT auth_providers (auto-link)
        else new user
            C->>DB: INSERT user + auth_providers
            C-->>M: 200 + flag "set password required"
        end
        C-->>M: { accessToken, refreshToken }
    end
```

## 6. Identity Cache Sync {#identity-sync}

Station caches member identities locally so it can authenticate users on LAN even when offline.

```mermaid
sequenceDiagram
    autonumber
    participant C as ☁️ Cloud
    participant S as 🏠 Station

    Note over C: member added/removed/updated
    C->>S: WSS notify identity_sync<br/>{ users: [...] }
    S->>S: cloudSyncRepo.upsert / delete
    S->>S: refresh local auth cache
    Note over S: TTL 14 days<br/>matches license grace
```

## 7. Kit Auto-Creation {#kit-creation}

When two ESP32 devices with the same `kit_tag` come online, Station auto-creates a kit and applies preset automations.

```mermaid
sequenceDiagram
    autonumber
    participant E1 as 🔘 esp32-switch-pir<br/>(kit_tag=k-sp-lt-a1b2)
    participant E2 as 💡 esp32-light<br/>(kit_tag=k-sp-lt-a1b2)
    participant B as 🖥️ Backend
    participant AE as 🤖 AutomationEngine

    E1->>B: MQTT handshake { kit_tag, type }
    E2->>B: MQTT handshake { kit_tag, type }
    B->>B: kitsService.tryMatchKit()
    Note over B: find devices same kit_tag + location
    B->>B: find auto_apply preset matching roles
    B->>B: createKitFromPreset()<br/>resolveTemplate $trigger/$target
    B->>B: INSERT device_kits + automations
    B->>AE: reload automations
```

## 8. BLE Provisioning {#ble-provisioning}

Adding a new ESP32 to the network — the **Station backend (RPi)** scans via BLE and provisions the device. The user interacts through the Station Frontend SPA.

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 User
    participant F as 🌐 Station Frontend
    participant B as 🖥️ Station Backend
    participant PY as 🐍 ble_bridge.py
    participant E as 💡 ESP32 (factory)

    Note over E: factory state → BLE advertise
    U->>F: open "Add Device"
    F->>B: WS provisioning_start
    B->>PY: startScan()
    PY-->>B: candidate { externalId, deviceType }
    B-->>F: WS provisioning_candidate

    U->>F: select + enter Wi-Fi credentials
    F->>B: POST /provisioning/add { externalId, ssid, psk }
    B->>PY: connectAndWrite(externalId, credentials)
    PY->>E: BLE write { ssid, psk, stationHost }
    E->>E: connect Wi-Fi

    E->>B: MQTT .../handshake { type, firmwareVersion, chip }
    B-->>E: MQTT .../handshake/ack { status: "ok" }
    B-->>F: WS device appeared
```
