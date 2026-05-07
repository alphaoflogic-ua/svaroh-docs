---
title: Deployment
sidebar_position: 3
---

# 🚀 Mobile Deployment

```mermaid
flowchart LR
    Tag["🏷️ git release v1.3.1"]
    Sync["🔄 sync app.json version<br/>(scripts/release.sh)"]
    Push["⬆️ git push --tags"]
    EAS["⚙️ EAS Build"]
    IPA["📦 .ipa (iOS)"]
    AAB["📦 .aab (Android)"]
    TF["✈️ TestFlight"]
    AS["🛍️ App Store"]
    PS["🛒 Play Store"]

    Tag --> Sync
    Sync --> Push
    Push --> EAS
    EAS --> IPA
    EAS --> AAB
    IPA --> TF --> AS
    AAB --> PS
```

## Version Sync

`scripts/release.sh` keeps `package.json` and `app.json` (`expo.version`) in sync — EAS Build refuses to publish if they differ.

## TestFlight Checklist

See full checklist in [TESTFLIGHT_DEPLOYMENT_CHECKLIST.md ↗](https://github.com/alphaoflogic-ua/smart-home-mobile/blob/main/docs/TESTFLIGHT_DEPLOYMENT_CHECKLIST.md).

## Reference

- [eas.json ↗](https://github.com/alphaoflogic-ua/smart-home-mobile/blob/main/eas.json)
- [APPSTORE_PLAN.md ↗](https://github.com/alphaoflogic-ua/smart-home-mobile/blob/main/APPSTORE_PLAN.md)
