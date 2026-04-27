# OnePlus 9 Pro Storage Fix Guide

[一加 9 Pro 存储清理指南（中文）](https://github.com/lingwsh/OnePlus-9-Pro-Storage-Fix-Guide/blob/main/README_CN.md)

**Recover 20–50+ GB from the mysterious "Other" storage on your OnePlus 9 Pro**

> [!WARNING]
> **Backup your phone data before proceeding.** Some commands in this guide (and any AI coding assistant such as Claude) may delete personal files and clean storage. Always back up your photos, contacts, WhatsApp chats, and other important data before running any commands.

## The Problem

OnePlus 9 Pro (LE2123) running OxygenOS 14 accumulates massive "Other" storage (30–50+ GB) that cannot be cleared through normal means. This is a **known, unfixed bug** affecting the OnePlus 9 series. OnePlus has never released a fix, and OOS 14 is the final major version for this device.

### Root Cause

The bloat comes from three sources stacked together:

1. **Modem ramdumps** (`/data/vendor/ramdump/`) — Qualcomm modem crash dumps (~269 MB each), generated multiple times per day on some firmware versions. OxygenOS never auto-deletes them.
2. **System logs** (`/data/oplus/log/`) — Engineering logs that accumulate indefinitely.
3. **Upgrade residue** — dalvik-cache from 4 major OS upgrades (OOS 11→12→13→14), stale OTA packages, and Play Store APK cache.

All of these are protected by SELinux and cannot be accessed by the user without root.

## Quick Start

### What You Need

- A computer with ADB installed ([Download Platform Tools](https://developer.android.com/tools/releases/platform-tools))
- A USB data cable (not charge-only) — **USB 2.0 recommended** (see USB Issues below)
- Your OnePlus 9 Pro with USB Debugging enabled

### Enable USB Debugging

1. **Settings → About Phone** → tap **Build number** 7 times
2. **Settings → Developer Options** → enable **USB Debugging**
3. **Settings → Developer Options → Default USB configuration** → set to **File Transfer**

### Step-by-Step Recovery

#### Step 1: Gallery Trash (0–50 GB)

On the phone: **Gallery → Collections → Recently Deleted → Empty trash**

This is the single biggest potential recovery. A known MediaStorage bug since 2021 causes deleted photos to accumulate indefinitely in the trash.

#### Step 2: Play Store Cache (0–15 GB)

```bash
adb shell du -sh /sdcard/Android/data/com.android.vending/
# If large (multiple GB), clear it:
adb shell rm -rf /sdcard/Android/data/com.android.vending/*
```

#### Step 3: LogKit Log Clear (2–30 GB)

On the phone: Open the **dialer** → type `*#800#`

- If **LogKit** opens → tap **"One key clearlog"**
- If **Feedback** app opens instead → LogKit is disabled on your build (common on EU OOS 14). Skip this step.

#### Step 4: ADB Cache and Dex Reset (2–8 GB)

```bash
# Trim all app caches
adb shell pm trim-caches 999G

# Reset ART compilation artifacts (apps will be slower for a few days)
adb shell pm compile --reset -a

# Trigger background re-optimization
adb shell pm bg-dexopt-job
```

#### Step 5: Clean Junk Folders on sdcard (0.5–2 GB)

```bash
# See what's there
adb shell du -sh /sdcard/*/ 2>/dev/null | sort -rh | head -20

# Common safe-to-delete junk folders
adb shell rm -rf /sdcard/tencent/*
adb shell rm -rf /sdcard/amap/*
adb shell rm -rf /sdcard/Qmap/*
adb shell rm -rf /sdcard/QQBrowser/*
adb shell rm -rf /sdcard/TbsReaderTemp/*
adb shell rm -rf /sdcard/MIUI/*
```

> **Do NOT delete**: `DCIM`, `Pictures`, `Download`, `Movies`, `Music`, `Documents`, `Android`

#### Step 6: Clear App Data on Phone (5–15 GB)

On the phone: **Settings → Apps** → clear cache for the biggest apps:

- **WeChat** (3–5 GB): WeChat → Me → Settings → General → Storage → Clear
- **Chrome** (0.5–1.5 GB): Chrome → ⋮ → Settings → Privacy → Clear browsing data
- **YouTube, Spotify, Outlook** etc.: Settings → Apps → [App] → Storage → Clear Cache
- **Software Update** (0–6 GB): Settings → Apps → ⋮ Show system → Software Update → Storage → Clear Data

#### Step 7: Measure Results

```bash
adb shell df -h /data
```

### Expected Recovery: 15–40 GB (without MSM reflash)

---

## The Remaining ~16 GB Hidden Storage

After all the above steps, you will likely still have ~16 GB of "Other" that cannot be cleared. This is the modem ramdump and system log data locked behind SELinux in `/data/vendor/ramdump/` and `/data/oplus/log/`.

### Option A: MSM Reflash (Recommended)

A factory-level reflash using Qualcomm's Emergency Download mode. This is what OnePlus service centers use.

| Feature | Normal Factory Reset | MSM Reflash |
|---|---|---|
| Clears app data | Yes | Yes |
| Clears hidden ramdump/logs | **No** | **Yes** |
| Rewrites all partitions | No | Yes |
| Bootloader state | Unchanged | Re-locked |
| Widevine L1 | Kept | **Kept** |
| Banking apps (MobilePay, etc.) | Work | **Work** |

**Requirements:**
- Windows PC
- USB data cable
- EU firmware for LE2123 (BA suffix)
- Full backup (Google backup + cloud sync photos + WhatsApp export)

**Steps:**
1. Download LE2123 EU MSM firmware package
2. Install Qualcomm USB drivers on Windows
3. Backup everything on the phone
4. Power off the phone
5. Hold **Vol Up + Vol Down**, plug in USB → enters EDL mode (black screen)
6. Run MsmDownloadTool → select firmware → Start
7. Wait ~5 minutes → phone reboots fresh
8. Restore from Google backup

### Option B: Root (NOT Recommended)

Rooting via `fastboot oem unlock` **will break**:
- **Widevine L1** → drops to L3 (Netflix stuck at 480p, ~30% chance permanent)
- **Play Integrity** → DEVICE check fails → **MobilePay stops working**
- **Banking apps** → may detect unlocked bootloader

Bypass modules (Magisk + PlayIntegrityFork) are unreliable and require constant updates.

---

## Known USB Issues

The OnePlus 9 Pro has a USB 3.0 descriptor negotiation bug. Symptoms:
- Phone charges but doesn't appear in `lsusb`
- Kernel log shows: `got a wrong device descriptor, warm reset device`

**Fixes:**
1. **Plug directly into the laptop** — not through a dock (especially Dell TB16)
2. **Use a USB 2.0 / USB-A port** — the bug only affects USB 3.0 SuperSpeed
3. **Use Wireless ADB** as a fallback (requires ADB 31+):
   ```bash
   adb pair <IP>:<PAIRING_PORT> <CODE>
   adb connect <IP>:<PORT>
   ```

---

## Storage Diagnosis Script

To understand exactly what's eating your storage, run:

```bash
adb shell dumpsys diskstats > diskstats.txt
```

Then parse it with the Python script in [fix-phone-storage.md](fix-phone-storage.md) to get a per-app breakdown.

### Understanding the Phone UI "Others" Number

| Component | Typical Size | Clearable? |
|---|---|---|
| Ramdump + system logs | ~16 GB | MSM reflash only |
| Filesystem overhead (ext4) | ~9 GB | No (normal) |
| App cache | 3–5 GB | Yes (per-app) |
| dalvik-cache | 3–5 GB | Yes (`pm compile --reset`) |
| Misc sdcard junk | 0.5–2 GB | Yes (ADB rm) |
| Play Store APK cache | 0–15 GB | Yes (ADB rm) |

The phone UI "Others" is the sum of ALL of the above — not just the ramdump/logs.

---

## Community Evidence

| Source | Finding |
|---|---|
| XDA 4321573 | User deleted ramdump_modem .elf files, recovered **84 GB** |
| XDA 4376113 | OP9P 128GB same issue; only MSM reflash fixed it |
| XDA 4525693 | OOS 13 did not fix it; "OOS stores logs that it never deletes" |
| Android Police (2021) | Russakovskii recovered 126 GB → 5 GB from Gallery trash alone |
| Gizmochina (2024) | OnePlus shrank `super` partition in OOS 15 (OP13) — indirect admission of wasted space |

---

## Preventive Maintenance

After recovery, repeat monthly:
1. Empty Gallery trash
2. `adb shell pm trim-caches 999G`
3. Clear WeChat/Chrome/YouTube cache
4. If LogKit works on your build: `*#800#` → One key clearlog

---

## Files in This Repository

- `README.md` — This guide (English)
- `README_CN.md` — This guide (Chinese)
- `fix-phone-storage.md` — Technical skill file with all commands and analysis scripts
- `find_lost_storage.md` — Original deep-dive research document

## License

This guide is provided as-is for the OnePlus 9 Pro community. Use at your own risk. MSM reflash will erase all data — always backup first.
