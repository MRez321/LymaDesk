# LymaDesk — RustDesk Rebrand

A rebranded remote desktop client built on the [RustDesk](https://github.com/rustdesk/rustdesk) codebase,
targeting the custom server at `desk.lymagroups.ir`.

## Overview

| Item | RustDesk (default) | LymaDesk (custom) |
|---|---|---|
| Application name | `RustDesk` | `LymaDesk` |
| Default rendezvous server | `id.rustdesk.com` | `desk.lymagroups.ir` |
| Default relay server | `id.rustdesk.com` | `desk.lymagroups.ir` |
| Default API server | `api.rustdesk.com` | `http://desk.lymagroups.ir` |
| Sidebar logo (large) | (none) | `lyma.png` (from repo root, full-width, height 56) |
| Sidebar nav logo | RustDesk logo | **Removed** — only `lyma.png` remains |
| Sidebar website link | — | `lymagroups.ir` → `https://lumagroups.ir` |
| Sidebar phone | — | `025-33808586` |
| Settings > Network fields | pre-filled | hidden (defaults work, user overrides) |
| Windows exe file properties | RustDesk | LymaDesk |
| Windows app icon | 48×48 ICO (1961 bytes) | 16–256 multi-res ICO (83 KB) |
| Installer | — | `LymaDesk-1.5.0-setup.exe` (self-extracting) |
| Theme accent blue | `#0071FF` | `#FBB12F` |
| Sidebar "powered by" | shown | removed |
| Tab title | RustDesk | LymaDesk |

## How Customization Works

`custom.txt` is a plain-text `key=value` file placed beside the executable.
`load_custom_client()` reads it at startup and calls `read_custom_client()`,
which falls back to `read_plain_custom_client()` when the content is not
base64-encoded signed JSON (which the user's `custom.txt` is).

Flow:

1. `load_custom_client()` → `src/common.rs:2353` reads `custom.txt` from the
   exe directory (release) or `./custom.txt` (debug).
2. `read_custom_client(config)` → `src/common.rs:2451` calls `decode64()`; on
   failure it delegates to `read_plain_custom_client()`.
3. `read_plain_custom_client(config)` → `src/common.rs:2528` parses
   `key=value` lines and populates `config::DEFAULT_SETTINGS` + `HARD_SETTINGS`.
4. `Config::get_options()` → `hbb_common/src/config.rs:1220` returns
   `DEFAULT_SETTINGS + CONFIG2.options + OVERWRITE_SETTINGS`, so the custom
   defaults are used even when the Settings dialog shows empty fields.
5. `is_custom_client()` → `src/common.rs:2600` returns `true` when
   `APP_NAME != "RustDesk"`, enabling custom-client behavior throughout.

## Changed Files

### 1. `custom.txt` (repo root)

Plain-text custom client configuration consumed at runtime by
`load_custom_client()`.

```
app-name=LymaDesk
host=desk.lymagroups.ir
key=Togh4B1VnEd3j5ttxvVp4qMKSNJBf55feBtQu51oB3g=
relay=desk.lymagroups.ir
api=http://desk.lymagroups.ir
```

| Key | Maps to | Effect |
|---|---|---|
| `app-name` | `config::APP_NAME` | Window title, `is_custom_client()` |
| `host` | `custom-rendezvous-server` | Default connection server |
| `relay` | `relay-server` | Default relay |
| `api` | `api-server` | Default API server |
| `key` | `key` | Encryption key for server connection |

### 2. `src/common.rs`

**`read_custom_client()` — L2451**

Added a plain-text fallback. When `decode64()` fails (the config is not
base64-encoded signed JSON), the function now calls
`read_plain_custom_client()` instead of returning early:

```rust
pub fn read_custom_client(config: &str) {
    let Ok(data) = decode64(config) else {
        // Not base64 — try plain-text key=value format (build-time custom configs).
        read_plain_custom_client(config);
        return;
    };
    // ... existing signed-JSON path ...
}
```

**`read_plain_custom_client()` — L2528 (new)**

Parses `key=value` lines and populates `DEFAULT_SETTINGS` + `HARD_SETTINGS`:

- `host` → `custom-rendezvous-server`
- `relay` → `OPTION_RELAY_SERVER` (`relay-server`)
- `api` → `OPTION_API_SERVER` (`api-server`)
- `key` → `OPTION_KEY` (`key`)
- `app-name` → `config::APP_NAME`
- any other key → inserted with underscores normalized to hyphens

`HARD_SETTINGS` entries lock the value so it cannot be overridden by
user changes at runtime.


### 3. `flutter/lib/desktop/pages/desktop_home_page.dart`

**Sidebar branding block (`buildTip()` — L389)** (called from `buildLeftPane()` at L83)

- **Removed old logo**: `loadLogo()` (which displayed `assets/logo.png`, the small 300×60 nav logo) has been **removed** from the sidebar's `children` list. Previously both `loadLogo()` and the `lyma-logo.png` image appeared — now only `lyma-logo.png` remains.
- **LYMA LOGO** (L389–L400): Displays `assets/lyma-logo.png` (width: 200, height: 56, `BoxFit.contain`) centered at the top of the sidebar, shown when `!isOutgoingOnly`.
- **LYMA CONTACT INFO** (L401–L435): Below the logo, a centered `Column` (`CrossAxisAlignment.center`) with:
  - Website link: displayed text `lymagroups.ir`, tap opens `https://lumagroups.ir`, colored `#FBB12F`, `fontSize: 14`, `fontWeight: w500`
  - Phone number: `025-33808586`, `fontSize: 13`, `color: Colors.grey.shade600`
- **Separator line** (L437–L445): `Container` with `height: 1`, `color: Colors.grey.shade300`, `width: double.infinity`, `margin: EdgeInsets.only(top: 12, bottom: 12)`. Visually separates the branding block from the ID/password displays below.

**Removed**: `loadPowered(context)` — the "powered by RustDesk" text/link that previously appeared at the top of the sidebar has been removed from `buildLeftPane()`.

**Sidebar structure** (after changes):
```
Sidebar (buildLeftPane)
├── buildPresetPasswordWarning()     — password warning banner
├── buildTip(context)                — LymaDesk branding (logo + contact info + separator)
├── buildIDBoard(context)            — 9-digit ID display
├── buildPasswordBoard(context)      — 6-digit password display
└── buildHelpCards()                 — help/support cards
```
The old `loadLogo()` step is **removed** — only `lyma-logo.png` appears, centered and full-width.


### 4. `flutter/lib/mobile/widgets/dialog.dart`

**`showServerSettings()` — L56 / `showServerSettingsWithValue()` — L68**

Added `showValues` parameter (default `true`). When `false`, the text field
controllers are initialized with empty strings instead of the server config
values. In `submit()`, empty field values fall back to the `serverConfig`
defaults (which come from `custom.txt` via `DEFAULT_SETTINGS`).

- `showServerSettings()` calls `showServerSettingsWithValue(..., showValues: !isCustomClient)`
  — hides defaults and keeps fields empty for custom clients like LymaDesk.
- `showServerSettingsWithValue()` in `scan_page.dart:159` (QR code scanner)
  uses the default `showValues: true` — shows scanned QR values.

**`submit()` fallback logic — L94–109:**
```dart
idServer: idCtrl.text.trim().isNotEmpty
    ? idCtrl.text.trim()
    : serverConfig.idServer,
relayServer: relayCtrl.text.trim().isNotEmpty
    ? relayCtrl.text.trim()
    : serverConfig.relayServer,
apiServer: apiCtrl.text.trim().isNotEmpty
    ? apiCtrl.text.trim()
    : serverConfig.apiServer,
key: keyCtrl.text.trim().isNotEmpty
    ? keyCtrl.text.trim()
    : serverConfig.key
```

This means: the Settings > Network fields appear empty, but when the user
clicks "Connect" without typing anything, the defaults from `custom.txt`
are used automatically. If the user enters a value, it overrides the default.

### 5. `flutter/windows/runner/Runner.rc`

**File-properties block — L88–L100**

Updated Windows file properties from "RustDesk" to "LymaDesk":

| Property | Value |
|---|---|
| `CompanyName` | `LymaDesk` |
| `FileDescription` | `LymaDesk Remote Desktop` |
| `InternalName` | `lymadeSk` |
| `LegalCopyright` | `Copyright (C) 2026 LymaDesk. All rights reserved.` |
| `OriginalFilename` | `lymadeSk.exe` |
| `ProductName` | `LymaDesk` |

### 6. `flutter/windows/runner/resources/app_icon.ico`

The Windows application icon. Replaced with a proper multi-resolution ICO
containing all standard Windows icon sizes (16, 24, 32, 48, 64, 128, 256).
The CI also regenerates this from `res/icon.png` via
`flutter pub run flutter_launcher_icons`.

| | `app_icon.ico` | `res/icon.png` |
|---|---|---|
| Before | 48×48, 1961 bytes | 42134 bytes |
| After | 16–256 multi-res, 83055 bytes | 42134 bytes |

### 7. `flutter/lib/common.dart` (theme colors)

**Theme accent blue**: `MyTheme.accent` changed from `#0071FF` (RustDesk blue)
to `#FBB12F` (LymaDesk orange). Also updated `accent50`, `accent80`, `button`,
`_msgboxColor()` default, theme `primary` colors (both light and dark), link
colors, and the `colorMap` "blue" entry to use the new accent.

### 8. `flutter/lib/desktop/pages/desktop_home_page.dart` (sidebar branding)

- **Removed**: `loadPowered(context)` ("powered by RustDesk" link) from the
  sidebar top. The navbar logo now uses `loadLogo()` directly.
- **Sidebar logo**: `assets/lyma-logo.png` is centered (`Alignment.center`),
  height 56, sourced from `lyma.png` in the repo root.
- **Contact info**: Website link and phone number are centered
  (`CrossAxisAlignment.center`). Website link uses `#FBB12F` color with
  `fontWeight: w500` and `fontSize: 14`. Phone uses `grey.shade600`.
- **Separator**: A grey divider line (`height: 1`, `grey.shade300`) with
  `margin: 12` top/bottom separates the branding block from the rest of the
  sidebar.

### 9. `flutter/lib/desktop/widgets/tabbar_widget.dart`

- Tab title text changed from `"RustDesk"` to `"LymaDesk"`.

### 10. `flutter/lib/desktop/pages/desktop_setting_page.dart` and
    `flutter/lib/mobile/pages/settings_page.dart`

- `"About RustDesk"` → `"About LymaDesk"` (both desktop and mobile).
- `"Keep RustDesk background service"` → `"Keep LymaDesk background service"`.
- Mobile "About" dialog: website URL from `https://rustdesk.com/` to
  `https://lumagroups.ir/`, display text from `rustdesk.com` to `lymagroups.ir`.

### 11. `flutter/lib/desktop/pages/server_page.dart`

- Server page header gradient: `#00bfe1` / `#0071ff` → `#FBB12F` / `#d99100`.

### 12. `.github/workflows/lyma-build.yml` (CI workflow)

GitHub Actions workflow that builds LymaDesk for Windows x64 and produces a
self-extracting installer.

**Jobs:**

1. `generate-bridge` — calls `bridge.yml` (reusable workflow) to generate the
   `flutter_rust_bridge` Dart/Rust bindings needed by `flutter build windows`.

2. `build-windows` — runs on `windows-2022` (17 steps):
   - Restores bridge files from `bridge-artifact`
   - Installs LLVM 15.0.6 (for Clang-based FFI builds)
   - Installs Flutter 3.24.5 (stable, x64)
   - Replaces the Flutter engine with the rustdesk custom build
   - Applies the Flutter 3.24.4 dropdown-menu patch
   - Installs Rust 1.75 toolchain
   - Sets up vcpkg (pinned commit `9e593bb…`, `x64-windows-static`)
   - Runs `flutter pub run flutter_launcher_icons` to regenerate
     `app_icon.ico` from `res/icon.png`
   - Creates `flutter/assets/lyma-logo.png` from `lyma.png` (sidebar logo)
   - `flutter/assets/icon.svg` is tracked in-repo and pre-replaced with LymaDesk logo (fallback for `loadIcon(16)`)
   - `logo*.png` nav logo files are still generated but no longer displayed in sidebar (old logo removed)
   - Runs `python3 ./build.py --portable --flutter --skip-portable-pack --hwcodec --vram`
   - Copies `custom.txt` into the `Release/` output directory
   - Renames the output directory to `LymaDesk/`
   - Builds the portable installer via `libs/portable/generate.py`
   - Renames the installer to `LymaDesk-1.5.0-setup.exe`
   - Uploads `LymaDesk-1.5.0-setup.exe` as artifact `lyma-desktop-windows-x64`

**Trigger:** `workflow_dispatch`, `push` to `main`, `pull_request` to `main`.

**Download:** Artifacts are available in the GitHub Actions "Artifacts"
section for each workflow run. The artifact is a single
`LymaDesk-1.5.0-setup.exe` self-extracting installer — no duplicate files.


### 13. `dev-guide.md` (new)
Development guide for hot-reload sidebar editing. Covers prerequisites,
quick-start commands, file locations, and step-by-step instructions for
changing the sidebar logo, contact info, colors, and styling.

## CI Environment Variables

| Variable | Value | Source |
|---|---|---|
| `SCITER_RUST_VERSION` | `1.75` | Matches `RUST_VERSION` discussion #7503 |
| `FLUTTER_VERSION` | `3.24.5` | Stable Flutter used by current RustDesk |
| `LLVM_VERSION` | `15.0.6` | LLVM with Clang for FFI |
| `VCPKG_COMMIT_ID` | `9e593bb…` | Pinned vcpkg version (2026.07.29) |
| `VCPKG_BINARY_SOURCES` | `clear;x-gha,readwrite` | GitHub Actions binary cache |
| `VERSION` | `1.5.0` | App version from `Cargo.toml` |


## Building Locally (Windows PowerShell)

### Prerequisites (Windows)

- Rust 1.75 (`rustup`, target `x86_64-pc-windows-msvc`)
- Flutter 3.24.5 (stable)
- LLVM 15.0.6+ with Clang
- vcpkg (pinned commit `9e593bb…`, triplet `x64-windows-static`)
- Python 3 (run as `python`, not `python3`)
- Visual Studio Build Tools (Desktop development with C++)

### Steps

```powershell
# 1. Generate bridge files (needs Rust 1.75 + Flutter)
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked
cd flutter
flutter pub get
flutter_rust_bridge_codegen `
  --rust-input ../src/flutter_ffi.rs `
  --dart-output ./lib/generated_bridge.dart
cd ..

# 2. Regenerate Windows icon from custom res\icon.png
cd flutter
flutter pub run flutter_launcher_icons
cd ..

# 3. Create logo assets (gitignored: *png rule in .gitignore)
cp lyma.png flutter\assets\lyma-logo.png
cp res\icon.png flutter\assets\logo.png
cp res\icon.png flutter\assets\logo_light.png
cp res\icon.png flutter\assets\logo_dark.png

# 4. Build
python .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram

# 5. Copy custom.txt into the Release output
cp custom.txt .\flutter\build\windows\x64\runner\Release\

# 6. Rename output directory
Rename-Item .\flutter\build\windows\x64\runner\Release\ -NewName LymaDesk

# 7. Build the installer
(Get-Content res\manifest.xml) -replace 'dpiAware.*', '' | Set-Content res\manifest.xml
cd libs\portable
pip install -r requirements.txt
python .\generate.py -f ..\..\LymaDesk\ -o . -e ..\..\LymaDesk\rustdesk.exe
cd ..\..
Move-Item .\target\release\rustdesk-portable-packer.exe .\LymaDesk-1.5.0-setup.exe
```

## Architecture: Default Server Configuration Flow

```
custom.txt (plain key=value)
  │
  ▼
load_custom_client()  [src/common.rs:2353]
  │  reads custom.txt from exe dir (release) or ./ (debug)
  │
  ▼
read_custom_client(config)  [src/common.rs:2451]
  │  decode64(config) fails → fallback
  │
  ▼
read_plain_custom_client(config)  [src/common.rs:2528]
  │  key=value → DEFAULT_SETTINGS + HARD_SETTINGS
  │
  ▼
Config::get_options()  [hbb_common/src/config.rs:1220]
  │  DEFAULT_SETTINGS + CONFIG2.options + OVERWRITE_SETTINGS
  │
  ▼
Flutter: mainGetOptions() → ServerConfig.fromOptions()
  │  reads custom-rendezvous-server, relay-server, api-server, key
  │
  ▼
Settings > Network dialog
  │  showValues: !isCustomClient → fields empty for LymaDesk, defaults hidden
  │  submit() falls back to serverConfig values when fields empty
  │
  ▼
Connect uses desk.lymagroups.ir (or user-entered override)
```

## Logo & Icon Reference

| UI element | File | Source |
|---|---|---|
| Sidebar logo (centered, full-width) | `flutter/assets/lyma-logo.png` | `lyma.png` (tracked, gitignored under `*png` with exception) |
| Top nav window icon (fallback) | `flutter/assets/icon.svg` | Replaced in-repo with LymaDesk logo (base64 from `res/icon.png`) |
| App icon (taskbar, Alt+Tab) | `flutter/windows/runner/resources/app_icon.ico` | `res/icon.png` (regenerated by `flutter_launcher_icons`) |

| Nav logo files (gitignored, no longer used in sidebar) | `flutter/assets/logo*.png` | `res/icon.png` (generated in CI, unused since old logo removed) |

To change any logo or icon:
1. Edit `lyma.png` (sidebar logo) or `res/icon.png` (app/nav icons)
2. CI will regenerate all icons automatically from these sources
3. For local testing: run `flutter pub run flutter_launcher_icons` and
   copy `lyma.png` to `flutter/assets/lyma-logo.png`, and `res/icon.png`
   to `flutter/assets/logo.png`, `logo_light.png`, and `logo_dark.png`
4. For the top-nav icon fallback (`icon.svg`), replace `flutter/assets/icon.svg`
   with a LymaDesk SVG (or embed `res/icon.png` as base64 in SVG)

## Coexistence with RustDesk

LymaDesk can be installed alongside RustDesk without conflicts because all
Windows instance-detection and IPC identifiers are namespaced by `APP_NAME`
("LymaDesk"), which `custom.txt` sets at startup:

| Mechanism | RustDesk identifier | LymaDesk identifier |
|---|---|---|
| Tray single-instance mutex | `Local\\RustDesk_tray` | `Local\\LymaDesk_tray` |
| IPC named pipe (Windows) | `\\.\pipe\RustDesk\query` | `\\.\pipe\LymaDesk\query` |
| IPC socket (Linux/macOS) | `/tmp/RustDesk/ipc…` | `/tmp/LymaDesk/ipc…` |
| Windows service name | `RustDesk` | `LymaDesk` |
| Install path (portable) | `.../RustDesk/RustDesk.exe` | `.../LymaDesk/LymaDesk.exe` |
| Virtual printer name | `RustDesk (Remote Printer)` | `LymaDesk (Remote Printer)` |
| Registry subkey (settings) | `…\RustDesk` | `…\LymaDesk` |

All of these use `config::APP_NAME` or `crate::get_app_name()`, both of which
resolve to the value from `custom.txt`. No additional changes are needed for
side-by-side installation.

## Known Limits

1. **Exe filename** remains `rustdesk.exe`. The `CMakeLists.txt`
   `BINARY_NAME "rustdesk"` is unchanged because `build.py`'s portable packer
   expects that name. The display name is "LymaDesk" via `APP_NAME` config
   + `Runner.rc`. `Runner.rc` uses `lymadeSk` as the internal/original name.

2. **`res/icon.png` is a custom PNG** (42,134 bytes, 500×500). The generated
  `app_icon.ico` contains all standard Windows sizes (16–256). `res/icon.ico`,
  `res/tray-icon.ico`, `res/logo.svg`, and various PNG sizes are also
  user-provided replacements for RustDesk defaults.

3. **Generated asset PNGs are git-ignored** (`lyma-logo.png`, `logo*.png` in
  `flutter/assets/`). They are created in CI from `lyma.png` (sidebar logo)
  and `res/icon.png` (nav logos). The source `lyma.png` itself is tracked via
  a `.gitignore` exception (`!lyma.png`).

4. **usbmmidd / printer drivers** are not included in this build.

5. **Bridge files** (`generated_bridge.dart`,
   `generated_bridge.freezed.dart`) are generated by `bridge.yml` and must
   be present before `flutter build windows`.

6. **CI only builds Windows x64.** macOS and Linux builds are not included.

7. **The installer (`LymaDesk-1.5.0-setup.exe`) is a self-extracting portable
   packer**, not an MSI installer. It extracts files to a user-chosen directory
   and runs the app — no system registry modifications or uninstall entry.

## Related Files

| File | Purpose |
|---|---|
| `dev-guide.md` | Development guide with hot-reload setup and sidebar editing instructions |
| `lyma.md` | This file — comprehensive change documentation |
| `custom.txt` | Default server configuration (host, relay, api, key, app-name) |
| `.github/workflows/lyma-build.yml` | CI workflow (Windows x64 installer build) |
