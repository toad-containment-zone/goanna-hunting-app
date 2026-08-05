# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, no-build, offline-first field data collection tool for cane toad hunters (Toad
Containment Zone / TCZ program) timing how long it takes to detect a goanna at a burrow, then
syncing the records to a KoboToolbox or ODK Central form.

- `goanna-hunting-app.html` — the entire application: HTML, CSS, and vanilla JS in one file, no
  dependencies, no bundler, no package.json.
- `goanna_burrow_detection.xlsx` — the companion XLSForm (sheets: `survey`, `choices`, `settings`)
  defining the KoboToolbox/ODK Central form that the app submits to. Field names in this workbook
  must line up with `defaultFieldMap` in the HTML's `<script>` (see below) — if one changes, the
  other needs to change too.

There is no build step, package manager, linter, or test suite. "Running" the app means opening
`goanna-hunting-app.html` directly in a browser (or serving it statically, e.g. `python3 -m http.server`).
Verify changes by opening it in a browser and exercising the flow — there is no automated test to lean on.

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
   `settings.team`) and kicks off a non-blocking `navigator.geolocation.getCurrentPosition()` call
   that fills in `sessionMeta.location` whenever it resolves (left blank if GPS fails — there's no
   manual override anymore). `currentMeta()` just returns `sessionMeta`, and it's attached to every
   record. `huntingActive` gates which card is visible (`#huntStartCard` vs `#timerCard`); team
   name is captured once in Settings instead, since it rarely changes between sessions.
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
   instance using `settings.fieldMap` (defaults in `defaultFieldMap`) to map internal field names
   to the target form's XML element names. `submitOne()` POSTs it as `xml_submission_file` in a
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

### Settings sheet field mapping

The "Advanced" section of the settings sheet (`.mapfield` inputs, `data-key` attributes) lets a
user remap internal field keys (`date`, `location`, `team`, `startTime`, `recordNumber`,
`durationSeconds`, `peopleSearching`, `digInspect`, `found`) to whatever XML element names their
specific Kobo/ODK form actually uses. This mapping is what `buildSubmissionXml` reads from —
changes to submission format should go through `settings.fieldMap`, not by hardcoding element
names.

### Styling

Plain CSS custom properties defined in `:root` (olive/dirt/rock/cream palette) — no CSS framework.
Layout is a single-column mobile app shell (`#app`, max-width 460px) with bottom-sheet modals
(`.overlay` / `.sheet`) for record entry and settings.

## Notes for editing

- Keep it a single self-contained HTML file unless explicitly asked to split it up — that's the
  deployment model (opened directly on a phone browser, no server required beyond the KoboToolbox/
  ODK Central endpoint it syncs to).
- The large base64 data URI in the `<header><img>` tag is an embedded logo image — leave it alone
  unless the logo itself needs to change.
- `crypto.randomUUID()` is used for record IDs (with a fallback for older browsers) and doubles as
  the OpenRosa `instanceID`, which servers use for submission deduplication — don't change ID
  generation without preserving uniqueness guarantees.
