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
| Sidebar logo | (none) | `assets/lyma-logo.png` |
| Sidebar contact | — | website + phone |
| Windows exe file properties | RustDesk | LymaDesk |

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
   defaults appear in the Flutter connection dialog.
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

**`buildTip()` — L389**

Added sidebar branding block inside `buildTip()`:

- **LYMA LOGO** (L398–L408): Displays `assets/lyma-logo.png` (height 40,
  `BoxFit.contain`) on the left side of the sidebar, shown when
  `!isOutgoingOnly`.
- **LYMA CONTACT INFO** (L410–L443): Below the logo, a `Column` with:
  - Website link `lymagroups.ir` (opens via `url_launcher` `launchUrl`)
  - Phone number `+98 912 000 0000` (placeholder — see *Known Limits*)

The Dart imports `url_launcher` is already present via `pubspec.yaml`
(`url_launcher: ^6.3.1`).

### 4. `flutter/windows/runner/Runner.rc`

**`STRINGTABLE` / file-properties block — L88–L100**

Updated Windows file properties from "RustDesk" to "LymaDesk" so Explorer
shows the custom product name and description:

| Property | Value |
|---|---|
| `CompanyName` | `LymaDesk` |
| `FileDescription` | `LymaDesk Remote Desktop` |
| `InternalName` | `lymadeSk` |
| `LegalCopyright` | `Copyright (C) 2026 LymaDesk. All rights reserved.` |
| `OriginalFilename` | `lymadeSk.exe` |
| `ProductName` | `LymaDesk` |

The exe file itself remains `rustdesk.exe` (see *Known Limits*).

### 5. `flutter/windows/runner/resources/app_icon.ico`

The Windows application icon. Regenerated from `res/icon.png` during CI via
`flutter pub run flutter_launcher_icons`. The committed copy is from the
user's custom `res/icon.png`.

### 6. `.github/workflows/lyma-build.yml` (new)

GitHub Actions workflow that builds LymaDesk for Windows x64 and uploads the
resulting build as a downloadable artifact.

**Jobs:**

1. `generate-bridge` — calls `bridge.yml` (reusable workflow) to generate the
   `flutter_rust_bridge` Dart/Rust bindings needed by `flutter build windows`.

2. `build-windows` — runs on `windows-2022`:
   - Restores bridge files from `bridge-artifact`
   - Installs LLVM 15.0.6 (for Clang-based FFI builds)
   - Installs Flutter 3.24.5 (stable, x64)
   - Replaces the Flutter engine with the rustdesk custom build
   - Applies the Flutter 3.24.4 dropdown-menu patch
   - Installs Rust 1.75 toolchain
   - Sets up vcpkg (pinned commit `9e593bb…`, `x64-windows-static`)
   - Runs `flutter pub run flutter_launcher_icons` to regenerate
     `app_icon.ico` from `res/icon.png`
   - Creates `flutter/assets/lyma-logo.png` from `res/icon.png`
   - Runs `python3 ./build.py --portable --flutter --skip-portable-pack --hwcodec --vram`
   - Copies `custom.txt` into the Release output directory
   - Renames the output to `lyma-desktop/`
   - Zips the output with PowerShell `Compress-Archive`
   - Uploads `lyma-desktop/` + `lyma-desktop-windows-x64.zip` as artifact
     `lyma-desktop-windows-x64`

**Trigger:** `workflow_dispatch`, `push` to `main`, `pull_request` to `main`.

**Download:** Artifacts are available in the GitHub Actions "Artifacts"
section for each workflow run.

## CI Environment Variables

| Variable | Value | Source |
|---|---|---|
| `SCITER_RUST_VERSION` | `1.75` | Matches `RUST_VERSION` discussion #7503 |
| `FLUTTER_VERSION` | `3.24.5` | Stable Flutter used by current RustDesk |
| `LLVM_VERSION` | `15.0.6` | LLVM with Clang for FFI |
| `VCPKG_COMMIT_ID` | `9e593bb…` | Pinned vcpkg version |
| `VCPKG_BINARY_SOURCES` | `clear;x-gha,readwrite` | GitHub Actions binary cache |
| `VERSION` | `1.5.0` | App version from `Cargo.toml` |

## Building Locally

### Prerequisites (Windows)

- Rust 1.75 (`rustup`, target `x86_64-pc-windows-msvc`)
- Flutter 3.24.5 (stable)
- LLVM 15.0.6+ with Clang
- vcpkg (pinned commit `9e593bb…`, triplet `x64-windows-static`)
- Python 3

### Steps

```bash
# 1. Generate bridge files (needs Rust 1.75 + Flutter 3.22.3 on Linux/macOS)
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked
pushd flutter && flutter pub get && popd
~/.cargo/bin/flutter_rust_bridge_codegen \
  --rust-input ./src/flutter_ffi.rs \
  --dart-output ./flutter/lib/generated_bridge.dart

# 2. Regenerate Windows icon from custom res/icon.png
pushd flutter && flutter pub run flutter_launcher_icons && popd

# 3. Copy logo asset (gitignored: *png rule in .gitignore)
cp res/icon.png flutter/assets/lyma-logo.png

# 4. Build
python3 .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram

# 5. Copy custom.txt into the Release output
cp custom.txt ./flutter/build/windows/x64/runner/Release/

# 6. (Output is at ./flutter/build/windows/x64/runner/Release/)
#    Rename to lyma-desktop and zip for distribution.
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
Connection dialog — fields pre-filled with desk.lymagroups.ir
```

## Known Limits

1. **Exe filename** remains `rustdesk.exe`. The `CMakeLists.txt`
   `BINARY_NAME "rustdesk"` is unchanged because `build.py`'s portable packer
   expects that name. The display name is "LymaDesk" via `APP_NAME` config
   + `Runner.rc`. `Runner.rc` uses `lymadeSk` as the internal/original name
   to avoid conflicts with Windows file properties.

2. **Phone number** `+98 912 000 0000` is a placeholder. No phone number
   was found in the repo or `custom.txt`. Update the value at
   `flutter/lib/desktop/pages/desktop_home_page.dart:435` with the real
   number.

3. **`lyma-logo.png`** is git-ignored (`*png` rule in `.gitignore`).
   It must be created in CI from `res/icon.png`. Do not commit it.

4. **usbmmidd / printer drivers** are not included in this build. They are
   optional and only needed for virtual-display (headless) and printer
   redirection features. The original RustDesk CI downloads them after build.

5. **Bridge files** (`generated_bridge.dart`,
   `generated_bridge.freezed.dart`) are generated by `bridge.yml` and must
   be present before `flutter build windows`. The `lyma-build.yml` workflow
   reuses `bridge.yml` via `workflow_call`.

6. **CI only builds Windows x64.** macOS and Linux builds are not included
   in this workflow. Add them to the matrix in `lyma-build.yml` if needed.
