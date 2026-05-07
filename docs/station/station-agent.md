---
title: station-agent
sidebar_position: 3
---

# 🤖 station-agent

Native Node.js binary on Raspberry Pi. Manages Docker containers for backend/frontend, self-updates from public GitHub releases.

[Source ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/station-agent)

## Responsibilities

- Pull and run Docker images for backend, frontend, postgres, mqtt
- Self-update — fetch new binary from `smart-home-updates` repo
- Provide Docker Hub credentials to the host (so app images can be pulled)
- Expose health/control endpoints for first-boot bootstrap UI (`smartstation.local`)

## Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Boot
    Boot --> CheckSelfUpdate: agent starts
    CheckSelfUpdate --> Apply: new agent version available
    CheckSelfUpdate --> Run: up to date
    Apply --> Boot: replace binary + restart
    Run --> Pull: docker compose pull
    Pull --> Up: docker compose up -d
    Up --> Watch
    Watch --> Pull: app version tag changed
    Watch --> [*]: shutdown
```

## Self-Update Flow

```mermaid
sequenceDiagram
    autonumber
    participant A as 🤖 station-agent
    participant U as 📦 smart-home-updates
    participant FS as 💾 RPi filesystem

    A->>U: GET /release.json
    U-->>A: { agentVersion, dockerComposeUrl, ... }
    A->>A: compare with current
    alt update available
        A->>U: GET station-agent-linux-arm64
        U-->>A: binary
        A->>FS: write to tmp + chmod +x
        A->>FS: atomic rename to active path
        A->>A: re-exec self
    end
```

## Distribution

- Built in `station-agent/` package (SEA — Single Executable Application)
- Published to [`smart-home-updates/station-agent/` ↗](https://github.com/alphaoflogic-ua/smart-home-updates/tree/main/station-agent) on release
- Installed by [`install-agent.sh` ↗](https://github.com/alphaoflogic-ua/smart-home-updates/blob/main/install-agent.sh)

## Reference

- [station-agent README ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/station-agent/README.md)
