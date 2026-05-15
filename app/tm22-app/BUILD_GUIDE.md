# TM22 Capacitor Build Guide

## Prerequisites
- **Node.js 18+** — `node -v` to check
- **Android Studio** — with SDK 33+ installed
- **USB debugging** enabled on your Android phone
- **Phone plugged in via USB** (or use wireless ADB)

## Step 1: Create project folder
```
mkdir tm22-app
cd tm22-app
```

## Step 2: Copy files from the ZIP into tm22-app/
You should have:
```
tm22-app/
  package.json
  capacitor.config.json
  www/
    index.html
```

## Step 3: Install dependencies
```
npm install
```
This installs Capacitor core, Android platform, and the BLE plugin.

## Step 4: Add Android platform
```
npx cap add android
```
This creates the `android/` folder with the full Android Studio project.

## Step 5: Add BLE permissions to Android manifest
Open `android/app/src/main/AndroidManifest.xml` and add these lines **inside** the `<manifest>` tag, **before** `<application>`:

```xml
<!-- BLE Permissions -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="false" />
```

Also add to keep the screen on during matches — inside `<activity>`:
```xml
android:keepScreenOn="true"
```

## Step 6: Sync web content to Android
```
npx cap sync android
```
This copies www/ into the Android project and registers the BLE plugin.

## Step 7: Open in Android Studio
```
npx cap open android
```
Android Studio opens with the project. Wait for Gradle sync to finish.

## Step 8: Build and run
- Select your phone from the device dropdown in Android Studio
- Click the green Run ▶ button
- App installs and launches on your phone

## Alternative: Run directly from terminal
```
npx cap run android
```
This builds and deploys without opening Android Studio (phone must be connected via USB with debugging enabled).

---

## After code changes

When you edit `www/index.html`:
```
npx cap sync android
npx cap run android
```
That's it. Two commands to update the phone.

---

## Testing BLE

1. Open the app on your phone
2. Tap the **BLE** tab (📡)
3. Tap **Connect** next to any device
4. Phone will request Bluetooth + Location permissions — grant them
5. It scans for 5 seconds and auto-connects to the first match

**Without hardware:**  The BLE tab shows "BLE not available" in browser. On Android it activates. All other tabs (HUD, Truing, Profiles, Guide) work identically in both environments.

**With Calypso:** Connect → weather data streams to HUD automatically. Temp and DA fields auto-populate. Green dot appears next to ENVIRONMENT header.

**With DOPE card:** Connect → any stage solution pushes to the card display automatically.

---

## Troubleshooting

**"BLE not available" on Android:**
- Check that @capacitor-community/bluetooth-le installed correctly: `ls node_modules/@capacitor-community/bluetooth-le`
- Re-run `npx cap sync android`
- Make sure permissions are in AndroidManifest.xml

**Gradle sync fails:**
- In Android Studio: File → Invalidate Caches → Restart
- Or delete `android/.gradle` and re-sync

**App installs but crashes:**
- Check logcat in Android Studio for errors
- Most likely: missing permissions or Babel transpile error
- Test www/index.html in Chrome on your PC first to verify no JS errors

**Data doesn't persist:**
- localStorage works in Capacitor WebView
- Data saves on every state change
- Survives app close and phone restart
- Does NOT survive app uninstall
