# OnePlus 9 Pro Storage Recovery Skill

You are helping the user diagnose and recover storage on their OnePlus 9 Pro (LE2123) running OxygenOS 14. The "Other" category in phone storage settings can bloat to 30–50+ GB due to a known, unfixed combination of: accumulated modem ramdumps, system logs, dalvik-cache residue from OS upgrades, Play Store cache, and app caches.

Work through the steps below **in order**. After each step, run measurement commands and report what was recovered before moving on. Stop and wait for user confirmation when a step requires manual phone interaction.

---

## Step 0 — ADB Connection

### Option A: USB Cable (preferred)

The OnePlus 9 Pro has a known USB 3.0 descriptor negotiation bug with many USB controllers and docking stations. If `lsusb` shows `got a wrong device descriptor`, the phone must connect via **USB 2.0**.

- Plug directly into the laptop (not a dock)
- Use a USB-A port if available (usually USB 2.0)
- If only USB-C ports exist, try a USB-C to USB-A adapter

Prerequisites on the phone:
1. Settings → About Phone → tap **Build number** 7 times → enables Developer Options
2. Settings → Developer Options → enable **USB Debugging**
3. Settings → Developer Options → Default USB configuration → **File Transfer**
4. When plugging in, accept the RSA fingerprint dialog on the phone

Verify:
```bash
adb devices
adb shell getprop ro.product.model        # should return LE2123
adb shell getprop ro.build.version.incremental
adb shell getprop ro.boot.slot_suffix      # returns _a or _b
```

### Option B: Wireless ADB (fallback if USB fails)

Requires ADB version 31+ (check with `adb version`). If your system ADB is older, download the latest platform-tools:
```bash
wget -q https://dl.google.com/android/repository/platform-tools-latest-linux.zip -O /tmp/platform-tools.zip
unzip -q /tmp/platform-tools.zip -d /tmp/
# Use /tmp/platform-tools/adb for all subsequent commands
```

On the phone: Settings → Developer Options → **Wireless debugging** → Enable → tap **Pair device with pairing code**.

```bash
# Kill old ADB server first to avoid version conflicts
adb kill-server

# Pair (one-time) — use the pairing port and code shown on phone
/tmp/platform-tools/adb pair <IP>:<PAIRING_PORT> <CODE>

# Connect — use the main port shown in Wireless debugging settings
/tmp/platform-tools/adb connect <IP>:<PORT>
/tmp/platform-tools/adb devices
```

**Note:** Wireless ADB has fewer permissions than USB ADB. `pm clear` will not work over wireless. USB is strongly preferred.

---

## Step 1 — Baseline Measurement

```bash
adb shell dumpsys diskstats > /tmp/diskstats_before.txt
adb shell df -h /data > /tmp/df_before.txt
adb shell du -sh /sdcard/Android/data/* 2>/dev/null | sort -hr | head -20
adb shell du -sh /sdcard/Android/media/* 2>/dev/null | sort -hr | head -10
adb shell du -sh /sdcard/DCIM /sdcard/Pictures /sdcard/Download /sdcard/Movies /sdcard/Music /sdcard/Documents 2>/dev/null
adb shell du -sh /sdcard/*/ 2>/dev/null | sort -rh | head -20
```

Parse diskstats to get per-app breakdown:
```bash
python3 - << 'PYEOF'
import re

data = open('/tmp/diskstats_before.txt').read()

def val(label):
    m = re.search(rf'^{label}: (\d+)', data, re.MULTILINE)
    return int(m.group(1)) if m else 0

def extract_quoted_list(text):
    m = re.search(r'Package Names: \[(.*)\]', text)
    if m:
        return re.findall(r'"([^"]+)"', m.group(1))
    return []

def extract_list(label, text):
    m = re.search(rf'{label}: \[([\d,]+)\]', text)
    if m:
        return [int(x) for x in m.group(1).split(',')]
    return []

categories = {
    'App Code':    val('App Size'),
    'App Data':    val('App Data Size'),
    'App Cache':   val('App Cache Size'),
    'Photos':      val('Photos Size'),
    'Videos':      val('Videos Size'),
    'Audio':       val('Audio Size'),
    'Downloads':   val('Downloads Size'),
    'Other':       val('Other Size'),
}

print("=== Storage Breakdown ===")
total = 0
for k, v in categories.items():
    gb = v / 1e9
    total += gb
    print(f"  {k:<12} {gb:>8.2f} GB")
print(f"  {'Sum':<12} {total:>8.2f} GB")

names = extract_quoted_list(data)
dat_sz = extract_list('App Data Sizes', data)
cch_sz = extract_list('Cache Sizes', data)

if names and dat_sz:
    combined = sorted(zip([d+c for d,c in zip(dat_sz, cch_sz)], dat_sz, cch_sz, names), reverse=True)[:15]
    print(f"\n=== Top 15 Apps (Data + Cache) ===")
    print(f"  {'Total MB':>9} {'Data MB':>9} {'Cache MB':>9}  Package")
    for t, d, c, pkg in combined:
        print(f"  {t/1e6:>9.1f} {d/1e6:>9.1f} {c/1e6:>9.1f}  {pkg}")
PYEOF
```

### Understanding the "Others" number

The phone UI "Others" includes multiple things added together:
- **diskstats "Other"** (~16 GB): ramdump files, system logs, dalvik-cache, OTA residue — SELinux-protected, shell cannot access
- **App cache** (~3-5 GB): clearable per-app
- **Filesystem overhead** (~9 GB): ext4 metadata, journals, reserved blocks — normal and unavoidable
- **Uncategorized sdcard files**: folders like `tencent/`, `amap/`, `Dictionary Library/`, etc.

This is why the phone may show 35+ GB "Others" while diskstats only reports 16 GB in the "Other" field.

---

## Step 2 — Gallery Trash (manual)

Tell the user:

> **On the phone:** Open Gallery → Collections → Recently Deleted → Empty trash.
> Also open Files by Google → Clean tab → run it.
> This can recover 0–50 GB if gallery trash has built up (known OnePlus 9 Pro MediaStorage bug since 2021).

Wait for confirmation, then measure:
```bash
adb shell df -h /data
```

---

## Step 3 — Play Store Cache

The Google Play Store (`com.android.vending`) can accumulate 5–15 GB of downloaded APK cache in `/sdcard/Android/data/com.android.vending/`. Check and clear:

```bash
adb shell du -sh /sdcard/Android/data/com.android.vending/
adb shell rm -rf /sdcard/Android/data/com.android.vending/*
adb shell df -h /data
```

---

## Step 4 — LogKit Log Clear (manual)

On the phone, open the **dialer** and type `*#800#`.

**Important:** On some EU OxygenOS 14 builds, `*#800#` opens the **Feedback** app instead of LogKit. If you see a "Feedback" screen with categories like "System experience", "Apps", etc., LogKit has been replaced on your build. Skip this step — the hidden logs can only be cleared via MSM reflash (see Step 9).

If LogKit opens correctly, tap **"One key clearlog"** (or "一键清除日志"). This clears `/data/oplus/log/`, `/data/anr/`, `/data/tombstones/`, and may clear `/data/vendor/ramdump/`. Expected recovery: 2–30 GB.

```bash
adb shell df -h /data
```

---

## Step 5 — ADB Cache Trim and Dex Reset

```bash
# Trim all app caches aggressively (system_server clears all /data/data/<pkg>/cache/)
adb shell pm trim-caches 999G

# Reset dalvik/ART compilation artifacts (frees 2–5 GB of .odex/.vdex/.art)
# Apps will cold-start slower for a few days until bg-dexopt rebuilds
adb shell pm compile --reset -a

# Trigger background dexopt (screen on + charging recommended)
adb shell pm bg-dexopt-job

adb shell df -h /data
```

---

## Step 6 — Clear Misc sdcard Folders

Scan for junk folders on sdcard that apps leave behind:
```bash
adb shell du -sh /sdcard/*/ 2>/dev/null | sort -rh | head -20
```

Common junk folders safe to clear (adjust based on scan results):
```bash
adb shell rm -rf /sdcard/tencent/*
adb shell rm -rf /sdcard/amap/*
adb shell rm -rf /sdcard/Qmap/*
adb shell rm -rf /sdcard/QQBrowser/*
adb shell rm -rf /sdcard/TbsReaderTemp/*
adb shell rm -rf /sdcard/MIUI/*
adb shell rm -rf /sdcard/voip-data/*
adb shell rm -rf /sdcard/.eudb_en/*
```

**Do NOT delete:** `DCIM`, `Pictures`, `Download`, `Movies`, `Music`, `Documents`, `Android` — these contain user data.

---

## Step 7 — Clear App Data (manual, on phone)

Open **Settings → Apps** on the phone. Clear cache/data for the largest offenders identified in Step 1. Common ones:

| App | Typical waste | Action |
|---|---|---|
| WeChat | 3–5 GB | WeChat → Me → Settings → General → Storage → Clear |
| Google Play Books | 1–2 GB | Settings → Apps → Play Books → Storage → Clear Data |
| Chrome | 0.5–1.5 GB | Chrome → ⋮ → Settings → Privacy → Clear browsing data |
| YouTube | 0.5–1 GB | Settings → Apps → YouTube → Storage → Clear Cache |
| Spotify | 0.5–1 GB | Settings → Apps → Spotify → Storage → Clear Cache |
| Outlook | 0.5–1 GB | Settings → Apps → Outlook → Storage → Clear Cache |
| Google TTS | 0.5–0.7 GB | Settings → Apps → Google TTS → Storage → Clear Data |

Also clear: Settings → Apps → (⋮ show system) → **Software Update** → Storage → Clear Data (removes 0–6 GB of stale OTA packages).

---

## Step 8 — Final Measurement

```bash
adb shell dumpsys diskstats > /tmp/diskstats_after.txt
adb shell df -h /data

python3 - << 'PYEOF'
import re

def parse(path):
    data = open(path).read()
    def val(label):
        m = re.search(rf'^{label}: (\d+)', data, re.MULTILINE)
        return int(m.group(1)) if m else 0
    return {k: val(k) for k in ['App Size','App Data Size','App Cache Size','Photos Size','Videos Size','Audio Size','Downloads Size','Other Size']}

b = parse('/tmp/diskstats_before.txt')
a = parse('/tmp/diskstats_after.txt')

print(f"{'Category':<16} {'Before':>8} {'After':>8} {'Saved':>8}")
print("-" * 46)
for k in b:
    bv, av = b[k]/1e9, a[k]/1e9
    print(f"{k:<16} {bv:>8.2f} {av:>8.2f} {bv-av:>+8.2f}")
total_b = sum(b.values()) / 1e9
total_a = sum(a.values()) / 1e9
print("-" * 46)
print(f"{'Total':<16} {total_b:>8.2f} {total_a:>8.2f} {total_b-total_a:>+8.2f}")
PYEOF
```

---

## Step 9 — MSM Reflash (nuclear option for remaining hidden storage)

If Steps 1–8 recovered less than expected and 15+ GB remains in hidden "Other" (confirmed by gap analysis), the only way to clear `/data/vendor/ramdump/` and `/data/oplus/log/` without root is a full MSM reflash.

### What MSM reflash does
- Writes every partition from scratch via Qualcomm EDL (Emergency Download) mode
- Completely wipes `/data` including SELinux-protected ramdump/log directories
- Re-locks bootloader — Widevine L1 preserved, Play Integrity passes
- MobilePay, Danske Bank, and all banking apps continue to work

### Requirements
- **Windows PC** (MsmDownloadTool is Windows-only)
- **USB data cable** (the one that works with your PC)
- **EU firmware** for LE2123 (filename contains `BA` suffix for EU builds)
- **Full backup**: Google backup, cloud sync photos, WhatsApp chat export

### Steps
1. Download the LE2123 EU MSM firmware package from OnePlus firmware repositories
2. Install Qualcomm USB drivers on the Windows PC
3. Extract MsmDownloadTool from the firmware package
4. **Backup everything**: Settings → System → Backup; sync photos to Google Photos; export WhatsApp chats
5. Power off the phone completely
6. Hold **Vol Up + Vol Down** simultaneously, then plug in USB cable → phone enters EDL mode (screen stays black, Qualcomm 9008 device appears in Windows Device Manager)
7. Open MsmDownloadTool → select the firmware → click Start
8. Wait for completion (~5–10 minutes) → phone reboots to fresh setup
9. Restore from Google backup during setup

### Do NOT root instead
- `fastboot oem unlock` wipes `/data` (same as MSM) BUT:
  - Widevine drops L1 → L3 (~30% chance permanent even after re-lock)
  - Play Integrity DEVICE fails → **MobilePay DK refuses to work**
  - Banking apps may detect unlocked bootloader
  - Magisk bypass modules require constant updates and are unreliable
- MSM reflash achieves the same cleanup without any of these risks

---

## Key Technical Facts

- **A/B device**: `/data` is shared by both slots. The inactive slot does NOT consume userdata space. `fastboot --set-active` will not help.
- **`/my_bigball`, `/my_heytap`, `/my_stock`** etc. are read-only logical partitions inside `super`, physically separate from userdata. They cannot cause the "Other" bloat.
- **OOS 14 is the final major version** for OnePlus 9 Pro. OnePlus has never released a cleanup tool and will not fix this.
- **OOS 14 restricts ADB shell permissions**: `pm clear` is blocked even via USB ADB. App data can only be cleared manually on the phone or via factory/MSM reset.
- **USB 3.0 bug**: OnePlus 9 Pro fails USB descriptor negotiation with many USB 3.0 controllers (especially Dell docks). Use USB 2.0 ports or wireless ADB as fallback.

---

## Expected Recovery Summary

| Step | Expected Recovery |
|---|---|
| Gallery trash | 0–50 GB |
| Play Store cache | 0–15 GB |
| LogKit clear (if available) | 2–30 GB |
| pm trim-caches + dex reset | 2–8 GB |
| Misc sdcard cleanup | 0.5–2 GB |
| Manual app data clear | 5–15 GB |
| MSM reflash (hidden ramdump/logs) | 10–20 GB |
| **Total possible** | **20–50+ GB** |
