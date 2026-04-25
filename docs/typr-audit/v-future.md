# Typr — Deferred Features (post v1)

Things explicitly left out of v1 to keep scope small. Pull from this list when
prioritizing v2 / v3 / Intercept Telehealth fork.

## v2 candidates (Chibi Tech internal)

### Stiki SSO + multi-tenant
- Auth via auth.chibitek.com using OAuth + loopback callback (system browser
  opens to login, Rust HTTP listener on `127.0.0.1:<port>/callback` catches
  the `#token=`).
- JWT in OS keychain.
- Rust validates RS256 locally with `CHIBITEK_JWT_PUBLIC_KEY` baked in,
  periodic `/api/me` revocation check.
- Vanilla-TS shim mirroring `@chibitek/auth`'s contract (`useUser`,
  `useOrg`, `hasRole`) — no React tax.
- Settings keyed by `(user_id, org_id)`. Org switcher in settings UI if user
  belongs to multiple orgs.
- CORS allowlist for `tauri://localhost` and `https://tauri.localhost` on
  `auth.chibitek.com/api/me`.
- Open question for Stiki: refresh token story for desktop (long-lived JWT
  vs refresh token). Without one, users get bumped mid-dictation when the
  JWT expires.

### Server-side audit log
- New `typr` Supabase project following the Mochii pattern.
- Table `dictation_events` (user_id, org_id, started_at, duration_ms,
  char_count, engine_used). Never the content.
- RLS by `org_id`.
- Rust posts events after each transcription via stiki-bridge Edge Function.
- Falls back to local queue if offline; flushes on reconnect.

### `@chibitek/ui` design tokens applied
- Import `tokens.css` into Typr's `style.css` entry.
- Replace hardcoded `--accent: #5e6ad2` etc with `var(--chibi-teal)`,
  `var(--chibi-amber)`, `var(--chibi-dark)`.
- CHIBITEK (teal) LABS (amber) wordmark in the sidebar header.

### FeedbackPopup
- Vanilla-TS port of `@chibitek/ui` FeedbackPopup (or wrap the React one if we
  bring React in).
- Posts to `feedback` table in the Typr Supabase.

### Cloud transcription proxy
- Users stop carrying their own Groq key. Backend at api.chibitek.com (or a
  Typr Supabase Edge Function) holds the key, proxies the audio.
- Per-tenant rate limits and usage tracking.
- Lets the org revoke a single user without rotating everyone's keys.

### Keystroke-typing instead of clipboard
- Drop clipboard entirely. Use `enigo`'s text-typing API on Windows; on macOS
  the existing `osascript keystroke` route is single-character but slow for
  long dictations — investigate a faster path.
- Removes the "clipboard manager scoops up transcripts" leak vector
  permanently. v1 does clear-after-paste as a stopgap.

### Code signing + signed updates
- macOS notarized build, Windows Authenticode signed.
- Tauri updater with signed update manifest. Without this, a compromised
  update channel can ship anything to a clinician's laptop.

### Memory-only audio
- Skip the temp WAV file. Pipe samples straight into whisper-cpp via stdin
  (it supports `-f -`), or use whisper-cpp as a library instead of a sidecar.
- Half the value of HIPAA hardening, useful even outside that context.

### Settings sync across devices
- Per-user settings (mic, hotkey, mode) sync to the Typr Supabase keyed by
  user_id. Org-level defaults override user prefs where set.
- Only meaningful once SSO is in.

## v3 / Intercept Telehealth fork

### HIPAA build profile (`hipaa` Cargo feature)
- Removes the cloud transcription path entirely (or swaps Groq for a
  BAA-signed provider — AWS Transcribe Medical, Deepgram enterprise, Azure
  Speech).
- Forces local Whisper.
- Disables all transcript-touching `println!`s at compile time.
- Keystroke-typing only, no clipboard fallback.
- Audit log mandatory.
- Stricter CSP.
- API keys from keychain only — no plaintext config fallback.

### Org-level controls (Intercept compliance side, not app code)
- BAAs with every vendor that touches PHI.
- Per-clinic risk assessment doc.
- Workforce training, access policies.
- Device requirements: FileVault/BitLocker, screen auto-lock, MDM.
- Incident response plan.
- Patient notice/consent if visits are dictated.

### Patient-facing safeguards
- Clinician must select patient/encounter before dictating (so audit log ties
  text to the right chart).
- Lockout after N minutes idle.
- Mandatory 2FA via Stiki.
