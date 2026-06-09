# Mobile Roadmap — Signalix-mobile

**As of:** v0.16.0 (foundation MVP, released 2026-06-09).

## v0.16.0 — Foundation (released)
- ✅ Expo SDK 54 + React Native 0.81 + React 19.1 + TS strict scaffolding.
- ✅ Auth (login / register / refresh-on-401 / logout) with `expo-secure-store`-backed session.
- ✅ Chat list with realtime updates + pull-to-refresh + unread badges + presence dots.
- ✅ Direct chat conversation (history + send + receive + auto-deliver + auto-read).
- ✅ Group chat conversation (text only; no group management UI).
- ✅ Settings (username / email / display name / version / sign out).
- ✅ Bottom tabs (Chats / Settings) + push to Conversation.
- ✅ Metro 0.83 config that pulls `@signalix/contracts` from the sibling repo via `watchFolders` + `extraNodeModules`.
- ❌ E2EE — plaintext only on mobile in v0.16.0 (see v0.17.0 below).
- ❌ Media / files / voice notes.
- ❌ Push notifications.
- ❌ Calls (voice / video).

## v0.17.0 — E2EE port (next theme)

**Goal:** match the web's v0.10.x+ E2EE so direct + group text round-trips end-to-end between mobile and web.

### Why this is the next theme

v0.16.0's per-conversation plaintext degradation (web↔mobile, mobile↔mobile) is documented but not acceptable long-term — every conversation that touches a mobile peer drops to plaintext. Closing this gap is more valuable than push or media, and it unblocks every downstream feature (push previews, encrypted media) that needs the crypto layer in place.

### Work breakdown

1. **Crypto adapter abstraction** *(estimated: 3-4 days)*
   - Extract a `CryptoBackend` interface from `Signalix-frontend/src/lib/crypto/` that abstracts the four primitive operations:
     - `x25519.generateKeyPair() / derive(privKey, pubKey)` — ECDH
     - `ed25519.generateKeyPair() / sign(privKey, msg) / verify(pubKey, msg, sig)` — signing
     - `aesGcm.encrypt(key, iv, plaintext, aad) / decrypt(key, iv, ciphertext, aad)` — envelope
     - `hkdf.derive(salt, ikm, info, length)` — key derivation
   - **Web backend** stays on Web Crypto (`crypto.subtle`).
   - **Mobile backend** uses `@stablelib/*` (pure-JS, audited):
     - `@stablelib/x25519`, `@stablelib/ed25519`, `@stablelib/aes-gcm`, `@stablelib/hkdf`, `@stablelib/sha256`.
   - Same wire format. Same envelope JSON. Same `signalix-v1-direct-text` / `signalix-v1-group-text` HKDF info strings.
   - **Reference files to read first** before starting:
     - `Signalix-frontend/src/lib/crypto/index.ts`
     - `Signalix-frontend/src/lib/crypto/envelope.ts`
     - `Signalix-frontend/src/lib/crypto/key-derivation.ts`
     - `Signalix-frontend/src/lib/crypto/identity.ts`

2. **Storage adapter abstraction** *(estimated: 2-3 days)*
   - Extract a `KeyStore` interface from `Signalix-frontend/src/lib/crypto/idb.ts` covering:
     - `identity` (1 row) — JWK keypairs + deviceId
     - `signed-pre-keys` (rotating) — JWK keypairs + signature
     - `pre-keys` (pool of 100) — JWK keypairs + consumed flag
     - `plaintext-cache` (per-message decrypted text for the sender's history)
     - `fingerprints` (peer safety numbers + verification snapshots)
   - **Web backend** keeps IndexedDB.
   - **Mobile backend**:
     - `expo-secure-store` for identity private key + signing private key (encrypted at rest by iOS Keychain / Android Keystore).
     - `AsyncStorage` for everything else (pre-key pool, fingerprints, plaintext cache).
   - **Important:** key material exported as JWK (web's format) so backups stay cross-platform.

3. **Device registration** *(estimated: 1 day)*
   - On first launch after install: generate identity + signing keypair → derive `deviceId` → register via `POST /crypto/devices/keys`.
   - Identical to the web flow. Endpoint is unchanged.
   - Mobile sets `device_type: 'mobile-expo'` in the registration payload (web uses `'web'`).

4. **Pre-key pool management** *(estimated: 1 day)*
   - Initial 100 pre-keys. Top-up to 100 when unconsumed drops below 20. Same v0.9.1 cadence.
   - Background task on app foreground: check `consumed_count`, mint + register replacements if needed.

5. **Reset detection** *(estimated: 0.5 day)*
   - Same partial-state / different-device-id heuristics from web v0.9.1 (`Signalix-frontend/src/lib/crypto/reset-detector.ts`).
   - Force-cleanup-once flag in `expo-secure-store` so the v0.17.0 deploy reaches a clean state automatically (mirrors web's `localStorage` cleanup flag).

6. **Backup / restore port** *(estimated: 2 days)*
   - Port the web's `bip39.ts` + `backup.ts` (v0.15.0).
   - Backup file is the same `.sbk` format (magic `SLXBKP01` + version + salt + IV + ciphertext).
   - Use the native share sheet (`expo-sharing`) for export. Use `expo-document-picker` for restore.

7. **Round-trip testing** *(estimated: 2-3 days)*
   - Test matrix: web → mobile, mobile → web, mobile → mobile, group with mixed peers.
   - Specific verification points:
     - Direct text decrypts on the receiving side (no `[Unable to decrypt]` placeholders).
     - Edit message envelope re-routes correctly (frontend v0.14.0 added edit envelope fields).
     - Group fan-out works when at least one peer is mobile.
     - Reconnect sync after offline (frontend v0.14.0 reconnect catch-up flow).
     - Backup created on web restores on mobile and vice versa.

**Total estimated effort:** 12-15 days for one engineer, sequentially.

### Out of scope for v0.17.0
- Push notifications.
- Group media / files / voice notes (E2EE for media is already beta on web; mobile gets it in v0.19.0).
- iOS production builds.

## v0.18.0 — Push notifications
- Expo Notifications + APNs / FCM.
- Backend already has `push_subscriptions` (v0.6.0); reuse with a `device_type: 'mobile-expo'` discriminator.
- Quiet-hours preference.
- Encrypted push previews (server forwards ciphertext; SW / native decrypts via the v0.17.0 crypto adapter).

## v0.19.0 — Media + voice notes
- Image / file upload (encrypted-blob path from web v0.11.0).
- Voice note recording with native audio APIs (`expo-av`).
- Camera roll picker (`expo-image-picker`).

## v0.20.0 — Calls (foundation)
- 1:1 voice via WebRTC (`react-native-webrtc`).
- Signal exchange over the existing WS channel.
- Push-to-wake for incoming calls (requires push from v0.18.0).
- Video calls follow as v0.21.0.

## v0.22.0+ — Long tail
- iOS production builds + TestFlight pipeline.
- Tablet layouts.
- Device verification UI (safety number QR scan on mobile).
- Group management UI (add/remove members, transfer ownership) — current placeholder is web-only.
- Theme switcher.
- Localization.

## How releases work in this project

Every Signalix release is a **lockstep tag** across all 7 repos. When v0.17.0 is cut:

1. `Signalix-mobile` does the real work (crypto + storage adapters, device registration, backup port).
2. `Signalix-frontend` may need shim changes if the crypto adapter is shared (web extracts its own backend behind the same interface).
3. `Signalix-contracts` may need no change OR add a `device_type` enum value (`'mobile-expo'`) if mobile registration uses a new discriminator.
4. `Signalix-api`, `Signalix-realtime`, `Signalix-infra` are typically no-ops (the existing `/crypto/devices/keys` endpoint already handles every device type).
5. All 7 repos: version bump, README v0.17.0 changelog section, lockstep commit + tag + push.

See `docs/HANDOFF.md` for the canonical release checklist.
