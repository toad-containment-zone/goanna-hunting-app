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
2. **No hunting-session concept** — each search stands alone; there is no Start/Finish-hunting
   wrapper and no `session_id` field. `#timerCard` is always visible. Stopping a search
   (`$('bigBtn')`'s stop branch, via `fetchSearchLocation()`) kicks off a non-blocking GPS
   capture for *that search's* fix, then opens the record-entry sheet. `$('saveBtn')` builds each
   record's `meta` object fresh at save time: `date`/`startTime` derived from that search's
   `startTs`, `locationLat`/`locationLon` from the just-resolved GPS fix, and `team` read directly
   from `settings` (still captured once in Settings, since it rarely changes between searches even
   though location now does).
   - **GPS capture is a sampler, not a single fix.** `getCurrentPosition()` returns the first fix
     the platform has, which on a cold start is often a coarse wifi/cell fix tens to hundreds of
     metres out; GNSS keeps refining for 30-60s after. So `sampleLocation()` (next to
     `getLocation()`, ported from `../toad-monitoring-app`) runs `watchPosition` for up to
     `GPS_SAMPLE_MAX_MS` (45s), keeps the smallest-`accuracy` reading, and stops early once it's
     within `GPS_TARGET_ACCURACY_M` (10m). `#gpsStatus` in the record sheet shows a live "±N m"
     readout (`updateGpsStatus()`). `getLocation()` itself — the old single-fix wrapper — is now
     used only for the passive page-load permission/hardware check. The sampler is aborted
     (`stopGpsSample()`) when the sheet closes without a save (backdrop tap, "reset timer",
     "continue timing") and after a successful save.
   - `coords.accuracy` drives the live readout and a **soft gate** only — it is not stored on the
     record or in the submission (`location_lat`/`location_lon` are unchanged). If the best fix
     was worse than `GPS_GATE_WARN_M` (25m), `$('saveBtn')` asks once via `confirm()` before
     saving (session-suppressed by `gpsGateWarned`, mirroring `geoWarned`).
   - **Each search also records a GPS track**, not just the stop point — a fix at timer start, one
     per minute while running (`trackTimer`, alongside the beep interval), and the stop-sampler's
     fix as the last point. Points accumulate in the module-level `trackPoints` array
     (`{ ms, lat, lon, alt, acc }`, `ms` = elapsed since `startTs`, **lat/lon stored
     full-precision** — rounding happens only at serialisation), reset on a fresh start and on
     "reset timer", carried across a "continue timing" pause (leaving a gap). Each vertex is a
     single `getLocation()` one-shot with a loose `opts` (`timeout: 25000`, `maximumAge: 30000` —
     a per-minute breadcrumb can wait, and this cuts the silent timeouts that happen under tree
     canopy / on a cold start); a failed fix just leaves a gap. At save, `trackPoints` is copied
     to `rec.track` and serialised by `buildSubmissionXml()` into two form fields:
     `search_track` (ODK `geotrace`: `"lat lon alt acc"` per point, lat/lon rounded to **6 dp**
     — ~0.11 m, matching ODK Collect — `;`-joined, **no trailing `;`**) and `search_track_seconds`
     (`;`-joined elapsed-seconds, index-aligned to `search_track` — same array mapped twice — since
     geotrace carries no timestamps; needed downstream to spot a forgotten timer + drive to the
     next site by segment speed). `prepareTrack()` (was `dedupeTrack()`) does the 6 dp rounding,
     drops only **exact** consecutive duplicates (full-precision jitter is always kept, so any
     real movement survives), and pops the last point once if it equals the first (geotrace needs
     last ≠ first). If fewer than 2 points remain (a search with 0–1 successful fixes, or a
     genuinely stationary one — a spec-valid geotrace is then impossible), **both fields go out
     empty**; the record still carries `location_lat`/`location_lon`, so the downstream ETL must
     fall back to those for the search-start location when `search_track` is empty.
     `location_lat`/`location_lon` are unchanged (still the accuracy-gated stop fix, 5 dp); when
     `search_track` is present, downstream uses its first point as the search-start location. A
     `#trackStatus` line on the timer card and a line per record in the history list both show the
     post-`prepareTrack` point count (`"stationary so far"` / `"not recorded (stationary search)"`
     when < 2), not the raw fix count.
   - Since a blank location has real consequences here, the app also proactively surfaces GPS
     trouble rather than staying silent about it: `getLocation()` is called once on page load and
     alerts (via `geoErrorMessage()`) if it fails, and `$('saveBtn')` alerts again — with a real
     choice, via `confirm()` — the first time a record would save with no location. Declining
     retries the fix and keeps the sheet open instead of saving; accepting sets the in-memory
     `geoWarned` flag, which silences further prompts for the rest of the session (a fresh page
     load re-arms both checks — `geoWarned` is intentionally not persisted).
   - **Native title area is no longer collected.** The app used to capture it once in Settings
     (`#cfgNta`) and stamp `settings.nativeTitleArea` onto every record, backed by an `nta_choices`
     list in `goanna_burrow_detection.xlsx` hand-copied from
     `../shared-taxonomy/taxonomy/native_title_areas.csv`. That was dropped (mirroring
     `../toad-monitoring-app`'s `cce170b`) once the intent settled on deriving NTA downstream in
     the `data-warehousing` ETL from each record's geolocation via spatial join against NTA
     boundaries — the reporting layer becomes the single source of truth. Removed end-to-end: the
     `<select>`, `FIELD_MAP` entry, `settings` default, `meta` field, `populateSettingsForm()` /
     `saveSettingsBtn` lines, the `<native_title_area>` element in `buildSubmissionXml()`, and the
     `native_title_area` survey row + `nta_choices` list in the XLSForm (re-validated with
     `xls2xform`). Central/Kobo keeps the historical `native_title_area` column; new submissions
     just omit it. The goanna-side warehouse ETL is not built yet — the derivation machinery
     (`core.native_title_area_crosswalk`, boundary polygons, `flatten_toad_survey.R`) already
     exists for the toad survey and would be reused. Note goanna GPS can still be blank on failure,
     so that ETL will need an "unresolved NTA" state as the toad ETL has.
3. **Timer** — a simple start/stop stopwatch (`running`, `startTs`, `tick()` on a 250ms interval)
   that measures burrow time-to-detection. Stopping opens the record-entry bottom sheet. While
   running, a Screen Wake Lock (`requestWakeLock()`/`releaseWakeLock()`) is held so the OS doesn't
   suspend the tab (and stall `tick()`) when the phone locks or enters low-power mode — it's
   re-acquired on `visibilitychange` since wake locks auto-release when the tab is hidden. A
   `setInterval(beep, 60000)` also fires a short Web Audio tone every minute while running, as a
   reminder for crews who forget to stop the timer. A parallel `trackTimer` (`setInterval(
   logTrackPoint, 60000)`, started/cleared in `startTiming()`/`stopTiming()`) records the search's
   GPS track — see the GPS-track bullet under section 2.
4. **Record lifecycle** — each search produces a record object with a `status` of
   `pending → synced` or `failed`. The record sheet requires an explicit number of searchers (no
   default — `people` starts `null` and must be set via the stepper), whether the area was
   recently burnt (`burnt`, a large driver of detectability), whether a burrow was found, and — if
   so — whether it was inspected at all (`dig`: `Yes`/`No`, maps to the `dig_or_inspect` XForm
   field — kept as Yes/No rather than repurposed for camera/dig, so its meaning and scoring stay
   consistent with every submission recorded before this distinction existed) before asking how
   (`method`: `camera` or `dig`, maps to a separate new `inspection_method` field, only asked if
   inspected) and what was found (`found`, asked either way a burrow was inspected, since camera
   inspection is more likely to yield "nothing" than digging). Records are pushed to `records`,
   saved to `localStorage` immediately, then a sync is attempted. Tapping a non-synced record in
   the history list retries it individually via `submitOne()`.
5. **XForms/OpenRosa submission** — `buildSubmissionXml(rec)` renders a record as an OpenRosa XML
   instance using the fixed `FIELD_MAP` constant to map internal field names to the target form's
   XML element names (there's no user-facing way to remap these — they must match
   `goanna_burrow_detection.xlsx`, see below). `submitOne()` POSTs it as `xml_submission_file` in a
   `multipart/form-data` body, matching the ODK Central / KoboToolbox submission API contract.
   Auth is carried in the URL itself (see the app-user-code note above) — there's no separate
   username/password. The instance is hand-written, so anything with a non-trivial serialisation
   is encoded here rather than by an XForms engine: `select_multiple` is space-joined; the
   `search_track` `geotrace` is `"lat lon alt acc"` per point (ODK's `lat lon` order, **not**
   GeoJSON's, lat/lon at 6 dp), points `;`-joined with no trailing `;`, and needs ≥2 points with
   first ≠ last or it's spec-invalid (`prepareTrack()` rounds + dedupes to enforce this, emitting
   an empty `search_track`/`search_track_seconds` pair when < 2 points remain — see the GPS-track
   bullet in section 2).
   If you add a form field, place its element in `buildSubmissionXml()` to match its position in
   the `survey` sheet and re-validate the XLSForm: `pip install pyxform` in a throwaway venv, then
   `xls2xform goanna_burrow_detection.xlsx /tmp/check.xml` — a clean "Conversion complete!" means
   pyxform and ODK Validate both accepted it.
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
