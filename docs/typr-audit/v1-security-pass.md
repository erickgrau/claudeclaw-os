# Typr v1 Security Pass (Chibi Tech build)

Scope: take vanilla Typr (https://github.com/albertshiney/typr) and make it
meaningfully more secure without adding architectural complexity. No auth, no
backend, no multi-tenant. Just close the easy leaks.

Anything bigger lives in `v-future.md`.

## Changes

### 1. Stop logging transcripts to stdout
- `src-tauri/src/transcribe_local.rs:39` — remove `println!("[Typr] Whisper output: {}", text);`
- `src-tauri/src/main.rs:214` — remove `println!("[Typr] Transcription: {}", result);`
- Keep state-change logs (recording started/stopped, errors). Just don't log
  the content.

### 2. Move Groq API key out of plaintext config.json
- Add `keyring = "3"` (or current) to `src-tauri/Cargo.toml`.
- New module `src-tauri/src/secrets.rs` wrapping macOS Keychain / Windows
  Credential Manager via the `keyring` crate. Service name `com.typr.app`,
  account `groq_api_key`.
- `Settings` keeps `groq_api_key: String` in the in-memory struct (so the UI
  hook doesn't change), but `Settings::save` strips it before writing JSON and
  pushes the value to keychain. `Settings::load` reads JSON, then pulls the key
  from keychain back into the struct.
- Existing config.json with a plaintext key: on first load, migrate it to
  keychain and rewrite config.json without the field.

### 3. Clear clipboard after paste
- `src-tauri/src/paste.rs` — after the paste keystroke fires, sleep ~150ms then
  `clipboard.clear()`. Slight risk: if the paste fails to land, the user can't
  Cmd+V again, but they can re-dictate. Acceptable tradeoff.
- Alternative (better, more work): drop the clipboard route entirely and use
  enigo's text-typing API to type the characters directly. Slower for long
  dictations but no clipboard leak. Defer the swap to v-future unless we hit
  problems with the clear-after-paste approach.

### 4. Allowlist `model_size`
- `src-tauri/src/transcribe_local.rs` — `model_filename` and
  `model_download_url` currently `format!()` the user-supplied string into a
  path and URL. Add a guard:
  ```rust
  fn validate_model_size(s: &str) -> Result<&str, String> {
      match s {
          "small" | "medium" => Ok(s),
          _ => Err(format!("Invalid model size: {}", s)),
      }
  }
  ```
- Call it at the entry points (`download_model`, `check_model_downloaded`,
  `transcribe_local`). Closes the path-traversal/URL-hijack edge case if
  config.json is ever written by something other than the UI.

### 5. Tighten CSP
- `src-tauri/tauri.conf.json` — replace `"csp": null` with something
  restrictive. Starting point:
  ```
  "csp": "default-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self' https://api.groq.com https://huggingface.co"
  ```
- Test that the dashboard, Google Fonts, model downloads, and Groq calls all
  still work. Tighten further if any directive turns out to be unused.

## Out of scope for v1 (parked in v-future.md)

- Stiki SSO + multi-tenant
- Cloud transcription proxy
- Per-tenant audit log
- Memory-only audio (no temp WAV on disk)
- HIPAA build profile for Intercept
- Code signing + signed update channel
- Settings sync across devices

## Test plan

- Build on macOS, run through dictation flow end-to-end.
- Verify transcripts no longer appear in `Console.app` or stdout.
- Verify Groq key survives app restart (read back from keychain), and
  config.json no longer contains it.
- Verify clipboard is empty after a dictation paste lands.
- Verify CSP doesn't break model download, Groq call, or font loading.
- Try poking `whisperModel: "../etc/passwd"` into config.json directly and
  confirm the app rejects it instead of writing/reading outside the app dir.
