# MiFitnessAdAway

English | [中文](README_zh.md)

An LSPosed module that removes ads and promotion popups from **Xiaomi Mi Fitness / 小米运动健康** (`com.mi.health`, and the international build `com.xiaomi.wearable`), built on the modern **libxposed API 102**.

> **v1.3.0, verified on device** — Redmi K70 Ultra / Android 16 / Mi Fitness 3.59.1 (and OnePlus PLQ110 for earlier releases): every page below stays clean, all normal features work, trial watchfaces export for third-party import, and band ↔ phone DND sync works again.

## What it removes

Grouped the same way as the settings cards, so you can find a switch by the screen it belongs to:

| Where | What is removed |
|---|---|
| Splash | Splash ads (image / video) |
| Popups | "Update available" dialog · full-screen VIP promo popup |
| Home (health) | Health-tab promotion banner cards |
| Device | Promotion cards · red dot on the bottom "Device" tab and on the home "System settings" entry |
| Sport | Carousel cards · the whole operation area below "training index" (+ page scrolling disabled) |
| Mine | VIP membership card · doctor consultation card |
| Health detail | PingAn-Health consultation cards on Sleep / Heart rate / SpO₂ / Stress · "AntBoy AI" interpretation card · sleep research & 21-day improvement cards |
| Weight | Personalized weight-loss plan card ("个性化减重方案") |
| Everywhere | Xiaomi verification-SDK hook detection (`SensorHelper`) |

## Settings UI

One in-app screen, 17 toggles, no external config needed.

| Card | Switches |
|---|---|
| Master | ad-removal master — own card, shown together with the "enabled x/11" summary |
| Splash & popups | splash ads · app-update dialog · VIP promo popup |
| Mine | VIP membership card · doctor consultation card |
| Sport | carousel cards · operation cards below "training index" |
| Device | red dots (bottom tab + system settings entry) · DND sync (phone ↔ band) |
| Health detail | consultation cards (Sleep / Heart rate / SpO₂ / Stress) · sleep research/improvement cards · weight plan card |
| Watchface | trial watchface auto-export (experimental, off by default) |
| Other | anti-hook detection · debug log · **hide launcher icon** (applies instantly) |

- Tap a group title to collapse/expand it — the state is remembered.
- Switch changes take effect after restarting Mi Fitness; no reboot.
- While the master switch is off, every switch that depends on it is dimmed and not tappable. Debug log, hide-icon and DND sync stay usable (they are not gated by the master).
- Follows the system dark/light theme.

## Extras

- **Trial watchface auto-export (experimental)** — as soon as a trial download finishes, a cache sentinel grabs the package: a background thread takes a cheap look at `WatchFace/<device>/<faceID>/` every 400ms (stat only, no reads), and reads a newly appeared package on the spot once it has been stable for 300ms and its content MD5 matches the file name — the cache names packages after their content MD5, so a half-downloaded file is never read (a trial package only sits in that folder briefly — measured on device: a package cached under its content MD5 was deleted by the app within a minute, and the push entry points `doInstall`/`preInstall` are never called at all on Xiaomi Smart Band 8 Pro, so hooks or a once-per-page scan both miss it). When the file name doesn't match, the package is located by the face ID embedded in its header (`resource.bin` on older builds; newer builds cache it under its content MD5 — 32 hex chars, no extension). The package is then re-ID'd (`12→19` prefix, same length) and written under its Chinese name (taken from `description.xml` in the cache folder, or from the name field in the package header when the newer cache layout has no such file), ready for third-party import (upstream verified with AstroBox on Xiaomi Smart Band 10 Pro). The destination is configurable in settings → Watchface → "Export location": a relative path such as `Download/表盘导出` or `Documents/2026/面孔` (first level limited to `Download` / `Documents`, which is all MediaStore allows third-party apps; anything else falls back to `Download`). Changing the location offers to re-export existing faces there, and deduplication follows the location so the new folder really receives files. Exported IDs are filtered out of the server-side cleanup list so sideloaded faces survive sync, and the exported cache is cleaned up on the next scan (snapshot-based, with handoff/push guards). Every scan reports via Toast/notification, including the real destination.
- **Band ↔ phone DND sync (Android 15+)** — the official sync is dead on Android 15 and later because `ZenUtils.isSupportZenMode()` returns `false` unconditionally there and gates the whole chain. With this toggle the gate is lifted, the `zen_mode` content observer is re-registered after app start, the sync flag is held on, and the band's rules are pulled once at startup. Needs DND (notification-policy) access for Mi Fitness, which the module can only remind you about — the system enforces it.
- **Hide launcher icon** — instantly hides the module's own icon; its settings page stays reachable from LSPosed.
- **Debug log** — verbose hook logging for troubleshooting.

## How it works

- **Data-layer interception first**: banner APIs, the splash cache, membership data, doctor data and PingAn-Health banner binders are emptied or skipped, which kills the ad before it is ever rendered.
- **View-layer fallback**: several pages are React Native (YRN). Cards there are removed by scanning the view tree on the shared RN host (`YRNCFragment`) and taking out the whole card, located by its title text — these pages have server-driven copy and no stable data hooks.
- **Reliability guards**: the "Mine" page collapses cards layer by layer and shifts following content up; the sport page removes everything below the "training index" anchor; hidden rows are tracked so repeated scans never double-shift.
- **Device red dots**: `PowerManager.isIgnoringBatteryOptimizations` is faked to `true` (equivalent to "battery optimization ignored"), plus the face-entrance red-dot getters return `false` — that clears both the bottom-tab dot and the home "System settings" entry dot.
- **Popups**: `AppUpgradeUtil.showUpdateDialogIfNeed` is skipped for the update prompt; `MembershipDialogManager.showMembershipExpiredFaceDialog` is skipped for the VIP promo popup, while still invoking the caller's dismiss callback so the birthday-medal flow it continues is left intact. A user-initiated purchase dialog is untouched.
- **Version tolerance**: every hook installs independently and fails in isolation, so entry points that differ on a given app version are skipped gracefully while the rest keep working.
- **Late-initialized classes**: classes whose static initializer needs the `Application` context (`DeviceSettingsPreference`, `ZenUtils`, `FitnessApp`) are loaded with `initialize = false`, because forcing the initializer during hook installation throws and would leave the hook uninstalled.

## Requirements

- Rooted device (KernelSU or Magisk) + LSPosed ≥ 2.1.1 (Zygisk)
- Xiaomi Mi Fitness `com.mi.health` / `com.xiaomi.wearable` 3.0+

## Install

1. Install `MiFitnessAdAway-*.apk` from [Releases](../../releases).
2. Enable the module in LSPosed — the static scope already contains both package names.
3. Reboot once, then open the module icon (or LSPosed → module settings) to adjust toggles.

## Build

Requires Gradle 9.5.1, AGP 9.2.1, JDK 17, compileSdk 37.

```powershell
gradle assembleRelease   # output: app/build/outputs/apk/release/app-release.apk
```

With `keystore/mifitnessadaway.keystore` and `keystore/signing.properties` present locally (both git-ignored) the release is signed with the real key; otherwise it falls back to the debug key. GitHub Actions builds every push and pull request automatically, but that artifact is debug-signed and only proves the code compiles.

## Repository layout

```
app/src/main/java/io/github/hao1196561270/mifitnessadaway/
├── AdAwayModule.java     # libxposed entry — all hooks
├── SettingsActivity.java # settings UI (grouped cards, dark/light adaptive)
├── MiFitnessApp.java     # XposedService bridge (RemotePreferences)
└── Prefs.java            # preference keys
app/src/main/resources/META-INF/xposed/  # module declarations (module.prop / java_init.list / scope.list)
.github/workflows/android.yml            # CI: build on push / pull request
```

## License

Apache License 2.0
