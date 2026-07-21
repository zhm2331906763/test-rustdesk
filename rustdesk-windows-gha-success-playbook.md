# RustDesk Windows GitHub Actions Build Runbook

This document is a reusable build recipe for producing a customized Windows RustDesk installer in GitHub Actions.

It is intentionally written as a standalone execution guide. A new agent should be able to follow it without any prior conversation history.

**Verified working**: This recipe has been tested end-to-end against the upstream RustDesk codebase (commit ~2026-07). Both the bridge generation and Windows x64 build complete successfully.

## Purpose

Build a Windows RustDesk installer with these customizable inputs:

- ID server
- Relay server
- Public key

## Parameter template

```text
ID_SERVER=""
RELAY_SERVER=""
PUBLIC_KEY=""
```

## Required repository layout

The build repository should include:

- `.github/workflows/build-rustdesk-windows.yml`

The workflow clones upstream RustDesk into `rustdesk-src` at runtime.

## Build inputs

Keep these tool versions fixed unless a new RustDesk release forces changes:

| Variable | Value | Notes |
|---|---|---|
| `RUST_VERSION` | `1.75` | pinned by upstream (sciter compat) |
| `FLUTTER_VERSION` | `3.24.5` | Windows x64 build target |
| `FRB_VERSION` | `1.80.1` | flutter_rust_bridge_codegen |
| `CARGO_EXPAND_VERSION` | `1.0.95` | required by FRB |
| `LLVM_VERSION` | `15.0.6` | pinned by upstream |
| `VCPKG_COMMIT_ID` | `120deac3062162151622ca4860575a33844ba10b` | must match vcpkg.json baseline |

The build command:

```powershell
python .\build.py --portable --hwcodec --flutter --vram
```

## Workflow structure

The official RustDesk CI uses **two jobs** — bridge generation and Windows build — because:

- flutter-rust-bridge codegen runs on **Linux** (faster, no Windows-specific issues)
- Generated bridge files are platform-independent and passed via artifacts

### Job 1: `generate-bridge` (runs-on: ubuntu-22.04)

1. Clone RustDesk source
2. Install Linux build prerequisites (clang, cmake, gtk3, nasm, etc.)
3. Install Rust 1.75 with `dtolnay/rust-toolchain`
4. Install Flutter 3.22.3 with `subosito/flutter-action`
5. Install `cargo-expand` and `flutter_rust_bridge_codegen`
6. Patch `flutter/pubspec.yaml` for Flutter 3.22.3 compat
7. Run `flutter pub get` in `flutter/` subdirectory
8. Generate bridge files with `flutter_rust_bridge_codegen`
9. Upload bridge files as artifact (`bridge-artifact`)

### Job 2: `build-windows` (runs-on: windows-2022, needs: generate-bridge)

1. Checkout build repository
2. Clone RustDesk source with submodules
3. Install MSVC tools (`microsoft/setup-msbuild@v2`)
4. Install LLVM/Clang (`KyleMayes/install-llvm-action@v2.0.9`)
5. Install Rust 1.75 with `dtolnay/rust-toolchain`
6. Apply `Swatinem/rust-cache` for faster rebuilds
7. Install Flutter 3.24.5 with `subosito/flutter-action`
8. Replace Flutter engine: download from `rustdesk/engine/releases` and extract to Flutter cache
9. Apply Flutter patch: `flutter_3.24.4_dropdown_menu_enableFilter.diff`
10. Setup vcpkg with `lukka/run-vcpkg@v11`
11. **Clean and reinstall** vcpkg dependencies (see gotchas below)
12. Download bridge artifact
13. Patch `config.rs` with custom server/key values
14. Build: `python build.py --portable --hwcodec --flutter --vram`
15. Rename installer EXE and upload

## KNOWN GOTCHAS

These were discovered through failed builds. An AI agent MUST handle each one.

### 1. LLVM installation on windows-2022

**DO NOT** use `choco install llvm`. The windows-2022 runner already has LLVM 20.x installed. Chocolatey will refuse to downgrade.

**Use instead:**
```yaml
- uses: KyleMayes/install-llvm-action@v2.0.9
  with:
    version: 15.0.6
```

This action handles installation without version conflicts.

### 2. Flutter installation

**DO NOT** use `git clone` for Flutter. The windows-2022 runner has Flutter in its tool cache; use `subosito/flutter-action` which installs from the cache and sets up PATH automatically.

```yaml
- name: Install Flutter
  id: flutter
  uses: subosito/flutter-action@v2.12.0
  with:
    channel: stable
    flutter-version: 3.24.5
    architecture: x64
```

The `id: flutter` is required — its `CACHE-PATH` output is used in the engine replacement step.

### 3. Flutter engine replacement

**DO NOT** use `flutter-desktop-embedding` (cloning and copying files is wrong). Instead:

```yaml
- name: Replace engine with rustdesk custom flutter engine
  run: |
    flutter precache --windows
    Invoke-WebRequest -Uri https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip -OutFile windows-x64-release.zip
    Expand-Archive -Path windows-x64-release.zip -DestinationPath windows-x64-release
    $flutterCache = "${{ steps.flutter.outputs['CACHE-PATH'] }}"
    mv -Force windows-x64-release/* "$flutterCache/bin/cache/artifacts/engine/windows-x64-release/"
```

The `CACHE-PATH` output from `subosito/flutter-action` points to the Flutter SDK root (e.g., `C:\hostedtoolcache\windows\flutter\stable-3.24.5-x64`).

### 4. Flutter SDK patch

The RustDesk build applies a patch to the Flutter SDK for dropdown menu compatibility. The patch file is in the RustDesk source at `.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff`.

```yaml
- name: Patch flutter
  shell: bash
  run: |
    cp rustdesk-src/.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff $(dirname $(dirname $(which flutter)))
    cd $(dirname $(dirname $(which flutter)))
    git apply flutter_3.24.4_dropdown_menu_enableFilter.diff
  continue-on-error: true  # safe if already applied
```

### 5. Rust toolchain

**DO NOT** use `rustup toolchain install` directly. Use the `dtolnay/rust-toolchain` action which properly sets up the Rust environment and adds the required target:

```yaml
- uses: dtolnay/rust-toolchain@stable
  with:
    toolchain: 1.75
    targets: x86_64-pc-windows-msvc
    components: rustfmt
```

### 6. vcpkg — stale packages cause ffmpeg header failures

**CRITICAL**: The windows-2022 runner has vcpkg pre-installed at `C:\vcpkg` with pre-existing packages. These can conflict with the RustDesk-required versions. The `hwcodec` crate depends on ffmpeg headers (`libavutil/pixfmt.h`, `libavcodec/avcodec.h`), and stale packages will cause `fatal error C1083: Cannot open include file`.

**The fix**: Delete the stale installed directory for the target triplet before installing:

```yaml
- name: Setup vcpkg
  uses: lukka/run-vcpkg@v11
  with:
    vcpkgDirectory: C:\vcpkg
    vcpkgGitCommitId: ${{ env.VCPKG_COMMIT_ID }}
    doNotCache: false

- name: Clean and install vcpkg dependencies
  shell: bash
  working-directory: rustdesk-src
  run: |
    rm -rf "$VCPKG_ROOT/installed/x64-windows-static"
    $VCPKG_ROOT/vcpkg install --triplet x64-windows-static --x-install-root="$VCPKG_ROOT/installed"
    ls "$VCPKG_ROOT/installed/x64-windows-static/include/libavutil/"  # verify
```

Also set these environment variables at the workflow level:

```yaml
env:
  VCPKG_BINARY_SOURCES: "clear;x-gha,readwrite"
  VCPKG_DEFAULT_HOST_TRIPLET: "x64-windows-static"
```

### 7. Bridge generation — MUST run on Linux, not Windows

**DO NOT** run `flutter_rust_bridge_codegen` on Windows. The official RustDesk CI generates bridge files on **Linux** (ubuntu-22.04) for two reasons:

- flutter-rust-bridge codegen has Windows-specific issues
- The bridge generation needs Flutter 3.22.3 (not 3.24.5 used for the Windows build), and separating them avoids Flutter version conflicts

Generate bridge files in a separate job, upload them as artifacts, and have the Windows job download them:

```yaml
generate-bridge:
  runs-on: ubuntu-22.04
  steps:
    - run: git clone --recurse-submodules https://github.com/rustdesk/rustdesk.git rustdesk-src
    - uses: dtolnay/rust-toolchain@stable
      with:
        toolchain: 1.75
        targets: x86_64-unknown-linux-gnu
    - uses: subosito/flutter-action@v2.12.0
      with:
        channel: stable
        flutter-version: "3.22.3"
    - run: |
        cargo install cargo-expand --version 1.0.95 --locked
        cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked
        sed -i -e 's/extended_text: 14.0.0/extended_text: 13.0.0/g' flutter/pubspec.yaml
        pushd flutter && flutter pub get && popd
        flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs \
          --dart-output ./flutter/lib/generated_bridge.dart \
          --c-output ./flutter/macos/Runner/bridge_generated.h
    - uses: actions/upload-artifact@v4
      with:
        name: bridge-artifact
        path: |
          rustdesk-src/src/bridge_generated.rs
          rustdesk-src/src/bridge_generated.io.rs
          rustdesk-src/flutter/lib/generated_bridge.dart
          rustdesk-src/flutter/lib/generated_bridge.freezed.dart
          rustdesk-src/flutter/macos/Runner/bridge_generated.h
          rustdesk-src/flutter/ios/Runner/bridge_generated.h
```

The `sed` line downgrades `extended_text` from 14.0.0 to 13.0.0 because Flutter 3.22.3 ships Dart 3.2 which does not support the 14.x constraint.

### 8. Rust caching

Use `Swatinem/rust-cache` after installing Rust but before the build step. This caches the `target/` directory and significantly speeds up subsequent builds (first build still compiles everything):

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    prefix-key: windows
```

Place this AFTER `dtolnay/rust-toolchain` and BEFORE any `cargo install` commands.

### 9. Source patching (config.rs)

The three replacement strings were verified against the actual `hbb_common/src/config.rs` in the upstream repo. Here is the exact current format:

```rust
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());
pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default();
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
```

The PowerShell replacement commands:

```powershell
$file = "libs\hbb_common\src\config.rs"
$content = Get-Content $file -Raw
$content = $content -replace 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new\(""\.to_owned\(\)\);', 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("<ID_SERVER>".to_owned());'
$content = $content -replace 'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default\(\);', 'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = RwLock::new(HashMap::from([("relay-server".to_owned(), "<RELAY_SERVER>".to_owned())]));'
$content = $content -replace 'pub const RS_PUB_KEY: &str = ".*?";', 'pub const RS_PUB_KEY: &str = "<PUBLIC_KEY>";'
Set-Content -Path $file -Value $content -Encoding UTF8
```

Note: `RS_PUB_KEY` uses `".*?"` (lazy match) because the actual key value may change between RustDesk versions.

## Validation checklist

Before pushing a change, verify these requirements (each one was learned through trial and error):

- [ ] LLVM uses `KyleMayes/install-llvm-action`, NOT `choco install llvm`
- [ ] Flutter uses `subosito/flutter-action`, NOT `git clone`
- [ ] Engine replacement downloads from `rustdesk/engine/releases`, NOT `flutter-desktop-embedding`
- [ ] Flutter patch is applied from `rustdesk-src/.github/patches/`
- [ ] Rust uses `dtolnay/rust-toolchain`, NOT `rustup install`
- [ ] `Swatinem/rust-cache` is placed after Rust install
- [ ] vcpkg uses `lukka/run-vcpkg` with commit pinning
- [ ] Stale vcpkg packages are deleted BEFORE `vcpkg install`
- [ ] `VCPKG_BINARY_SOURCES` and `VCPKG_DEFAULT_HOST_TRIPLET` are set
- [ ] ffmpeg headers are verified after vcpkg install
- [ ] Bridge generation runs on Linux (ubuntu-22.04), NOT Windows
- [ ] Bridge files are uploaded and downloaded as artifacts
- [ ] Config patching regexes match the exact upstream format
- [ ] Build command is `python build.py --portable --hwcodec --flutter --vram`
- [ ] Installer EXE is renamed and uploaded as artifact

## Operational notes

- If a build fails, check the vcpkg step first (most common failure point). Verify `libavutil/pixfmt.h` exists at `$VCPKG_ROOT/installed/x64-windows-static/include/`.
- The bridge generation job takes ~6 minutes. The Windows build job takes ~40-50 minutes. Total wall time: ~1 hour.
- Node.js 20 deprecation warnings are harmless.
- If upstream RustDesk changes the config.rs format, update the regex replacement patterns accordingly.

## Expected output

Each workflow run produces:

- a `bridge-artifact` (intermediate, ~6 files)
- a `custom-rustdesk-windows` artifact containing the portable installer EXE (renamed)

## Complete working workflow

See `.github/workflows/build-rustdesk-windows.yml` in this repository for the current verified workflow. Copy it as a starting point, then replace `ID_SERVER`, `RELAY_SERVER`, and `PUBLIC_KEY` with your values.
