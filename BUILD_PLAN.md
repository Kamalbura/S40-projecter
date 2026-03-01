# Custom Clean OS Build Plan — Orange Box S40 (Allwinner H723)

## Executive Summary

After extensive research across GitHub, XDA, community projects, and Allwinner documentation, there are **3 viable approaches** to getting a clean OS on this projector, ranked by risk and complexity:

| # | Approach | Risk | Complexity | Preserves Keystone |
|---|----------|------|------------|-------------------|
| **1** | **Deep Debloat + Clean Launcher** (RECOMMENDED FIRST) | Very Low | Low | YES |
| **2** | **GSI Flash** (Generic System Image) | Medium | Medium | MAYBE |
| **3** | **Custom Firmware Build** | High | Very High | Requires porting |

**Recommendation**: Start with Approach 1 immediately (zero risk, reversible). If unsatisfied, attempt Approach 2. Approach 3 is a long-term project.

---

## Device Quick Reference

| Property | Value |
|----------|-------|
| SoC | Allwinner H723 (sun50iw15p1) |
| CPU | Quad-Core ARM Cortex-A53 (64-bit capable) |
| Kernel | armv7l (32-bit kernel!) |
| Userspace | armeabi-v7a (32-bit), zygote32 |
| Binder | Needs verification (likely 32-bit given armv7l kernel) |
| Android | 14 (API 34), VNDK 34 |
| RAM | 816 MB |
| Storage | 7.6 GB eMMC |
| Partitions | A/B, dynamic, super=3.5GB |
| Treble | YES (`ro.treble.enabled=true`) |
| Bootloader | Unlockable (`oem_unlock_supported=1`) |
| Boot State | Orange (dm-verity disabled) |
| SELinux | Permissive |
| Root | YES (ADB root) |
| Overlayfs | Active (system/vendor/product all RW) |

---

## APPROACH 1: Deep Debloat + Clean Launcher (RECOMMENDED FIRST)

### Why This First
- **Zero risk** — all operations are reversible with `pm enable`
- **No flashing** — works over ADB immediately
- **Preserves ALL hardware features** — keystone, motor, gsensor, HDMI-CEC, IR remote
- **Already has root** — dm-verity disabled, overlayfs active, adb root
- **Malware already removed** — 38 items eliminated in Phase 2
- **Community proven** — HY300 owners report success with Projectivy Launcher

### What We Gain
- Clean Android TV launcher (Projectivy Launcher)
- All bloatware disabled/removed
- Security apps installed (AdGuard, NetGuard)
- YouTube TV, Chrome, lightweight apps
- Custom DNS (encrypted)
- C2 blocking (already active)
- All projector features preserved

### Step-by-Step Plan

#### Step 1: Verify Current State
```bash
adb connect 192.168.0.106:5555
adb shell

# Check what's still running
pm list packages -e  # enabled packages
dumpsys activity top | grep "ACTIVITY"  # current activity
cat /proc/meminfo | head -5  # memory usage
```

#### Step 2: Enumerate ALL Packages & Classify
```bash
# List all packages with paths
pm list packages -f > /sdcard/all_packages.txt
adb pull /sdcard/all_packages.txt

# Categories:
# KEEP (essential for hardware):
#   - com.softwinner.tcorrection (KeystoneCorrection)  
#   - com.softwinner.videoinputservice (VideoInputService)
#   - com.softwinner.awmanager (AwManager)
#   - com.android.providers.tv (TV Provider)
#   - com.softwinner.awlivetv (HDMI/ATV input)
#   - com.android.tv.settings (System Settings)
#   - com.google.android.youtube.tv (YouTube TV)
#   - Any com.android.* core services
#
# REMOVE (bloatware/malware):
#   - com.chihi.* (CHIHI malware - already removed)
#   - com.hx.* (HX malware - already removed)
#   - Any Chinese app stores
#   - Factory test apps
#   - Preinstalled 3rd party apps (Netflix unless wanted)
#   - CHIHI_Launcher
#
# EVALUATE (might be needed):
#   - com.softwinner.* (check each one)
#   - com.allwinner.* (platform utilities)
```

#### Step 3: Install Clean Launcher
```bash
# Option A: Projectivy Launcher (BEST for projectors)
# Download from: https://play.google.com/store/apps/details?id=com.spocky.projengmenu
# Or APKMirror: https://www.apkmirror.com/apk/spocky/projectivy-launcher-android-tv/

adb install projectivy_launcher.apk

# Option B: FLauncher (open source, lightweight)
# https://f-droid.org/en/packages/me.efesser.flauncher/
adb install flauncher.apk

# Set as default launcher
# Method 1: Via Settings > Apps > Default apps > Home app
# Method 2: Via ADB
cmd package set-home-activity com.spocky.projengmenu/.ui.main.MainActivity

# Disable CHIHI Launcher (if not already removed)
pm disable-user --user 0 com.chihi.launcher
# Or if we have root:
pm uninstall --user 0 com.chihi.launcher
```

#### Step 4: Aggressive Bloatware Removal
```bash
# Safe to disable/remove (verify package names on device first):
# Chinese app stores and SDKs
pm uninstall --user 0 com.chihi.store          # if exists
pm uninstall --user 0 com.hx.appcleaner        # if exists

# Factory test apps (identified in Phase 2)
pm uninstall --user 0 com.allwinner.schemetest
pm uninstall --user 0 com.softwinner.firewalltest
pm uninstall --user 0 com.softwinner.dragonbox
pm uninstall --user 0 com.softwinner.dragonfire
pm uninstall --user 0 com.softwinner.dragonphone

# Unnecessary preinstalls
pm uninstall --user 0 com.waxrain.airplayer2     # AirPlayer
pm uninstall --user 0 com.apowersoft.mirror.tv   # ApowerMirror

# Evaluate Netflix (user preference)
# pm uninstall --user 0 com.netflix.mediaclient
```

#### Step 5: Install Essential Apps
```bash
# YouTube TV (already installed - keep it)
# Chrome/Browser
adb install chrome_tv.apk  # or use existing Browser2

# Security: AdGuard for Android TV
# Download: https://adguard.com/en/adguard-android-tv/overview.html
adb install adguard_tv.apk

# Alternative: NetGuard (free, open source firewall)
# Download: https://github.com/M66B/NetGuard/releases
adb install netguard.apk

# File manager (lightweight)
# Solid Explorer or MiXplorer

# VLC (lightweight media player)
adb install vlc_android.apk
```

#### Step 6: DNS Security Configuration
```bash
# Set encrypted DNS (via ADB as root)
settings put global private_dns_mode hostname
settings put global private_dns_specifier dns.adguard.com

# OR use Cloudflare
settings put global private_dns_specifier one.one.one.one

# Verify
settings get global private_dns_mode
settings get global private_dns_specifier
```

#### Step 7: Additional Hardening
```bash
# Verify C2 block script is active
cat /system/etc/init/block_c2.rc

# Block unknown sources (optional - makes sideloading harder)
settings put secure install_non_market_apps 0

# Disable USB debugging when not in use
settings put global adb_enabled 0  # BE CAREFUL — you'll need physical access to re-enable

# Verify no unexpected network connections
netstat -tlnp
cat /proc/net/tcp
```

#### Step 8: Memory Optimization
```bash
# Kill background processes aggressively
settings put global always_finish_activities 1

# Reduce animation scales (faster + less RAM)
settings put global window_animation_scale 0.5
settings put global transition_animation_scale 0.5
settings put global animator_duration_scale 0.5

# Set background process limit
settings put global background_process_limit 2
```

---

## APPROACH 2: GSI Flash (Generic System Image)

### Feasibility Assessment

| Requirement | Status | Notes |
|-------------|--------|-------|
| Treble support | YES | `ro.treble.enabled=true` |
| A/B partitions | YES | slot_suffix=_a |
| Dynamic partitions | YES | super partition |
| VNDK version | 34 | Android 14 |
| Architecture | armv7l kernel | **CRITICAL: Need to verify binder type** |
| Bootloader unlock | YES | `oem_unlock_supported=1` |

### Critical Unknown: Binder Architecture

Before attempting GSI, MUST verify:
```bash
# Check kernel binder configuration
adb shell cat /dev/binderfs/binder-control 2>/dev/null
adb shell ls -la /dev/binder*
adb shell cat /proc/config.gz | gunzip | grep -i binder
# OR
adb shell zcat /proc/config.gz | grep -i BINDER

# Check if 64-bit binder
adb shell getprop ro.hardware.binder
adb shell getprop ro.binder.64bit
```

**If armv7l kernel with 32-bit binder** → Need `arm32` (a-only) GSI  
**If 64-bit kernel with 32-bit userspace** → Need `arm32_binder64` (a64) GSI  
**Our likely case**: armv7l = 32-bit kernel = 32-bit binder = need `arm32` GSI

### GSI Options (if compatible)

#### Option A: phhusson's AOSP GSI (most tested)
- Latest: AOSP 12.1 v416 (phhusson stopped at Android 12)
- Has `system-squeak-arm32_binder64-ab-vndklite-gogapps.img.xz` (686 MB)
- Has `system-squeak-arm32_binder64-ab-vanilla.img.xz` (457 MB)
- **Problem**: Android 12 GSI + Android 14 vendor = VNDK mismatch
- May not boot or have issues

#### Option B: AndyYan's LineageOS 21 GSI
- Based on Android 14 (LineageOS 21 = Android 14)
- Available on SourceForge: `sourceforge.net/projects/andyyan-gsi/files/`
- Has `a64` (arm32_binder64) and `arm64` variants
- **Need arm32 variant** — may not be available (most GSI builders dropped pure arm32)
- TrebleDroid based, better vendor compatibility

#### Option C: Google's Official GSI
- `developer.android.com/topic/generic-system-image/releases`
- Android 14 GSI available
- Only `arm64` variant officially — not suitable for our 32-bit kernel

### GSI Flash Procedure (if compatible GSI found)

**WARNING: This replaces the system partition. BACKUP FIRST.**

```bash
# 1. Full backup of current system (we already have firmware dumps)
# Verify firmware_dump/ has all partitions

# 2. Verify partition layout
adb shell ls -la /dev/block/by-name/
adb shell cat /proc/partitions

# 3. Check super partition details
adb shell cat /proc/device-tree/firmware/android/super_partition_name
adb shell lpdump  # list logical partitions

# 4. Flash GSI via fastboot
adb reboot bootloader
# Wait for fastboot mode

fastboot getvar all  # verify device info
fastboot erase system
fastboot flash system gsi_image.img

# If dynamic partitions:
fastboot reboot fastboot  # enter fastbootd
fastboot flash system gsi_image.img

# Disable verified boot
fastboot --disable-verification flash vbmeta vbmeta.img

# Wipe data
fastboot -w

# Reboot
fastboot reboot
```

### GSI Risks & Limitations
1. **Keystone correction may not work** — vendor HIDL services may conflict with new system
2. **Display might not initialize** — MIPS co-processor needs vendor blobs
3. **WiFi might not work** — vendor kernel modules needed
4. **Motor control lost** — sysfs interface exists in vendor but system apps needed
5. **IR remote might not work** — depends on vendor HAL
6. **No rollback without firmware reflash** — need PhoenixSuit or USB flash

### Mitigation: Keep Vendor Partition Intact
The Treble architecture means:
- `system` partition = Android framework (replaceable with GSI)
- `vendor` partition = Hardware drivers/HALs (keep)
- `product` partition = OEM customizations (can be cleaned)

If we flash GSI to system only, vendor HIDL services should still provide:
- `IDisplayConfig@1.0` (keystone)
- `ITvServer@1.0` (TV system)  
- `IDisplay@1.0` (display control)

---

## APPROACH 3: Custom Firmware Build

### Community Research Results

#### HY300 Linux Porting Project (Most Relevant)
- **Repo**: `github.com/shift/sun50iw12p1-research`
- **SoC**: Allwinner H713 (sun50iw12p1) — sister chip to H723
- **Status**: 8 of 9 phases complete, Phase IX (hardware testing) in progress
- **Has**: U-Boot, device tree, kernel modules, MIPS co-processor driver
- **Missing**: Not Android — targets mainline Linux (Kodi/NixOS)
- **Relevance**: Shows it's possible but VERY different SoC (H713 ≠ H723)

#### lolmam's HY300-H713-Research
- **Repo**: `github.com/lolmam/HY300-H713-Research`
- **Focus**: Root + eMMC dump + Magisk patch on HY300 Pro+
- **Method**: Dump boot_a → Magisk patch → fastboot flash back
- **Relevance**: Similar approach possible for our H723

#### XDA Tutorial
- **Thread**: `xdaforums.com/t/beta-android-tv-and-root-tutorial-for-allwinner-h713-based-projectors.4744020/`
- **Method**: PhoenixSuit + imgRePacker + Magisk
- **Requires**: Factory firmware .img file
- **Our advantage**: We already have root and full dumps

#### Key Finding: No H723-Specific BSP Available
- **lindenis-org** (Allwinner official BSP partner): Only has V5/V536/V833/V853 SDKs
- **No H723/sun50iw15 kernel sources found** on GitHub
- **No community ROM exists** for H723 specifically
- **Allwinner SDK is not open source** — only available to licensed OEMs

### Why Full Custom Build Is Not Recommended Now
1. No H723 BSP available publicly
2. SoC has MIPS co-processor for display — needs proprietary display.bin firmware
3. GPU (Mali) needs proprietary blobs or Panfrost (incomplete for this SoC)
4. WiFi driver (likely AIC8800 or similar) needs proprietary firmware
5. Building AOSP from source requires BSP + ~200GB disk + powerful Linux machine
6. Even the HY300 (H713) Linux project hasn't achieved hardware testing yet after 127 commits

---

## Hardware Flash Methods

### Method 1: PhoenixSuit (Windows, USB)
1. Install PhoenixSuit + Allwinner USB drivers
2. Connect projector to PC via USB-A to USB-A cable
3. Enter FEL mode: Disconnect power → Connect USB → Press power button
4. PhoenixSuit detects device → Select firmware → Flash
5. **Used by**: XDA tutorial, HY300 community

### Method 2: USB Stick Auto-Update (Safest)
1. Format USB stick as FAT32
2. Create `update/` folder at root
3. Place `auto_update.txt` containing: `sunxi_flash write update/update.img firmware`
4. Place firmware as `update/update.img`
5. Disconnect projector power
6. Insert USB stick → Plug power (don't press power button)
7. Green progress bar appears → Wait until complete
8. **Note**: All filenames must be lowercase

### Method 3: fastboot (if bootloader accessible)
```bash
adb reboot bootloader
# or
adb reboot fastboot  # fastbootd for dynamic partitions
```

### Method 4: PhoenixCard (SD Card Boot)
- Uses PhoenixCard Windows tool to create bootable SD card
- Two modes: **Boot card** (boot from SD) or **Mass production card** (flash to eMMC)
- **Boot card** is the safe testing method

### Method 5: ADB + Root (Current Method — Safest)
- Already have root access via ADB
- overlayfs active on system/vendor/product
- Can modify anything without flashing
- Fully reversible

---

## Security Plan

### Layer 1: Network Security (Already Done)
- [x] C2 server blocking script active (`/system/etc/init/block_c2.rc`)
- [x] Malware packages removed (38 items)

### Layer 2: DNS Security
- [ ] Configure Private DNS (AdGuard DNS or Cloudflare)
- [ ] Install AdGuard for Android TV (ad blocking + DNS filtering)

### Layer 3: Firewall
- [ ] Install NetGuard (per-app network control, no root required)
- [ ] OR use AdGuard's built-in firewall
- [ ] Block all apps from internet except: YouTube, Chrome, Google Play

### Layer 4: App Hardening
- [ ] Disable installation from unknown sources
- [ ] Remove/disable all unnecessary system apps
- [ ] Set background process limit to 2
- [ ] Disable ADB when not in use

### Layer 5: Monitoring
- [ ] Set up periodic `netstat` checks via script
- [ ] Monitor `/proc/net/tcp` for unexpected connections
- [ ] Check running processes periodically

---

## Recommended Apps

| App | Purpose | Source | Size |
|-----|---------|--------|------|
| **Projectivy Launcher** | Clean TV launcher | Play Store | ~15 MB |
| **YouTube TV** | Video (already installed) | Pre-installed | — |
| **Chrome** | Web browser | APKMirror (TV version) | ~180 MB |
| **AdGuard TV** | Ad blocking + DNS | adguard.com | ~25 MB |
| **NetGuard** | Firewall | F-Droid/GitHub | ~5 MB |
| **VLC** | Media player | F-Droid/Play Store | ~35 MB |
| **Solid Explorer** | File manager | Play Store | ~15 MB |
| **Send Files to TV** | File transfer | Play Store | ~5 MB |

**Total additional space needed**: ~280 MB (well within available storage)

---

## Next Steps (In Order)

### IMMEDIATE (Approach 1 — Debloat)
1. [ ] Connect to device via ADB
2. [ ] List ALL enabled packages with paths
3. [ ] Classify each package (KEEP / REMOVE / EVALUATE)
4. [ ] Install Projectivy Launcher
5. [ ] Set as default launcher
6. [ ] Disable/remove all bloatware
7. [ ] Install AdGuard TV + configure DNS
8. [ ] Install NetGuard + configure firewall rules
9. [ ] Memory optimization settings
10. [ ] Full verification & testing

### IF UNSATISFIED (Approach 2 — GSI)
1. [ ] Verify binder architecture (32-bit vs 64-bit)
2. [ ] Find compatible GSI image (ARM32 + Android 14 + A/B)
3. [ ] Test PhoenixCard SD boot capability
4. [ ] Create full backup via PhoenixSuit
5. [ ] Flash GSI to system partition only
6. [ ] Test keystone, WiFi, display, remote
7. [ ] If problems, restore from backup

### LONG TERM (Approach 3 — Custom Build)
1. [ ] Monitor HY300 Linux porting project
2. [ ] Wait for H723 BSP leak or Allwinner SDK access
3. [ ] Contribute device tree for sun50iw15p1
4. [ ] Build custom Android from vendor BSP when available

---

## Key Community Resources

| Resource | URL | Relevance |
|----------|-----|-----------|
| HY300 Hacking Gist | `gist.github.com/probonopd/3ad6b7777caea1503f00d5fe7710ad06` | Debloat, FEL, Treble, launcher |
| HY300 Linux Porting | `github.com/shift/sun50iw12p1-research` | H713 device tree, kernel |
| HY300 Root Research | `github.com/lolmam/HY300-H713-Research` | eMMC dump, Magisk, root |
| XDA H713 Tutorial | `xdaforums.com/t/...4744020/` | PhoenixSuit, Android TV |
| phhusson GSI | `github.com/phhusson/treble_experimentations` | GSI images (AOSP 12.1) |
| AndyYan GSI | `sourceforge.net/projects/andyyan-gsi/` | LineageOS 21 GSI |
| Projectivy Launcher | Play Store: `com.spocky.projengmenu` | Clean TV launcher |
| AdGuard TV | `adguard.com/en/adguard-android-tv/` | DNS + ad blocking |
| NetGuard | `github.com/M66B/NetGuard` | Per-app firewall |
| Allwinner Flash Guide | PhoenixSuit + PhoenixCard | Firmware tools |

---

## Architecture Decision Record

### Why Debloat First (Not GSI or Custom Build)

1. **We already have everything we need**: Root access, RW filesystem, malware removed
2. **The launcher is NOT the OS**: CHIHI_Launcher is just an app — replacing it doesn't require a new OS
3. **Keystone works without the launcher**: Community confirmed (HY300 users) that keystone correction works even with launcher disabled — the hardware control is in vendor HIDL services, not the launcher
4. **GSI risk**: Our device uses armv7l (32-bit) kernel which is unusual — most GSI builds target arm64 or arm32_binder64. Finding a compatible GSI is uncertain.
5. **No community support for H723**: Unlike H713 (which has active community), our H723 has zero community ROM support
6. **RAM concern**: 816 MB is tight — a debloated stock ROM uses less RAM than a fresh GSI that needs setup
7. **Reversibility**: Everything in Approach 1 can be undone with `pm enable` commands

### The "Clean OS" User Experience

After Approach 1, the user experience will be:
- Boot → Projectivy Launcher (clean, customizable, no ads)
- YouTube TV works
- Chrome for browsing
- Keystone correction via Settings or dedicated app
- HDMI input switching works
- IR remote works
- No malware, no Chinese apps, no tracking
- Encrypted DNS, firewall protection
- ~50% less RAM usage than factory state

This is effectively a "custom clean OS" without the risks of reflashing.
