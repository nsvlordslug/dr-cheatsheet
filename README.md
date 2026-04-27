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
- `assets/mindrot-logo.jpg` — source for icon generation
- `docs/` — design spec and implementation plan

## Regenerating icons

If the source logo changes, regenerate icons with ImageMagick. See Task 5 in `docs/superpowers/plans/2026-04-27-dr-cheatsheet-pwa.md` for exact commands.
