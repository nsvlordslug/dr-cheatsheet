# DaVinci Resolve Cheatsheet PWA — Design

**Date:** 2026-04-27
**Status:** Design approved (awaiting spec review)

## Problem

Sal has an HTML cheatsheet for DaVinci Resolve workflows on a TourBox Elite. The file currently lives at `C:\Users\cereb\Downloads\sal_cheatsheet.html`. Opening it on Android via `file://` is broken in two ways:

1. Chrome on Android does not offer "Add to Home Screen" for `file://` URLs.
2. When Chrome closes, the tab is lost and Sal has to re-navigate the local file system to reopen it.

He wants the cheatsheet to:

- Be installable as a real-feeling app on his Android home screen.
- Work fully offline once installed (no internet at usage time).
- Be updatable as he learns more DaVinci Resolve, without reinstalling.
- Use a custom icon based on the brain from the Mindrot Studios logo.
- Not require a Play Store listing — purely personal use.

## Approach

Convert the static HTML into a **Progressive Web App** hosted on **GitHub Pages** from a public repo. Add a `manifest.json` and a service worker so Chrome treats the URL as installable and caches every asset for offline use.

This is the smallest viable change: the existing HTML is already self-contained (CSS + SVG inlined), so we add three new files (manifest, service worker, icons) and update a few lines in the HTML.

## Naming

| Item | Value |
|---|---|
| GitHub repo | `nsvlordslug/dr-cheatsheet` (public) |
| Local path | `C:\Users\cereb\Desktop\Claude projects\dr-cheatsheet\` |
| Live URL | `https://nsvlordslug.github.io/dr-cheatsheet/` |
| HTML `<title>` | DaVinci Resolve Cheatsheet |
| Header `<h1>` | DaVinci Resolve Cheatsheet |
| PWA `name` | DaVinci Resolve Cheatsheet |
| PWA `short_name` (home screen label) | DR Cheatsheet |

## File structure

```
dr-cheatsheet/
├── index.html                  # retitled cheatsheet
├── manifest.json               # PWA install metadata
├── service-worker.js           # offline cache + update strategy
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   ├── icon-maskable-512.png   # with ~10% safe-area padding
│   └── apple-touch.png         # 180x180 fallback
├── README.md                   # editing/update notes
├── .gitignore
└── docs/
    └── superpowers/specs/
        └── 2026-04-27-dr-cheatsheet-design.md   # this file
```

## Components

### `index.html`

Changes from current `sal_cheatsheet.html`:

- `<title>` → "DaVinci Resolve Cheatsheet"
- Header `<h1>` → "DaVinci Resolve Cheatsheet"
- `apple-mobile-web-app-title` meta → "DR Cheatsheet"
- Add `<link rel="manifest" href="./manifest.json">` in `<head>`
- Add `<link rel="apple-touch-icon" href="./icons/apple-touch.png">`
- Add a small `<script>` at end of body that registers the service worker
- The existing "Add to Home Screen" instructions in the Tips tab stay — they're still accurate

### `manifest.json`

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

### `service-worker.js`

Cache-first strategy with a versioned cache.

- `CACHE_VERSION` constant (string like `'v1'`) — bumped on every content release.
- `install` event: opens the cache, fetches every URL in a hardcoded list (HTML, manifest, all icons), stores them.
- `activate` event: deletes any cache whose name doesn't match the current `CACHE_VERSION`.
- `fetch` event: cache-first, network fallback. On network success, the response is also written back to the cache so the latest version is preserved.

**Update mechanics:** when the service worker file itself changes (which happens whenever `CACHE_VERSION` is bumped), the browser detects the byte-diff and triggers a reinstall. The new SW takes over on the next page load. From the user's perspective: open the app while online → next time you open it, it's the new version.

### Icons

**Source:** the green/purple brain in the Mindrot Studios logo (user-supplied image, see Q4 in decision log). Just the brain — no "DR" text overlay (Q5).

**Generation:**

- Crop the brain region from the logo image.
- Resize to 192px and 512px square PNGs.
- For the maskable variant, add ~10% padding on all sides so Android adaptive icon shapes (circle, squircle, rounded square — varies by launcher) don't crop the brain.
- Apple touch icon: 180px square.

**Tooling:** ImageMagick is the simplest option on Windows. The implementation plan must verify availability and install if missing before generating icons.

### `README.md`

Brief notes for future-Sal:

- What this project is.
- How to update — the three workflows below.
- **Reminder: bump `CACHE_VERSION` in `service-worker.js` on every content change**, otherwise the phone keeps serving the cached version forever.

## Update workflows

Three options, all available — Sal picks per change (Q7):

1. **Through Claude (preferred for substantive changes)** — describe what was learned in a session; Claude edits the HTML, bumps `CACHE_VERSION`, commits, pushes.
2. **GitHub web UI (small tweaks from anywhere, including phone)** — edit the file in browser, bump `CACHE_VERSION` in the SW, commit. Tedious for big edits.
3. **Local edit + git** — full control on the dev machine.

Phone picks up the new version automatically the next time the app is opened while online.

## Setup steps (one-time)

1. Create local folder `C:\Users\cereb\Desktop\Claude projects\dr-cheatsheet\`.
2. Copy `sal_cheatsheet.html` → `dr-cheatsheet/index.html` and apply title/header/meta changes.
3. Verify ImageMagick (or alternative) is available; generate icon PNGs from Mindrot brain.
4. Write `manifest.json`, `service-worker.js`, `README.md`, `.gitignore`.
5. `git init`, create public GitHub repo `nsvlordslug/dr-cheatsheet` via `gh`, push.
6. Enable GitHub Pages on `main` branch / root via `gh` or web UI.
7. Wait ~1 minute for first Pages build; verify URL responds.
8. On Android: open URL in Chrome → menu → "Add to Home Screen" → confirm icon + "DR Cheatsheet" label.
9. **Verify offline:** enable airplane mode, reopen app from home screen, confirm it loads and is fully functional.

## Out of scope

- Custom subdomain (`cheatsheet.mindrotstudios.com` — Sal declined; the github.io URL is fine for personal use).
- "DR" text overlay on the icon (Q5 — declined).
- Native APK / TWA wrapper (Q3 — PWA model is sufficient).
- Play Store listing.
- Auth or access control (public repo, public Pages site, no sensitive content).
- Any visual redesign of the cheatsheet itself — the HTML stays as-is except for the title/header rename.

## Decision log

| # | Question | Decision | Reason |
|---|---|---|---|
| Q1 | Why doesn't current setup work? | `file://` URLs can't be installed as PWAs in Chrome | Browser restriction |
| Q2 | Where to host? | GitHub Pages | Cloudflare Pages declined per user's experience with reliability |
| Q3 | Offline model? | Install once online, offline forever | Standard PWA mechanics; meets the requirement |
| Q4 | Icon source? | Brain from Mindrot Studios logo | User-supplied |
| Q5 | "DR" placement on icon? | None — brain only | Cleaner at small icon sizes; home screen label provides identification |
| Q6 | Hosting confirmation | GitHub Pages | Confirmed |
| Q7 | Update workflow? | All three (Claude / web / local) | Maximum flexibility, no extra cost |
| Q8 | Repo visibility? | Public | No sensitive content; required for free GitHub Pages |
