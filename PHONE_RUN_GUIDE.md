# CompanyBox phone run guide

## What I verified

I checked the project structure and code in the Android app. The app is a React Native project (`react-native@0.88.0`) with an Android native module (`PhoneConfigModule.kt`) that reads the phone Wi‑Fi IP, gateway, RSSI, and link speed.

I also verified the JavaScript app logic in `CompanyBox/App.tsx`:

- It fetches state from `http://${routerIp}:8000/api/state`.
- It sends commands to `http://${routerIp}:8000/cmd/...`.
- It expects the phone to be on the same Wi‑Fi/LAN as the server.
- There is a fallback IP set to `192.168.1.128`, but this must be changed if your router/server is on a different subnet.

I ran the app checks from this environment:

- `npm install` succeeded.
- `npm test -- --watch=false` passed.
- Android deployment could not be completed here because no `adb`/Android SDK was available and no physical Android device was attached to this machine.

## Required environment to run on a real phone

1. Install Node.js 22.13 or newer (React Native 0.88 requires this).
2. Install Android Studio.
3. Install Android SDK + Android SDK Platform Tools (`adb`).
4. On the phone:
   - Enable Developer Options.
   - Enable USB debugging.
   - Connect the phone to the same Wi‑Fi network as the controlling backend/server.
5. Confirm the phone is visible with:

```powershell
adb devices
```

## Run the app on the phone

From the project directory:

```powershell
cd C:\Users\FDRkinggon\Desktop\licenceTech\CompanyBox\CompanyBox
npm install
npx react-native start
```

Then in another terminal:

```powershell
cd C:\Users\FDRkinggon\Desktop\licenceTech\CompanyBox\CompanyBox
npx react-native run-android
```

If Metro is not reachable from the phone, run:

```powershell
adb reverse tcp:8081 tcp:8081
```

Then reload the app from the Android device.

## Important notes

- The app uses cleartext HTTP traffic, so the backend must be on a trusted internal network.
- If the phone is not connected to the correct LAN, the app will show OFFLINE.
- The app currently relies on a hardcoded fallback IP. In production, it is safer to let the phone discover the gateway automatically or configure the correct server IP from the app settings.

## Why I could not physically run it on the phone from this machine

This environment does not have:

- an attached Android phone,
- an installed `adb` binary in the PATH,
- an Android SDK platform-tools installation.

Because of that, I could not complete the final USB install step from here. The steps above are the exact procedure to use on your machine or on the phone-connected workstation.
