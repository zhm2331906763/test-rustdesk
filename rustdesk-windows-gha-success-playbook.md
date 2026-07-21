# RustDesk Windows GitHub Actions Build Runbook

This document is a reusable build recipe for producing a customized Windows RustDesk installer in GitHub Actions.

It is intentionally written as a standalone execution guide. A new agent should be able to follow it without any prior conversation history.

## Purpose

Build a Windows RustDesk installer with these customizable inputs:

- ID server
- Relay server
- Public key

Leave the values blank in the template below and fill them in for the target deployment.

## Parameter template

Use these placeholders in the workflow or in your local notes:

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

- `RUST_VERSION = 1.75`
- `FLUTTER_VERSION = 3.24.5`
- `FLUTTER_RUST_BRIDGE_VERSION = 1.80.1`
- `CARGO_EXPAND_VERSION = 1.0.95`
- `LLVM_VERSION = 15.0.6`
- `VCPKG_COMMIT_ID = 120deac3062162151622ca4860575a33844ba10b`

The build command should remain:

```powershell
python .\build.py --portable --hwcodec --flutter --vram
```

## Workflow steps

The GitHub Actions job should execute in this order:

1. Checkout the build repository
2. Clone upstream RustDesk with submodules
3. Set up MSVC build tools
4. Install LLVM and Clang
5. Install Flutter
6. Replace the Flutter engine with the RustDesk custom engine
7. Patch the Flutter SDK for RustDesk compatibility
8. Install Rust
9. Install vcpkg and Windows dependencies
10. Generate flutter-rust-bridge files
11. Patch RustDesk config values
12. Build the installer
13. Rename the installer EXE
14. Upload the artifact

## Source patching strategy

Modify this file only:

- `rustdesk-src\libs\hbb_common\src\config.rs`

Use PowerShell string replacement inside the workflow. Do not introduce a separate patch file unless a future upstream change requires it.

### Replace exactly these items

1. `PROD_RENDEZVOUS_SERVER`
2. `DEFAULT_SETTINGS` entry for `relay-server`
3. `RS_PUB_KEY`

### Workflow snippet

Fill in the placeholder values before use:

```powershell
$file = "rustdesk-src\libs\hbb_common\src\config.rs"
$content = Get-Content $file -Raw
$content = $content -replace 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new\(""\.to_owned\(\)\);', 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("<ID_SERVER>".to_owned());'
$content = $content -replace 'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default\(\);', 'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = RwLock::new(HashMap::from([("relay-server".to_owned(), "<RELAY_SERVER>".to_owned())]));'
$content = $content -replace 'pub const RS_PUB_KEY: &str = ".*?";', 'pub const RS_PUB_KEY: &str = "<PUBLIC_KEY>";'
Set-Content -Path $file -Value $content -Encoding UTF8
```

## Why this approach

This method is preferred because it is:

- simple
- deterministic
- easy to reproduce
- less sensitive to patch-format drift
- easy for an automated agent to execute

It avoids:

- custom patch files that can become invalid
- runtime source injection
- extra cleanup helpers
- additional unrelated feature toggles

## Recommended workflow structure

Keep the workflow focused on build orchestration only.

Suggested step names:

- `Checkout this repo`
- `Checkout RustDesk source`
- `Setup MSVC build env`
- `Install LLVM and Clang`
- `Install Flutter`
- `Replace Flutter engine with RustDesk custom engine`
- `Patch Flutter SDK for RustDesk`
- `Setup Rust toolchain`
- `Setup vcpkg`
- `Install vcpkg dependencies`
- `Generate flutter-rust-bridge files`
- `Patch RustDesk custom config`
- `Build installer EXE`
- `Rename installer EXE`
- `Upload installer artifact`

## Validation checklist

Before pushing a change, verify:

- the workflow still clones upstream RustDesk with `--recurse-submodules`
- the workflow still runs `flutter pub get`
- the workflow still generates flutter-rust-bridge files
- the workflow only modifies `config.rs`
- the workflow still builds with `--portable --hwcodec --flutter --vram`
- the workflow still renames the output EXE

## Expected output

The GitHub Actions run should produce:

- a Windows installer EXE
- a renamed artifact such as `smwh-rustdesk.exe`

The upload step should point to the renamed file under `rustdesk-src`.

## Operational notes

- Keep the source modification step minimal.
- Do not add extra config keys unless they are part of a new requirement.
- If a build fails, first check whether the `config.rs` replacement still matches the upstream RustDesk source exactly.
- If upstream RustDesk changes the target line text, update the replacement string accordingly.

## Compact execution summary

If you need a short version to hand to another agent, use this:

1. Clone the build repository.
2. Let Actions clone upstream RustDesk.
3. Install MSVC, LLVM, Flutter, Rust, and vcpkg.
4. Generate flutter-rust-bridge files.
5. Fill in `ID_SERVER`, `RELAY_SERVER`, and `PUBLIC_KEY`.
6. Replace those values in `rustdesk-src\libs\hbb_common\src\config.rs`.
7. Build with `python .\build.py --portable --hwcodec --flutter --vram`.
8. Rename the EXE.
9. Upload the artifact.

