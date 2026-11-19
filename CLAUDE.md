# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Mexiko a Střední Amerika" (repo `Mexiko-a-Stredni-Amerika`) is a personal trip tracker for one overland route, Cancún → Buenos Aires. It is a static PWA hosted on GitHub Pages, with no backend, no build step, no package manager and no tests. The UI and all code comments are in **Czech**, and commit messages are Czech too (e.g. "Aktualizace trasy"). Keep it free and self-contained: no paid services, no API keys, no AI. The Python scripts use only the standard library. See [PRODUCT.md](PRODUCT.md) for users and principles, and [DESIGN.md](DESIGN.md) for the visual system (dark glassmorphism only, design tokens).

## Running

- App: serve the folder statically, e.g. `python -m http.server`, then open `http://localhost:8000/`. The service worker only registers over `https:`, so offline behaviour can be tested only on the deployed site.
- Alerts script: `python zpravy.py` rewrites `news.json` (it reads `state.json` for stops and the current position).
- Photo script: `python fotky.py` fetches Wikipedia/Commons photos for stops without a built-in photo into `photos/places/auto/` and records them in `photos/places/auto.json`.

## Architecture

**[index.html](index.html) is the entire app**: about 1600 lines of inline CSS and one `<script>`, split into sections marked `/* ================= NAME ================= */` (ZÁKLAD, STAV, GITHUB, FOTKY, ZPRÁVY, the per-tab views, UDÁLOSTI, START). Use these markers to navigate. Rendering is plain string templating plus `refresh()`. There is no framework.

**Data model and persistence (read the STAV and GITHUB sections together):**
- `S` is the public trip state (`stops`, `cur`, `diary`, `places`, `photos`). It is persisted to `state.json` in this repo. `SEED` inside index.html is only a fallback copy, used when `state.json` can't be loaded.
- `P` is private data (budget, tasks, bookings, packing list, notes). It is kept in `localStorage` (`priv_local`) and optionally synced to `private.json` in a separate private repo (`GH.pr`). A redacted subset (`sharedOut()`, without booking `ref`s or contacts) is published to `sdilene.json` so that visitors can view Rozpočet/Plán/Výbava read-only.
- **Edit mode** turns on when a GitHub token is stored in `localStorage` (`gh_token`, with `gh_cfg`). Saving writes files directly through the GitHub Contents API (`ghWrite`, with a retry on sha conflicts). So every "Aktualizace trasy" commit comes from the app, not from a developer. Visitors without a token get read-only mode and poll for new data every 3 minutes.
- The `dirty.{pub,priv,shr}` flags track which of the three files need saving.

**Automation (GitHub Actions; bots commit straight to `main`):**
- [.github/workflows/zpravy.yml](.github/workflows/zpravy.yml) runs daily at 05:00 UTC: `zpravy.py` pulls USGS, GDACS, NOAA NHC, Google News RSS, GDELT (often rate-limited, skipped after repeated 429s), and travel-advisory changes from UK FCDO and the US State Dept (`type: "adv"`), and writes route-aware alerts to `news.json`, including distance from the current stop, a source, and an `expires`/TTL per type. USGS and GDACS items also carry `lat`/`lon`, which the app uses to show the distance from the current stop and to hide far-away alerts on Přehled.
- [.github/workflows/fotky.yml](.github/workflows/fotky.yml) runs on every push touching `state.json` or `fotky.py`, and weekly. It fills in photos for new stops.
- Because bots and the app both commit to `main`, run `git pull --rebase` before pushing local changes.

**Cross-file couplings to keep in sync:**
- Built-in stop photos: `PHOTO_BY_STOP` / `PHOTO_CREDITS` in index.html ↔ `BUILTIN` in fotky.py ↔ files in `photos/places/` (+ `credits.json`). Keys are normalized names (lowercase, no diacritics).
- Country names are Czech (`Mexiko`, `Kostarika`, …). The mapping tables in zpravy.py (`ISO3`, `EN`, `NAMES`) and fotky.py (`COUNTRY_EN`) must cover every country used in stops. In index.html each such country also needs an `INFO` entry, `ZX` (capital, entry rules, budget, food, slang…), `WX` climate plus `WXLL` coordinates, and `CCUR`. Border crossings live in `HR` keyed `"A|B"`; `ZX_CHECKED` records when entry rules and fees were last reviewed.
- [sw.js](sw.js) is network-first with a cache fallback. Bump its cache name (`cesta-vN`) when the caching behaviour changes.

## Design work

The `.impeccable/` folder holds the design-tool state (`design.json`, surface brief). DESIGN.md documents the shipped design. When you change visuals, follow its tokens and the direction contract in `.impeccable/surfaces/index-html.md`: dark only, frosted glass over a blurred photo of the current stop, a bottom tab bar (a centred dock on desktop, never a side rail), and the pink-orange gradient reserved for "now" and the primary action.
