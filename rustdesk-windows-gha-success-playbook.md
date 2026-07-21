# RustDesk Windows GitHub Actions Build Runbook

This document is a **complete, self-contained build recipe** for producing a customized Windows RustDesk installer via GitHub Actions. Any AI agent can follow these instructions without prior conversation history or external context.

**Status: VERIFIED WORKING** — tested end-to-end against upstream `rustdesk/rustdesk` (commit ~2026-07). Both the bridge generation (Linux) and Windows x64 build complete successfully in ~45 minutes.

---

## 1. What You Will Build

A portable Windows RustDesk client with these values pre-configured:

| Setting | Effect |
|---|---|
| ID / rendezvous server | The server your client connects to for peer discovery |
| Relay server | The server that relays traffic when direct P2P fails |
| API server | HTTP API for user management (used with `lejianwen/rustdesk-server`) |
| Public key | If empty, traffic is not end-to-end encrypted |
| Fixed password | Pre-set password for unattended access |

---

## 2. Repository Setup

Your build repository must contain exactly **one file**:

```
.github/workflows/build-rustdesk-windows.yml
```

That's it. The workflow will clone the actual RustDesk source code from `https://github.com/rustdesk/rustdesk.git` at runtime.

**Important**: The repository must be **PUBLIC** if you use a free GitHub account, because GitHub Actions does not run on private repositories for free-tier users (you get a billing error: "recent account payments have failed").

---

## 3. The Workflow YAML — Line by Line

Below is the complete workflow. Each section is explained so an AI can adapt it correctly.

### 3.1 Triggers and Environment Variables

```yaml
name: Build RustDesk Windows

on:
  workflow_dispatch:    # allows manual trigger from GitHub Actions UI
  push:
    branches: [main]   # auto-triggers on push to main

env:
  # === CUSTOMIZE THESE VALUES ===
  ID_SERVER: "abc.zhm666.top"
  RELAY_SERVER: "abc.zhm666.top"
  API_SERVER: "http://abc.zhm666.top:21114"
  FIXED_PASSWORD: "Zhm@123456"
  PUBLIC_KEY: ""
  # === TOOL VERSIONS (do not change unless upstream forces it) ===
  RUST_VERSION: "1.75"              # pinned by rustdesk for sciter compat
  FLUTTER_VERSION: "3.24.5"         # windows x64 build target
  FRB_VERSION: "1.80.1"             # flutter_rust_bridge_codegen
  CARGO_EXPAND_VERSION: "1.0.95"    # required by FRB codegen
  LLVM_VERSION: "15.0.6"            # pinned by upstream
  VCPKG_COMMIT_ID: "120deac3062162151622ca4860575a33844ba10b"
  VCPKG_BINARY_SOURCES: "clear;x-gha,readwrite"
  VCPKG_DEFAULT_HOST_TRIPLET: "x64-windows-static"
```

**What each env var does:**
- `ID_SERVER` → written into `config.rs` as `PROD_RENDEZVOUS_SERVER`
- `RELAY_SERVER` → written into `config.rs` `DEFAULT_SETTINGS` with key `relay-server`
- `API_SERVER` → written into `config.rs` `DEFAULT_SETTINGS` with key `api-server`
- `FIXED_PASSWORD` → read at compile time by `option_env!("FIXED_PASSWORD")` in `Config::load()`
- `PUBLIC_KEY` → overwrites `RS_PUB_KEY` in `config.rs`
- `VCPKG_BINARY_SOURCES` + `VCPKG_DEFAULT_HOST_TRIPLET` → required to prevent ffmpeg header errors

### 3.2 Bridge Generation Job (Linux)

```yaml
jobs:
  generate-bridge:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout RustDesk source
        run: git clone --recurse-submodules https://github.com/rustdesk/rustdesk.git rustdesk-src
```

**Why Linux?** The official RustDesk CI generates flutter-rust-bridge files on Linux because:
- The codegen tool has Windows-specific issues
- It uses Flutter 3.22.3 (different from the 3.24.5 used for the actual build)
- Linux runners are faster and cheaper

```yaml
      - name: Install prerequisites
        run: |
          sudo apt-get update -y
          sudo apt-get install -y clang cmake curl gcc git g++ libclang-dev libgtk-3-dev llvm-dev nasm ninja-build pkg-config wget
```

These are needed to compile the FRB codegen tool and its dependencies.

```yaml
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: ${{ env.RUST_VERSION }}
          targets: x86_64-unknown-linux-gnu
          components: rustfmt

      - uses: Swatinem/rust-cache@v2
        with:
          prefix-key: bridge
```

The `dtolnay/rust-toolchain` action installs Rust properly (do NOT use `rustup install` directly). The `Swatinem/rust-cache` caches the cargo build artifacts for faster subsequent runs.

**Note**: `Swatinem/rust-cache` will log a non-fatal error "could not find `Cargo.toml`" because we clone into a subdirectory. This error is harmless — the action falls back gracefully.

```yaml
      - name: Install Flutter
        uses: subosito/flutter-action@v2.12.0
        with:
          channel: stable
          flutter-version: "3.22.3"
          cache: true
```

Uses Flutter 3.22.3 specifically for bridge generation. The `cache: true` improves speed.

```yaml
      - name: Generate flutter-rust-bridge
        working-directory: rustdesk-src
        run: |
          cargo install cargo-expand --version ${{ env.CARGO_EXPAND_VERSION }} --locked
          cargo install flutter_rust_bridge_codegen --version ${{ env.FRB_VERSION }} --features "uuid" --locked
          sed -i -e 's/extended_text: 14.0.0/extended_text: 13.0.0/g' flutter/pubspec.yaml
          pushd flutter && flutter pub get && popd
          ~/.cargo/bin/flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs --dart-output ./flutter/lib/generated_bridge.dart --c-output ./flutter/macos/Runner/bridge_generated.h
          cp ./flutter/macos/Runner/bridge_generated.h ./flutter/ios/Runner/bridge_generated.h
```

Key details:
- The `sed` command downgrades `extended_text` because Flutter 3.22.3 ships Dart 3.2, which does not support the 14.x version constraint
- The `flutter_rust_bridge_codegen` generates:
  - `src/bridge_generated.rs` — Rust FFI glue
  - `src/bridge_generated.io.rs` — Rust serialization helpers
  - `flutter/lib/generated_bridge.dart` — Dart FFI bindings
  - `flutter/lib/generated_bridge.freezed.dart` — Dart immutable models
  - `flutter/macos/Runner/bridge_generated.h` — C header for macOS
  - `flutter/ios/Runner/bridge_generated.h` — C header for iOS

```yaml
      - name: Upload bridge artifact
        uses: actions/upload-artifact@v4
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

The generated files are uploaded as a named artifact (`bridge-artifact`) so the Windows job can download them.

### 3.3 Windows Build Job

```yaml
  build-windows:
    runs-on: windows-2022
    needs: generate-bridge     # waits for bridge job to finish
    steps:
      - uses: actions/checkout@v4
```

The `needs: generate-bridge` ensures the bridge files exist before the Windows build starts.

#### Step: Checkout RustDesk source
```yaml
      - name: Checkout RustDesk source
        run: git clone --recurse-submodules https://github.com/rustdesk/rustdesk.git rustdesk-src
```
Must use `--recurse-submodules` to get the `hbb_common` submodule (which contains `config.rs`).

#### Steps:  MSVC + LLVM
```yaml
      - uses: microsoft/setup-msbuild@v2

      - name: Install LLVM and Clang
        uses: KyleMayes/install-llvm-action@v2.0.9
        with:
          version: ${{ env.LLVM_VERSION }}
```
**Do NOT use `choco install llvm`**. The windows-2022 runner already has LLVM 20.x installed and chocolatey will refuse to downgrade. The `KyleMayes/install-llvm-action` handles version selection cleanly.

#### Step: Rust toolchain
```yaml
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: ${{ env.RUST_VERSION }}
          targets: x86_64-pc-windows-msvc
          components: rustfmt

      - uses: Swatinem/rust-cache@v2
        with:
          prefix-key: windows
```
Target `x86_64-pc-windows-msvc` is required for Windows. The `rustfmt` component is needed by some RustDesk build scripts.

#### Step: Flutter
```yaml
      - name: Install Flutter
        id: flutter
        uses: subosito/flutter-action@v2.12.0
        with:
          channel: stable
          flutter-version: ${{ env.FLUTTER_VERSION }}
          architecture: x64
```
The `id: flutter` is critical — the `CACHE-PATH` output is used in the next step for engine replacement.

#### Step: Replace Flutter engine
```yaml
      - name: Replace engine with rustdesk custom flutter engine
        run: |
          flutter precache --windows
          Invoke-WebRequest -Uri https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip -OutFile windows-x64-release.zip
          Expand-Archive -Path windows-x64-release.zip -DestinationPath windows-x64-release
          $flutterCache = "${{ steps.flutter.outputs['CACHE-PATH'] }}"
          mv -Force windows-x64-release/* "$flutterCache/bin/cache/artifacts/engine/windows-x64-release/"
```
**Do NOT use `flutter-desktop-embedding`** (an old, incorrect approach). The correct approach downloads the custom engine from `github.com/rustdesk/engine/releases` and replaces the stock Flutter engine. `${{ steps.flutter.outputs['CACHE-PATH'] }}` resolves to something like `C:\hostedtoolcache\windows\flutter\stable-3.24.5-x64`.

#### Step: Patch Flutter SDK
```yaml
      - name: Patch flutter
        shell: bash
        run: |
          cp rustdesk-src/.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff $(dirname $(dirname $(which flutter)))
          cd $(dirname $(dirname $(which flutter)))
          git apply flutter_3.24.4_dropdown_menu_enableFilter.diff
        continue-on-error: true
```
The patch file comes from the RustDesk source repository (cloned into `rustdesk-src`). It fixes a dropdown menu filter issue. `continue-on-error: true` is safe because the patch might already be applied.

#### Steps: vcpkg setup and install
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
          echo "=== Verifying ffmpeg headers ==="
          ls "$VCPKG_ROOT/installed/x64-windows-static/include/libavutil/" 2>/dev/null || echo "MISSING: libavutil headers"
          ls "$VCPKG_ROOT/installed/x64-windows-static/include/libavcodec/" 2>/dev/null || echo "MISSING: libavcodec headers"
```

**CRITICAL**: The windows-2022 runner has vcpkg pre-installed with stale packages that conflict. The `rm -rf "$VCPKG_ROOT/installed/x64-windows-static"` removes old packages before installing fresh ones. Without this, the `hwcodec` crate fails with `fatal error C1083: Cannot open include file: 'libavutil/pixfmt.h'`.

The vcpkg manifest (`vcpkg.json`) is in the `rustdesk-src` directory — it defines all required dependencies (ffmpeg, libvpx, opus, etc.). The install takes ~20 minutes on first run.

#### Step: Restore bridge files
```yaml
      - name: Restore bridge files
        uses: actions/download-artifact@v4
        with:
          name: bridge-artifact
          path: rustdesk-src
```
Downloads the bridge files generated by the Linux job into the `rustdesk-src` directory at the correct paths.

#### Step: Patch config.rs (THE KEY STEP)
```yaml
      - name: Patch RustDesk custom config
        shell: bash
        working-directory: rustdesk-src
        run: |
          python3 << 'PYEOF'
          import re
          with open("libs/hbb_common/src/config.rs", "r", encoding="utf-8") as f:
              content = f.read()

          # 1. Set ID server
          # Changes: PROD_RENDEZVOUS_SERVER from "" to "abc.zhm666.top"
          content = re.sub(
              r'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new\(""\.to_owned\(\)\);',
              'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("abc.zhm666.top".to_owned());',
              content
          )

          # 2. Set relay-server AND api-server in DEFAULT_SETTINGS
          # Changes: Default::default() to HashMap with relay-server and api-server
          content = re.sub(
              r'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default\(\);',
              'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = RwLock::new(HashMap::from([("relay-server".to_owned(), "abc.zhm666.top".to_owned()), ("api-server".to_owned(), "http://abc.zhm666.top:21114".to_owned())]));',
              content
          )

          # 3. Clear public key
          content = re.sub(
              r'pub const RS_PUB_KEY: &str = ".*?";',
              'pub const RS_PUB_KEY: &str = "";',
              content
          )

          # 4. Inject fixed password check into Config::load()
          # The marker is: fn load() -> Config {\n        let mut config = Config::load_::<Config>("");
          # After this line, we insert code that sets password from FIXED_PASSWORD env var
          load_marker = 'fn load() -> Config {\n        let mut config = Config::load_::<Config>("");'
          password_inject = '\n        if config.password.is_empty() {\n            if let Some(pwd) = option_env!("FIXED_PASSWORD") {\n                if !pwd.is_empty() {\n                    config.password = pwd.to_owned();\n                }\n            }\n        }\n'
          if load_marker in content:
              content = content.replace(load_marker, load_marker + password_inject, 1)

          with open("libs/hbb_common/src/config.rs", "w", encoding="utf-8") as f:
              f.write(content)
          PYEOF
```

**Why Python instead of PowerShell?** Multiline string replacements in PowerShell's `-replace` operator inside a YAML `run: |` block are extremely fragile — any indentation mismatch breaks the YAML parser. Using `python3 << 'PYEOF'` (a bash here-document) avoids this entirely.

**Why `re.sub()` vs `str.replace()`?**
- `re.sub()` with regex patterns is used for replacements where the text has regex-special characters like `(`, `)`, `.`
- `str.replace()` is used for the password injection because it's a simple literal string match
- The `load_marker` string uses `\n` as Python escape sequences for newlines, avoiding actual newlines in the Python script

**Why `content.replace(marker, new, 1)` with `1`?**
Without the `1` limit, `.replace()` replaces ALL occurrences. The `Config::load()` function appears exactly once, but to be safe, the `1` prevents accidental multiple matches.

**How `option_env!("FIXED_PASSWORD")` works:**
- `option_env!("FIXED_PASSWORD")` is a Rust compile-time macro
- It reads the environment variable `FIXED_PASSWORD` when `rustc` compiles `config.rs`
- The env var is set in the workflow's `env:` block, which propagates to child processes (`cargo build` → `rustc`)
- If the env var is set, it returns `Some("Zhm@123456")`; otherwise `None`
- The injected code only sets `config.password` if it's currently empty, so user-set passwords are preserved

**What the injected Rust code looks like after patching:**
```rust
fn load() -> Config {
    let mut config = Config::load_::<Config>("");
    if config.password.is_empty() {
        if let Some(pwd) = option_env!("FIXED_PASSWORD") {
            if !pwd.is_empty() {
                config.password = pwd.to_owned();
            }
        }
    }
    // ... rest of load() ...
}
```

#### Step: Verify patches
```yaml
      - name: Verify config patches
        shell: bash
        working-directory: rustdesk-src
        run: |
          echo "=== Checking config.rs patches ==="
          echo "ID server:"
          grep -n "abc.zhm666.top" libs/hbb_common/src/config.rs | head -3
          echo "API server:"
          grep -n "api-server" libs/hbb_common/src/config.rs | head -3
          echo "Public key:"
          grep "RS_PUB_KEY" libs/hbb_common/src/config.rs
          echo "Password injection:"
          grep -n "FIXED_PASSWORD\|option_env" libs/hbb_common/src/config.rs
```
This step confirms the patches were applied. If any grep returns empty, the patch failed and the build will likely fail later. Check for:
- `abc.zhm666.top` should appear in `PROD_RENDEZVOUS_SERVER` and `relay-server`
- `api-server` should appear in `DEFAULT_SETTINGS`
- `RS_PUB_KEY = ""` should be the public key
- `FIXED_PASSWORD` should appear in `Config::load()`

#### Steps: Build, Rename, Upload
```yaml
      - name: Build rustdesk
        working-directory: rustdesk-src
        run: |
          python build.py --portable --hwcodec --flutter --vram

      - name: Rename installer
        working-directory: rustdesk-src
        run: |
          Get-ChildItem -Filter "*.exe" | ForEach-Object {
            $newName = "custom-rustdesk-$($_.Name)"
            Rename-Item -Path $_.FullName -NewName $newName
          }

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: custom-rustdesk-windows
          path: rustdesk-src/*.exe
```

The `build.py` script handles everything: building the Rust backend, compiling the Flutter frontend, and packaging into a portable executable. Flags:
- `--portable` — creates a standalone portable EXE
- `--hwcodec` — enables hardware codec support (NVENC, AMF, etc.)
- `--flutter` — builds the Flutter GUI
- `--vram` — enables VRAM-based encoding (NVENC)

---

## 4. Understanding config.rs and What Gets Patched

The file `libs/hbb_common/src/config.rs` in the RustDesk submodule contains several key values:

### 4.1 `PROD_RENDEZVOUS_SERVER` (ID server)

```rust
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());
```
This is the default rendezvous server — the server clients use to discover each other. When empty, the client prompts the user to enter one. Our patch sets it to your server.

### 4.2 `DEFAULT_SETTINGS` (relay server, API server, and more)

```rust
pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default();
```
This is a HashMap of default configuration options. The config keys are defined as constants in the `keys` module of `config.rs`:

| Constant | Key string | Purpose |
|---|---|---|
| `OPTION_RELAY_SERVER` | `relay-server` | Relay server for proxied connections |
| `OPTION_API_SERVER` | `api-server` | HTTP API for user management |
| `OPTION_KEY` | `key` | Public key for end-to-end encryption |

The `api-server` setting is read by the RustDesk client to communicate with a management API (like `lejianwen/rustdesk-api`).

### 4.3 `RS_PUB_KEY` (public key)

```rust
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
```
This is the default public key for end-to-end encryption. When set to `""`, encryption is disabled.

### 4.4 `Config::password` (NOT a settings key!)

```rust
pub struct Config {
    pub password: String,  // NOT in options HashMap!
    pub salt: String,
    pub id: String,
    ...
}
```

**Critical**: The password is NOT stored in `DEFAULT_SETTINGS`. It's a separate field in the `Config` struct. Adding `password` to `DEFAULT_SETTINGS` has NO effect. The password must be set via `config.password = ...`.

The password uses an encoded format (version prefix `"01"` + base64-encrypted hash). However, the comparison code also accepts plaintext passwords (checked by `password_is_empty_or_not_hashed()`). Setting a plaintext password like `"Zhm@123456"` works because the code falls back to plaintext comparison.

---

## 5. Common Failure Modes and Fixes

### 5.1 YAML parsing failure (0-second build failure)

**Symptom**: Build fails instantly with "This run likely failed because of a workflow file issue."

**Cause**: In YAML `run: |` (literal block scalar), ALL lines must have the SAME or GREATER indentation than the first content line. A single line with LESS indentation breaks the parser.

**Example of WRONG YAML** (the `let mut config` line has 8 spaces, less than the 10-space block indent):
```yaml
        run: |
          $content = $content -replace '...', 'fn load() -> Config {
        let mut config = Config::load_::<Config>("");
```

**Fix**: Use `python3 << 'PYEOF'` for complex multiline scripts. Python's triple-quoted strings handle multiline content without YAML indentation issues.

### 5.2 `default_password` defined multiple times

**Symptom**: `error[E0428]: the name 'default_password' is defined multiple times`

**Cause**: `content.replace("lazy_static::lazy_static!", func, count)` without the `count=1` limit replaces ALL 6 occurrences of `lazy_static::lazy_static!` in the file.

**Fix**: Always pass `count=1` to `str.replace()` when inserting into a marker that appears multiple times.

### 5.3 `default_password` not found in scope

**Symptom**: `error[E0425]: cannot find function 'default_password' in this scope`

**Cause**: The function was inserted before a `lazy_static!` that's inside a submodule, making it inaccessible from the outer module scope.

**Fix**: Don't add a separate function. Inject the check directly inline in `Config::load()`.

### 5.4 ffmpeg header not found

**Symptom**: `fatal error C1083: Cannot open include file: 'libavutil/pixfmt.h'`

**Cause**: The windows-2022 runner has stale vcpkg packages that don't include ffmpeg for the `x64-windows-static` triplet.

**Fix**: Delete the stale installed packages before installing:
```bash
rm -rf "$VCPKG_ROOT/installed/x64-windows-static"
$VCPKG_ROOT/vcpkg install --triplet x64-windows-static
```

### 5.5 Private repo billing

**Symptom**: "The job was not started because recent account payments have failed"

**Fix**: Set the repository to PUBLIC. GitHub Actions requires a paid plan for private repositories.

### 5.6 `Swatinem/rust-cache` cargo.toml error

**Symptom**: "Error: could not find `Cargo.toml`"

**Cause**: The cache action looks for `Cargo.toml` in the workspace root, but we clone into `rustdesk-src/`.

**Fix**: This error is harmless. The action falls back gracefully and cache is skipped. The build still works.

---

## 6. How to Use This Runbook

### For an AI agent

1. Create a GitHub repository with the workflow YAML
2. Set the repo to **PUBLIC**
3. Customize the `env:` values (ID_SERVER, RELAY_SERVER, API_SERVER, FIXED_PASSWORD, PUBLIC_KEY)
4. Push to `main`
5. Wait ~45 minutes for the build
6. Download the `custom-rustdesk-windows` artifact

### Verification checklist

Before pushing, verify:

- [ ] Repo is PUBLIC (not private)
- [ ] `env:` has all 5 customizable values filled in
- [ ] Tool version env vars (RUST_VERSION, etc.) match the current upstream pins
- [ ] `python3 << 'PYEOF'` is used for config patching (not PowerShell multiline)
- [ ] `content.replace(marker, new, 1)` has the `1` argument
- [ ] `config.rs` regex patterns match the ACTUAL upstream source (check before pushing)
- [ ] vcpkg stale directory is deleted before install (`rm -rf "$VCPKG_ROOT/installed/x64-windows-static"`)
- [ ] Flutter engine URL (`rustdesk/engine/releases`) is still valid
- [ ] Bridge artifact paths match the files generated by `flutter_rust_bridge_codegen`

### If the build fails

1. Check if it's a YAML parsing error (0-second build) → fix indentation, use Python here-doc
2. Check vcpkg step for ffmpeg header errors → verify the stale dir was cleaned
3. Check Rust compilation errors in `hbb_common` → verify config.rs patches match upstream format
4. Check if `option_env!("FIXED_PASSWORD")` is found → verify the marker string matches the actual `fn load()` signature
5. All other errors → likely upstream code changes, check the official RustDesk CI for updated tool versions

---

## 7. Reference: Checking Upstream Patterns

Before using this runbook, verify the config.rs patterns match the current upstream:

```bash
# Fetch the current config.rs patterns
gh api repos/rustdesk/hbb_common/contents/src/config.rs | \
  jq -r '.content' | base64 -d | grep -E "PROD_RENDEZVOUS_SERVER|DEFAULT_SETTINGS|RS_PUB_KEY"
```

If the output differs from:
```
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());
pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default();
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
```

...then update the workflow's regex patterns accordingly.

Also verify the `fn load() -> Config` function signature:
```bash
gh api repos/rustdesk/hbb_common/contents/src/config.rs | \
  jq -r '.content' | base64 -d | grep -A1 "fn load()"
```

Expected:
```rust
fn load() -> Config {
    let mut config = Config::load_::<Config>("");
```

The `load_marker` string in the Python script must match this exactly (with the correct indentation).

---

## 8. Expected Output

After a successful run, you get:

1. **`bridge-artifact`** (intermediate, ~6 files) — can be ignored
2. **`custom-rustdesk-windows`** — a Windows portable EXE with all your settings pre-configured

Download the artifact from the GitHub Actions run page and distribute the EXE to your users.
