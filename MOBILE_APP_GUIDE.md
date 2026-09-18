# Building the Android & iOS Apps (with background auto-updates)

This turns the same app into a real installable APK (Android) and IPA (iOS),
fully offline from first install, with a self-hosted background-update
mechanism — no app store review, no third-party update service, no account.

Read this in full before starting — a few steps happen in a specific order.

## What you'll need

**For Android:**
- A computer (Windows, Mac, or Linux all work)
- [Node.js](https://nodejs.org) (LTS version)
- [Android Studio](https://developer.android.com/studio) — free

**For iOS — genuinely more involved, not optional:**
- A **Mac** — Xcode only runs on macOS, there's no way around this
- [Xcode](https://apps.apple.com/us/app/xcode/id497799835) — free, from the Mac App Store
- An **Apple Developer Program membership — $99/year** — required to install
  on a real iPhone at all, even just for yourself. There's no free tier for
  this. If iOS isn't essential, it's fine to do Android first and decide
  about this later — nothing below locks you out of coming back to iOS.

If you only have Android available right now, skip every "iOS" step below
and everything still works for Android alone.

## 1. One-time project setup

Open a terminal, navigate to an empty folder where you want this project to
live, and run:

```
npm init -y
npm install @capacitor/core @capacitor/app @capacitor/preferences @capacitor/splash-screen @capgo/capacitor-updater
npm install -D @capacitor/cli @capacitor/android @capacitor/ios
npx cap init "Parade State" "com.yourunit.paradestate" --web-dir www
```

Replace `com.yourunit.paradestate` with your own unique app ID if you like
(reverse-domain style — it doesn't need to be a real domain, just unique to
you). Whatever you choose, use the **same one** in `capacitor.config.json`
below.

This creates a `capacitor.config.json` in your project — **replace it** with
the one in this package (already configured for self-hosted manual updates).
Alternatively just copy the `"plugins"` block from ours into the one it
generated.

Now create a `www` folder in your project root and copy these files into it:
- `index.html`
- `manifest.json`
- `icon-192.png`
- `icon-512.png`

(Leave `service-worker.js` out of `www` — it's for the browser/GitHub Pages
version specifically and isn't needed inside the native shell. Leave
`update-manifest.json` out too — see why in the next section.)

Then add the native platforms:

```
npx cap add android
npx cap add ios
```

(Skip `npx cap add ios` if you're not doing iOS right now.)

## 2. The update-check code is already in index.html — one thing to set

Open `index.html` and search for `CAPACITOR OTA UPDATE CHECK` — you'll find a
block near the bottom, inside the same `DOMContentLoaded` handler as
everything else. It's guarded by `if (window.Capacitor...)`, so it does
nothing at all when this same file is opened as a plain webpage or installed
as the PWA — it only activates inside the native app.

**Before your first build**, check the `UPDATE_MANIFEST_URL` line right
above the `checkForUpdate` function and make sure it's your actual GitHub
Pages URL — it needs to be a full `https://...` address, not a relative
path. This matters more than it looks: the app's web content is bundled
*locally* inside the APK/IPA, so a relative path would resolve against that
local copy and just read back whatever was bundled at build time, forever —
never actually reaching the internet to check for something newer. An
absolute URL is what makes this a live check instead of reading its own
tail.

This is also why `update-manifest.json` doesn't go in `www/`: anything in
`www/` gets bundled into the app and becomes a fixed, unchanging snapshot
from that point on. `update-manifest.json` needs to be the opposite — a
file that stays live and editable on the internet, so you can change what
it says *after* the app is already installed on someone's phone. Put it at
the root of your GitHub Pages repo instead (alongside wherever `index.html`
is hosted for the browser/PWA version), so it's reachable at a fixed URL
that never changes even as its contents do.

What the code does, once that URL is right:
- Every time the app comes to the foreground, it fetches that live
  `update-manifest.json`
- If the manifest's version doesn't match what's currently installed, it
  downloads the new bundle quietly in the background
- The moment the app is backgrounded (switched away from), it applies the
  update — so it's never interrupting anything you're doing
- Next time you open the app, it's already the new version

## 3. Build and run for the first time

```
npx cap sync
```

**Android:**
```
npx cap open android
```
This opens Android Studio. Let it finish indexing/syncing Gradle (can take a
few minutes the first time), then press the green ▶ Run button with a
connected device or emulator selected. This installs a debug build directly
— no signing needed yet for testing on your own device.

**iOS** (Mac + Xcode only):
```
npx cap open ios
```
In Xcode, select your device (or a simulator) from the dropdown at the top
and press ▶. The **first** time you run on a real iPhone, Xcode will prompt
you to select a "Team" under Signing & Capabilities — this requires the
Apple Developer account mentioned above.

At this point you have a real, installed app, working fully offline, on
whichever device you ran it on.

## 4. Shipping an update, going forward

This is the part that replaces "push to GitHub, get a new APK" — from here
on, most updates never need a new APK/IPA build at all.

**There's no auto-incrementing version anywhere in this** — the version
number is just a value you choose and write into `update-manifest.json`
yourself each time. The app has no idea what "the next version" is; it only
ever compares "what does the live manifest say" against "what did I last
apply," so bumping that number is what actually triggers an update at all.
Forget to change it, and the app checks, sees the same version it already
has, and does nothing — which is correct behavior, not a bug, but worth
knowing so a forgotten bump doesn't look like a broken update system.

1. Make your changes to `index.html` (same as always).
2. Decide on a new version number — anything goes as long as it's different
   from last time, e.g. `1.0.1`. Nothing has to match anywhere else; it's
   purely a label you and the app agree on via the manifest.
3. Zip up the updated file(s) into `bundle-1.0.1.zip`. `index.html` is the
   only one that actually needs to be in there — it's the whole app (all the
   CSS and JS are inlined in that one file). `manifest.json` and the icons
   are read by browsers for "Add to Home Screen," not by the native app
   itself, so there's no need to include them in an update bundle unless
   you've actually changed them.
4. Host that zip somewhere reachable by URL — the same GitHub Pages repo
   you're already using works fine; a plain `/updates/` folder in that repo
   is enough.
5. Update `update-manifest.json` (also hosted on that same site) to:
   ```json
   { "version": "1.0.1", "url": "https://yourname.github.io/your-repo/updates/bundle-1.0.1.zip" }
   ```
6. Push. That's it — every installed copy of the app picks this up
   automatically next time it's opened, no reinstall, no app store.

You only need to go back through Android Studio/Xcode and build a fresh
APK/IPA when you change something that isn't just the web content — adding
a new native plugin, changing app icons/permissions, or the like. Ordinary
feature and bug-fix work, like everything we've built in this whole
conversation, ships through the steps above instead.

## Being upfront about what's verified vs. not

The `capacitor.config.json` content, the dependency versions, and the
`download`/`set` update API are pulled directly from Capgo's current
documentation, so I'm confident in those. The exact plugin-access pattern in
the update-check code (`window.Capacitor.Plugins.CapacitorUpdater`) is the
standard, documented way to reach a Capacitor plugin from plain JavaScript
without a bundler — but I haven't been able to run this end-to-end myself,
since it needs Android Studio/Xcode and real devices I don't have access to
here. If step 3 above throws an error the first time you run it, paste it
back to me — a real error message is far more useful than me guessing
further in the dark, and this is a very fixable category of problem.
