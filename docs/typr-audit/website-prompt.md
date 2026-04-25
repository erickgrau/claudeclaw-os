# Website prompt for Mabel marketing + billing site

Paste this entire prompt into a fresh Claude Code session, v0, or whatever
you're using to scaffold the marketing site. It's self-contained.

---

## START PROMPT

Build a marketing and billing website for **Mabel**, a privacy-first voice
dictation app for macOS. The website's job is to convert visitors into Free
downloads and Pro subscribers, then act as the customer's account portal.

This site is a separate Next.js project deployed to Vercel. It does NOT
contain the dictation app source code. The dictation app is a Tauri/Rust
desktop binary distributed via DMG and Homebrew Cask, built in a different
repo. This site only handles marketing, payment, and license issuing.

### Stack

- **Next.js 15 (App Router) + TypeScript**
- **Tailwind CSS**
- **Stripe** for payments (or Lemon Squeezy / Polar as merchant of record if
  the maintainer prefers — flag the choice early so it's a deliberate decision)
- **Supabase** for Postgres + magic-link auth for the customer portal
- **Resend** for transactional email (license delivery, receipts)
- Deploy target: **Vercel**

### Positioning (use this voice everywhere)

Mabel is **forever local**. Your audio never leaves your Mac. Most other
voice AI tools, ambient AI assistants, and dictation apps stream your audio
to a cloud server, transcribe it there, and store the recording for "model
improvement" or session history. Mabel does not. Transcription happens
on-device. Audio is deleted the moment the transcription finishes. There
is no upload, no cloud account required to dictate, and no telemetry that
phones home with what you said.

Tone: confident, direct, no hype, no buzzwords, no AI clichés. Treat
visitors like adults who can read. No "🚀 Revolutionary AI". No "Welcome to
the future". No emojis in body copy.

Marketing line everywhere: **"Other voice AI tools send your audio to the
cloud. Mabel keeps it on your Mac."**

Do NOT name specific competitors anywhere in copy. Use phrases like "other
voice AI tools", "cloud-based dictation apps", "ambient AI assistants",
"other dictation tools". The point is positioning against the category, not
attacking individual companies.

### Tier structure

#### Free (forever, no usage caps)
- Unlimited local Whisper transcription
- Bring-your-own-key cloud Whisper if the user prefers
- Configurable global hotkey
- Toggle and push-to-talk modes
- Basic punctuation and capitalization cleanup
- Microphone selection
- Macos Keychain for any keys

#### Pro
- $10/month
- $99/year (save $21)
- $129 lifetime (one-time, never expires)

Pro adds:
- **AI cleanup** — a small language model runs on your Mac to fix grammar,
  remove filler words ("um", "uh", "like"), and tighten punctuation. Runs
  on-device, never sends text to a server.
- **App-aware formatting** — Mabel notices which app you're typing into and
  adjusts the cleanup style. Casual in chat apps, professional in email,
  code-comment style in editors.
- **Custom vocabulary** — names, acronyms, and technical terms get
  recognized correctly.
- **Voice commands** — "new line", "new paragraph", "delete that", "undo".
- **iCloud settings sync** — preferences and vocabulary sync across your
  Macs. Your transcripts never sync because they never leave your Mac.
- **Priority support**

The hard "we don't do this, ever" list, surfaced on the privacy page:
- No cloud-stored transcripts
- No audio retention beyond a single transcription
- No web dashboard with playback
- No usage analytics that phone home with what you said
- No training on user dictation
- No required account to use the free tier

### Pages required

| Path | Purpose |
|------|---------|
| `/` | Landing. Above-fold hook, the marketing line, a 60-second product video or animated demo, the "everything you don't have to worry about" privacy block, download Free CTA, See Pro CTA. |
| `/pricing` | Three tiers (Free, Pro Monthly, Pro Annual + Lifetime toggle). Stripe checkout. Comparison table to a generic "cloud dictation" category, no competitor names. |
| `/privacy-by-design` | The privacy story in detail. What we don't collect. What we don't store. What permissions Mabel asks for and why. The CISO-friendly version. |
| `/faq` | Common questions: how does the LLM run locally, what about Whisper accuracy, why Mac-only, can I use it for HIPAA-regulated work (answer: not officially supported in v2, see /enterprise for the future), refund policy, etc. |
| `/download` | Direct DMG download + Homebrew install instructions. Detect non-macOS visitors and tell them clearly. |
| `/account` | Magic-link login. Shows current subscription, license key, button to manage subscription via Stripe Customer Portal, button to deactivate license. |
| `/login` | Magic-link entry. |
| `/auth/callback` | Magic-link callback handler. |
| `/terms` | Plain-English ToS. Have a lawyer review before launch. |
| `/privacy-policy` | Plain-English privacy policy reflecting that we collect almost nothing. Have a lawyer review. |
| `/enterprise` | Holding page: "Multi-seat, BAA, and HIPAA-compatible deployments coming. Contact sales." Mailto link, no form yet. |

### API routes

| Route | Purpose |
|-------|---------|
| `POST /api/checkout` | Create a Stripe checkout session. Body: `{ sku: 'monthly' \| 'annual' \| 'lifetime', email }`. Returns checkout URL. |
| `POST /api/webhooks/stripe` | Stripe webhook handler. Handles `checkout.session.completed`, `invoice.paid`, `customer.subscription.deleted`, `customer.subscription.updated`. Issues / revokes licenses, sends license email via Resend. |
| `POST /api/license/verify` | Called by the Mabel desktop app. Body: `{ token: string }`. Returns `{ valid: bool, sku, expires_at \| null, email }`. Rate limited per IP. |
| `POST /api/license/deactivate` | Called by the Mabel desktop app on user request. Marks the device as deactivated, lets user reactivate elsewhere. |
| `GET /api/auth/session` | Magic-link session check for `/account`. |

### Database schema (Supabase Postgres)

```sql
create table customers (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  stripe_customer_id text unique,
  created_at timestamptz default now()
);

create table licenses (
  id uuid primary key default gen_random_uuid(),
  customer_id uuid references customers(id) on delete cascade,
  token text unique not null,
  sku text not null check (sku in ('monthly','annual','lifetime')),
  status text not null check (status in ('active','cancelled','past_due','expired')),
  current_period_end timestamptz,
  stripe_subscription_id text unique,
  device_fingerprint text,
  last_validated_at timestamptz,
  created_at timestamptz default now()
);

create index licenses_token_idx on licenses(token);
create index licenses_customer_idx on licenses(customer_id);
```

**Critical: this schema only stores billing data.** No transcripts, no audio
URLs, no usage logs of what was dictated. If a future feature wants to log
something, it must not include user content.

### Visual direction

- Restrained, professional, slightly serious. The audience is people who
  care about privacy.
- Type-led. Lots of whitespace. Big claims, small text.
- One accent color (suggest a deep indigo or muted teal — pick one and stick
  to it). Black/white otherwise.
- A monospace font for the hotkey demo and any code snippets.
- Hero animation: cursor in a generic mail/chat-style window, user holds the
  hotkey, words appear in the field with a tiny waveform overlay, then the
  waveform fades when transcription completes. No stock photos of people
  with headsets.
- A privacy-stat block on the landing page: "0 audio recordings stored. 0
  transcripts uploaded. 0 cloud accounts required. 100% on your Mac."
- A "permissions Mabel asks for" block on the privacy page, listing each
  permission and explaining what it does and what it does NOT do.

### Copy don'ts

- No em dashes anywhere in copy.
- No exclamation points.
- No "Welcome to", "Discover", "Unlock", "Revolutionary", "Game-changing".
- No emojis in headlines or body text. UI accents OK if subtle.
- No naming specific competitors. Always category language.
- No "AI-powered" as a feature claim. Be specific about what the AI does.

### Functional requirements

- Stripe checkout in test mode by default. Add a clear `STRIPE_LIVE_MODE`
  env var to flip live.
- Webhook signature verification on `/api/webhooks/stripe`.
- License token format: UUID v7 (sortable, time-encoded). Generated
  server-side, emailed to customer, never logged in plaintext to any server.
- License email via Resend with a copy-paste-able token, plus a deep link
  the desktop app can register: `mabel://activate?token=...`. The desktop
  app registers a URL scheme that opens it and pre-fills activation.
- 30-day offline grace: the desktop app can validate the license once and
  consider it valid for 30 days without re-checking. After that, Pro features
  lock until the next successful `/api/license/verify` call.
- Customer portal at `/account` shows: current subscription, plan, next
  renewal date, license key, "Manage subscription" button (Stripe Customer
  Portal redirect), "Deactivate this device" button.
- Refund policy: 14 days, no questions asked. Surface it on the pricing page
  and the FAQ.

### Email templates (via Resend)

- License delivery: token + activation deep link + getting-started link.
- Receipt: handled by Stripe automatically.
- Cancellation confirmation: thanks for trying it, you keep Pro until the
  end of the period, here's how to come back.
- Subscription renewed: short thank-you, no upsell, no marketing copy.

### Out of scope for v1 of the website

- Affiliate program
- Team / multi-seat plans (the `/enterprise` holding page covers this)
- A blog (add later if there's something to say)
- Localization (English only at launch)
- Marketing automation, drip campaigns, retargeting

### Deliverables

1. The Next.js codebase, deployable to Vercel.
2. A `.env.example` listing every env var needed (Stripe keys, Supabase URL +
   anon + service role keys, Resend key, marketing site URL, app URL scheme).
3. A `README.md` covering local dev, env setup, Stripe test mode, deploying
   to Vercel, and how to point the desktop app at the right license endpoint.
4. SQL migration files for the Supabase schema.
5. Stripe webhook setup instructions.

Stop and ask the maintainer before writing copy that names competitors,
makes claims about HIPAA/SOC 2/etc compliance, or commits to specific
performance numbers. Those need verification.

## END PROMPT
