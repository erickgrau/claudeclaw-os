# Applying the Typr v1 security pass

This patch hardens the upstream `albertshiney/typr` repo with the five v1
changes described in `v1-security-pass.md`. It assumes you've already forked
upstream to `erickgrau/typr` (or similar) with no commits added yet.

## On your Mac

```bash
# 1. Clone your fork
git clone git@github.com:erickgrau/typr.git
cd typr

# 2. Make a branch
git checkout -b v1-security-pass

# 3. Pull this patch from the audit branch of claudeclaw-os
curl -L \
  https://raw.githubusercontent.com/erickgrau/claudeclaw-os/claude/audit-typr-repo-HlTnj/docs/typr-audit/v1-security-pass.patch \
  -o v1-security-pass.patch

# (or download via the GitHub UI from PR #1's file list)

# 4. Apply
git apply --check v1-security-pass.patch   # dry run, should say nothing
git apply v1-security-pass.patch
rm v1-security-pass.patch

# 5. Build
cd src-tauri
cargo build                                 # pulls keyring crate, drops enigo
cd ..
npm install
npm run tauri dev                           # smoke test

# 6. Commit and push
git add -A
git commit -m "v1 security pass

See docs in claudeclaw-os/docs/typr-audit/ for the full rationale."
git push -u origin v1-security-pass
```

## What changed

8 files modified, 1 new file. Full per-change rationale lives in
`v1-security-pass.md`.

```
src-tauri/Cargo.toml              | drop enigo, add keyring (apple-native)
src-tauri/src/lib.rs              | export new secrets module
src-tauri/src/main.rs             | strip transcript prints; handle Result from model_filename/url
src-tauri/src/paste.rs            | macOS-only; clear clipboard 150ms after paste
src-tauri/src/recorder.rs         | handle Result from model_filename
src-tauri/src/settings.rs         | split Settings (in-memory) from DiskSettings (no key); migrate
src-tauri/src/transcribe_local.rs | allowlist model_size; remove whisper-output print
src-tauri/tauri.conf.json         | tighten CSP from null to scoped policy
src-tauri/src/secrets.rs          | NEW: keychain wrapper for Groq API key
```

## Smoke tests after applying

- `cargo build` succeeds.
- `npm run tauri dev` launches; settings UI loads.
- Enter a Groq key in the cloud settings, save, restart the app — key persists.
- Quit the app, inspect `~/Library/Application Support/com.typr.app/config.json`
  — should NOT contain `groqApiKey`.
- Run `security find-generic-password -s com.typr.app -a groq_api_key` — the
  key is in the macOS keychain.
- Set `whisperModel` to `"../etc/passwd"` in `config.json` directly, restart —
  app should reject the model size, not write outside the app dir.
- Trigger dictation, paste lands, then check the clipboard manually — should
  be empty.
- Watch `Console.app` while dictating — no transcript text should appear.

## Rolling back

```bash
git checkout main
git branch -D v1-security-pass
```

The patch is a single commit; reverting is one `git revert`.
