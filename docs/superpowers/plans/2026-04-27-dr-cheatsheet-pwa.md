# DaVinci Resolve Cheatsheet PWA Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `sal_cheatsheet.html` into an installable, offline-capable PWA hosted on GitHub Pages, accessed at `https://nsvlordslug.github.io/dr-cheatsheet/`.

**Architecture:** Static site with PWA manifest + cache-first service worker. No build step. Public repo on GitHub Pages, branch `main`, root path. The existing self-contained HTML stays largely as-is; we add manifest, service worker, icons, and a few HTML hooks.

**Tech Stack:** HTML / CSS / JS (no framework), PWA Web App Manifest, Service Worker API, ImageMagick (icon generation), Git, GitHub CLI (`gh`), Python (one-off local HTTP server for verification).

---

## File Structure

```
dr-cheatsheet/
├── index.html                  # the cheatsheet (modified copy of sal_cheatsheet.html)
├── manifest.json               # PWA install metadata
├── service-worker.js           # offline cache + update strategy
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   ├── icon-maskable-512.png   # safe-area padded for Android adaptive icons
│   └── apple-touch.png         # 180x180 fallback
├── assets/
│   └── mindrot-logo.jpg        # source for icon generation (JPG; ImageMagick converts to PNG)
├── README.md
├── .gitignore
└── docs/superpowers/
    ├── specs/2026-04-27-dr-cheatsheet-design.md
    └── plans/2026-04-27-dr-cheatsheet-pwa.md
```

All paths below are relative to `C:\Users\cereb\Desktop\Claude projects\dr-cheatsheet\` unless absolute.

---

## Task 1: Initialize project structure and git

**Files:**
- Create: `.gitignore`
- Existing: `docs/superpowers/specs/2026-04-27-dr-cheatsheet-design.md`
- Existing: `docs/superpowers/plans/2026-04-27-dr-cheatsheet-pwa.md`

- [ ] **Step 1: Create .gitignore**

Write to `.gitignore`:

```
# OS
.DS_Store
Thumbs.db
desktop.ini

# Editor
.vscode/
.idea/
*.swp

# Local
*.log
```

- [ ] **Step 2: Initialize git with main as default branch**

Run from `dr-cheatsheet/`:

```bash
git init -b main
```

Expected output: `Initialized empty Git repository in .../dr-cheatsheet/.git/`

- [ ] **Step 3: First commit (spec + plan + .gitignore)**

```bash
git add .gitignore docs/
git commit -m "Initial: design spec, implementation plan, and gitignore"
```

Expected: 1 commit with 3 files (`.gitignore`, the spec, the plan).

---

## Task 2: Create index.html with PWA hooks

**Files:**
- Source: `C:\Users\cereb\Downloads\sal_cheatsheet.html`
- Create: `index.html`

- [ ] **Step 1: Copy source HTML to project as index.html**

PowerShell:

```powershell
Copy-Item "C:\Users\cereb\Downloads\sal_cheatsheet.html" "index.html"
```

- [ ] **Step 2: Update `<title>`**

Find: `<title>Sal's Edit Cheatsheet</title>`
Replace with: `<title>DaVinci Resolve Cheatsheet</title>`

- [ ] **Step 3: Update `apple-mobile-web-app-title` meta**

Find: `<meta name="apple-mobile-web-app-title" content="Sal's Cheatsheet">`
Replace with: `<meta name="apple-mobile-web-app-title" content="DR Cheatsheet">`

- [ ] **Step 4: Update header `<h1>`**

Find: `<h1>Sal's Edit Cheatsheet</h1>`
Replace with: `<h1>DaVinci Resolve Cheatsheet</h1>`

- [ ] **Step 5: Add manifest + apple-touch-icon links**

Insert just before the closing `</head>`:

```html
<link rel="manifest" href="./manifest.json">
<link rel="apple-touch-icon" href="./icons/apple-touch.png">
```

- [ ] **Step 6: Add service worker registration**

The existing `<script>` block at end of `<body>` defines `showTab` and `tog`. Append a second `<script>` block immediately after it:

```html
<script>
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('./service-worker.js')
      .then(function(reg) { console.log('SW registered:', reg.scope); })
      .catch(function(err) { console.log('SW registration failed:', err); });
  });
}
</script>
```

- [ ] **Step 7: Sanity-check HTML opens**

Open `index.html` directly in any browser. Expected: page loads, all 5 tabs work, no JS errors in console.

Note: the service worker WILL fail to register over `file://` — that's expected. We verify SW registration over HTTP in Task 9.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add index.html with PWA hooks (manifest link, SW registration)"
```

---

## Task 3: Verify ImageMagick availability

**Files:** none

- [ ] **Step 1: Check if ImageMagick is installed**

```bash
magick -version
```

Expected: prints version starting with `Version: ImageMagick 7.x`. If you get "command not found" or similar, proceed to Step 2. Otherwise skip to Task 4.

- [ ] **Step 2: Install via winget**

```powershell
winget install ImageMagick.ImageMagick
```

Then **close and reopen the shell** so the new PATH takes effect.

- [ ] **Step 3: Re-verify**

```bash
magick -version
```

Expected: prints version. If still missing, troubleshoot before continuing — icon generation needs this.

---

## Task 4: Source the Mindrot Studios logo

**Files:**
- Existing: `assets/mindrot-logo.jpg` (already in place — user provided as JPG)

- [ ] **Step 1: Confirm logo file is in place**

The Mindrot Studios brain logo was already saved to `assets/mindrot-logo.jpg` before plan execution. Verify it exists:

```bash
ls "assets/mindrot-logo.jpg"
```

Expected: file listed. If missing, ask the user to save the Mindrot logo image to that path before continuing.

- [ ] **Step 2: Verify the file exists and check dimensions**

```bash
magick identify "assets/mindrot-logo.jpg"
```

Expected output similar to: `assets/mindrot-logo.jpg JPEG 1024x1024 1024x1024+0+0 8-bit sRGB ...`

Note the dimensions (W × H). The icon-generation crops in Task 5 are tuned for ~1024x1024 AI-generated images; if your dimensions differ significantly, adjust the crop offsets proportionally.

- [ ] **Step 3: Commit the source asset**

```bash
git add assets/mindrot-logo.jpg
git commit -m "Add Mindrot Studios brain logo as icon source (JPG)"
```

---

## Task 5: Generate icon files

**Files:**
- Create: `icons/icon-512.png`
- Create: `icons/icon-192.png`
- Create: `icons/apple-touch.png`
- Create: `icons/icon-maskable-512.png`

**Approach:** in the source logo, the brain occupies roughly the left 45% of the image, vertically centered around the upper-middle. Crop a square region around the brain, then resize for each icon size. The maskable variant adds safe-area padding so Android's adaptive icon shapes (circle, squircle, etc.) don't crop the brain.

- [ ] **Step 1: Create the icons directory**

```powershell
New-Item -ItemType Directory -Path "icons" -Force
```

- [ ] **Step 2: Determine crop region**

For a 1024×1024 source image, use:
- Crop size: `460x460`
- X offset: `50` (5% from left)
- Y offset: `200` (~20% from top)

If your source is significantly different (e.g., 2048×2048), scale these values proportionally. The goal is a square crop centered on the brain with a small amount of glow around it and **no Mindrot Studios text visible**.

- [ ] **Step 3: Generate icon-512.png**

```bash
magick "assets/mindrot-logo.jpg" -crop 460x460+50+200 +repage -resize 512x512 -background "#0C0C0F" -gravity center -extent 512x512 "icons/icon-512.png"
```

The `-extent` with the dark background guarantees a 512×512 canvas even if resize undershoots a pixel.

- [ ] **Step 4: Visually inspect icon-512.png**

Open `icons/icon-512.png`. Verify:
- Brain is centered
- No "Mindrot Studios" text visible
- Dark background fills any empty space
- Brain looks recognizable

If the crop is off (text visible, brain off-center), adjust the offsets in Step 3 and regenerate. Iterate until it looks right before continuing.

- [ ] **Step 5: Generate icon-192.png from the verified 512**

```bash
magick "icons/icon-512.png" -resize 192x192 "icons/icon-192.png"
```

- [ ] **Step 6: Generate apple-touch.png (180×180)**

```bash
magick "icons/icon-512.png" -resize 180x180 "icons/apple-touch.png"
```

- [ ] **Step 7: Generate maskable variant (brain inside 80% safe zone)**

Android adaptive icons crop ~10% off each side. The maskable icon must keep the brain inside an inner 410×410 region of a 512×512 canvas.

```bash
magick "icons/icon-512.png" -resize 410x410 -background "#0C0C0F" -gravity center -extent 512x512 "icons/icon-maskable-512.png"
```

- [ ] **Step 8: Verify all 4 icons exist with correct dimensions**

```bash
magick identify "icons/icon-192.png" "icons/icon-512.png" "icons/icon-maskable-512.png" "icons/apple-touch.png"
```

Expected: 4 lines, with dimensions `192x192`, `512x512`, `512x512`, `180x180` respectively.

- [ ] **Step 9: Commit icons**

```bash
git add icons/
git commit -m "Add PWA icons generated from Mindrot brain logo"
```

---

## Task 6: Write manifest.json

**Files:**
- Create: `manifest.json`

- [ ] **Step 1: Write manifest.json**

```json
{
  "name": "DaVinci Resolve Cheatsheet",
  "short_name": "DR Cheatsheet",
  "description": "Personal DaVinci Resolve + TourBox cheatsheet",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#0C0C0F",
  "theme_color": "#0C0C0F",
  "icons": [
    { "src": "./icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "./icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "./icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

- [ ] **Step 2: Validate JSON parses**

```bash
python -c "import json; json.load(open('manifest.json')); print('OK')"
```

Expected: prints `OK`. If it errors, fix the JSON syntax.

- [ ] **Step 3: Commit**

```bash
git add manifest.json
git commit -m "Add PWA web app manifest"
```

---

## Task 7: Write service-worker.js

**Files:**
- Create: `service-worker.js`

- [ ] **Step 1: Write service-worker.js**

```javascript
const CACHE_VERSION = 'v1';
const CACHE_NAME = `dr-cheatsheet-${CACHE_VERSION}`;
const PRECACHE_URLS = [
  './',
  './index.html',
  './manifest.json',
  './icons/icon-192.png',
  './icons/icon-512.png',
  './icons/icon-maskable-512.png',
  './icons/apple-touch.png'
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) { return cache.addAll(PRECACHE_URLS); })
      .then(function() { return self.skipWaiting(); })
  );
});

self.addEventListener('activate', function(event) {
  event.waitUntil(
    caches.keys().then(function(keys) {
      return Promise.all(
        keys.filter(function(key) { return key !== CACHE_NAME; })
            .map(function(key) { return caches.delete(key); })
      );
    }).then(function() { return self.clients.claim(); })
  );
});

self.addEventListener('fetch', function(event) {
  if (event.request.method !== 'GET') return;
  event.respondWith(
    caches.match(event.request).then(function(cached) {
      if (cached) return cached;
      return fetch(event.request).then(function(response) {
        if (response && response.status === 200 && response.type === 'basic') {
          var clone = response.clone();
          caches.open(CACHE_NAME).then(function(cache) {
            cache.put(event.request, clone);
          });
        }
        return response;
      });
    })
  );
});
```

- [ ] **Step 2: Verify JS syntax**

If Node is installed:

```bash
node --check service-worker.js
```

Expected: no output (means syntax OK). If Node isn't installed, skip — the browser will surface syntax errors at registration time.

- [ ] **Step 3: Commit**

```bash
git add service-worker.js
git commit -m "Add service worker with cache-first offline strategy"
```

---

## Task 8: Write README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README.md**

```markdown
# DR Cheatsheet

Personal DaVinci Resolve + TourBox Elite cheatsheet, installed as a PWA on Android.

**Live URL:** https://nsvlordslug.github.io/dr-cheatsheet/

## How to update

Pick whichever fits the change:

1. **Through Claude** — best for substantive changes. Describe what you learned in a Claude Code session; Claude will edit the HTML, bump `CACHE_VERSION` in the service worker, commit, and push.
2. **GitHub web UI** — quickest for small tweaks from anywhere (including phone). Edit `index.html` on github.com, then edit `service-worker.js` to bump `CACHE_VERSION`, then commit.
3. **Local edit + git** — full control on the dev machine.

**Always bump `CACHE_VERSION`** in `service-worker.js` (e.g. `v1` → `v2`) on any content change. Otherwise the phone keeps serving the cached version forever.

## How updates reach the phone

When the phone is online and the app is opened, the browser silently checks for a new service worker. If `CACHE_VERSION` changed, the new SW takes over on the next page load and re-caches everything. No reinstall needed.

## Project files

- `index.html` — the cheatsheet
- `manifest.json` — PWA install metadata
- `service-worker.js` — offline cache (bump `CACHE_VERSION` on content changes)
- `icons/` — app icons (192, 512, maskable, apple-touch)
- `assets/mindrot-logo.png` — source for icon generation
- `docs/` — design spec and implementation plan

## Regenerating icons

If the source logo changes, regenerate icons with ImageMagick. See Task 5 in `docs/superpowers/plans/2026-04-27-dr-cheatsheet-pwa.md` for exact commands.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "Add README with update workflows"
```

---

## Task 9: Verify the full PWA locally with an HTTP server

Service workers don't run on `file://` URLs — only `http(s)://` or `localhost`. We use a local HTTP server to verify everything before deploying.

**Files:** none (verification only)

- [ ] **Step 1: Start a local HTTP server**

From `dr-cheatsheet/`:

```bash
python -m http.server 8000
```

Expected output: `Serving HTTP on :: port 8000 (http://[::]:8000/) ...`

Leave this running. Open a new terminal for any other commands.

(If Python isn't installed, use `npx http-server -p 8000` or any equivalent.)

- [ ] **Step 2: Open the app in Chrome**

Open `http://localhost:8000` in Chrome (desktop). Expected: cheatsheet loads, all 5 tabs (TourBox, Color, Pipeline, Fairlight, Tips) work, no JS errors in console.

- [ ] **Step 3: Verify service worker registered**

Chrome DevTools → Application → Service Workers.

Expected: `service-worker.js` listed, status "activated and is running".

If registration failed, fix the error and recommit before continuing.

- [ ] **Step 4: Verify manifest loaded correctly**

Chrome DevTools → Application → Manifest.

Expected: name "DaVinci Resolve Cheatsheet", short_name "DR Cheatsheet", icons listed (3), no warnings or errors. The brain icon should preview correctly.

- [ ] **Step 5: Verify offline cache populated**

Chrome DevTools → Application → Cache Storage → expand `dr-cheatsheet-v1`.

Expected entries:
- `/` (or `http://localhost:8000/`)
- `index.html`
- `manifest.json`
- All 4 icons in `icons/`

- [ ] **Step 6: Test offline mode**

Chrome DevTools → Network → toggle "Offline". Reload the page (Ctrl+R).

Expected: page loads fully, all tabs work, no errors. This proves the offline experience your phone will have.

- [ ] **Step 7: Stop the local server**

In the server terminal, press Ctrl+C.

If any verification step in this task failed, fix the underlying issue (re-edit a file, re-commit), restart the server, and retest. Do not proceed to deployment until all 6 verification checks pass.

---

## Task 10: Push to GitHub

**Files:** none (remote setup)

- [ ] **Step 1: Verify gh CLI is authenticated**

```bash
gh auth status
```

Expected: includes `Logged in to github.com as nsvlordslug`. If not, run `gh auth login` and complete the flow.

- [ ] **Step 2: Create the public repo and add origin remote**

From `dr-cheatsheet/`:

```bash
gh repo create nsvlordslug/dr-cheatsheet --public --source=. --remote=origin --description "Personal DaVinci Resolve + TourBox cheatsheet PWA"
```

Expected output: `✓ Created repository nsvlordslug/dr-cheatsheet on GitHub` and `✓ Added remote https://github.com/nsvlordslug/dr-cheatsheet.git`.

- [ ] **Step 3: Push main branch and set upstream**

```bash
git push -u origin main
```

Expected: pushes all commits, sets `origin/main` as upstream.

- [ ] **Step 4: Confirm the repo is visible on GitHub**

```bash
gh repo view nsvlordslug/dr-cheatsheet --web
```

Or just open `https://github.com/nsvlordslug/dr-cheatsheet` in a browser. Expected: repo with all files visible, public.

---

## Task 11: Enable GitHub Pages

**Files:** none

- [ ] **Step 1: Enable Pages on main / root**

```bash
gh api -X POST "repos/nsvlordslug/dr-cheatsheet/pages" -f "source[branch]=main" -f "source[path]=/"
```

Expected: HTTP 201 with JSON describing the Pages site. If you get 409 with "Pages already enabled", that's fine — continue.

- [ ] **Step 2: Get the Pages URL**

```bash
gh api "repos/nsvlordslug/dr-cheatsheet/pages" --jq .html_url
```

Expected output: `https://nsvlordslug.github.io/dr-cheatsheet/`

- [ ] **Step 3: Wait for the first build to finish**

Poll the build status:

```bash
gh api "repos/nsvlordslug/dr-cheatsheet/pages/builds/latest" --jq .status
```

Expected progression: `queued` → `building` → `built`. The first build usually takes 1–2 minutes.

Re-run the command every 20 seconds until it returns `built`. If it returns `errored`, run `gh api "repos/nsvlordslug/dr-cheatsheet/pages/builds/latest"` to see the full error message.

- [ ] **Step 4: Verify the live URL responds with HTTP 200**

```bash
curl -sI https://nsvlordslug.github.io/dr-cheatsheet/ | head -1
```

Expected: `HTTP/2 200`

- [ ] **Step 5: Verify the manifest is reachable**

```bash
curl -sI https://nsvlordslug.github.io/dr-cheatsheet/manifest.json | head -1
```

Expected: `HTTP/2 200`

- [ ] **Step 6: Verify the service worker file is reachable**

```bash
curl -sI https://nsvlordslug.github.io/dr-cheatsheet/service-worker.js | head -1
```

Expected: `HTTP/2 200`

---

## Task 12: Phone install + offline verification (USER TASK)

This is a manual user task on the Android phone. Cannot be automated. Have the user perform these steps and confirm each one before checking it off.

- [ ] **Step 1: Open the URL in Chrome on Android**

URL: `https://nsvlordslug.github.io/dr-cheatsheet/`

Expected: cheatsheet loads with the dark theme.

- [ ] **Step 2: Add to Home Screen**

Chrome menu (⋮) → "Install app" or "Add to Home screen" → confirm.

Expected: a new icon appears on the home screen showing the brain, labeled "DR Cheatsheet".

If the install option doesn't appear: scroll the page once, wait ~5 seconds (Chrome's installability check needs the SW to fully install first), reopen the menu. If still missing, check DevTools remote debugging (`chrome://inspect`) for SW errors.

- [ ] **Step 3: Open the app from home screen**

Tap the brain icon. Expected: opens full-screen, no Chrome browser bar, looks like a real app.

- [ ] **Step 4: Verify offline mode**

Enable airplane mode. Force-close the app from recents. Reopen from home screen.

Expected: app opens fully and is fully usable. All 5 tabs work, all SVG button visuals render, all card expansions work.

- [ ] **Step 5: Disable airplane mode**

Done. The cheatsheet is installed, offline-capable, and ready to update via any of the three workflows in `README.md`.

---

## Self-review checklist

After completing all tasks above, verify:

- [ ] All 8 design questions (Q1–Q8) are addressed in the implementation
- [ ] No "TODO" or "TBD" markers in any committed file
- [ ] `CACHE_VERSION = 'v1'` defined in `service-worker.js`
- [ ] All 4 icon files exist in `icons/` with correct dimensions
- [ ] Live URL returns HTTP 200 in `curl -sI`
- [ ] Service worker successfully registers on the phone (Step 12.2 install option appeared)
- [ ] Offline mode works on the phone (Step 12.4 succeeded)
- [ ] README.md documents the three update workflows and the `CACHE_VERSION` rule
