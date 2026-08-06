# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A no-build, offline-first field data collection tool for cane toad hunters (Toad Containment Zone
/ TCZ program) timing how long it takes to detect a goanna at a burrow, then syncing the records
to a KoboToolbox or ODK Central form. The app itself is one HTML file; a few small support files
sit alongside it so it's installable, works with zero signal, and has a clean URL when hosted on
GitHub Pages (see "PWA shell" below).

- `goanna-hunting-app.html` — the entire application: HTML, CSS, and vanilla JS in one file, no
  dependencies, no bundler, no package.json.
- `goanna_burrow_detection.xlsx` — the companion XLSForm (sheets: `survey`, `choices`, `settings`)
  defining the KoboToolbox/ODK Central form that the app submits to. Field names in this workbook
  must line up with `FIELD_MAP` in the HTML's `<script>` (see below) — if one changes, the
  other needs to change too.
- `manifest.json` / `sw.js` — web app manifest and service worker that make the app installable
  and precache the shell for offline use. See "PWA shell" below before touching these.
- `index.html` — a bare redirect to `goanna-hunting-app.html`, so the site's root URL (e.g. GitHub
  Pages' `https://<org>.github.io/<repo>/`) works without visitors needing to know or type the
  actual filename. It carries no app logic of its own.

There is no build step, package manager, linter, or test suite. "Running" the app means serving
the directory statically (e.g. `python3 -m http.server`) and opening `goanna-hunting-app.html` —
the service worker requires a real HTTP(S) origin (or `localhost`) and won't register over a bare
`file://` URL, so a static server is needed to test the offline/install behavior, not just the app
logic. Verify changes by opening it in a browser and exercising the flow — there is no automated
test to lean on.

## Architecture

Everything lives in one `<script>` block at the bottom of the HTML file, organized into clearly
commented sections (search for the `// ---------- ... ----------` markers):

1. **Settings & persistence** — `settings` (sync config) and `records` (search history) are the
   only two pieces of durable state, persisted to `localStorage` under `tcz_goanna_settings_v1`
   and `tcz_goanna_records_v1`. There is no backend database; the phone's local storage is the
   source of truth until a record syncs. The settings sheet accepts either a plain submission URL
   or an ODK Central "app user" code (the base64 zlib-compressed QR payload Central now issues
   instead of a bare token) — `resolveSubmissionUrl()` tells them apart and, for the latter,
   decodes it with the built-in `DecompressionStream('deflate')` (no library) to pull out
   `general.server_url` and append `/submission`. The raw pasted value is kept as
   `settings.appUserCode` so the field can be repopulated; the derived endpoint is
   `settings.submissionUrl`, which is what actually gets POSTed to.
2. **Hunting session** — there's no per-session metadata form. Tapping **Start hunting**
   (`startHunting()`) stamps `sessionMeta` (date/startTime from `new Date()`, `team` from
   `settings.team`, `nativeTitleArea` from `settings.nativeTitleArea`) and kicks off a non-blocking
   `navigator.geolocation.getCurrentPosition()` call that fills in `sessionMeta.location` whenever
   it resolves (left blank if GPS fails — there's no manual override anymore). `currentMeta()` just
   returns `sessionMeta`, and it's attached to every record. `huntingActive` gates which card is
   visible (`#huntStartCard` vs `#timerCard`); team name and native title area are captured once in
   Settings instead, since they rarely change between sessions.
   - `nativeTitleArea` is manually selected from a fixed dropdown (`#cfgNta`), not auto-detected —
     GPS in this app is best-effort and left blank on failure, so it can't be relied on as the sole
     source for a governance-relevant tag (see `../shared-taxonomy` for why NTA matters here). The
     dropdown's values (`NTA-KJ`, `NTA-NYKJ`, `NTA-NY`, `NTA-YWR`, `NTA-NML`, plus `none`) are a
     hand-copied snapshot of the *active* rows in `../shared-taxonomy/taxonomy/native_title_areas.csv`
     — that repo is the authoritative source; re-sync this dropdown (and the matching `nta_choices`
     list in `goanna_burrow_detection.xlsx`) if it changes. The submitted value is the NTA's mnemonic
     `nta_id`, not a display label, so it joins directly against that table downstream.
3. **Timer** — a simple start/stop stopwatch (`running`, `startTs`, `tick()` on a 250ms interval)
   that measures burrow time-to-detection. Stopping opens the record-entry bottom sheet.
4. **Record lifecycle** — each search produces a record object with a `status` of
   `pending → synced` or `failed`. Records are pushed to `records`, saved to `localStorage`
   immediately, then a sync is attempted. Tapping a non-synced record in the history list retries
   it individually via `submitOne()`. **Finish hunting** (`#finishHuntBtn`) ends the session: if a
   search is still running it simulates a `#bigBtn` click first (same stop → record-sheet flow as
   normal) via the `finishPending` flag, then calls `endHuntingSession()` once that sheet closes
   (saved or cancelled) to flip the UI back to `#huntStartCard`.
5. **XForms/OpenRosa submission** — `buildSubmissionXml(rec)` renders a record as an OpenRosa XML
   instance using the fixed `FIELD_MAP` constant to map internal field names to the target form's
   XML element names (there's no user-facing way to remap these — they must match
   `goanna_burrow_detection.xlsx`, see below). `submitOne()` POSTs it as `xml_submission_file` in a
   `multipart/form-data` body, matching the ODK Central / KoboToolbox submission API contract.
   Auth is carried in the URL itself (see the app-user-code note above) — there's no separate
   username/password.
6. **Sync orchestration** — `trySyncAll()` walks all non-synced records and submits them
   sequentially; it's triggered on save, on manual "Sync now", on the `online` browser event, and
   on a 30s interval. `updateStatusBar()` reflects online/offline state and pending count in the
   header status bar.
7. **QR code scanning** — jsQR v1.4.0 is vendored inline (minified, own `<script>` block, Apache
   2.0, attributed at the top of the block) rather than using the native `BarcodeDetector` API,
   because Safari/iOS doesn't implement `BarcodeDetector` and field crews use a mix of Android and
   iPhone devices. `startScan()`/`scanTick()` grab camera frames onto a canvas and feed them to the
   global `jsQR()` to populate the Central app user code field.

### PWA shell (`manifest.json`, `sw.js`, `index.html`)

The app is registered as a service worker (`navigator.serviceWorker.register('sw.js')`, near the
top of the main `<script>` block) so it opens instantly and works offline even on first launch
after being added to the home screen — important since it's used in areas with no signal. `sw.js`
precaches `./`, `index.html`, `goanna-hunting-app.html`, and `manifest.json` on install, then
serves from cache on every load while updating the cache in the background (stale-while-revalidate)
so it stays current whenever there *is* signal without ever blocking on the network. `index.html`
is precached too (not just `goanna-hunting-app.html`) so the bare root URL still resolves offline
for anyone who bookmarked or installed from `/` instead of the direct filename. This only covers
the app shell loading — the actual record sync to ODK Central still needs connectivity, which is
what the existing pending/sync-queue UI already handles.

`manifest.json`'s icon (and the `<link rel="apple-touch-icon">` in the HTML `<head>`, for iOS,
which doesn't use the manifest for "Add to Home Screen") is a generated 512×512 PNG: the header
logo centered on an olive (`#838851`) square background, embedded as a `data:` URI so no extra
image files are needed on disk. Regenerate it (via Pillow or similar) if the logo changes; don't
hand-edit the base64.

Bump `CACHE_NAME` in `sw.js` whenever you change what needs to be cached — it's also how old
caches get cleaned up on `activate`.

### Styling

Plain CSS custom properties defined in `:root` (olive/dirt/rock/cream palette) — no CSS framework.
Layout is a single-column mobile app shell (`#app`, max-width 460px) with bottom-sheet modals
(`.overlay` / `.sheet`) for record entry and settings.

## Notes for editing

- Keep the *application* in the single `goanna-hunting-app.html` file — no dependencies, no
  bundler — unless explicitly asked to split it up. `manifest.json` and `sw.js` are the one
  sanctioned exception: browsers refuse to register a service worker from a `blob:`/`data:` URL,
  so they can't be inlined and have to ship as real files next to the HTML. Everything else
  (jsQR, the logo, the generated app icon) stays embedded inline as before.
- The large base64 data URI in the `<header><img>` tag is an embedded logo image — leave it alone
  unless the logo itself needs to change.
- `crypto.randomUUID()` is used for record IDs (with a fallback for older browsers) and doubles as
  the OpenRosa `instanceID`, which servers use for submission deduplication — don't change ID
  generation without preserving uniqueness guarantees.
