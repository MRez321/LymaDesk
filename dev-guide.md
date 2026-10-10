# LymaDesk Development Guide

This guide explains how to set up a hot-reload development environment for
editing the LymaDesk sidebar and other UI elements without rebuilding the
entire app from CI.

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Flutter SDK | 3.24.5 (stable) | `git clone https://github.com/flutter/flutter.git --branch 3.24.5` then add `flutter/bin` to `PATH` |
| Rust toolchain | 1.75 | `rustup install 1.75` |
| cargo-expand | latest | `cargo install cargo-expand` |
| flutter_rust_bridge_codegen | 1.80.1 | `cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked` |
| CMake | 3.24+ | https://cmake.org/download/ |
| Visual Studio Build Tools (Windows) | 17.x | Select "Desktop development with C++" |
| LLVM/Clang | 15.0.6+ | https://releases.llvm.org/ |
| vcpkg | pinned commit | `git clone https://github.com/microsoft/vcpkg.git` then `git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d` |

> **Tip**: If you only need to edit sidebar **styles** (colors, padding, text),
> you can skip Rust entirely. Build once via CI, then use `flutter run` for
> hot reload on subsequent edits. The Dart side reloads without rebuilding Rust.

## Quick Start (Sidebar Editing)



```powershell
# 1. One-time setup: install Flutter + Rust deps
flutter --version                             # should print 3.24.5
rustc --version                               # should print 1.75.x
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked

# 2. Generate bridge files (needed once, or after src/flutter_ffi.rs changes)
cd flutter
flutter pub get
flutter_rust_bridge_codegen `
  --rust-input ../src/flutter_ffi.rs `
  --dart-output ./lib/generated_bridge.dart

# 3. Build the Rust library (debug, needed once or after Rust changes)
cd ..
cargo build --features flutter --lib

# 4. Create logo assets (gitignored: *png rule in .gitignore)
cp res\icon.png flutter\assets\lyma-logo.png
cp res\icon.png flutter\assets\logo.png
cp res\icon.png flutter\assets\logo_light.png
cp res\icon.png flutter\assets\logo_dark.png

# 5. Run Flutter with hot reload
cd flutter
flutter run
```

**Hot reload**: After step 5, press `r` (reload) or `R` (restart) in the
terminal to see style changes instantly without rebuilding Rust.
**Style-only changes** (colors, padding, text): just press `r` — no Rust
rebuild needed. Structural changes (adding widgets): press `R` (full restart).

### What Each File Controls

| File | What It Controls |
|---|---|
| `flutter/lib/desktop/pages/desktop_home_page.dart` | **The sidebar** (left panel). The `buildTip()` method at line 389 renders the LymaDesk logo, website link, and phone number. |
| `flutter/lib/common.dart` | App logo (`_Logo` class, line 3779), server config class (`ServerConfig`, line 2961), and Dart bindings to Rust. |
| `flutter/lib/common/widgets/dialog.dart` | Server settings dialog (`showServerSettings`, line 56). This is where the ID/Relay server fields are shown/hidden. |
| `flutter/lib/mobile/widgets/dialog.dart` | **Server settings dialog implementation** (`showServerSettingsWithValue`, line 68). This is where `showValues` parameter controls whether default values are visible. |
| `flutter/assets/lyma-logo.png` | The sidebar logo (git-ignored; CI copies from `lyma.png` in repo root). Replace `lyma.png` to change it. |
| `flutter/assets/logo.png` | Small nav logo at the top of the sidebar (shown in `_Logo` widget). |
| `flutter/assets/logo_light.png` | Light-theme variant of the nav logo. |
| `flutter/assets/logo_dark.png` | Dark-theme variant of the nav logo. |
| `custom.txt` | Default server address, relay, API, encryption key — read at app startup. |
| `flutter/windows/runner/Runner.rc` | Windows file properties (exe right-click → Properties). |
| `flutter/windows/runner/resources/app_icon.ico` | Windows app icon (taskbar, Alt+Tab). Regenerate with `flutter pub run flutter_launcher_icons`. |

## Sidebar Structure

The sidebar is built by `_DesktopHomePageState.build()` (line 59 in
`desktop_home_page.dart`). The left panel is `buildLeftPane()` (line 81):

```
Sidebar (buildLeftPane)
├── buildPresetPasswordWarning()       — password warning banner
├── loadLogo()                         — small nav logo (assets/logo*.png)
├── buildTip(context)                  — **LymaDesk branding block (edit here!)**
│   ├── LYMA LOGO                      — assets/lyma-logo.png (height: 56, centered)
│   ├── LYMA CONTACT INFO
│   │   ├── Website link               — lymagroups.ir → https://lumagroups.ir (#FBB12F)
│   │   └── Phone number               — 025-33808586
│   └── SEPARATOR LINE                  — grey divider (grey.shade300)
├── buildIDBoard(context)              — 9-digit ID display
├── buildPasswordBoard(context)        — 6-digit password display
└── buildHelpCards()                   — help/support cards
```

**Note**: The "powered by RustDesk" text/link was removed from the sidebar
(previously shown via `loadPowered(context)`).

### Editing the Sidebar (Step by Step)

1. **Change the sidebar logo image**:
   - Edit `lyma.png` in the repo root (`lyma.png` is tracked via `.gitignore`
     exception). CI copies it to `flutter/assets/lyma-logo.png`.
   - For local testing:
     ```powershell
     cp your-logo.png flutter\assets\lyma-logo.png
     ```
   - **In CI**: the sidebar logo is generated from `lyma.png` (repo root).
     The nav logo (`logo.png`, `logo_light.png`, `logo_dark.png`) is generated
     from `res/icon.png`.

2. **Change the logo size**:
   - Edit `desktop_home_page.dart` line 404:
     ```dart
     height: 40,  // ← change this (e.g., 60 for larger, 20 for smaller)
     ```

3. **Change the website link**:
   - Edit two lines in `buildTip()` (lines 419 and 425):
     ```dart
     final uri = Uri.parse('https://lumagroups.ir');   // ← URL to open
     ...
     'lymagroups.ir',                                    // ← displayed text
     ```
   - The displayed text and the URL can differ (as they do now: text shows
     `lymagroups.ir`, tap opens `https://lumagroups.ir`).

4. **Change the phone number**:
   - Edit line 435 in `buildTip()`:
     ```dart
     '025-33808586',   // ← current value
     ```

5. **Change colors, fonts, padding**:
   - The contact info `Text` and `TextStyle` widgets in `buildTip()`
     (lines 426–438) support any Flutter `TextStyle` property:
     ```dart
     style: TextStyle(
       color: Colors.blue,       // ← link color
       fontSize: 13,             // ← font size
       fontWeight: FontWeight.bold, // ← add weight
       fontStyle: FontStyle.italic, // ← italic
     ),
     ```
   - Padding is controlled by `EdgeInsets` at line 413:
     ```dart
     padding: const EdgeInsets.only(top: 4.0, bottom: 4.0),
     ```

6. **Change the nav logo** (small logo at top of sidebar):
   - Replace `flutter/assets/logo.png`, `logo_light.png`, `logo_dark.png`.
   - Max dimensions: 300 × 60 (enforced by `_LogoState` at `common.dart:3811`).

## Hot Reload Tips

- **Style-only changes** (colors, padding, text): press `r` in the Flutter
  terminal — changes appear instantly without recompiling Rust.
- **New image assets**: add the file to `flutter/assets/`, then press `R`
  (full restart) so Flutter re-scans the assets directory.
- **Custom fonts**: add to `flutter/pubspec.yaml` under `flutter:` → `fonts:`.
- **Structural changes** (adding/removing widgets): press `r` — Flutter
  hot reload supports structural changes in most cases.

## Building a Release (Windows PowerShell)

For local release builds (testing without CI), run:

```powershell
python .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram
cp custom.txt .\flutter\build\windows\x64\runner\Release\
Rename-Item .\flutter\build\windows\x64\runner\Release\ -NewName LymaDesk
```

Then build the installer:
```powershell
(Get-Content res\manifest.xml) -replace 'dpiAware.*', '' | Set-Content res\manifest.xml
cd libs\portable
pip install -r requirements.txt
python .\generate.py -f ..\..\LymaDesk\ -o . -e ..\..\LymaDesk\rustdesk.exe
cd ..\..
Move-Item .\target\release\rustdesk-portable-packer.exe .\LymaDesk-1.5.0-setup.exe
```

The resulting `LymaDesk-1.5.0-setup.exe` is a self-extracting installer.
