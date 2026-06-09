# Mobile Setup — Signalix-mobile

**Applies to:** Signalix-mobile v0.16.0.

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 22.x | Expo SDK 51 requirement |
| npm | 10+ | bundled with Node 22 |
| Watchman (macOS) | latest | RN file watching |
| Android Studio + emulator | Jellyfish (2023.3.1) or later | Android target |
| Java JDK | 17 | Gradle 8 |
| Xcode (macOS only) | 15+ | iOS target (optional for v0.16.0) |

Get the Expo CLI:
```bash
npm install -g expo-cli  # global, optional — `npx expo` also works
```

## 0. Backend reachable

The mobile app expects `Signalix-api` and `Signalix-realtime` to be running. Easiest path: spin up the full stack from `Signalix-infra`:
```bash
cd Signalix-infra
./scripts/up.sh
```

Verify:
- API: `curl http://localhost:4000/health` → `{ "status": "ok" }`
- Realtime: open `ws://localhost:5000` in a tool like `websocat`.

## 1. Build shared contracts

The mobile app consumes `@signalix/contracts` via the `file:` protocol. The contracts package must be **built** before `npm install` runs in the mobile repo (otherwise the `dist/` import resolves to nothing).

```bash
cd Signalix-contracts
npm install
npm run build
cd ..
```

## 2. Install + start

```bash
cd Signalix-mobile
npm install
npm run start          # opens the Expo Dev Tools (Metro + QR code)
```

Then in the same terminal:
- press `a` to open the Android emulator
- press `i` to open the iOS simulator (macOS only)
- scan the QR code from the Expo Go app on a physical device

## 3. Emulator → host networking

The default `app.json` ships with:
```json
"extra": {
  "apiUrl": "http://10.0.2.2:4000",
  "wsUrl":  "ws://10.0.2.2:5000"
}
```

The `10.0.2.2` address is the **Android emulator's** loopback to the host machine. For other targets:

| Target | apiUrl | wsUrl |
|---|---|---|
| Android emulator | `http://10.0.2.2:4000` | `ws://10.0.2.2:5000` |
| iOS simulator | `http://localhost:4000` | `ws://localhost:5000` |
| Physical Android device (LAN) | `http://192.168.x.y:4000` | `ws://192.168.x.y:5000` |
| Physical iOS device (LAN) | same as above | same as above |
| Production | `https://api.signalix.example` | `wss://realtime.signalix.example` |

Edit `app.json` (no rebuild needed; Metro picks it up) and reload the app with `r` in the Expo CLI.

## 4. Typecheck

```bash
npm run typecheck
```

## 5. Production build (Android)

v0.16.0 is foundation only. A signed release build is out of scope for this version — the supported flow is **Expo Go dev experience** with `npm run start`. When v0.18.0+ adds push notifications, a real native build via EAS will become necessary.

If you do want to produce an APK for distribution today:
```bash
npx eas-cli build --platform android --profile preview
```
This requires an Expo account and an `eas.json` (not committed in v0.16.0).

## Troubleshooting

**`Unable to resolve module @signalix/contracts`** — you skipped step 1. Build the contracts package, then `rm -rf node_modules` + `npm install` again.

**`Network request failed` on Android emulator** — verify `apiUrl` is `10.0.2.2` (not `localhost`). The emulator can't see the host's `localhost`.

**WebSocket connects then immediately closes** — usually a JWT mismatch. The `JWT_SECRET` in `Signalix-api`'s `.env` must match `Signalix-realtime`'s `.env`. The `Signalix-infra/docker-compose.yml` wires them automatically.

**`expo-secure-store` error on Web fallback** — there's no web fallback for v0.16.0; Expo's `expo start --web` will throw. The app is Android-first (iOS works incidentally). Web support is not on the v0.16.0 roadmap.

**Tabs don't render** — make sure `react-native-screens` and `react-native-safe-area-context` are installed at the exact versions in `package.json`; React Navigation has tight version constraints.

## Running on a physical device

1. Install **Expo Go** from the Play Store.
2. Make sure your device is on the same LAN as your dev machine.
3. Update `app.json` → `extra.apiUrl` and `wsUrl` to your machine's LAN IP (not `10.0.2.2`).
4. Run `npm run start` and scan the QR code with Expo Go (Android) or the Camera app (iOS).
