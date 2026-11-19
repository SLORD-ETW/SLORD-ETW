# Camper Integration Plan

Bringing the camper's existing kit together: **power monitoring, heating, internet,
entertainment and a single dashboard**, with **no paid connectivity plans**.

> Status legend: ✅ known · 🔧 to check/confirm · ❓ unknown · 💡 suggestion
>
> Companion doc: [Fire TV Stick guide](fire-tv-stick-guide.md) (SmartTube, no Shorts, debloat).

---

## Contents

1. [Equipment status](#1-equipment-status)
2. [Target setup (overview)](#2-target-setup-overview)
3. [Network: 5G hub](#3-network-5g-hub)
4. [Victron (no GX device)](#4-victron-no-gx-device)
5. [Truma Combi D 6](#5-truma-combi-d-6)
6. [Home Assistant: the glue](#6-home-assistant-the-glue)
7. [Remote access (Tailscale)](#7-remote-access-tailscale)
8. [Android head unit](#8-android-head-unit)
9. [Entertainment](#9-entertainment)
10. [Old Apple TV](#10-old-apple-tv)
11. [Power budget & wiring](#11-power-budget--wiring)
12. [Shopping list](#12-shopping-list)
13. [Build order](#13-build-order)
14. [Open questions](#14-open-questions)

---

## 1. Equipment status

| Item | Status | Details / to do |
|---|---|---|
| 5G hub | ✅ Generic rebadged Chinese CPE (branded splash screen) | 🔧 Model/chipset, 12V input range, LAN port, SIM lock |
| Heater | ✅ Truma Combi D 6 (diesel heating + hot water) | 🔧 Panel type: CP classic / CP plus / iNet X |
| Victron | ✅ No GX device | 🔧 List devices (SmartShunt, MPPT, Orion, inverter…) and ports (Bluetooth / VE.Direct) |
| Head unit | ✅ 10.1" budget quad-core, "Android 15", 4+64 GB, wireless CarPlay/AA, GPS, Wi-Fi | 🔧 Real SoC + Android version (CPU-Z), MCU version. Limited settings, voice broken, mic not switchable, weak Bluetooth |
| Apple TV | ❓ ~15 years old | 🔧 Model number on underside |
| TV / screen | ❓ | 🔧 Size, 12V or 230V, 4K or HD |
| Camper Fire Stick | 💡 Use a spare from home | Pick model to match TV (see section 9) |
| Small computer | 💡 Raspberry Pi (if owned) | For Home Assistant |

---

## 2. Target setup (overview)

```
                         ┌───────────────────────────┐
     Internet ◄── 5G ───►│   5G hub (Wi-Fi + LAN)     │
                         └──────┬─────────┬───────────┘
                    Wi-Fi/LAN   │         │   Wi-Fi
          ┌─────────────────────┘         └──────────────────────┐
          ▼                                                      ▼
 ┌──────────────────┐   Wi-Fi   ┌───────────────────┐   ┌──────────────────┐
 │ Raspberry Pi     │◄─────────►│ ESP32 (near        │   │ Head unit        │
 │ Home Assistant   │           │ battery/heater)    │   │ HA dashboard,    │
 │ + Tailscale      │           │ • Victron BLE read │   │ SmartTube, maps  │
 └──────────────────┘           │ • Truma LIN bus    │   └──────────────────┘
          ▲                     └───┬──────────┬─────┘   ┌──────────────────┐
          │ Tailscale               │BLE       │LIN      │ Fire Stick + TV  │
          ▼                         ▼          ▼         │ SmartTube, HA,   │
   Phone (anywhere)          Victron devices  Truma      │ Jellyfin         │
                                              CP plus    └──────────────────┘
```

- **5G hub** = the only internet connection. Everything joins its Wi-Fi.
- **ESP32** sits near the Victron gear (and Truma panel), reads them locally, and sends data over Wi-Fi. This solves the head unit's weak Bluetooth range.
- **Home Assistant** on a Pi pulls it all into one dashboard and runs automations.
- **Tailscale** gives remote access from your phone despite mobile CGNAT.
- **Head unit / Fire Stick / phone** all show the same dashboard.

---

## 3. Network: 5G hub

### Checks
- [ ] **Model/chipset:** label underneath, or admin page *About / Device info* (try `192.168.0.1`, `192.168.1.1`, `192.168.8.1`). The original OEM model can unlock better settings/firmware.
- [ ] **Splash screen / captive portal:** look for a setting to disable it. If it can't be removed it only affects a browser's first page, not the Fire Stick, Victron or Home Assistant.
- [ ] **Power input:** usually 12V DC barrel. Battery voltage swings ~11.5–14.6V, so confirm the accepted range or fit a **12V regulated DC-DC converter**. Fuse the feed.
- [ ] **LAN port:** use Ethernet for the Pi if possible (more reliable than Wi-Fi).
- [ ] **SIM lock:** confirm it takes any network's SIM.
- [ ] **DHCP reservations:** give the Pi, ESP32, head unit and Fire Stick fixed addresses, if supported.

### Notes
- Mobile networks use **CGNAT**, so **port forwarding won't work**. See [section 7](#7-remote-access-tailscale).
- Power draw ~8–15W ≈ **15–25 Ah/day** if left on 24/7. Consider auto-off at low battery.
- 💡 *Optional later:* a GL.iNet travel router behind the hub gives a proper interface, DHCP reservations and built-in Tailscale. The hub then just supplies the 5G connection.

---

## 4. Victron (no GX device)

| Option | How | Cost | Notes |
|---|---|---|---|
| **A. ESP32 reading Victron BLE** ⭐ | Small board near the battery reads the Bluetooth "Instant readout" broadcasts and sends them over Wi-Fi to Home Assistant | ~£5–10 | Needs each device's **encryption key**: VictronConnect → device → ⚙ → *Product info → Instant readout*. No wiring into the Victron system. |
| B. Home Assistant Victron BLE integration | Pi reads the broadcasts directly | Pi only | Pi must be in Bluetooth range of the Victron devices. |
| C. Pi running Venus OS + VE.Direct cables | Pi acts like a Cerbo GX: VRM portal, Remote Console, full data | Pi + ~£15–30 per cable | Wired, most reliable. Only for devices with a VE.Direct port. "Large" image adds Node-RED and Signal K. |

**Recommended:** start with **A**. VictronConnect on your phone still works for setup and firmware updates.

### To do
- [ ] List every Victron device and whether it has Bluetooth / VE.Direct
- [ ] Record encryption keys (keep them private)

| Device | Model | Bluetooth? | VE.Direct? | Key recorded? |
|---|---|---|---|---|
| | | | | |
| | | | | |

---

## 5. Truma Combi D 6

Integration depends on the **control panel**:

| Panel | How to recognise | Options |
|---|---|---|
| **CP classic** | Two analog rotary dials | No digital interface. Not worth hacking. |
| **CP plus** | One rotary knob + small digital display ("iNet ready") | **Truma iNet Box** (official, Bluetooth app) or **DIY ESP32 + LIN transceiver running the community ESPHome `truma_inetbox` project**, plugged into the panel's spare LIN port → Combi in Home Assistant |
| **iNet X** | Large colour touchscreen | Truma app over Bluetooth. HA support is newer and more limited. |

- With **CP plus + ESP32**: control heating and hot water from your phone over 5G, with no Truma subscription or SIM.
- The **same ESP32** may be able to do both the Truma LIN bus and Victron BLE. Check the projects support that combination; otherwise use two boards.
- 🔧 Confirm the community project lists the **Combi D 6** (diesel) as supported before buying parts.

### To do
- [ ] Identify panel type (photo it)
- [ ] Check for a spare LIN port on the panel / heater

---

## 6. Home Assistant: the glue

Run **Home Assistant OS** on a Raspberry Pi (Pi 4 or 5 recommended; a Pi 3 works for a light setup).

### Integrations
- **ESPHome:** the ESP32 (Victron data + Truma control)
- **Victron BLE** (option B), if not using the ESP32 for Victron
- **Tailscale** add-on for remote access
- Optional: temperature/humidity sensors, tank levels, door sensors, 5G hub (if it exposes an API)

### Dashboard ideas
- Battery % and voltage, solar input, load, time remaining
- Heater: mode, target temperature, hot water on/off, current temperature
- Inside/outside temperature
- Data usage (if the hub exposes it)

Keep a **simple, light dashboard** for the head unit; it's a low-power device.

### Automation ideas
- Battery below X% → turn off 5G hub / TV (via a 12V relay or smart switch)
- Pre-heat before arrival (from your phone over Tailscale)
- Frost protection: heater on if inside temperature drops below 5°C
- Alert to phone if battery is low or temperature is out of range

---

## 7. Remote access (Tailscale)

Mobile networks use **CGNAT**, so you can't open ports to reach the camper.

- **Tailscale** (free for personal use): install the add-on in Home Assistant and the app on your phone. Open the HA app from anywhere as if you were in the camper.
- Alternative: **Home Assistant Cloud (Nabu Casa)**, paid.
- **VRM** works without either (outbound connection) – but only with a GX device or Venus OS (option C in section 4).

---

## 8. Android head unit

### What it probably is
"4-core, 4+64 GB, Android 15, entry-level" fits the cheapest class of Chinese head units (budget quad-core such as AC8227L/MTK, Allwinner or Rockchip). The "Android 15" label is often an older Android underneath. That explains partial settings, broken voice and fixed mic routing.

- [ ] Install **CPU-Z**: note real SoC, Android version and RAM
- [ ] Note **MCU version** (Settings → About)

### Don't flash firmware
Community ROMs and mods (XDA) are for **FYT / UIS7862** 8-core units, not these. Wrong firmware is the most common way to brick a unit. Treat this one as a **screen for native Android apps and the camper dashboard**. 💡 If you ever want a moddable unit, an 8-core UIS7862 (FYT) is the step up.

### CarPlay is too restrictive → use native apps instead
CarPlay only allows Apple-approved driving apps (no YouTube, browser, Home Assistant…). The head unit runs Android, so install apps on it directly and connect it to the **5G hub's Wi-Fi** (or a phone hotspot when the hub is off).

| Need | App on head unit |
|---|---|
| YouTube | **SmartTube** (Shorts hidden, no ads, SponsorBlock) – see Fire TV guide |
| Camper dashboard | **Home Assistant** app or browser |
| Victron | **VictronConnect**, or the HA dashboard |
| Heater | HA (via ESP32), or Truma app if supported |
| Navigation | Google Maps, Waze, **OsmAnd / Organic Maps** (offline) |
| Offline films | **Jellyfin** or **VLC** (USB stick or Pi) |
| Music | Spotify, YouTube Music, etc. |

### Fixes without flashing
- [ ] **Full Android Settings:** install **Activity Launcher** → find `com.android.settings`
- [ ] **Factory settings** (often Car settings → Factory, behind a code; try `8888`, `1234`, `3368`, `1617`, `0000`): look for **MIC internal/external** and mic gain
- [ ] **Voice:** unit's mic is often wired to the Bluetooth phone module only, so Android apps can't use it. Try:
  - Pair the iPhone as a plain Bluetooth phone; long-press the voice button (unit or steering wheel) to trigger **Siri** over hands-free
  - A **USB microphone** for Android apps that need voice input
- [ ] **iPhone screen:** check the CarPlay app (ZLink / AutoKit) for an **AirPlay / Mirror** mode – shows the full iPhone screen without CarPlay's restrictions (usually no touch control, some lag)
- [ ] Optional cleaner home screen: **AGAMA Car Launcher** or **Car Launcher Pro** (check it works with the unit's custom interface)
- [ ] Give the unit a fixed IP on the hub

### Notes
- Weak Bluetooth range doesn't matter for Victron/Truma – the ESP32 relays over Wi-Fi.
- Set SmartTube quality to **720p/1080p** to save mobile data.
- **UK law:** no video while driving. SmartTube when parked only.

---

## 9. Entertainment

- **Fire Stick on the camper TV:** SmartTube with Shorts hidden, SponsorBlock on (see [Fire TV guide](fire-tv-stick-guide.md)).
  - 4K TV → 4K Max (AV1). HD TV → any Fire OS stick (set codec VP9/AVC on older ones).
  - Powered from 5V USB – use a quality 12V USB socket.
- **Save data:** cap SmartTube at 720p/1080p.
- **No signal:** films on a USB drive or the Pi via **Jellyfin** (or a network share), played on the Fire Stick (Jellyfin/Kodi) or head unit.
- **Home Assistant on the TV:** browser or an HA TV app on the Fire Stick.
- **AirPlay:** old Apple TV as a receiver for iPhones (section 10).

---

## 10. Old Apple TV

Check model number on the underside:

| Model | Year | What it can still do |
|---|---|---|
| **A1218** (silver, hard drive) | 2007 | Small PC inside; can run Linux/Kodi, but slow and power-hungry. Not worth it. |
| **A1378** (small black box), 2nd gen | 2010 | Apps mostly dead. Works as an **AirPlay receiver** (up to 720p). Older firmware can be jailbroken for an old Kodi – hobby project. |
| **A1427 / A1469**, 3rd gen | 2012–13 | AirPlay receiver, 1080p, very limited apps. |

**Verdict:** AirPlay target for iPhones, otherwise use a spare Fire Stick. Needs 230V (inverter) – another reason to prefer a USB-powered Fire Stick.

- [ ] Model number: ________

---

## 11. Power budget & wiring

### Estimated draw (fill in real figures)

| Device | Approx. power | Hours/day | Ah/day @12V |
|---|---|---|---|
| 5G hub | 8–15 W | 24 | 15–30 |
| Raspberry Pi 4 | 3–6 W | 24 | 6–12 |
| ESP32 | <1 W | 24 | ~1–2 |
| Head unit (in use) | 10–20 W | ? | |
| TV (12V) | 15–40 W | ? | |
| Fire Stick | 3–5 W | ? | |
| **Total** | | | |

### Wiring notes
- Fuse every feed at the distribution point; size cable for the current and run length.
- Use a **regulated 12V DC-DC converter** for anything sensitive to voltage swing (hub, possibly TV).
- Use a good-quality **5V USB-C supply** for the Pi (undervoltage causes crashes and SD card corruption).
- 💡 A **12V relay** (switched by an ESP32 or a Victron device's relay/load output) lets Home Assistant turn the hub/TV off at low battery.
- Consider a **separate always-on circuit** for hub + Pi + ESP32 so they survive when the main 12V is switched off.

---

## 12. Shopping list

Prices approximate. Only buy after the checks above.

| Item | Purpose | Approx. cost | Needed? |
|---|---|---|---|
| ESP32 dev board (e.g. ESP32-WROOM / ESP32-S3) | Victron BLE + Truma | £5–10 | ✅ |
| LIN transceiver module (e.g. TJA1020-based) + RJ12 lead | Truma CP plus LIN bus | £5–10 | If CP plus |
| 12V→5V buck converter | Power the ESP32 | £3–5 | ✅ |
| Raspberry Pi 4/5 + case + SSD or good SD card | Home Assistant | £60–100 | If not owned |
| Quality 12V→5V USB-C supply for Pi | Stable Pi power | £10–15 | ✅ |
| 12V regulated DC-DC converter | Hub (and TV if needed) | £10–20 | If hub needs it |
| USB microphone | Voice for Android apps on head unit | £10 | Optional |
| VE.Direct USB cables | Option C only | £15–30 each | Optional |
| GL.iNet travel router | Better network features | £30–80 | Optional |
| 12V relay module | Auto-off at low battery | £5 | Optional |

---

## 13. Build order

1. **Network**
   - [ ] Identify hub, sort 12V power, disable splash screen if possible
   - [ ] Set fixed IPs for key devices
2. **Head unit**
   - [ ] CPU-Z, Activity Launcher, factory settings (mic)
   - [ ] Connect to hub Wi-Fi; install SmartTube, maps, VictronConnect
3. **Entertainment**
   - [ ] Set up a spare Fire Stick for the camper TV (Fire TV guide)
4. **Home Assistant**
   - [ ] Pi with Home Assistant OS; Tailscale add-on; phone app
5. **Victron**
   - [ ] ESP32 with Victron BLE (keys from VictronConnect) → ESPHome → HA
6. **Truma**
   - [ ] Identify panel; if CP plus, ESP32 + LIN → ESPHome `truma_inetbox` → HA
7. **Dashboard & automations**
   - [ ] Light dashboard for head unit; phone dashboard; low-battery and frost automations
8. **Power tidy-up**
   - [ ] Measure real draw, fill in section 11, add relay auto-off if needed

---

## 14. Open questions

- [ ] 5G hub model / chipset, 12V range, LAN port, SIM lock
- [ ] Truma panel type (CP classic / CP plus / iNet X)
- [ ] Victron device list and ports
- [ ] Head unit real SoC, Android version, MCU version
- [ ] Apple TV model number
- [ ] Camper TV: size, 12V or 230V, 4K or HD
- [ ] Is there a Raspberry Pi already, or buy one?
- [ ] iPhone or Android phone(s) in use? (affects mirroring/voice options)
