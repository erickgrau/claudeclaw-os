# Mabel v2: Pro tier implementation plan

Locked positioning: **forever local, privacy-first, free unlimited, Pro $10/mo
or $99/yr or $129 lifetime.** Other voice AI tools send your audio to the
cloud. Mabel keeps it on your Mac.

This plan covers what gets built in v2 to unlock paid features, in build
order. v1 stays as-is and remains the free tier.

---

## Tier feature matrix

### Free (forever, no caps)
- Unlimited local Whisper transcription
- BYOK cloud Whisper via the user's own Groq key
- Configurable global hotkey
- Toggle and push-to-talk modes
- Basic text cleanup (capitalize, end-punctuation)
- Mic selection
- Macos Keychain for any keys

### Pro ($10/mo · $99/yr · $129 lifetime)
- **Local LLM cleanup** — Qwen 2.5 3B or Gemma 3n via MLX, runs on-device,
  no cloud. Removes filler words, tightens grammar, fixes punctuation in
  context.
- **App-context formatting** — detects the focused app (Slack, Mail, Cursor,
  VS Code, Google Docs, etc.) and routes a different system prompt to the
  cleanup model. Requires macOS Accessibility permission.
- **Custom vocabulary** — names, acronyms, technical terms recognized
  correctly. Implemented as a Whisper initial prompt plus a post-process
  dictionary.
- **Voice commands** — "new line", "new paragraph", "delete that", "undo",
  basic editing without leaving voice mode.
- **Settings sync** — preferences and custom vocab sync across the user's
  Macs via iCloud (CloudKit). Transcripts NEVER sync.
- **Priority support** — direct email channel.

### Hard "no, never" list
- Cloud-stored transcripts
- Audio retention beyond the duration of a single transcription
- Web dashboard with playback
- Server-side analytics that phone home with usage data
- Training on user dictation
- Required cloud account for free tier

---

## Existing code can handle this

Quick architecture review of v1 against the v2 requirements. Conclusion:
**no major rework needed**. The license layer slots in alongside existing
modules.

| v2 need | v1 has | Gap |
|---------|--------|-----|
| Per-user license state | `Settings` struct in `settings.rs` | Add a `License` struct + `license.rs` module mirroring `secrets.rs` (keychain-backed token) |
| Phone-home for license validation | `reqwest` already a dep | New `license_client.rs` calling mabel-cloud's API |
| Pro feature gating | Tauri commands in `main.rs` | Wrap commands with a `require_pro()` check |
| Local LLM runtime | Nothing | New `cleanup_llm.rs` module, MLX dep |
| App-context detection | Nothing | New `focused_app.rs` using macOS Accessibility |
| Custom vocab | Nothing | New `vocabulary.rs`, plumb into Whisper prompt + post-process |
| Settings sync | Local `config.json` only | iCloud CloudKit container, mirror non-secret settings |

Total new modules: 5. No refactors of existing modules required.

---

## Build order

### 0. Apple Developer ID and notarization pipeline (1 day, $99/yr)

Prerequisite for everything paid. Without it, first-launch Gatekeeper
warnings tank conversion.

- Enroll in Apple Developer Program ($99/yr).
- Generate Developer ID Application certificate, Developer ID Installer
  certificate.
- Create app-specific password for `notarytool`.
- Add to Tauri config:
  ```json
  "macOS": {
    "signingIdentity": "Developer ID Application: Erick Grau (TEAM_ID)",
    "providerShortName": "TEAM_ID",
    "entitlements": "src-tauri/entitlements.plist"
  }
  ```
- Github Actions workflow that builds, signs, notarizes, and uploads the DMG
  to a release. Stores certs in Actions secrets.
- Test: download the notarized DMG, double-click, no Gatekeeper warning.

### 1. Local LLM cleanup (2-3 weeks)

The headline Pro feature. Biggest unlock, strongest differentiator.

- **Runtime choice: MLX**. Apple's framework, optimized for Apple Silicon
  unified memory. Alternative: `llama.cpp` via Rust bindings. MLX wins for
  speed on M-series and Apple's first-party tooling.
- **Model choice**: Qwen 2.5 3B Instruct quantized to 4-bit (~2GB on disk,
  ~3GB RAM at runtime). Fast (~500ms for typical dictation) and good enough
  at cleanup. Alternatives: Gemma 3n 4B if license terms work, Llama 3.2 3B.
- **Bundle vs download**: don't bundle in the DMG (bloats download to ~2.5GB).
  Download on first activation of Pro features, with progress UI similar to
  the existing Whisper model download.
- New `src-tauri/src/cleanup_llm.rs`. Public API: `pub async fn cleanup(text:
  &str, context: AppContext) -> Result<String, String>`.
- Integration point: `recorder.rs::stop_and_transcribe`. After raw Whisper
  output, if Pro and feature enabled, run cleanup_llm, otherwise fall through
  to the existing basic cleanup.
- System prompt template: "You are a writing assistant. Clean up dictated
  text. Fix grammar, remove filler words, add punctuation. Do not change
  meaning. Do not add content. Do not summarize. Output the cleaned text only."
- Settings UI: toggle "AI cleanup", with a hint that it requires the model
  download.

### 2. App-context detection (3-5 days)

- Add `entitlements.plist` requesting Accessibility.
- New `src-tauri/src/focused_app.rs`. Public API: `pub fn focused_app() ->
  Option<AppContext>`. Returns `{ name: String, bundle_id: String }`.
- Implementation: `NSWorkspace.shared.frontmostApplication` via `objc2` or
  shell out to `osascript` calling System Events. The shell-out is simpler;
  the FFI is faster. Start with osascript.
- Map bundle IDs to context strings: `com.tinyspeck.slackmacgap` → "casual
  chat", `com.apple.mail` → "professional email", `com.todesktop.230313mzl4w4u92`
  (Cursor) → "code comment", etc. Default fallback: "neutral".
- Plumb `AppContext` from `focused_app()` into `cleanup_llm::cleanup()` call
  site. The system prompt branches on context.
- Settings UI: add a "Permissions" section with an Accessibility check and a
  button to open System Settings if not granted.
- Graceful degradation: if permission denied, cleanup still runs with neutral
  context. Log a warning, don't error.

### 3. License module + mabel-cloud scaffold (1-2 weeks)

#### Mabel side (this repo)

- New `src-tauri/src/license.rs`. State: `Free | Pro { expires_at, sku }`.
  Stored in keychain via the existing `secrets.rs` pattern.
- New `src-tauri/src/license_client.rs`. Calls `https://mabel.app/api/license/verify`
  with the token. Caches the result for 7 days. Falls back to last-known-good
  for up to 30 days if the network is unreachable. After 30 days offline,
  Pro features lock until next successful validation.
- New Tauri commands: `activate_license(token: String)`, `deactivate_license()`,
  `license_status() -> License`.
- Settings UI: new "Account" section with "Activate Pro", "Manage subscription"
  (opens browser to mabel-cloud customer portal), "Deactivate".
- Pro gates: every Pro feature checks `state.license.is_pro()` and degrades
  gracefully if not.

#### mabel-cloud (new repo)

Next.js 15 app on Vercel. Single project, no microservices.

- Marketing site at `/` (landing, features, screenshots, comparison).
- Pricing page at `/pricing` with three SKUs: monthly $10, annual $99,
  lifetime $129.
- FAQ at `/faq`. Terms at `/terms`. Privacy at `/privacy`.
- Stripe checkout at `/api/checkout`.
- Stripe webhook at `/api/webhooks/stripe` handling `checkout.session.completed`,
  `invoice.paid`, `customer.subscription.deleted`, `customer.subscription.updated`.
- License issuing: on successful payment, generate a license token (UUID v7),
  store in Supabase tied to email + sku + expires_at, email it to the customer.
- License validation API at `/api/license/verify`. Input: token. Output:
  `{ valid: bool, sku, expires_at, email }`.
- Customer portal at `/account`. Magic-link login (Supabase Auth). Shows
  current sub, lets them upgrade/downgrade/cancel via Stripe Customer Portal.
- Stripe Tax enabled for VAT. Or use Lemon Squeezy / Polar as merchant of
  record to dodge tax compliance entirely. **Decide MoR vs Stripe direct
  during this sprint.**

#### Schema (Supabase)

```sql
create table customers (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  stripe_customer_id text unique,
  created_at timestamptz default now()
);

create table licenses (
  id uuid primary key default gen_random_uuid(),
  customer_id uuid references customers(id),
  token text unique not null,
  sku text not null check (sku in ('monthly','annual','lifetime')),
  status text not null check (status in ('active','cancelled','past_due','expired')),
  current_period_end timestamptz,
  stripe_subscription_id text unique,
  created_at timestamptz default now()
);
```

That's it. No transcripts, no audio, no usage analytics. Just billing.

### 4. Custom vocabulary + voice commands (1 week)

- New `src-tauri/src/vocabulary.rs`. Stores a list of `{ phrase, replacement
  }` pairs in the keychain-backed settings.
- Whisper integration: pass the vocab list as the `--prompt` argument to
  whisper-cpp.
- Post-process: regex replace common Whisper mishears using the vocab map.
- Voice commands: a small command grammar parsed from the raw transcript
  before LLM cleanup. "new line" → `\n`, "new paragraph" → `\n\n`, "delete
  that" → suppresses paste, "undo" → simulates Cmd+Z. Configurable on/off.
- Settings UI: new "Vocabulary" section. Add/remove pairs.

### 5. Settings sync via iCloud (3-5 days)

- Add iCloud entitlement (`com.apple.developer.icloud-container-identifiers`).
- CloudKit container for non-secret settings: hotkey, mode, vocab, mic
  preference, Pro toggles. Excludes Groq key (stays in Keychain, which iCloud
  Keychain can sync separately if user enabled it).
- New `src-tauri/src/sync.rs`. Watches local settings, pushes to CloudKit on
  save. Pulls on launch, merging local + remote.
- Conflict resolution: last-write-wins. Two devices touching the same setting
  is rare enough to not warrant CRDTs.

### 6. Public launch (1 week of marketing prep)

- Notarize the v2 build, upload DMG to Github Releases.
- Update Cask formula to point at the new release URL with new SHA256.
- Marketing site goes live.
- Stripe live mode enabled.
- Public announcement: HN, X, indie dev forums.

---

## Total timeline

Assuming part-time (~15 hrs/week):

| Phase | Time | Cumulative |
|-------|------|-----------|
| Apple Developer + signing | 1 day | week 1 |
| Local LLM cleanup | 2-3 weeks | week 4 |
| App-context detection | 3-5 days | week 5 |
| License module + mabel-cloud | 1-2 weeks | week 7 |
| Vocabulary + voice commands | 1 week | week 8 |
| iCloud sync | 3-5 days | week 9 |
| Launch prep | 1 week | week 10 |

10 weeks part-time, 5-6 weeks full-time.

---

## Open decisions to make during build

- **Stripe direct vs Lemon Squeezy / Polar (merchant of record).** MoR adds
  3-5% on top of Stripe's 2.9% + 30¢ but eliminates VAT/sales-tax compliance.
  For a solo founder, MoR usually wins. Decide during phase 3.
- **Bundle the LLM model or download on demand.** On-demand is friendlier to
  the DMG size; bundled gets Pro working immediately on activation. Probably
  on-demand with a fast first-time download UI.
- **License token format.** UUID v7 is fine. If you want offline-friendly
  validation later (signed JWT instead of round-trip), can add as a v3
  optimization.
- **Free tier piracy.** Won't matter. Free is unlimited; nobody pirates a
  product they can already use unlimited for free. The Pro license check is
  the only thing worth caring about, and even there: spend zero effort on
  obfuscation. Easy to pay, easy to validate, ignore the rest.
- **Cancellation grace.** When a subscription cancels, do Pro features stop
  immediately or at end-of-period? Recommendation: end-of-period. Less
  customer-hostile and Stripe handles the date.
