# Reolink Camera Alert — Home Assistant Blueprint for Android  
**⚠️ Status: WORK IN PROGRESS: This blueprint is still being developed and may change or break. Please test carefully before relying on it for security notifications.
---

# Android

## Install (Still a WiP but less changes compated to the beta)

[![Open your Home Assistant instance and show the blueprint import dialog](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FNS086%2FHomeAssistantBlueprints%2Frefs%2Fheads%2Fmain%2FAndroidreolinkfrigate.yaml)


## Beta (regular changes)

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FNS086%2FHomeAssistantBlueprints%2Frefs%2Fheads%2Fmain%2Fbeta.yaml)

---

-----------------------------------------------

# IOS Still in Alpha - All features have not been tested

[![Open your Home Assistant instance and show the blueprint import dialog](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FNS086%2FHomeAssistantBlueprints%2Frefs%2Fheads%2Fmain%2FIOSAlpha.yaml)


A reusable automation blueprint for Reolink cameras/doorbells that:

- Captures a snapshot and sends it in a **Home Assistant Companion App** notification  
- Adds **action buttons**: _Open Frigate_ and _Open Reolink_  
- Includes a **Snooze** button (default 6 hours) using a Timer helper  
- Uses a **single rolling file** (`*_latest.jpg`) with a cache-buster to avoid media bloat  
- Lets you pick the **target mobile device** (no need to type `notify.mobile_app_*`)  
- Supports **Android notification channel** name & importance  
- Works for plain cameras (**person detect**) and doorbells (optional **visitor/pressed**)

---

## 🙏 Credits

Huge thanks to **reinaldoarrosi** for reverse-engineering the Reolink deep-link format and sharing it on the Home Assistant forums. This blueprint uses that intent to open the Reolink app directly to a specific camera/channel.

- Forum user: **reinaldoarrosi**  
- Post: https://community.home-assistant.io/t/reolink-how-do-i-create-a-home-assistant-notification-with-a-link-that-opens-the-reolink-camera-app/517971/17

Huge thanks to **SgtBatten** whos frigate blueprint was used as inspiration for this
- Github user: SgtBatten
- Github Url: https://github.com/SgtBatten/
---

## Requirements

- Home Assistant Companion App installed on the target phone (Android) 
- A **Timer helper** for snoozing (e.g., `timer.alert_snooze`)  
- Reolink **UID**, **device name**, and (if using an NVR) **channel bitmask**  
- Camera entity and binary sensors for **person** (and optionally **visitor/pressed** for doorbells)


---

## Create the Timer helper

**Settings → Devices & Services → Helpers → + Create Helper → Timer**  
Name it (e.g.) `alert_snooze` → entity id: `timer.alert_snooze`.

---

## Use the blueprint

**Automations → Create Automation → From Blueprint → “Reolink Camera Alert…”**  
Fill the inputs:

- **Person sensor**: e.g. `binary_sensor.reolinkoffice_person`  
- **Additional event sensors (optional)**: e.g. `binary_sensor.reolinkdoorbell_visitor` (for doorbells)  
- **Camera entity**: e.g. `camera.reolinkoffice_fluent`  
- **Phone to notify**: select your mobile device (integration: `mobile_app`)  
- **Snapshot filename stem**: e.g. `office` or `doorbell`  
- **Frigate URL**: PWA origin, e.g. `https://frigate.example.com/`  
- **Reolink UID / Device name / Channel**: see **Reolink deep-link** below  
- **Snooze timer**: pick `timer.alert_snooze`  
- **Snooze duration**: default `06:00:00`  
- **Android channel**: e.g. `Security` (created on first use; then adjust in Android settings)  
- **Android importance**: `max | high | default | low | min`  
- **Max queued runs**: default `10` (prevents dropped back-to-back events)

---

## Reolink deep-link (Android): how to fill it

Thanks to @reinaldoarrosi, the Reolink app can be opened to a specific camera with this intent. 

### **Please note the Blueprint now does the bitmask calculation for you, so you can just enter the channel number**

```
intent://scan/#Intent;scheme=reolink;package=com.mcu.reolink;action=android.intent.action.VIEW;
S.UID=<UID>;
S.DEVNAME=<DEVICE_NAME>;
S.ALMTYPE=<ALARM_TYPE>;
S.ALMCHN=<CHANNEL>;
S.ALMNAME=Detection;
S.ALMTIME=<ALARM_DATE>;
end
```

**Parameters:**

- `<UID>` — Reolink **camera or NVR UID**  
- `<DEVICE_NAME>` — device name as shown in the Reolink app/NVR  
- `<ALARM_TYPE>` — use `PEOPLE` or `VEHICLE` (any non-empty string works; keep it present)  
- `<CHANNEL>` 
  - This should be the camera channel number in the NVR. 
  - For direct-to-camera (no NVR), set `1` and rely on the **UID** to select the device  
- `<ALARM_DATE>` — ISO-8601 timestamp  
  - Within ~2 minutes of “now” will open the **Live View**  
  - Older than two minutes will open **Playback** at that time  
  - In HA templates: `{{ now().isoformat() }}`

### Example: doorbell on NVR channel 8

Assume:
- NVR UID `998877AABBCC` and name `MyNvr`  
- Doorbell on **channel 8** → bitmask `2^(8-1) = 128` (This is calculated for you automatically in the template)

The blueprint’s **Open Reolink** action becomes:

```
intent://scan/#Intent;scheme=reolink;package=com.mcu.reolink;action=android.intent.action.VIEW;S.UID=998877AABBCC;S.DEVNAME=MyNvr;S.ALMTYPE=PEOPLE;S.ALMCHN=128;S.ALMNAME=Detection;S.ALMTIME={{ now().isoformat() }};end
```

---

## What the blueprint does

- **Triggers**: person sensor (required) and optional extra sensors (visitor/pressed). Multiple triggers are **OR**.  
- **Mode**: `queued` (no lost events; back-to-back runs queue up).  
- **Snooze**: pressing Snooze 30m fires a `mobile_app_notification_action` event that disables the automation for the specified amount of time
- **Snapshot**: `camera.snapshot` writes to `/media/reolink/<stem>_latest.jpg`, then the notification loads `/media/local/...?_cache_bust=<timestamp>` to force a fresh image while keeping only one file on disk.  
- **Notification title**: defaults to **“Person detected at _<Camera Friendly Name>_”** (doorbells show “Doorbell pressed at …”).  
- **Android channel**: channel is created on first use with your chosen name/importance; tune sound/vibration per-channel in Android Settings.  

---

## Troubleshooting

- **Image not showing**  
  - Ensure the Companion App can reach HA (local/VPN/remote URL).  

- **Buttons do nothing**  
  - Verify the **device** chosen under “Phone to notify” matches the phone you’re testing.  
  - On Android, confirm the Reolink app is installed and the intent scheme is supported.

- **Wrong camera opens in Reolink**  
  - Double-check the **UID** and the **channel**

---

## Security & privacy tips

- Avoid publishing real **UIDs**, **internal domains/IPs**, **device ids**, and camera names that reveal locations.  
- If you share examples, use placeholders (e.g., `REOLINK_UID_EXAMPLE`, `https://frigate.example.com`).  
- Consider a dedicated Android **“Security”** notification channel with your preferred sound/urgency.

---

## FAQ

**Can I notify multiple phones?**  
Yes

**Can I change Snooze duration per camera?**  
Yes — set **Snooze duration** per automation instance. Share one timer for a **global** snooze, or use separate timers per camera.



---

## Changelog

- **v1.1** - Updates include:
  - Removing Android notification importance as this impacts notification channels. By default the priority is set to the highest. Note you will require a new channel for this to work.
- **v1.0** — Initial public draft (snapshot, actions, snooze, Android channel, optional visitor trigger).
