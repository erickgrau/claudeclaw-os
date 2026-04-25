# Applying the Mabel patches (formerly Typr)

The audit branch ships a series of patches that build on each other. Apply
them in order. Each patch's full rationale lives next to it in this directory.

## Prerequisites

A working Rust + Node + Tauri toolchain on macOS, and the whisper.cpp sidecar
binary in place (see "Whisper sidecar" below).

## Quickstart

```bash
# 1. Clone your fork (run this OUTSIDE any existing typr dir)
git clone git@github.com:erickgrau/typr.git
cd typr
git checkout -b mabel-v1.1

# 2. Build the whisper.cpp sidecar (one-time, ~30 sec on M-series)
#    The Rust build refuses to start without this binary present.
cd ..
git clone --depth 1 https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
cmake -B build
cmake --build build --config Release -j
mkdir -p ../typr/src-tauri/binaries
cp build/bin/whisper-cli ../typr/src-tauri/binaries/whisper-cpp-aarch64-apple-darwin
chmod +x ../typr/src-tauri/binaries/whisper-cpp-aarch64-apple-darwin
cd ../typr

# 3. Pull and apply v1 (security pass) then v1.1 (rename + configurable hotkey)
BASE=https://raw.githubusercontent.com/erickgrau/claudeclaw-os/claude/audit-typr-repo-HlTnj/docs/typr-audit
curl -L $BASE/v1-security-pass.patch -o p1.patch
curl -L $BASE/v1.1-rename-mabel-and-configurable-hotkey.patch -o p2.patch
git apply --check p1.patch p2.patch          # dry run
git apply p1.patch p2.patch
rm p1.patch p2.patch

# 4. Build and smoke test in dev mode
cd src-tauri && cargo build && cd ..
npm install
npm run tauri dev

# 5. Build a release .app you can drop in /Applications
npm run tauri build
# Output: src-tauri/target/release/bundle/macos/Mabel.app
#         src-tauri/target/release/bundle/dmg/Mabel_0.1.0_aarch64.dmg

# 6. Commit and push
git add -A
git commit -m "mabel v1.1: security pass, rename, configurable hotkey"
git push -u origin mabel-v1.1
```

## Patches in this directory

| Patch | What it does |
|-------|-------------|
| `v1-security-pass.patch` | Strip stdout transcript leaks. Move Groq key to OS keychain. Clear clipboard after paste. Allowlist model_size. Tighten CSP. macOS-only (drop Windows enigo path). |
| `v1.1-rename-mabel-and-configurable-hotkey.patch` | Rename Typr → Mabel everywhere user-facing (productName, identifier, window title, log prefix, package names, keychain service). Make the global hotkey rebindable: click the kbd, press a combo, the binding updates live without a restart. |

## Whisper sidecar

Tauri sidecars are platform-specific binaries the app shells out to. The repo
gitignores `src-tauri/binaries/`, so the binary needs to be built once. The
Rust build looks for the file with the target triple suffix, which on Apple
Silicon is `whisper-cpp-aarch64-apple-darwin`.

If you forget this step, the build fails at:

```
resource path `binaries/whisper-cpp-aarch64-apple-darwin` doesn't exist
```

## Smoke tests after applying

### v1 (security pass)
- `cargo build` succeeds.
- `npm run tauri dev` launches; settings UI loads.
- Enter a Groq key in cloud settings, save, restart — key persists.
- Quit app, inspect `~/Library/Application Support/com.mabel.app/config.json` —
  should NOT contain `groqApiKey`.
- `security find-generic-password -s com.mabel.app -a groq_api_key` — key is
  in the macOS keychain.
- Set `whisperModel` to `"../etc/passwd"` in `config.json` directly, restart —
  app rejects, no write outside app dir.
- Trigger dictation, paste lands; clipboard should be empty after.
- Watch `Console.app` while dictating — no transcript text appears.

### v1.1 (rename + hotkey)
- Window title reads "Mabel".
- App identifier in About is `com.mabel.app`.
- Log prefix in `Console.app` reads `[Mabel]`.
- Click the hotkey kbd → it shows "Press keys..." and pulses.
- Press Esc → reverts to current binding, no change.
- Press Cmd+Shift+M (or whatever) → binding updates immediately.
- Trigger the new hotkey from any app → recording starts. Old hotkey no longer
  fires.
- Restart the app → new hotkey persists.
- Try a single bare letter (e.g. just "A") → frontend ignores it; you have to
  add a modifier.

## Rolling back

```bash
# Single revert per patch:
git revert HEAD                              # undo v1.1
git revert HEAD                              # undo v1

# Or nuke the branch entirely:
git checkout main
git branch -D mabel-v1.1
```
