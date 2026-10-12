# Poco M4 Pro 4G (`miel` / SKU `mielp`): device notes

Everything learned about my phone: live adb dumps, a snapshot of the LineageOS device tree, and research for a fingerprint-sensor gesture app.

Collected 2026-10-12 over USB adb (no root). Personal identifiers (serial, IMEI, MACs, android_id, ICCID, fingerprint IDs) are replaced with `<REDACTED…>`.

## Identity

| Item | Value |
|---|---|
| Marketing name | POCO M4 Pro (4G), India |
| Model number | `2201117PI` |
| Codename | **`miel`** (`ro.boot.hwname`, `ro.product.vendor.device`) |
| Hardware SKU | **`mielp`** (`ro.boot.product.hardware.sku`) = POCO-branded `miel` |
| Product name | `miel_p_in`, mod device `fleur_in_global` |
| Sibling codenames | `fleur` (Redmi Note 11S / POCO M4 Pro 4G global, with NFC), `fleurp`, `miel`, `mielp`. All built from one device tree |
| Boot image built as | `Redmi/fleur/fleur` (shared platform build) |
| Region / hwc | IN / India, hw version 1.19.5 |

## Software

| Item | Value |
|---|---|
| OS | HyperOS 1.0 (`OS1.0`), build `V816.0.10.0.TKEINXM` |
| Android | 13 (SDK 33), first API level 30 (launched on Android 11) |
| Security patch | 2024-12-01 (build date 2024-12-09) |
| Kernel | Linux 4.19.191 (`4.19.191-g6e5440f6a542`), aarch64 |
| Baseband | `MOLY.LR13.R2.MP.V16.1.P89` |
| Bootloader | **locked** (`ro.boot.flash.locked=1`, vbmeta `locked`, verified boot `green`) |
| Partitions | A/B (slot `_a`), dynamic `super`, virtual A/B with compression; FBE + metadata encryption |

## Hardware

| Item | Value |
|---|---|
| SoC | MediaTek MT6781 (Helio G96): 2× Cortex-A76 (`0xd0b`) + 6× Cortex-A55 (`0xd05`) |
| RAM | 6 GB (MemTotal 5.86 GB) |
| Storage | UFS (sda/sdb/sdc), `/data` 48 GB → 64 GB model |
| Display | 1080×2400, 440 dpi, 60/90 Hz, centre punch-hole (93 px), HDR10-type 1, 500 nits max reported |
| Touchscreen | FocalTech `fts_ts` |
| Fingerprint | **FPC**, Truly module, side-mounted in power button (`ro.hardware.fp.sideCap=true`) |
| Cameras | 4 camera IDs. Rear main 4.71 mm, reports 4624×3472 (binned output of the 64 MP sensor); rear ultrawide 1.65 mm, 3280×2464 (8 MP); rear macro 2.0 mm; front 3.49 mm, reports 2320×1744 (binned output of the 16 MP sensor) |
| Sensors | Accel/gyro Bosch BMI220, magnetometer AKM09918, light ROHM BU27030NUC, **Elliptic Labs ultrasound proximity** (virtual), Xiaomi pickup + large-area-touch sensors |
| IR blaster | yes (`android.hardware.consumerir`) |
| NFC | **no** (only `fleur`/`fleurp` enable the ST21NFC HAL) |
| Battery | 5000 mAh design, 4953 mAh full charge reported, **795 cycles** at collection time |
| Input devices | `uinput-fpc` (event4), `fts_ts` (event3), `mtk-kpd` (event2, power/volume), `ACCDET` (event0, headset jack), `swtp-input` (event1) |

## Fingerprint-sensor gestures (main finding)

The FPC HAL (`fpc_fingerprint_hal`) runs a navigation loop (`nav_loop2`) and logs every gesture it classifies:

| Gesture | HAL event | Reaches Android input? |
|---|---|---|
| Swipe **up** | `4` (`SLIDE_UP`, dx/dy) | **Yes**: kernel `KEY_UP` (103) on `uinput-fpc` → `KeyEvent keyCode=0 (UNKNOWN)`, delivered to apps |
| Swipe down | none | No. Sensor IRQs fire but the HAL never classifies it |
| Tap | `1` | No: "Unrecognized trigger_event", only visible in logcat |
| Hold | `2` | No: logcat only |
| Double tap | (two `1`s) | No. HyperOS has its own built-in shortcut: `settings system fingerprint_double_tap` |

- Stock `/system/usr/keylayout/uinput-fpc.kl` maps only 105 → `DPAD_CENTER` and 106 → `BUTTON_C`, which the HAL never emits. 103/108 are commented out.
- Plan: an `AccessibilityService` with `flagRequestFilterKeyEvents` catches scanCode 103 from `uinput-fpc` → `performGlobalAction(GLOBAL_ACTION_NOTIFICATIONS)`. No root needed.
- Full research, routes and next steps: [`research/fingerprint-gestures-handoff.md`](research/fingerprint-gestures-handoff.md). Raw evidence: [`logs/`](logs/).

## Repo layout

| Path | Contents |
|---|---|
| `adb/props/` | full `getprop` (redacted) |
| `adb/hardware/` | cpuinfo, cpufreq, meminfo, camera summary, device-tree model |
| `adb/kernel/` | kernel version, modules, platform drivers |
| `adb/input/` | `getevent -pl`, `dumpsys input` |
| `adb/keylayout/` | stock `.kl`/`.idc` for fingerprint and keypad |
| `adb/display/` | `dumpsys display`, panel modes, SurfaceFlinger head |
| `adb/power/` | battery, power_supply uevent |
| `adb/sensors/` | `dumpsys sensorservice` |
| `adb/storage/` | partitions by-name, df, mounts |
| `adb/packages/` | system packages, features, libraries |
| `adb/settings/` | `settings list system/secure/global` (redacted) |
| `adb/fingerprint/` | fp props, vendor fp libs, dev nodes |
| `adb/misc/` | `lshal` + service list, vendor VINTF manifest, vendor HAL binaries |
| `device-tree-snapshot/` | full copy of the LineageOS tree (see below) |
| `research/` | fingerprint-gesture research/handoff notes |
| `logs/` | getevent + logcat captures of fingerprint gestures |

## Device tree and upstream sources

Snapshot in `device-tree-snapshot/` is `mt6781-devs/android_device_xiaomi_fleur` @ `lineage-23.2`, commit `45289f6f6e94fc90870d27477d42a89c735fcff5` (2026-08-02), Apache-2.0 (The LineageOS Project). Notable files:
- `sku/build_{fleur,fleurp,miel,mielp}.prop`: per-SKU names (one tree serves all four)
- `configs/keylayout/uinput-fpc.kl`, `uinput-goodix.kl`: all mappings commented out
- `overlay/FrameworkOverlayFleur/res/values/config.xml`: `config_is_powerbutton_fps=true`, no fingerprint-gesture flag
- `BoardConfig.mk`: kernel `kernel/xiaomi/mt6781`, `fleur_defconfig`, boot header v2, super 9.1 GB, vendor/odm erofs
- `proprietary-files.txt`: FPC + Goodix fingerprint HALs, Goodix biometrics 2.1

mt6781-devs organisation repos (all branch `lineage-23.2`):
- Device tree: https://github.com/mt6781-devs/android_device_xiaomi_fleur
- Kernel: https://github.com/mt6781-devs/android_kernel_xiaomi_mt6781
- MediaTek hardware: https://github.com/mt6781-devs/android_hardware_mediatek
- MediaTek vendor sepolicy: https://github.com/mt6781-devs/android_device_mediatek_sepolicy_vndr
- Releases: https://github.com/mt6781-devs/releases

Other references:
- Android `FingerprintGestureController`: https://developer.android.com/reference/android/accessibilityservice/FingerprintGestureController
- Key Mapper fingerprint gestures: https://keymapper.app/user-guide/fingerprint-gestures/
- backtap (logcat approach): https://github.com/xarantolus/backtap
- GSMArena review: https://www.gsmarena.com/poco_m4_pro-review-2397p2.php

## Re-collecting

adb on Windows came from `winget install Google.PlatformTools`. Most dumps here are plain `adb shell getprop`, `dumpsys <service>`, `getevent -pl`, `pm list …`, and `settings list …`.
