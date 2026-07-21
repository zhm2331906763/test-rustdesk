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
API_SERVER=""
FIXED_PASSWORD=""
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

## Customizable features

### API server

The RustDesk client supports an API server URL that enables server-side user management. When configured, the client connects to this API for authentication and management instead of using direct ID-based connections.

The API server is set by adding `api-server` to `DEFAULT_SETTINGS` in `config.rs`:

```rust
pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = RwLock::new(HashMap::from([
    ("relay-server".to_owned(), "<RELAY_SERVER>".to_owned()),
    ("api-server".to_owned(), "<API_SERVER>".to_owned()),
]));
```

The config key is `api-server` (with hyphen), matching the constant `pub const OPTION_API_SERVER: &str = "api-server"` in the RustDesk source.

### Fixed password

RustDesk stores passwords in its config file using an encoded format (version prefix + encrypted hash). Setting a fixed password at compile time is NOT as simple as adding `password` to `DEFAULT_SETTINGS` — the password field is a separate field in the `Config` struct, not a regular settings key.

The working approach: inject the password check in `Config::load()` using `option_env!("FIXED_PASSWORD")`, which reads an environment variable at compile time:

```python
# In the Python patching script:
load_marker = 'fn load() -> Config {\n        let mut config = Config::load_::<Config>("");'
password_inject = '\n        if config.password.is_empty() {\n            if let Some(pwd) = option_env!("FIXED_PASSWORD") {\n                if !pwd.is_empty() {\n                    config.password = pwd.to_owned();\n                }\n            }\n        }\n'
content = content.replace(load_marker, load_marker + password_inject, 1)
```

This generates Rust code that, at every startup, sets the password from the compile-time env var if no password is already configured.

**Why this approach works:**
- On first run (no config file), `Config::default()` returns `password: ""`
- `Config::load()` is called, the injected code sets the password from the env var
- The password is now available for authentication
- When the user later sets a password via the UI, it overrides the default
- The injected code only sets the password if `is_empty()`, so it won't overwrite user changes

**Why simpler approaches FAIL:**
- ❌ Adding `password` to `DEFAULT_SETTINGS` — `password` is NOT in `KEYS_SETTINGS`, it's a separate struct field
- ❌ Adding a `default_password()` function with `#[serde(default = "default_password")]` — the function must be at module scope and careful with YAML indentation; also doesn't handle the first-run case
- ❌ Using `impl Default for Config` — requires removing `Default` from derive, complex structural changes

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

The source patching is done via a **Python script** (not PowerShell regex, which has limitations with multiline replacements). Use `python3` with a here-document in bash.

The replacements target these exact lines in `hbb_common/src/config.rs`:

```rust
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());
pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default();
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
```

#### Simple replacements (ID server, relay+API servers, key)

Use `re.sub()` in Python for each:

```python
import re

# 1. ID server
content = re.sub(
    r'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new\(""\.to_owned\(\)\);',
    'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("<ID_SERVER>".to_owned());',
    content
)

# 2. Relay server + API server
content = re.sub(
    r'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default\(\);',
    'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = RwLock::new(HashMap::from([("relay-server".to_owned(), "<RELAY_SERVER>".to_owned()), ("api-server".to_owned(), "<API_SERVER>".to_owned())]));',
    content
)

# 3. Public key (empty = no encryption)
content = re.sub(
    r'pub const RS_PUB_KEY: &str = ".*?";',
    'pub const RS_PUB_KEY: &str = "";',
    content
)
```

Note: `RS_PUB_KEY` uses `".*?"` (lazy match) because the actual key value may change between RustDesk versions.

#### Fixed password injection (Config::load())

The password is injected into the `Config::load()` function. Use `str.replace()` with literal matching:

```python
load_marker = 'fn load() -> Config {\n        let mut config = Config::load_::<Config>("");'
password_inject = '\n        if config.password.is_empty() {\n            if let Some(pwd) = option_env!("FIXED_PASSWORD") {\n                if !pwd.is_empty() {\n                    config.password = pwd.to_owned();\n                }\n            }\n        }\n'
content = content.replace(load_marker, load_marker + password_inject, 1)
```

The `\n` in the Python strings are escape sequences that become actual newlines. The injected Rust code uses 8-space indentation (matching the surrounding code style).

**IMPORTANT**: Use `content.replace(marker, marker + inject, 1)` with the `1` limit to replace only the first occurrence. The `load()` function appears exactly once in the file.

#### Why use Python instead of PowerShell

- Multiline replacements in PowerShell `-replace` are prone to YAML indentation errors in GitHub Actions workflow files
- The `shell: python` or `python3 << 'PYEOF'` approach avoids YAML block scalar indentation issues
- Python's `\n` escape sequences are more reliable for multiline strings in YAML than actual newlines

## Config patching — common pitfalls (debugging history)

These issues were discovered over several failed builds. Understanding them will save hours of debugging.

### Pitfall 1: Multiline strings in YAML break indentation

When using `shell: powershell` or `shell: bash` with `run: |` (YAML literal block scalar), ALL lines in the block must be indented at the SAME level or more. A single line with less indentation breaks the YAML parser.

**Symptoms**: Workflow fails in 0 seconds with "This run likely failed because of a workflow file issue."

**Fix**: Use `python3 << 'PYEOF'` (bash here-document) for complex multiline scripts. This avoids YAML block indentation issues entirely.

### Pitfall 2: `option_env!()` inserts function at wrong scope

When modifying `config.rs` to add a `default_password()` function, inserting it before `lazy_static::lazy_static!` with `.replace("lazy_static::lazy_static!", func + "lazy_static::lazy_static!")` causes issues:

- If `lazy_static::lazy_static!` appears 6 times in the file (it does), `.replace()` replaces ALL of them unless you pass `count=1`
- Even with `count=1`, the function might be inserted inside a submodule scope if the first `lazy_static!` is nested, making it inaccessible from the outer module

**Fix**: Inject the password check directly in `Config::load()` instead. The `load()` function is a single, unique method at module scope.

### Pitfall 3: `password` is NOT a regular settings key

The `Config` struct has `password: String` as a separate field, NOT in `options: HashMap<String, String>`. This means:

- Adding `password` to `DEFAULT_SETTINGS` does NOTHING
- The password must be set via `config.password = ...`
- The password comparison code checks this field directly against the incoming password
- Plaintext passwords ARE supported (the `password_is_empty_or_not_hashed` function confirms this)

### Pitfall 4: GitHub Actions private repo billing

GitHub Actions does NOT run on private repositories for free-tier accounts if payment is not configured. The error message is: "The job was not started because recent account payments have failed."

**Fix**: Set the repository to public before triggering builds.

### Pitfall 5: YAML `run: |` and PowerShell string escape interactions

When using PowerShell `-replace` with multiline replacement strings in a YAML `run: |` block:
- The replacement string's internal newlines must be at the correct indent level
- Using `@'...'@` PowerShell here-strings can help but create complex YAML
- Python is more reliable for complex string transformations

### Pitfall 6: `re.sub` vs `str.replace` in Python

- `re.sub()` uses regex patterns — escape `(` `)` `\` `.` etc. with `\`
- `str.replace()` uses literal strings — no escaping needed
- For SED-style replacements, `re.sub` is more flexible
- For inserting at a known marker, `str.replace(marker, new, 1)` is simpler and more reliable

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
- [ ] Config patching uses Python (not PowerShell multiline) to avoid YAML indent issues
- [ ] `content.replace(marker, new, 1)` has the `1` limit to avoid multiple insertions
- [ ] API server is set with key `api-server` (hyphenated) in DEFAULT_SETTINGS
- [ ] Fixed password uses `option_env!("FIXED_PASSWORD")` in `Config::load()`, NOT `DEFAULT_SETTINGS`
- [ ] Build command is `python build.py --portable --hwcodec --flutter --vram`
- [ ] Installer EXE is renamed and uploaded as artifact

## Operational notes

- If a build fails, check the vcpkg step first (most common failure point). Verify `libavutil/pixfmt.h` exists at `$VCPKG_ROOT/installed/x64-windows-static/include/`.
- The bridge generation job takes ~3 minutes. The Windows build job takes ~40-45 minutes. Total wall time: ~45-50 minutes.
- Node.js 20 deprecation warnings are harmless.
- Cache service errors ("Failed to save/restore") are harmless — they don't affect the build.
- If upstream RustDesk changes the config.rs format, update the replacement patterns accordingly.
- To avoid YAML multiline indentation issues, use `shell: bash` with `python3 << 'PYEOF'` for complex patching scripts.

## Expected output

Each workflow run produces:

- a `bridge-artifact` (intermediate, ~6 files)
- a `custom-rustdesk-windows` artifact containing the portable installer EXE (renamed)

## Complete working workflow

See `.github/workflows/build-rustdesk-windows.yml` in this repository for the current verified workflow. Copy it as a starting point, then replace these env vars with your values:

```yaml
env:
  ID_SERVER: "your-server.com"
  RELAY_SERVER: "your-server.com"
  API_SERVER: "http://your-server.com:21114"
  FIXED_PASSWORD: "YourPassword123"
  PUBLIC_KEY: ""
```

If you don't need API server or fixed password, leave them out or set to empty strings.
