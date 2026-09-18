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
version specifically and isn't needed inside the native shell.)

Then add the native platforms:

```
npx cap add android
npx cap add ios
```

(Skip `npx cap add ios` if you're not doing iOS right now.)

## 2. The update-check code is already in index.html

Open `index.html` and search for `CAPACITOR OTA UPDATE CHECK` — you'll find a
block near the bottom, inside the same `DOMContentLoaded` handler as
everything else. It's guarded by `if (window.Capacitor...)`, so it does
nothing at all when this same file is opened as a plain webpage or installed
as the PWA — it only activates inside the native app. You don't need to
change this code, just know it's there and what it does:

- Every time the app comes to the foreground, it fetches
  `update-manifest.json` (hosted at the same place as `index.html`)
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
on, most updates never need a new APK/IPA build at all:

1. Make your changes to `index.html` (same as always).
2. Bump the version number somewhere you'll remember it (e.g. in a comment
   at the top of the file) — say, to `1.0.1`.
3. Zip up the `www` folder's contents (`index.html`, `manifest.json`, the
   icons) into `bundle-1.0.1.zip`.
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
