# Applying the Mabel patches

The audit branch ships a series of patches that build on each other. Apply
them in order.

## Patches

| Patch | What it does |
|-------|-------------|
| `v1-security-pass.patch` | Strip stdout transcript leaks. Move Groq key to OS keychain. Clear clipboard after paste. Allowlist model_size. Tighten CSP. macOS-only. |
| `v1.1-rename-mabel-and-configurable-hotkey.patch` | Rename Typr → Mabel everywhere user-facing. Make the global hotkey rebindable from the UI. |
| `v1.2-mabel-redesign.patch` | Full app redesign: new sidebar with Pro upsell items, dashboard with local stats, settings modal, help modal, fresh README, Persian cat placeholder icon. |

## Quickstart for a fresh fork

```bash
# 1. Clone your fork (run OUTSIDE any existing typr/mabel dir)
git clone git@github.com:erickgrau/typr.git mabel
cd mabel
git checkout -b mabel-v1.2

# 2. Build the whisper.cpp sidecar (one-time, ~30 sec on M-series)
cd ..
git clone --depth 1 https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
cmake -B build
cmake --build build --config Release -j
mkdir -p ../mabel/src-tauri/binaries
cp build/bin/whisper-cli ../mabel/src-tauri/binaries/whisper-cpp-aarch64-apple-darwin
chmod +x ../mabel/src-tauri/binaries/whisper-cpp-aarch64-apple-darwin
cd ../mabel

# 3. Pull and apply all three patches
BASE=https://raw.githubusercontent.com/erickgrau/claudeclaw-os/claude/audit-typr-repo-HlTnj/docs/typr-audit
curl -L $BASE/v1-security-pass.patch -o p1.patch
curl -L $BASE/v1.1-rename-mabel-and-configurable-hotkey.patch -o p2.patch
curl -L $BASE/v1.2-mabel-redesign.patch -o p3.patch
git apply --check p1.patch p2.patch p3.patch       # dry run
git apply p1.patch p2.patch p3.patch
rm p1.patch p2.patch p3.patch

# 4. Generate the icon files from the source SVG (one-time)
#    Pre-req: ImageMagick or rsvg-convert. brew install imagemagick if needed.
#    OR skip this step and use the existing Tauri icons until you swap in a real Mabel photo.
magick src-tauri/icons/mabel-source.svg -resize 1024x1024 /tmp/mabel-icon.png
npm run tauri icon /tmp/mabel-icon.png

# 5. Build and smoke test in dev
cd src-tauri && cargo build && cd ..
npm install
npm run tauri dev

# 6. Build the release .app + dmg
npm run tauri build
# Output:
#   src-tauri/target/release/bundle/macos/Mabel.app
#   src-tauri/target/release/bundle/dmg/Mabel_0.1.0_aarch64.dmg

# 7. Commit and push
git add -A
git commit -m "mabel v1.2: redesign with dashboard, sidebar, modals, stats"
git push -u origin mabel-v1.2
```

## If you've already applied v1 + v1.1 (incremental update)

```bash
cd ~/Antigravity/typr   # or wherever your local checkout is
curl -L https://raw.githubusercontent.com/erickgrau/claudeclaw-os/claude/audit-typr-repo-HlTnj/docs/typr-audit/v1.2-mabel-redesign.patch -o v1.2.patch
git apply --check v1.2.patch
git apply v1.2.patch
rm v1.2.patch
npm run tauri dev   # smoke test
```

## Real Mabel icon (replace the placeholder)

The included `src-tauri/icons/mabel-source.svg` is a stylized Persian cat
based on Mabel's coloring (silver-grey, white chest, pink nose, green eyes).
It works as a placeholder. For the real icon:

1. Pick a flattering photo of Mabel from her Instagram.
2. Run it through a generation tool (Recraft, ChatGPT image gen, Midjourney)
   with a prompt like:

   > Stylize this cat photo as a macOS app icon. Squircle background with
   > soft 3D depth, centered subject, friendly Persian cat face with pink
   > nose and green eyes, professional macOS Big Sur icon style, 1024x1024.
   > No text, no watermarks.

3. Save the output as a 1024x1024 PNG.
4. Run `npm run tauri icon path/to/mabel.png`. Tauri generates all sizes
   into `src-tauri/icons/` automatically.
5. `npm run tauri build` again.

## Smoke tests after applying v1.2

- `cargo build` succeeds.
- `npm run tauri dev` launches with the new dashboard visible at startup.
- Sidebar shows: Home, then Dictionary / Snippets / Style / Transforms /
  Scratchpad with PRO badges and greyed icons.
- Footer shows Activate Pro / Settings / Help.
- Click any Pro item → upsell page appears in the right pane with "Activate
  Pro" CTA.
- Click Activate Pro anywhere → modal opens with v2 feature list and
  pricing ($10/mo / $99/yr / $129 lifetime).
- Click Settings → modal opens with internal nav: General / Engine / Privacy
  / Account / About.
- Settings → General: change mic, recording mode, hotkey. Hotkey rebinds
  live (click kbd, press combo, see it accepted).
- Settings → Engine: download a Whisper model, switch between Local and Cloud.
- Settings → Privacy: shows the privacy claim, "Open in Finder" opens the app
  data dir, "Reset stats" zeroes the counters.
- Press your global hotkey from another app → recording starts, transcribes,
  pastes, and the dashboard's Today / All-time / Streak stats bump up.
- Quit and relaunch → stats persist.

## Rolling back

```bash
# Revert just v1.2:
git apply -R v1.2.patch

# Or per commit if committed:
git revert HEAD
```
