# Fire TV Stick: De-bloat, Ad-free YouTube & No Shorts

Personal reference for cleaning up a mixed collection of Amazon Fire TV Sticks.
Main goal: **ad-free YouTube with no Shorts, minimal Amazon UI.**

> ⚠️ Things change with Fire OS updates. Double-check current XDA threads for your
> device codename before doing anything risky (root, disabling system packages).

---

## Contents

1. [Check which OS the stick runs](#1-check-which-os-the-stick-runs)
2. [Model reference table](#2-model-reference-table)
3. [Built-in settings (no hacking)](#3-built-in-settings-no-hacking)
4. [Enable Developer Options & ADB](#4-enable-developer-options--adb)
5. [Identify each stick](#5-identify-each-stick)
6. [Debloat](#6-debloat)
7. [SmartTube (ad-free YouTube, no Shorts)](#7-smarttube-ad-free-youtube-no-shorts)
8. [Hide the Amazon UI (launcher + Button Mapper)](#8-hide-the-amazon-ui-launcher--button-mapper)
9. [Network-wide ad blocking](#9-network-wide-ad-blocking)
10. [Rooting (old models only)](#10-rooting-old-models-only)
11. [Which stick goes where](#11-which-stick-goes-where)
12. [YouTube Premium vs SmartTube](#12-youtube-premium-vs-smarttube)
13. [Per-stick checklist](#13-per-stick-checklist)
14. [Risks](#14-risks)

---

## 1. Check which OS the stick runs

**Settings → My Fire TV → About**

- **Fire OS** (Android-based) – most sticks. Everything in this guide applies.
- **Vega OS** (Linux-based) – Fire TV Stick 4K Select (2025) and newer Vega devices.
  **No sideloading, no custom launcher, no SmartTube.** Don't buy these if you want to customize.

---

## 2. Model reference table

From memory – confirm codename/firmware on XDA.

| Model (year) | Codename | Fire OS (Android) | Root? | Home-button hijack | Best use |
|---|---|---|---|---|---|
| Fire TV Stick 1st gen (2014) | montoya | FO5 (5.1) | Old exploits only | Easy | Retire / basic Kodi box. Many modern apps unsupported. |
| Fire TV Stick 2nd gen (2016) / Basic Edition | tank | FO5 (5.1) | **Yes** (bootrom exploit) | Easy | Root it, remove Amazon UI. Good practice stick. |
| Fire TV Stick 4K 1st gen (2018) | mantis | FO6 → FO7 (7.1/9) | **Yes** (open case, short test point) | Easy when rooted | **Best stick to hack.** Pure SmartTube box. |
| Fire TV Stick 3rd gen / Lite (2020) | sheldon / sheldonp | FO7 (9) | Community unlocks existed on some firmware | Blocked on 7.6.x+ | Debloat + launcher. Check XDA. |
| Fire TV Stick 4K Max 1st gen (2021) | kara | FO7 (9) | No public method | Blocked on newer firmware | Debloat, Projectivy/Button Mapper, DNS blocking |
| Fire TV Stick 4K / 4K Max 2nd gen (2023), Stick HD (2024) | check `getprop` | FO8 (11) | No | Blocked → workarounds | Same as above. Main viewing sticks. |
| Fire TV Stick 4K Select (2025+) | – | **Vega OS** | No | No sideloading | Use stock or give away |

**Key cutoffs**
- **Fire OS 5–6:** almost anything works (home redirect, disabling stock launcher, root on tank/mantis).
- **Fire OS 7.6.x / 8:** Home-button hijack blocked, updates re-enable things. Debloat + launcher workaround + DNS blocking.
- **Vega OS:** nothing.

---

## 3. Built-in settings (no hacking)

- **Settings → Preferences → Privacy Settings:** turn off *Device Usage Data*, *Collect App Usage Data*, *Interest-based Ads*.
- **Settings → Preferences → Featured Content:** turn off video and audio autoplay.
- **Settings → Preferences → Notification Settings:** silence app notifications.
- **Settings → Applications → Manage Installed Applications:** uninstall anything removable.

---

## 4. Enable Developer Options & ADB

1. **Settings → My Fire TV → About** → click the device name **7 times** → Developer Options appears.
2. **Developer Options:** turn on **ADB Debugging** and **Install Unknown Apps** (allow *Downloader*).
3. Note the IP: **About → Network**.
4. First connection from a computer shows an "Allow USB debugging?" prompt on the TV – accept and tick **Always allow**.

No computer? Phone apps like **Bugjaeger**, **Remote ADB Shell**, or **TV Tools** work too.

---

## 5. Identify each stick

```bash
IP=192.168.1.50
adb connect $IP:5555
adb -s $IP:5555 shell getprop ro.product.device         # codename
adb -s $IP:5555 shell getprop ro.build.version.name     # Fire OS version
adb -s $IP:5555 shell getprop ro.build.version.release  # Android version
```

Record results here:

| Room / label | IP | Codename | Fire OS | Android | Notes |
|---|---|---|---|---|---|
| | | | | | |
| | | | | | |
| | | | | | |

---

## 6. Debloat

```bash
adb shell pm list packages | grep amazon                 # see what's installed
adb shell pm disable-user --user 0 <package.name>        # reversible: pm enable <package.name>
adb shell pm uninstall --user 0 <package.name>           # removed for user; factory reset restores
```

**Common targets**
- Unused Amazon apps (Photos, Music, Kindle bits, games, Freevee/Prime extras).
- OTA updaters (stop Amazon undoing hacks – **but you lose security patches**):
  - `com.amazon.device.software.ota`
  - `com.amazon.tv.forcedotaupdater.v2`

**⚠️ Do NOT disable** `com.amazon.tv.launcher`, Settings, or framework packages unless you know what
you're doing – risk of no home screen / boot loop → factory reset.

**GUI option:** *Fire Toolbox* (XDA, Windows) – debloat, launcher replacement, ad tweaks. Check it supports your firmware.

### Batch debloat across all sticks

```bash
#!/usr/bin/env bash
STICKS="192.168.1.50 192.168.1.51 192.168.1.52"
PKGS="com.amazon.device.software.ota com.amazon.tv.forcedotaupdater.v2"   # add your own

for ip in $STICKS; do
  adb connect "$ip:5555" >/dev/null
  echo "== $ip: $(adb -s $ip:5555 shell getprop ro.product.device) / $(adb -s $ip:5555 shell getprop ro.build.version.name)"
  for p in $PKGS; do
    adb -s "$ip:5555" shell pm disable-user --user 0 "$p" 2>&1 | sed "s/^/   /"
  done
done
```

---

## 7. SmartTube (ad-free YouTube, no Shorts)

Free, open-source YouTube client for Android TV / Fire TV.

- **No ads** (pre-roll, mid-roll, banners)
- **SponsorBlock** built in – skips sponsor segments, intros, "like and subscribe" reminders
- **Hide Shorts** – from Home, Subscriptions and search
- Sign in with Google → subscriptions, history, playlists
- Choose quality/codec (4K, HDR, 60fps where supported)
- Phone casting works (link the phone's YouTube app in SmartTube settings)

### Install

1. Install **Downloader** from the Amazon Appstore.
2. Get the APK **only** from the official GitHub: **github.com/yuliskov/SmartTube** (Releases).
   ⚠️ Fake "SmartTube" sites spread malware – avoid lookalike domains.
3. Install via Downloader, then delete the APK to free space.

Or push it from a computer to all sticks:

```bash
for ip in 192.168.1.50 192.168.1.51; do
  adb connect $ip:5555
  adb -s $ip:5555 install -r SmartTube.apk
done
```

### Settings to change

- [ ] Sign in with Google account (or a secondary account – see section 12)
- [ ] **Hide Shorts** everywhere (Home, Subscriptions, search) – look under the Shorts / content-filter options (location varies by version)
- [ ] Enable **SponsorBlock** and choose which segment types to auto-skip
- [ ] Set **codec** per stick (see section 11): AV1 on 4K Max, VP9/AVC on older sticks
- [ ] Link phone for casting
- [ ] Leave auto-update on (Google breaks it occasionally; fixes usually arrive fast)

---

## 8. Hide the Amazon UI (launcher + Button Mapper)

### Launchers

| Launcher | Notes |
|---|---|
| **Projectivy Launcher** | Most popular. Clean, themeable, has an override option for Home. |
| Wolf Launcher | Very customizable. |
| FLauncher | Open source, minimal. |
| ATV Launcher | Simple Android-TV-style layout. |

### Setup

1. Install **Projectivy Launcher**. Keep only SmartTube (+ one or two others) on it. Enable its override option.
2. Install **Button Mapper**:
   - Remap **Home** → Projectivy.
   - Remap a **sponsored app button** on the remote (Netflix / Prime / etc.) → **SmartTube**. One press to YouTube.
3. Optional: set Projectivy to open SmartTube on startup (if your version offers it).
4. Turn off Featured Content autoplay (section 3) for anything that slips through.

**Notes:** Amazon's screen still flashes briefly at boot. On Fire OS 8 updates can reset the Home
mapping – redo it if so.

---

## 9. Network-wide ad blocking

Covers every device, including Vega OS sticks.

- **Pi-hole** or **AdGuard Home** (Raspberry Pi / home server)
- **NextDNS** or **AdGuard DNS** (hosted, no hardware)

On the stick: **Settings → Network → (your Wi-Fi) → Advanced** → enter DNS.

- Blocks Amazon home-screen ads and telemetry.
- **Does NOT block YouTube ads** (served from the same domains as videos) – that's what SmartTube is for.
- Aggressive blocklists can break Prime Video / Appstore – allowlist Amazon domains if needed.

---

## 10. Rooting (old models only)

- **Rootable:** Fire TV Stick 2nd gen (**tank**), Fire TV Stick 4K 1st gen (**mantis**) via bootrom exploits. Some others had unlocks on certain firmware.
- **Not rootable:** 4K / 4K Max 2nd gen (2023) and newer.
- Usually involves opening the case and shorting a test point. **Real brick risk.**
- Find your codename (`getprop ro.product.device`) and follow the XDA Forums thread for it.
- Payoff: no Amazon UI at all, a true default launcher, no forced updates.

---

## 11. Which stick goes where

| Stick | Role | SmartTube codec |
|---|---|---|
| **4K Max 2nd gen** (or 1st gen) | Main TV | **AV1** (best 4K quality) |
| 3rd gen / Lite / 4K 1st gen | Secondary TVs | **VP9 or AVC** (no AV1 → 4K may stutter otherwise) |
| 4K 1st gen (mantis) | Root experiment → pure SmartTube box | VP9 |
| 2014–2016 sticks | Retire (slow, old SmartTube builds only) | AVC |
| 4K Select (Vega OS) | Can't run SmartTube | – |

---

## 12. YouTube Premium vs SmartTube

**Premium removes ads but NOT Shorts.** The official TV app has no setting to turn Shorts off.

### Official app workarounds (partial, unreliable)
- Use only the **Subscriptions** tab (fewer Shorts, not zero).
- **"Not interested"** on the Shorts shelf (⋮ menu) – may hide it for a while, comes back.
- **Pause watch history** (myactivity.google.com) – empties Home recommendations, but loses useful recs and the Shorts shelf may still appear.

### SmartTube
- Actually hides Shorts everywhere. Main reason to switch.
- Against YouTube's terms of service. In practice Google breaks the app technically rather than
  banning accounts – no known bans just for using it, but possible in principle.
  If worried: use a **secondary Google account**, or don't sign in and import subscriptions.

### Combine both
- Keep Premium (if subscribed) for phone, YouTube Music and supporting creators.
- Use SmartTube on the TV for a Shorts-free feed.
- Caveat: SmartTube viewing may not count toward Premium creator revenue share.

---

## 13. Per-stick checklist

Copy for each stick:

```
Stick: __________   Room: __________   IP: __________
Codename: ________  Fire OS: ________

[ ] Check About: Fire OS (not Vega OS)
[ ] Privacy settings off (usage data, app usage, interest-based ads)
[ ] Featured Content autoplay off
[ ] Developer Options → ADB on, Install Unknown Apps on
[ ] getprop recorded in section 5 table
[ ] Debloat (unused Amazon apps; optionally OTA updaters)
[ ] Install SmartTube (official GitHub only)
    [ ] Signed in
    [ ] Shorts hidden everywhere
    [ ] SponsorBlock on
    [ ] Codec set (AV1 / VP9 / AVC)
    [ ] Phone linked for casting
[ ] Install Projectivy → only SmartTube + essentials, override on
[ ] Install Button Mapper → Home → Projectivy, app button → SmartTube
[ ] DNS set to Pi-hole / AdGuard / NextDNS
[ ] (Old models) consider root
```

---

## 14. Risks

- Updates can re-enable disabled apps and break launcher workarounds.
- Blocking OTA updates = no security patches.
- Disabling the wrong package may force a factory reset.
- Rooting voids warranty and can brick the device.
- SmartTube can break for a few days when Google changes YouTube.
- Only download APKs from official sources.
