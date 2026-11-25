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
| Vehicle | ✅ Mercedes Sprinter (W907-style dash) | – |
| Head unit | ✅ 10.1" budget quad-core, model **K2401P**, sold as "Android 15", 4+64 GB, wireless CarPlay/AA, GPS, Wi-Fi. USB updater app V3.7.6. On the hub's Wi-Fi (`192.168.0.x`) | ✅ CPU-Z: 4× Cortex-A53 1.8 GHz, PowerVR GE8300, **real Android 10 (API 29)**, 32-bit, patch 2021-01. 🔧 Exact chip name, MCU version. Limited settings, voice broken, mic not switchable, weak Bluetooth |
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

### Fleet (discovered via BLE scan, Nov 2026)

All five are Victron **Smart** (Bluetooth) devices → all support **Instant Readout** broadcasts, so one BLE gateway (ESP32/Pi) can read the lot without pairing.

| Device | Serial | Role | Instant Readout key recorded? |
|---|---|---|---|
| SmartShunt / BMV | HQ2023WN5JS | Battery monitor (SOC, V, A, W) | ☐ |
| SmartSolar MPPT | HQ2031A2JHW | Solar charge controller | ☐ |
| Orion Smart DC-DC #1 | HQ20309ARM1 | Alternator → leisure charger | ☐ |
| Orion Smart DC-DC #2 | HQ2110LRRUC | DC-DC charger | ☐ |
| BatteryProtect | HQ2002V8CSQ | Low-voltage load disconnect | ☐ |

Live sample: BMV 100% SOC, 14.00 V, 44.75 A, **627 W in** (solar + both Orions, engine running).

### To do
- [ ] For **each** device: VictronConnect → ⚙ → Product info → **Instant Readout ON** + copy the **encryption key** (keep private; not in this repo)

> **Why not read Victron on the tablet/head unit directly:** the head unit's Android Bluetooth is `null` (unusable). The Fire tablet's Bluetooth *works* with VictronConnect (per-device viewing), but it **cannot feed a custom web panel** — Fully Kiosk's webview can't scan BLE adverts, and Termux/Python can't use Android's Bluetooth stack. So a **unified custom dashboard needs a BLE gateway** (ESP32 ESPHome `victron_ble`, or a Pi) → Home Assistant → tablet/head-unit browser. VictronConnect remains the quick per-device view.

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

### ✅ Confirmed (Nov 2026): CP plus

Panel board labelled **Truma CP plus — HW 04.00, SW C 04.03.00**; wall panel shows **"iNet ready"**; the **CI-BUS connector** (black RJ-style socket) is present and accessible behind the panel. → Full monitor **and** control via **ESP32 + ESPHome `truma_inetbox`** tapping the CI-BUS. Still verify the project lists **Combi D 6 (diesel)** before buying the LIN transceiver.

### To do
- [x] Identify panel type → **CP plus**
- [x] CI-BUS connector present behind panel ✅
- [ ] Confirm `truma_inetbox` supports Combi D 6 (diesel)

### As-built van catalog (Nov 2026)

**Victron (power)** — all Bluetooth → ESP32 `victron_ble`:
- BMV-712 Smart (SN HQ2023WN5JS, PN BAM030712000R) — head at the Truma/Victron panel, shunt at battery; has a NO/COM/NC relay. PUK recorded privately (not in repo).
- SmartSolar MPPT 75/15 (HQ2031A2JHW)
- 2× Orion Smart DC-DC (HQ20309ARM1, HQ2110LRRUC) — **under passenger seat**
- BatteryProtect (HQ2002V8CSQ)
- VE.Smart Network "Rebel ve" links BMV ↔ SmartSolar.

**Truma (heat):** CP plus panel (above) + Combi D 6 diesel + Truma Gasfernschalter (gas remote switch).

**CBE (van 12 V control system) — separate, more closed:**
- CBE **PT622** — tank levels (fresh/waste)
- CBE **PT638** — temperature (Int/Ext sensors)
- CBE distribution/charger behind the panel.
- Integration is harder (CBE bus is not well documented). **Easier path:** take room temp from Truma, add a DS18B20 on an ESP32 for outside temp, add own tank senders later — rather than tapping CBE.

**Install bay:** the Victron BMV head and Truma CP plus are **side by side** with a large accessible void behind — fits the Pi + ESP32(s). Orions are a separate location (passenger seat), so a 2nd ESP32 there may be needed if BLE range from the main bay doesn't reach them.

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

### Findings so far
- **Model:** K2401P (Settings → About car device). Storage 64 GB.
- **CPU-Z results (confirmed):**

  | Item | Value | Meaning |
  |---|---|---|
  | CPU | 4× ARM Cortex-A53 @ 1.8 GHz | Entry-level quad-core |
  | GPU | PowerVR Rogue GE8300 | Budget GPU. Consistent with a MediaTek MT8766-class (or Allwinner A133-class) chip; `getprop ro.board.platform` will name it |
  | Android version shown | 15 | **Fake label** |
  | API level | **29** | **Real version: Android 10** |
  | Security patch | 2021-01-05 | Old; avoid sensitive logins (banking etc.) on this unit |
  | Build | K2401_NWD_S212802.20251217 | Firmware built Dec 2025 |
  | Kernel | 4.9.170, armv8l | **32-bit Android** on 64-bit cores |
  | Root | No | |
  | Google Play Services | 26.32.34 | Up to date, Play Store works |
  | Uptime | 12 days | Unit sleeps rather than powering off with the ignition |

- **What this means:**
  - Choose **32-bit (armeabi-v7a / arm)** APK versions when sideloading (e.g. SmartTube's armeabi-v7a build). A few newer apps are 64-bit only and won't install.
  - Fine for maps, music, VictronConnect, a light Home Assistant dashboard, and SmartTube at **720p/1080p** (set codec to AVC/VP9; no AV1).
  - No community ROMs for this hardware: **don't flash**. Debloat with ADB `pm disable-user` instead (same as the Fire Sticks).
  - Because it sleeps instead of powering off, check its standby draw on whichever battery feeds it.
- **USB updater** (runs automatically when a USB stick is plugged in), translated:

  | On screen | Meaning | Result |
  |---|---|---|
  | MCU升级状态 | MCU upgrade | No upgrade file found |
  | 拷贝固件 | Copy firmware | No upgrade file found |
  | 工厂设置 | Factory settings (import config) | No factory config found |
  | 拷贝地图 | Copy map data | No map data found |
  | 清除文件 | Delete files | No delete config found |
  | APP版本 | Updater version | V3.7.6 |
  | 所有操作已完成，设备可拔出! | All done, USB can be removed | Nothing was changed |

  The updater looks for specific files on the stick (MCU update, firmware, a factory config file, maps). **Only use files supplied by the seller for the K2401P.** A factory config file from the seller is a safe way to change hidden settings such as microphone routing.

### Find the real chip with ADB (same method as the Fire Sticks)
1. Settings → About car device → tap **Build number** 7 times → Developer options.
2. Turn on **USB debugging** / **ADB over network** (if offered).
3. From a laptop on the same Wi-Fi:
   ```bash
   adb connect <head-unit-ip>:5555
   adb shell getprop ro.board.platform      # chip family
   adb shell getprop ro.build.version.release  # real Android version
   adb shell getprop | grep -i -E "mcu|hardware|product"
   ```

### Ask the seller for
- [ ] Factory settings password
- [ ] A factory config file that sets the **external microphone**
- [ ] The correct firmware for the K2401P (only if a fix is needed)

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

### Teardown & ADB investigation (deep dive, Nov 2026)

Confirmed hardware/firmware identity:

| Attribute | Value |
|---|---|
| Model | **K2401P** |
| Platform | **NWD** (packages `com.nwd.*`, `nwdapp`; build string `K2401_NWD_S212802`) |
| SoC | **Allwinner A133** (`sun50iw10p1`; 4× Cortex-A53 @1.8 GHz, PowerVR GE8300) |
| OS | **Android 10** (API 29) — falsely labelled "Android 15"; 32-bit userspace (armv8l) |
| RAM/Storage | 4 GB / 64 GB |
| Firmware builds seen | running `...20251217`; newer `...20260324`, `...20261017` present on device |

**Vehicle / CAN profile** (from `canconfig.xml` on the file share):
- CAN decoder = **"Oudi"**, protocol family **2E**, UART at **38400 baud** (matches the live `du <- 2E 08 ...` packets in the unit's Debug→Log monitor; those carry the RTC clock, etc.)
- Profile: **Benz Sprinter "2006–2018 (2nd gen / W906)"** — this profile only appeared *after* a firmware update.
- Flags: `hasAir:false`, `hasReverseControlSystem:false` → climate/reverse not integrated via CAN.
- ⚠️ For Home Assistant vehicle data, don't tap this UART (the head unit owns it). Use a separate OBD-II BLE dongle or CAN reader.

**Built-in file-share server** — the unit runs a web file server (Settings → the FileShare feature) on a **high port (e.g. :36715)**, a Vue/Vite SPA called "FileShare". It serves **/sdcard read-write** (download + upload). Useful folders: `Boot Logo`/`boot.logo`/`.boot_logo` (splash), `themeData` (UI skin), `canconfig.xml`, `upgrade` (holds the full firmware .zip packages), `preset_install`. ⚠️ It exposes storage to anyone on the Wi-Fi — don't leave it running; keep the hotspot password private.

**ADB enablement — methods tried and results:**

| Method | Result |
|---|---|
| Developer options (tap Build number ×7) | **Password-locked** ("Please Input Password") |
| `need_enable_console` empty file on USB (known NWD trick) | **No effect**, even after reboot with stick inserted |
| USB-A ↔ USB-A cable + `adb devices` | Empty — **port is host-only** (a `filter_usb` sysfs node exists, may be why) |
| Network scan (`nmap -p-`) | Only ports 3152 / 14035 / 57677 open; **5555 closed**. Those three are vendor services, **not** ADB (adb shows `offline`) |
| boot.img inspection | System-as-root ramdisk (`fstab.sun50iw10p1`), no ADB props present |
| vendor.img inspection | **Stock Allwinner**, no `nwd` customisation → console/adb gate is elsewhere |
| system.img inspection | No adb/console trigger in `init.rc`; `nwd` settings/factory app is in the **product** partition |

**Candidate factory/developer passwords** (community-reported for K2401/NWD; unverified — try in the password box): `345543` (dev-options input), `16176699` + `1617` (factory), `111333` (UI/style), `7890`, `8888`, `1234`, `0000`; some NWD firmware also accepts **today's date** as `YYYY.MM.DD` or `7890`+current hour.

**Firmware package internals** (Allwinner "sunxi" OTA, ~1.1 GB zip). Reusable extraction recipe on a Linux laptop:
```bash
sudo apt install -y brotli p7zip-full
wget https://raw.githubusercontent.com/xpirt/sdat2img/master/sdat2img.py
# per partition (system / vendor / product):
brotli -d <part>.new.dat.br
python3 sdat2img.py <part>.transfer.list <part>.new.dat <part>.img
7z x -o<part>_out <part>.img          # 7z reads ext4 without mounting/root
```
`boot.img` unpacks with `abootimg -x`; ramdisk is system-as-root (no props). `boot-resource.fex` holds boot-logo resources.

**Key sysfs controls found in `/system/init.rc`** (most are `chmod 0666`, i.e. writable from a shell once ADB works):
- **`/sys/devices/platform/nwdpower/mic_select`** → **internal/external microphone switch**. This is the fix for the broken voice input: `echo 1 > …/mic_select` (or `0`). Writable by any shell; MacroDroid (pre-installed) may also be able to toggle it.
- `/sys/kernel/nwdtest_common/config_logo` → boot-logo control (custom splash).
- `/sys/kernel/nwdtest_common/config_car` → car/CAN config.
- `/sys/kernel/nwdtest_common/filter_usb` → USB filter (suspected cause of the host-only USB port).
- Car/CAN config partitions mounted at `/data/nwdconfig` and `/data/nwdappconfig` (from `/dev/block/by-name/nwdconfig`, `carconfig`).

**Breakthrough — ADB is pre-configured, just gated.** `/system/build.prop` ships with:
```
persist.service.adb.enable=1     # adbd meant to run
service.adb.tcp.port=5555        # on TCP 5555
ro.adb.secure=0                  # NO "Allow debugging?" prompt
ro.debuggable=1                  # adb shell runs as ROOT
```
So when ADB comes up it is **root, no auth, over the network** — the best case. It's held off only by an `init.rc` gate:
```
on property:persist.sys.nwd.console.enable=1   # (init.rc ~line 973)
on property:persist.sys.nwd.console.enable=0   # default 0 → port 5555 closed
```
The `need_enable_console` USB file is meant to set that property to 1 but did **not** fire on this firmware.

**No-password route to try next.** The USB updater scans for a factory-config file ("工厂设置 / import factory config"), and `/system/config/app/FactoryConfig.ini` already sets `persist.sys.nwd.*` properties. Adding `persist.sys.nwd.console.enable=1` through that import path should flip the gate and bring up **root ADB with no seller password**. Still to confirm: the exact factory-config filename/format the updater imports (grep the images for `FactoryConfig` / the import handler).

**Fallbacks:** the factory/developer password from the eBay seller, or decompile the nwd settings/car app (`pout/app/SettingsSetup`, `pout/priv-app/SettingsIntelligence`) to read the password check.

**Status:** ADB not yet enabled, but the path is clear and promising. **ADB is only needed for debloating** — SmartTube, Victron, maps and the custom boot logo all work without it.

### Bluetooth (head unit) — no usable Android radio

- Android's "Pair new device" shows **"Bluetooth address: null"** and finds nothing → the SoC's Android Bluetooth adapter is **non-functional**.
- The only working Bluetooth is a **separate hands-free call module** wired to the MCU, used by the unit's own BT/Phone app. It pairs a **phone** for calls / voice / A2DP music only — **not** arbitrary BLE devices.
- **Consequence:** VictronConnect (or any BLE app) **cannot** run on the head unit over its own Bluetooth. Victron data must come via the **ESP32 → Home Assistant** relay, with the head unit showing the HA dashboard in a browser over Wi-Fi (visible while driving or parked — and no Bluetooth needed).
- **USB Bluetooth dongle (maybe):** an OTG dongle *might* give Android a real adapter. Verify before buying by checking the extracted firmware for a USB BT driver:
  ```bash
  find ~/Downloads/fw/vout ~/Downloads/fw/sout -iname 'btusb*' -o -iname '*bluetooth*'
  find ~/Downloads/fw -iname '*rtl8761*' -o -iname '*.hcd' -o -iname 'rtlbt*'
  ```
  Prefer a **CSR8510 (BT 4.0)** dongle (generic `btusb`, no firmware upload) over Realtek RTL8761 units (e.g. **TP-Link UB600**), which need `btrtl` + a firmware file these kernels usually lack. BT 4.0 is plenty for Victron BLE.

### Troubleshooting log

- **Google apps (Maps/Waze) start then crash:** caused by out-of-date **Google Play Services**. Fix: Play Store → update Google Play Services (+ Maps/Waze). Clock being wrong causes the same symptom (SSL fails) — check date/time first.

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
