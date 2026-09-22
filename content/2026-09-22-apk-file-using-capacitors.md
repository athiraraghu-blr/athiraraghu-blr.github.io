Title: From Web App to APK: Packaging Your Site with Capacitor
Date: 2026-09-20
Category: Article
Tags: capacitor, android, apk, hybrid-apps, mobile-development, javascript, ionic
Slug: web-app-to-apk-capacitor-guide

If you already have a web app — React, Vue, Angular, Svelte, or even plain HTML/CSS/JS — and you want to hand someone an installable Android APK without rewriting everything in Kotlin or Java, Capacitor is one of the most direct paths there. It wraps your existing web build in a native Android (or iOS) shell, gives you access to native device APIs through plugins, and lets your normal web tooling keep working.

This article walks through what Capacitor actually does, how to set it up, and how to produce a real, installable APK at the end.

# **What Capacitor Is (and Isn't)**

Capacitor, built by the Ionic team, is a native runtime for web apps. It creates a native Android Studio (or Xcode) project that hosts your web app inside a WebView, and bridges JavaScript calls to native code so you can use things like the camera, filesystem, geolocation, push notifications, and hundreds of community plugins.

It's worth being clear about what this means in practice:

1. Your UI is still HTML/CSS/JS rendered in a WebView — not native UI widgets like a true native app would use.

2. Performance is generally good for typical business apps, dashboards, forms, and content-driven apps, but won't match a fully native game or graphics-heavy app.

3. You get a real native project (an actual Android Studio project) that you own and can modify directly — Capacitor doesn't hide it from you the way some older tools did.

This makes it a good fit for teams that already have a web app and want a mobile presence without maintaining a separate native codebase, or for internal tools where "good enough" native feel is fine and development speed matters more.

**Prerequisites**

Before starting, make sure you have:

1. **Node.js and npm** installed

2. **Android Studio** installed, with the Android SDK and at least one platform version configured

3. A working **web app** with a build step that outputs static files (a dist, build, or similar folder)

4. Basic familiarity with the command line

You don't need to know Kotlin or Java to get a working APK, though it helps if you later want to customize native behavior.

# **Step 1: Install Capacitor in Your Project**

From the root of your existing web project:

bash

    npm install @capacitor/core
    npm install -D @capacitor/cli

Then initialize Capacitor:

bash

    npx cap init

You'll be prompted for an app name and an app ID (a reverse-domain identifier like com.yourcompany.appname). This ID is important — it becomes your app's unique package identifier on the Play Store and can't be changed later without effectively publishing a new app.

This step creates a capacitor.config.ts (or .json) file in your project root, which controls settings like the web directory, app ID, and plugin configuration.

# **Step 2: Point Capacitor at Your Web Build**

Capacitor needs to know where your compiled web assets live. Open capacitor.config.ts and check the webDir field:

ts

    import { CapacitorConfig } from '@capacitor/cli';

    const config: CapacitorConfig = {
    appId: 'com.yourcompany.appname',
    appName: 'YourAppName',
    webDir: 'dist',
    };

    export default config;

Set webDir to whatever folder your build tool outputs (dist for Vite/Vue, build for Create React App, etc.). Run your normal build command first:

bash

    npm run build

Capacitor copies the contents of this folder into the native project — it doesn't build your web app for you.

# **Step 3: Add the Android Platform**

Install the Android platform package and add it to the project:

bash

    npm install @capacitor/android
    npx cap add android

This generates a full native android/ folder in your project — a real Android Studio project using Gradle. Every time you change your web code, you'll rebuild the web assets and then sync them into this native project:

bash

    npm run build
    npx cap sync android

cap sync copies the latest web build into the native project and updates any native dependencies for installed plugins. Get in the habit of running this after every meaningful web change.

# **Step 4: Open and Configure the Native Project**

Open the Android project directly in Android Studio:

bash

    npx cap open android

Android Studio will index the project and resolve Gradle dependencies, which can take a few minutes the first time. While you're in here, it's worth checking a few things:

1. **App icon and splash screen** — Capacitor has an official assets generator (@capacitor/assets) that can produce all required icon and splash sizes from a single source image.

2. **Permissions** — declared in android/app/src/main/AndroidManifest.xml. If you're using plugins like camera or geolocation, confirm the relevant permissions were added automatically (most official plugins do this for you).

3. **build.gradle version info** — versionCode and versionName live in android/app/build.gradle and matter once you're preparing real releases.

# **Step 5: Build a Debug APK**

The fastest way to get something installable on a device for testing is a debug build. From Android Studio:

1. Go to **Build → Build Bundle(s) / APK(s) → Build APK(s).**

2. Android Studio compiles the project and shows a notification when it's done, with a link to locate the file.

3. The output lands in android/app/build/outputs/apk/debug/app-debug.apk.

You can also do this from the command line using Gradle directly, without opening the IDE:

bash

    cd android
    ./gradlew assembleDebug

This debug APK is signed with a default debug key, which is fine for testing on your own devices or sharing informally, but Android and the Play Store will reject it for public distribution.

# **Step 6: Build a Signed Release APK**

For anything beyond local testing — sharing with testers outside your team, or publishing to the Play Store — you need a signed release build.

**Generate a keystore** (only needs to be done once; keep this file and its passwords safe, since losing it means you can't update the app later under the same identity):

bash

    keytool -genkey -v -keystore my-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-key-alias

**Configure signing** in android/app/build.gradle by adding a signingConfigs block, or, more conveniently, sign the already-built APK afterward using apksigner from the Android SDK command-line tools:

bash

    cd android
    ./gradlew assembleRelease

This produces an unsigned release APK at android/app/build/outputs/apk/release/app-release-unsigned.apk. Sign and align it:

bash

    zipalign -v -p 4 app-release-unsigned.apk app-release-aligned.apk
    apksigner sign --ks my-release-key.jks --out app-release-signed.apk app-release-aligned.apk

The result, app-release-signed.apk, is a proper release build ready to install directly on a device or upload for distribution. (If you're specifically targeting the Play Store, note that Google now generally expects an **AAB — Android App Bundle** — rather than a raw APK; Capacitor projects can produce this too via ./gradlew bundleRelease, but a signed APK remains the right format for direct installs, sideloading, or internal distribution.)

# **Common Gotchas**

A few issues come up often enough to flag in advance:

1. **Forgetting to run cap sync** — changes to your web code won't show up in the native app until you rebuild the web assets and sync again.

2. **White screen on launch** — usually means webDir points to the wrong folder, or the web build wasn't run before syncing.

3. **Plugin not working** — most Capacitor plugins need npx cap sync after installation to wire up native dependencies, not just npm install.

4. **App ID mismatches** — changing the app ID after your first build can leave stale references in the native project; regenerating the platform (npx cap add android again after removing the old folder) is often the cleanest fix.

5. **Network requests to localhost in dev** — during local development, Capacitor apps run against your bundled build, not your dev server, unless you explicitly configure a live-reload server URL in capacitor.config.ts.

# **Wrapping Up**

Capacitor's core value is that it doesn't ask you to abandon your web stack — it builds a bridge from it to a real native Android project you can build, sign, and ship like any other Android app. The workflow boils down to a simple loop: **build your web app → sync into the native project → build the APK in Android Studio or via Gradle**. Once that loop is comfortable, adding native functionality through plugins, customizing app icons, and preparing real signed releases are all incremental steps from there rather than separate systems to learn.