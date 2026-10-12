# Handoff: fingerprint-sensor gestures app for Poco M4 Pro 4G

Written 2026-10-12.

## 1. What the user wants

An Android app that turns gestures on the phone's **side-mounted fingerprint sensor** (in the power button) into actions.
The headline example: **swipe on the sensor → open the notification shade.** Other actions (quick settings, torch, screenshot, launch app) are nice-to-have.
The user first wanted to know whether Poco/Xiaomi provides an API for this, or any other way to do it.

## 2. Setup

Device testing is done over USB adb from a Windows laptop.

## 3. Phone facts

| Item | Value | Confidence |
|---|---|---|
| Model | Poco M4 Pro **4G** (not the 5G) | from user |
| Codename / SoC | `fleur`, MediaTek MT6781 (Helio G96) | from public device trees |
| Sensor | capacitive, inside the power button, right side | confirmed (GSMArena review) |
| Sensor vendor | either **FPC** or **Goodix**; the phone ships with one of the two | device tree has HALs for both; user's unit unknown |
| OS version | MIUI 13/14 or HyperOS, Android 11–13+ | **unknown, ask the user** (Settings → About phone) |

## 4. Research findings

### 4.1 No Poco/Xiaomi API
There is no public Xiaomi/Poco SDK for fingerprint gestures.

### 4.2 Built-in shortcut (no app needed, double-tap only)
MIUI 14 / HyperOS phones with a side sensor have **Settings → Additional settings → Gesture shortcuts → "Double-tap the fingerprint sensor"**.
Fixed action list: screenshot, torch, Control Centre, **notification shade**, Google Assistant, camera, calculator, silent mode, bug-report logs.
Sources say the Poco M4 Pro has it. **Not yet confirmed on the user's phone.** No swipes, no custom actions.

### 4.3 Android's official API: `FingerprintGestureController` (API 26+)
- Delivered to an `AccessibilityService`. The service config needs `android:canRequestFingerprintGestures="true"` and the `flagRequestFingerprintGestures` flag. Callbacks give `FINGERPRINT_GESTURE_SWIPE_UP/DOWN/LEFT/RIGHT`.
- Works **only if the OEM enabled it**. Check at runtime with `fingerprintGestureController.isGestureDetectionAvailable()`. In AOSP this depends on the framework config `config_fingerprintSupportsGestures` (from memory, verify).
- The Key Mapper app (`io.github.sds100.keymapper`) uses this API. An XDA user got swipe-down → notification shade working on a **Redmi Note 7** (rear sensor, MIUI) with Key Mapper. So some Xiaomi phones do enable it.
- Key Mapper's docs list three device classes: (1) gestures work for third-party apps, (2) the phone detects gestures but blocks third-party apps, (3) no gesture detection.

### 4.4 Evidence from this phone's device tree (most important clue)
Repo: `mt6781-devs/android_device_xiaomi_fleur`, branch `lineage-23.2` (LineageOS tree, files usually copied from stock vendor):

- `configs/keylayout/uinput-fpc.kl`: **every mapping is commented out**, but the comments name the kernel codes:
  ```
  # Swipe Up      #key 103   DPAD_UP VIRTUAL
  # Swipe Down    #key 108   DPAD_DOWN VIRTUAL
  # Single Tap    #key 105   DPAD_CENTER VIRTUAL
  # Double Tap    #key 106   BUTTON_C VIRTUAL
  # also commented: key 304 DPAD_CENTER, key 102 HOME, 108/103 SYSTEM_NAVIGATION_DOWN
  ```
- `configs/keylayout/uinput-goodix.kl`: also all commented: `353` single click, `306` double click, `102`, `105`, `106`.
- `overlay/FrameworkOverlayFleur/res/values/config.xml`: `config_is_powerbutton_fps=true`. **No fingerprint-gesture flag is set.**
- `proprietary-files.txt`: `fingerprint.fpc.default.so`, `fingerprint.goodix.default.so`, `libgf_ca.so`, `libgf_hal.so`, `libgoodixhwfingerprint.so`, Goodix biometrics HAL 2.1. `init` touches `/dev/goodix_fp`.

**Inference (unverified):** the fingerprint HAL probably creates a virtual input device (`uinput-fpc` or `uinput-goodix`) and can send swipe and tap key events, but Android is configured to ignore them, so the official API in 4.3 will likely report "not available". Stock MIUI's own `.kl` files have not been checked yet.

### 4.5 Unverified hypothesis worth testing (no-root path)
In AOSP, when a device has a key layout but a scancode is not mapped, the input system still sends the key with `keyCode = KEYCODE_UNKNOWN` and the raw `scanCode` set (from memory of `KeyboardInputMapper`, verify). If so, an `AccessibilityService` with `FLAG_REQUEST_FILTER_KEY_EVENTS` could see `onKeyEvent` with scanCode 103/108/105/106 from the fingerprint device. That would be a no-root, no-Shizuku solution. This works only if the HAL actually sends events while the phone is unlocked.

### 4.6 Other prior art
- `xarantolus/backtap`: a Magisk (root) module that detects fingerprint taps by **watching logcat**. Taps, double taps and holds only, no swipes. Tested only on a Mi Mix 2 running LineageOS.

## 5. Routes, ranked

| Route | Root? | Detects | Odds on this phone | Notes |
|---|---|---|---|---|
| A. Built-in double-tap | No | double tap | high | Zero code. Check it first. |
| B. Official `FingerprintGestureController` | No | swipes | low (see 4.4) | Simplest app if it works |
| B2. Accessibility `onKeyEvent` raw scanCodes (4.5) | No | whatever the HAL sends | unknown | Cheap to test with a probe app |
| C. Shizuku reads `/dev/input/eventN` | No | whatever the HAL sends | good if events exist | Shizuku runs as the shell user, which can read input devices. On Android 11+ Shizuku can be restarted on the phone itself via Wireless debugging after each reboot, no PC needed. |
| D. Root (Magisk): override `/vendor/usr/keylayout/uinput-*.kl` to map 103/108/105/106 to real keycodes, or read evdev directly | Yes | swipes + taps | highest | Bootloader unlock wipes the phone. Xiaomi unlock has a waiting period. |
| E. Watch logcat for finger down/up | Needs `READ_LOGS` via `adb shell pm grant` | taps only | fragile | Android 13+ shows a log-access consent prompt |

Every route ends the same way: the app's `AccessibilityService` calls `performGlobalAction(GLOBAL_ACTION_NOTIFICATIONS)` (or `GLOBAL_ACTION_QUICK_SETTINGS`, `GLOBAL_ACTION_TAKE_SCREENSHOT`, and so on). Torch uses `CameraManager.setTorchMode`.

## 6. Where we are now

- Research: **done** (above).
- Device testing: done, see 7b.
- No code written yet.

## 7. Next steps

### Step 1: quick checks on the phone (user, about 2 min)
1. Settings → About phone: note the **MIUI/HyperOS version and Android version**.
2. Check the built-in double-tap shortcut exists (4.2). Set it to "Notification shade" and see whether it works.
3. Install **Key Mapper** from the Play Store, try to add a *fingerprint gesture* map, and note whether it says unsupported.

### Step 2: connect adb
- Phone: About phone → tap MIUI/OS version 7× → Additional settings → Developer options → **USB debugging** ON. Plug in a data cable and accept the "Allow USB debugging?" prompt.
- Or Wi-Fi: Developer options → **Wireless debugging** → Pair device with pairing code, then `adb pair <ip>:<pair-port>` and `adb connect <ip>:<port>`.
- MIUI gotcha: installing APKs over adb needs Developer options → **Install via USB** ON, which may require being signed into a Mi account (and sometimes a SIM).

### Step 3: diagnostics (Claude runs these; the user touches the sensor when asked)
```
adb shell getprop ro.product.device            # expect fleur
adb shell getprop ro.build.version.release
adb shell getprop ro.miui.ui.version.name
adb shell getprop ro.mi.os.version.name        # set on HyperOS

adb shell getevent -pl                          # list input devices; find uinput-fpc / uinput-goodix / gf_* / fpc*
adb shell dumpsys input | grep -i -B2 -A15 -E 'uinput|fpc|goodix'   # which .kl file each device uses
adb shell ls /vendor/usr/keylayout/ | grep -i -E 'fpc|goodix|gf'
adb shell cat /vendor/usr/keylayout/uinput-fpc.kl      # stock mapping (also try uinput-goodix.kl)

# Record ONLY the fingerprint device (replace N) for ~20 s while the user taps, double-taps, swipes up, swipes down:
adb shell timeout 20 getevent -lt /dev/input/eventN
```
Run the recording in **three conditions**: (a) unlocked, MIUI double-tap shortcut OFF; (b) unlocked, double-tap shortcut ON (enabling it may keep the sensor awake after unlock); (c) screen on but locked.
For route E, also capture `adb logcat -v time | grep -i -E 'finger|fpc|goodix|gf_'` while touching the sensor.

### Step 4: decide
- Key Mapper gestures work, or `isGestureDetectionAvailable()` is true → **Route B**.
- `getevent` shows events while unlocked → build the **probe app** (below) to test **B2**. If B2 fails → **Route C (Shizuku)**.
- No `getevent` events, but logcat shows finger down/up → **Route E** (taps only), or use the built-in double-tap.
- Nothing at all → built-in double-tap (A), or root (D) if the user accepts wiping the phone.

### Step 5: probe app first (small, answers B and B2 in one install)
A Kotlin app with one `AccessibilityService` configured with `canRequestFingerprintGestures="true"`, `flagRequestFingerprintGestures` and `flagRequestFilterKeyEvents`, which:
- shows `isGestureDetectionAvailable()` on screen and logs every `FingerprintGestureCallback`;
- logs every `onKeyEvent` (keyCode, **scanCode**, deviceId, `InputDevice.getDevice(id).name`) and returns `false` so input isn't swallowed.
Install with `adb install` (apps installed via adb avoid Android 13+ "restricted settings"; a sideloaded APK file needs App info → ⋮ → *Allow restricted settings* before Accessibility can be enabled).

### Step 6: the real app (after the route is known)
- Kotlin, minSdk 26. A gesture-source interface with one implementation per route (FingerprintGestureSource / KeyEventSource / ShizukuEvdevSource / LogcatSource).
- A mapping screen: gesture (swipe up, swipe down, tap, double tap) → action.
- Actions via `AccessibilityService.performGlobalAction`, plus torch and launching apps.
- MIUI survival: tell the user to enable **Autostart**, set Battery saver to **No restrictions**, and lock the app in Recents, or MIUI may kill the accessibility service.

## 7b. Device findings (2026-10-12, adb over USB)
- Device: `miel` / model 2201117PI (Poco M4 Pro 4G India), Android 13 (SDK 33), HyperOS OS1.0, build V816.0.10.0.TKEINXM.
- Sensor: **FPC** (`vendor.sys.fp=fpc`, module Truly). Input device `uinput-fpc` = `/dev/input/event4`, uses `/system/usr/keylayout/uinput-fpc.kl` (no vendor copy).
- Stock `.kl` maps only `105 → DPAD_CENTER` (single tap) and `106 → BUTTON_C` (double tap); 103/108 commented out.
- Built-in double-tap shortcut exists: `settings system fingerprint_double_tap=launch_control_center`.
- HAL (`fpc_fingerprint_hal`, logcat) classifies touches in `nav_loop2` → `Got navigation event: N, dx, dy`:
  - `1` = tap → "Unrecognized trigger_event", **not sent** to input.
  - `2` = hold → not sent.
  - `4` = slide up (dy > 0) → **sent as EV_KEY 103** (KEY_UP) on event4, down+up. Seen in `getevent`.
  - No double-tap event (two event-1s instead). Key 105/106 never emitted.
  - **Swipe down: never classified** (sensor IRQs fire, but no navigation event). Not supported.
  - Swipes are flaky: ~2 of 6 attempts registered as slides, rest became taps.
- **Hypothesis 4.5 confirmed at framework level:** each slide-up reaches the framework as `KeyEvent keyCode:0 (UNKNOWN)`, deviceId 0 (`uinput-fpc`), policy passes it through and it is published to the foreground app (`MiuiInputKeyEventLog` / `MIUIInput [KeyEvent] publisher`). An AccessibilityService with `flagRequestFilterKeyEvents` should see it (scanCode 103). Swipes with fast full-length motion register reliably (7/7 on 2nd try).
- So: swipe up is the only gesture reaching the input system (unmapped scancode 103). Taps/holds only visible via logcat (route E).

## 8. Open questions
1. MIUI/HyperOS + Android version on the user's phone.
2. FPC or Goodix sensor on this unit (shown by `getevent -pl` device name).
3. Does the sensor send any events while the phone is unlocked? Can it physically register a swipe along the short power-button sensor?
4. Is the user willing to root (route D)? Assume **no** unless they say so.

## 9. Sources
- Android API: https://developer.android.com/reference/android/accessibilityservice/FingerprintGestureController
- Key Mapper fingerprint gestures: https://keymapper.app/user-guide/fingerprint-gestures/
- Key Mapper implementation issue: https://github.com/sds100/KeyMapper/issues/378
- XDA, Redmi Note 7 + Key Mapper: https://xdaforums.com/t/fix-fingerprint-gestures-on-redmi-note-7.4050139/
- Device tree (fleur): https://github.com/mt6781-devs/android_device_xiaomi_fleur (branch lineage-23.2)
- MIUI double-tap shortcut: https://www.gizchina.com/how-to/xiaomi-phone-fingerprint-sensor-shortcuts
- XiaomiTime on side-sensor double-tap: https://xiaomitime.com/unleash-secret-functions-of-your-xiaomis-side-fingerprint-sensor-24134/
- backtap (logcat approach): https://github.com/xarantolus/backtap
- GSMArena Poco M4 Pro review (sensor placement): https://www.gsmarena.com/poco_m4_pro-review-2397p2.php
