# Orange Box S40 Projector — Deep Hacker Analysis

**Device**: Orange Box S40 (Chinese projector, "TV303" internal codename)  
**SoC**: Allwinner H723 (`sun50iw15p1`)  
**Platform codename**: `hermes` (from VINTF manifest: `device/softwinner/hermes/`)  
**Build**: `h723_evb1-userdebug 14 UP1A.231105.001.A1` (Android 14, API 34)  
**Kernel**: `5.15.167-gd490be1a1370` — **armv7l 32-bit** (NOT arm64)  
**GPU**: Mali-G31, mali_kbase r20p0-01rel0 (`arm,mali-midgard`)  
**RAM**: 816 MB (MemTotal: 835452 kB)  
**Storage**: 7.6 GB eMMC  
**Display**: 1024x600 native, 60Hz, "Built-in Screen" (internal)  
**Status**: Rooted, SELinux permissive, overlayfs active, dm-verity off, OEM unlock supported

---

## 1. DISPLAY ARCHITECTURE — Generic or SoC-Specific?

### Verdict: **100% SoC-Specific Proprietary Stack**

The display pipeline is NOT using standard Linux DRM/KMS, NOT using standard Allwinner DISP2, and NOT using anything in mainline Linux. It is a completely proprietary "TV Panel" architecture.

### Display Driver Chain
```
Application Layer (Android)
    ↓
SurfaceFlinger (PID 318, 12MB)
    ↓
HWComposer HAL: hwcomposer.hermes.so (270KB, namespace: sunxi::HWComposer)
    ↓ android.hardware.graphics.composer@2.2-service (PID 279)
    ↓
Kernel Module Stack:
    tvpanel.ko (Allwinner, GPL, 82KB)
    ├── sunxi_tvtop.ko (built-in)
    │   └── vs_display.ko (V-Silicon Semiconductor, "Display Team")
    │       └── Panel drivers:
    │           ├── panel_dsi_gen.ko (generic DSI panel)
    │           ├── panel_lvds_gen.ko (generic LVDS panel)
    │           └── z20hd720m.ko (specific LCD panel: Z20HD720M)
    ├── sunxi_ksc.ko (Allwinner KSC — Keystone Correction driver)
    │   └── vs_osd.ko (V-Silicon Semiconductor, OSD overlay layer)
    └── backlight.ko → /sys/class/backlight/tv (0-100)
    ↓
Framebuffer: /dev/graphics/fb0 ("TV303 SVP OSD3")
    Resolution: 1024x600 native, 1024x1200 virtual, 32bpp, stride 4096
    Mode: U:1024x600p-0
```

### Why This Matters
- **No /dev/dri/** — no DRM subsystem at all
- `CONFIG_AW_DRM is not set` (explicitly disabled in kernel config)
- `CONFIG_AW_DISP2 is not set` (not using standard Allwinner display either)
- Uses `CONFIG_AW_TVPANEL=m` — a proprietary TV Panel framework
- V-Silicon (`vs_display.ko`, `vs_osd.ko`) is a semiconductor IP company; their drivers are NOT open-source
- The display pipeline is entangled with keystone correction (`sunxi_ksc.ko` + `libksc.so`)

### Key Display Libraries (vendor partition)
| Library | Size | Purpose |
|---------|------|---------|
| `hwcomposer.hermes.so` | 270KB | Hardware composer (sunxi::HWComposer) |
| `gralloc.hermes.so` | 59KB | Graphics memory allocator |
| `libksc.so` | 26KB | Keystone math: projection matrix, coordinate warping, LUT |
| `libdisplayconfig.so` | 14KB | IDisplayConfig HIDL implementation |
| `vendor.display.config@1.0-impl.so` | 17KB | Display configuration service |
| `vendor.aw.homlet.tvsystem.tvserver@1.0.so` | 656KB | TV system server |

### Vendor HIDL Services (display-related)
| Service | PID | Status |
|---------|-----|--------|
| `vendor.display.config@1.0::IDisplayConfig/default` | 279 | Running (inside graphics composer) |
| `vendor.sunxi.tv.graphics@1.0::IDisplay/default` | 195 | Running (inside tvserver) |
| `vendor.aw.homlet.tvsystem.tvserver@1.0::ITvServer/default` | 195 | Running |

---

## 2. Can We Replace with "Proper Android" (GSI)?

### Verdict: **Nearly Impossible**

#### Problem 1: Pure ARM32 Binder (CRITICAL BLOCKER)
```
CONFIG_ANDROID_BINDER_IPC=y
CONFIG_ANDROID_BINDERFS=y
# CONFIG_64BIT is not set     ← CRITICAL
# CONFIG_ARM64 is not set     ← CRITICAL
# CONFIG_COMPAT is not set    ← No 32/64 compatibility layer
```
The kernel runs pure 32-bit ARM despite the Cortex-A53 being 64-bit capable. Almost ALL modern GSI images require `binder64` (arm64 binder). The number of GSI builders supporting pure arm32 binder is essentially zero for Android 14.

#### Problem 2: Proprietary Display Stack
GSI replaces `/system` but inherits `/vendor`. The vendor partition contains the entire proprietary TV Panel display pipeline. A GSI built for standard DRM/KMS or DISP2 would get **no display output** because the vendor's HWComposer expects the proprietary tvpanel/vs_display/vs_osd kernel modules.

In theory, since Treble is enabled and VNDK=34, a GSI that works with this exact vendor partition might work. But no such GSI exists — nobody else uses the H723 TV303 display stack.

#### Problem 3: 816MB RAM
Modern Android 14 GSIs are designed for devices with 2-4GB RAM. With only 816MB, even Go edition would struggle. The current system already uses ~500MB for core Android services.

#### Problem 4: Kernel Architecture
The 5.15.167 kernel is custom-built with Allwinner's proprietary modules compiled in/against it. A GSI can replace `/system` but NOT the kernel. The kernel MUST remain to drive the display, GPU, keystone, motor, and IR receiver.

### What Would Need to Happen for GSI
1. Find/build an arm32 (NOT arm64) GSI for Android 14 with VNDK 34
2. The GSI would need to work with `graphics.composer@2.2` from this vendor partition
3. It must not require DRM-based display
4. It should be lightweight enough for 816MB RAM
5. **Probability: <1%** — effectively not possible without building from AOSP source with Allwinner BSP

---

## 3. Does the SoC Have a Linux Image? Can We Run Linux?

### Verdict: **Theoretically Possible but Extremely Difficult**

#### Mainline Linux Status for Allwinner H723
- `sun50iw15p1` has **ZERO mainline Linux support** as of 2025
- No device tree in mainline kernel (`arch/arm64/boot/dts/allwinner/`)
- No sunxi community support for this specific SoC
- The display pipeline (tvpanel/vs_display/vs_osd/ksc) has NO open-source equivalent

#### Related Projects
| Project | SoC | Status | Relevance |
|---------|-----|--------|-----------|
| HY300 Linux | H713 (sun50iw12p1) | Active, community | Sister chip, different display pipeline |
| Allwinner H616 | sun50iw9p1 | Good mainline | Different architecture |
| Allwinner H6/H5/A64 | Various | Mature mainline | Different generation |

#### Why Linux Would Fail on This Display
The display uses a custom V-Silicon + Allwinner "TV Panel" pipeline:
- `tvpanel.ko` → custom timing controller, DSI/LVDS mux
- `vs_display.ko` → V-Silicon display engine (no mainline driver)
- `vs_osd.ko` → V-Silicon OSD compositor
- `sunxi_ksc.ko` → Hardware keystone correction engine
- `z20hd720m.ko` → Specific LCD panel initialization

None of these have mainline equivalents. Running mainline Linux = **blank screen**.

#### What Could Work (Hybrid Approach)
1. **Keep the vendor kernel** (5.15.167 with all proprietary modules)
2. Replace the rootfs with a minimal Linux userspace (Buildroot/Yocto)
3. Use the framebuffer (`/dev/graphics/fb0`) directly for rendering
4. This gives you a "Linux" but with the proprietary kernel — it's essentially Android without the Java framework
5. **Risk: HIGH** — you lose Android's SurfaceFlinger compositing, touch input pipeline, and all HIDL services

#### The Nuclear Option
Port the HY300 community's work to H723:
- Map register differences between H713 and H723
- Reverse-engineer the tvpanel/vs_display/vs_osd kernel modules
- Write DRM/KMS drivers for the TV303 display pipeline
- Effort: **Months of full-time work by an experienced kernel developer**

---

## 4. Debloat Results — What We Did

### Malware Killed & Disabled
| Package | Type | RAM Freed | Action |
|---------|------|-----------|--------|
| `com.chihihx.launcher` | Malware launcher | ~92MB | APK deleted, uninstalled |
| `com.hx.videotest` | Persistent spy service | ~44MB | APK deleted, disabled (FLAG_PERSISTENT) |
| `com.hx.appcleaner` | Fake cleaner/adware | ~44MB | APK deleted, disabled |
| `com.android.nfhelper` | NFX backdoor | ~44MB | APK deleted, disabled |
| `com.hx.autoupdatenotifier` | Auto-update malware | - | APK deleted, disabled |
| `com.hx.bluetoothspeaker` | BT speaker hijack | - | APK deleted, disabled |
| `com.hx.httpspeedtest` | Network spy | - | APK deleted, disabled |
| `com.hx.httpvideotest` | Video spy | - | APK deleted, disabled |
| `com.hx.projectortest` | Factory spy | - | APK deleted, disabled |
| `com.hx.devicemonitor` | Device monitor | - | APK deleted |
| `com.hx.music` | HX Music player | - | APK deleted |
| `com.waxrain.airplayer2` | Casting bloat | - | Fully uninstalled |
| `com.apowersoft.mirror.tv` | Screen mirror bloat | - | Fully uninstalled |

### Factory Test Apps Disabled
| Package | Action |
|---------|--------|
| `com.softwinner.agingdragonbox` | Disabled |
| `com.softwinner.dragonatt` | Disabled |
| `com.softwinner.dragonbox` | Disabled |
| `com.softwinner.runin` | Disabled |
| `com.android.dreams.basic` | Disabled |
| `com.google.android.partnersetup` | Disabled |

### Clean Launcher Installed
- **FLauncher v0.18.0** installed and set as default home
- Open-source, no ads, no tracking
- Package: `me.efesser.flauncher`
- Set via: `cmd package set-home-activity me.efesser.flauncher/.MainActivity`

### Package Statistics
| Metric | Before | After |
|--------|--------|-------|
| Enabled packages | ~128 | 118 |
| Disabled packages | 0 | 12 |
| Malware processes running | 4 | 0 |
| MemAvailable | ~250MB | ~375MB |

---

## 5. What All Can Be Done — Hacker's Playbook

### ✅ SAFE — Do Right Now
| Action | Difficulty | Impact |
|--------|-----------|--------|
| Install apps via Google Play Store | Trivial | Full app ecosystem available |
| Install Kodi for media playback | Easy | Best media center for projector |
| Private DNS (dns.adguard.com) | Easy | System-wide ad blocking |
| `settings put global stay_on_while_plugged_in 3` | Easy | Prevent screen-off |
| Install file manager (Solid Explorer) | Easy | Browse/manage files |
| Install VLC for direct playback | Easy | Universal media player |
| Adjust display density: `wm density 120` | Easy | More screen real estate |
| Set up Tailscale/WireGuard VPN | Medium | Secure remote access |

### ⚡ MODERATE RISK — Carefully Executable
| Action | Difficulty | Impact |
|--------|-----------|--------|
| Disable Google Play Services for RAM | Medium | Saves ~250MB but loses Play Store |
| Install microG as GMS replacement | Medium | Lighter Google services (~50MB vs ~250MB) |
| Block all HX/CHIHI domains in hosts file | Medium | Prevent any revival/phone-home |
| Build custom boot animation | Medium | Replace vendor branding |
| Write a Tasker/MacroDroid automation | Medium | Auto-launch apps, scheduling |
| Modify build.prop for performance | Medium | Adjust heap sizes, dalvik params |

### ⚠️ HIGH RISK — Expert Only
| Action | Difficulty | Impact |
|--------|-----------|--------|
| Flash custom boot.img (debloated) | Hard | Permanent bloat removal, risks brick |
| Re-partition super for more userdata | Hard | More storage, needs fastboot |
| Port Projectivy Launcher (projector-specific) | Hard | Best projector UI |
| Compile custom kernel modules | Very Hard | Custom features, needs Allwinner BSP/toolchain |
| Port H713 Linux to H723 | Extreme | Full Linux, months of work |

### 🚫 NOT FEASIBLE
| Action | Why |
|--------|-----|
| Flash arm64 GSI | Pure 32-bit binder, no CONFIG_64BIT |
| Use standard DRM/KMS display | tvpanel/vs_display is the only path to the LCD |
| Replace vendor partition | Would kill display, keystone, IR, motor |
| Run mainline Linux kernel | No sun50iw15p1 support, proprietary display |
| Increase RAM | Hardware limitation, soldered |

---

## 6. Critical Hardware Services — DO NOT TOUCH

These services/packages are essential for the projector's hardware to function:

| Package/Service | Function |
|----------------|----------|
| `com.softwinner.tcorrection` | Keystone correction UI |
| `com.softwinner.vis` | Video input service (HDMI) |
| `com.softwinner.awlivetv` | Live TV / HDMI input |
| `com.softwinner.awmanager` | System hardware manager |
| `com.softwinner.settingsassist` | Hardware settings |
| `com.softwinner.awtvinputservice` | TV input framework |
| `tvserver` (native) | TV system HIDL server |
| `gpioservice` (native) | GPIO control (motor, LED) |
| `systemmixservice` (native) | System mix (IR, power) |
| `surfaceflinger` (native) | Display compositor |

---

## 7. Hardware Properties — Reference

### Display Properties
```
persist.vendor.disp.screensize=1024x600
persist.vendor.launcher.platform=H723
ro.surface_flinger.max_frame_buffer_acquired_buffers=2
```

### Keystone Properties
```
persist.display.keystone_ltx=66
persist.display.keystone_lty=77
persist.display.keystone_lbx=0
persist.display.keystone_lby=0
persist.display.keystone_rtx=0
persist.display.keystone_rty=0
persist.display.keystone_rbx=0
persist.display.keystone_rby=0
persist.display.keystone_zoom_h=0
persist.display.keystone_zoom_v=0
```

### Audio
```
persist.vendor.audio.output.active=OUT_SPK
```

### Boot
```
androidboot.selinux=permissive
androidboot.dynamic_partitions=true
boot_type=2
console=ttyAS0,115200
```

---

## 8. Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    USER SPACE                             │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │  FLauncher   │  │  Apps    │  │  Kodi/VLC/etc     │   │
│  └──────┬───────┘  └────┬─────┘  └────────┬──────────┘   │
│         │               │                 │              │
│  ┌──────▼───────────────▼─────────────────▼──────────┐   │
│  │              SurfaceFlinger (PID 318)              │   │
│  └──────────────────────┬────────────────────────────┘   │
│                         │                                │
│  ┌──────────────────────▼────────────────────────────┐   │
│  │    HWComposer (hwcomposer.hermes.so)              │   │
│  │    graphics.composer@2.2-service (PID 279)        │   │
│  └──────────────────────┬────────────────────────────┘   │
│         ┌───────────────┼───────────────┐                │
│  ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐        │
│  │ DisplayCfg  │ │  tvserver   │ │  gpioservice │        │
│  │ HIDL @1.0   │ │ HIDL @1.0  │ │ (motor/LED)  │        │
│  └─────────────┘ └─────────────┘ └──────────────┘        │
├──────────────────────────────────────────────────────────┤
│                    KERNEL SPACE                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │  tvpanel.ko (Allwinner, master module)            │    │
│  │  ├── vs_display.ko (V-Silicon display engine)     │    │
│  │  ├── vs_osd.ko     (V-Silicon OSD compositor)     │    │
│  │  ├── sunxi_ksc.ko  (Keystone correction)          │    │
│  │  └── z20hd720m.ko  (LCD panel init)               │    │
│  └──────────────────────┬───────────────────────────┘    │
│  ┌──────────────────────▼───────────────────────────┐    │
│  │  mali_kbase.ko (Mali-G31 GPU driver, r20p0)      │    │
│  └──────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Framebuffer: /dev/graphics/fb0                   │    │
│  │  "TV303 SVP OSD3" — 1024x600, 32bpp, 60Hz        │    │
│  └──────────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────────┤
│                    HARDWARE                               │
│  CPU: Cortex-A53 x4 (ARMv7l mode)                       │
│  GPU: Mali-G31                                           │
│  RAM: 816MB (DDR3?)                                      │
│  eMMC: 7.6GB                                             │
│  LCD: Z20HD720M (1024x600)                               │
│  Motor: Stepper (focus), GPIO-controlled                  │
│  IR: sunxi-ir-rx                                         │
│  WiFi: xr829 (XR829)                                    │
│  BT: Bluetooth via xr829                                 │
└──────────────────────────────────────────────────────────┘
```

---

## 9. Recommended Next Steps

### Immediate (this session)
1. ✅ Malware killed and disabled
2. ✅ FLauncher installed as default home
3. Install Kodi or Smart YouTube TV for media
4. Set private DNS: `settings put global private_dns_mode hostname_and_port; settings put global private_dns_specifier dns.adguard.com`
5. Add HX/CHIHI domain blocks to `/system/etc/hosts`

### Short-Term
6. Consider disabling Google Play Services and using microG (saves ~250MB RAM)
7. Install a lightweight file manager
8. Set up auto-start for preferred apps

### Medium-Term
9. Dump and archive full firmware for recovery (`dd` backup of mmcblk0)
10. Research if Allwinner releases H723 SDK/BSP publicly
11. Monitor HY300/H713 community for portability to H723

### Long-Term (if motivated)
12. Contact Allwinner for BSP access (they sometimes provide to developers)
13. Reverse-engineer tvpanel.ko module interface for potential Linux DRM wrapper
14. Build custom AOSP from source with Allwinner toolchain

---

*Last Updated: 2026-03-01*  
*Author: Automated hacker analysis via ADB surgical probing*
