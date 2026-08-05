# Goanna Burrow Timer — TCZ

A field app for the Toad Containment Zone (TCZ) program: times how long it takes to detect a
goanna at a burrow, then syncs the records to our KoboToolbox/ODK Central form. Works fully
offline once installed — no signal needed out in the field.

**App URL:** https://popbiolgen.github.io/goanna-hunting-app/

## Setup (do this once)

### 1. Request app user credentials

Before the app can sync your records to the server, you need an ODK Central **app user** code for
this project. Contact **tcz@curtin.edu.au** to have one created for you — you'll be sent either a
QR code image or a long text code from Central's "App Users" page.

### 2. Install the app on your phone

**iOS (must use Safari — other browsers don't support this):**
1. Open the app URL above in Safari.
2. Tap the **Share** button (square with an arrow pointing up).
3. Scroll down and tap **Add to Home Screen**, then tap **Add**.

**Android (Chrome):**
1. Open the app URL above in Chrome.
2. Tap the **⋮** menu (top right).
3. Tap **Add to Home screen** (or **Install app**, if Chrome offers it directly), then confirm.

Either way you'll get an app icon on your home screen. Open it once while you have signal so it
can finish setting itself up for offline use — after that it'll launch instantly with zero bars.

### 3. Add your app user code and team names

1. Open the app and tap the ⚙ settings icon.
2. Under **Hunting team names**, enter the names for your team (set once — applied to every
   record from this device).
3. Under **Central app user code**, either tap **📷 Scan QR code instead** and scan the code you
   were sent, or paste the text code directly.
4. Tap **Save settings**.

If your phone asks for camera or location permission the first time you use the QR scanner or
start a hunt, allow it — the camera is only used to read the QR code, and location is only used to
tag each record with GPS coordinates.

## Using it in the field

- Tap **Start hunting** to begin a session — this captures the date, time and GPS location
  automatically.
- Use the timer to record each burrow search as normal.
- Tap **Finish hunting** when you're done for the day.
- Records sync automatically whenever you have signal (or tap **Sync now**). Nothing is lost if
  you're offline — everything queues on the phone until it can sync.

## For developers

See [`CLAUDE.md`](CLAUDE.md) for how the app is built and how to make changes.
