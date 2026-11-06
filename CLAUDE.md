# יומן אימונים — עותק שני (workout-app-2)

A separate copy of the workout log (repo `home-workout`, v1.61) for a second person: own URL, own D1 database, own password, own data.
He is 28, advanced, trains 6 days a week in a full gym (same gym as the original user: no leg-curl machine, cables available), goal: muscle mass.

- **Live URL:** https://workout-app-2.plants-app.workers.dev/ (Cloudflare Worker `workout-app-2`)
- **Plan:** push / pull / legs twice a week — `d1` דחיפה א, `d2` משיכה א, `d3` רגליים א, `d4` דחיפה ב, `d5` משיכה ב, `d6` רגליים ב. `PLANNED_PER_WEEK = 6`.
  Weekly working sets roughly: chest 17, back 20, shoulders 20, quads 10, hamstrings 12, biceps 9 direct, triceps 6 direct + pressing, calves 8.
  Shared ids across days (one progression): `leg_curl_standing_cable` (d3, d6), `abs_cable` (d3, d6). `SAME_MOVEMENT` is empty (dumbbell and cable lateral raises are apart).
- **Removed from the original** (they belonged to the first user's plan and data): the plan migrations in `applyBarDefaults` (only the bar defaults are left), `repairV157Merges` on start, the automatic training break of 29.10–4.11 and the pinned "next workout" `d1`.

## Layout
- `public/` — static files served as-is (Workers static assets). **Every file here is part of the live app; a deploy replaces the whole set, so never delete one.**
  - `index.html` — the single-page app, Hebrew, RTL, dark theme (accent `#d7ff3f`)
  - `storage.js` (defines `window.storage`), `foods.js`, `sw.js` (network-first cache), `manifest.webmanifest`, `icons/`
- `src/worker.js` — handles `/api/*` only: password login (secret `APP_PASSWORD`, set in the dashboard) and a key/value sync API on D1.
- `wrangler.jsonc` — worker name `workout-app-2`, assets folder, D1 binding `DB` → database `workout-2`. The worker creates its tables (`kv`, `usage`, `backups`) by itself on first use.

## Where the data lives
`storage.js` is local-first: every write goes to the browser's localStorage and is pushed in the background to D1 (`/api/kv`); on load the newer copy per key wins.
So the data is on the phone **and** in D1 table `kv` (key, value, updated). A device that is not logged in shows only its own local data.
Still remind the user to back up (in-app export) before any change that touches storage keys.

## Data model (never rename these storage keys — they hold the user's real history)
All persistence goes through `window.storage.get/set/list(key, false)`, values are JSON strings.
- `day-data:<dayKey>` → `{ history:[{date, exercises:{exId:{sets:[{reps,weight,warmup}], note}}}], last:{exId:rec}, prs:{exId:{weight,reps,e1rm,date}} }`
- `set-log:<dayKey>` → newest-first array of individually saved sets `{date,time,exerciseId,exerciseName,setIndex,reps,weight,warmup}` (max 300)
- `day-overrides`, `rotation-last`, `bodyweight-log`, `nutrition-goal`, `my-foods`, `nut-recent`, `nut-meals`, `deload-start`, `note-drafts` (v1.39: `{"<dayKey>|<exId>": {date, text}}`, exercise notes typed before "סיום"), `weekly-seen` (v1.46: ISO date of the Saturday whose weekly summary was auto-shown), `waist-log` (v1.52: newest-first `[{date:'D.M.YYYY', cm}]`), `next-workout` (v1.61: `{day, set}` or `{}`)
- Weights are stored in kg; `kgToDisplayUnit` / `displayUnitToKg` convert for display. PRs compare by estimated 1RM (`est1RM`, `prEst`).

## Working on it
- Test in Playwright with a stub `storage.js` that keeps `window.storage` in memory (note: the real `get` throws when a key is missing), served over `python3 -m http.server` from a copy of `public/`.
- Check at phone width (390px). UI text is Hebrew.
- Deploy: push to `main`. Cloudflare Workers Builds (connected to this repo) runs `npx wrangler deploy`. Do **not** use "Edit code" → Deploy in the dashboard — it knows only the worker, not the assets.
- The service worker is network-first, so a new version shows on the next load with signal.
- **Version:** bump `APP_VERSION` and `APP_VERSION_DATE` in `public/index.html` on every change that is deployed (shown at the bottom of "עוד"), and add a line below. Tell the user the new version number so they can confirm the phone loaded it.

## History of changes
- 2026-11-06 · v1.0: copy of home-workout v1.61 with the 6-day push/pull/legs plan above; worker creates `kv` and `usage` tables if missing.
