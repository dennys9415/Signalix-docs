# Mobile Roadmap — Signalix-mobile

**As of:** v0.16.0 (foundation MVP).

## v0.16.0 — Foundation (this release)
- ✅ Expo SDK 51 + RN 0.74 + TS strict scaffolding.
- ✅ Auth (login / register / refresh-on-401 / logout) with `expo-secure-store`-backed session.
- ✅ Chat list with realtime updates + pull-to-refresh + unread badges + presence dots.
- ✅ Direct chat conversation (history + send + receive + auto-deliver + auto-read).
- ✅ Group chat conversation (text only; no group management UI).
- ✅ Settings (username / email / display name / version / sign out).
- ✅ Bottom tabs (Chats / Settings) + push to Conversation.
- ❌ E2EE — plaintext only on mobile in v0.16.0 (see v0.17.0 below).
- ❌ Media / files / voice notes.
- ❌ Push notifications.
- ❌ Calls (voice / video).

## v0.17.0 — E2EE port (next theme)

**Goal:** match the web's v0.10.x+ E2EE so direct + group text round-trips end-to-end between mobile and web.

Work breakdown:

1. **Crypto adapter**. Replace the web's hard dependency on `crypto.subtle` with a thin abstraction. Two backends:
   - **Web** keeps the existing Web Crypto path.
   - **Mobile** uses `@stablelib/*` (pure-JS, audited):
     - `@stablelib/x25519` for ECDH (signed pre-key + ephemeral).
     - `@stablelib/ed25519` for SPK signature + verify.
     - `@stablelib/aes-gcm` for envelope encryption.
     - `@stablelib/hkdf` + `@stablelib/sha256` for HKDF-SHA-256.
   - Same wire format. Same envelope JSON. Same `signalix-v1-direct-text` HKDF info.
2. **Storage adapter**. Replace `IndexedDB` with:
   - `expo-secure-store` for identity private keys, signing private key.
   - `AsyncStorage` for the pre-key pool, signed pre-keys, fingerprints, plaintext cache.
   - Shared `KeyStore` interface so the web's existing code paths port with minimal changes.
3. **Device registration**. On first launch, generate identity + signing keypair, derive a `deviceId`, register via `POST /crypto/devices/keys`. Identical to the web flow.
4. **Pre-key pool management**. Initial 100, top-up to 100 when unconsumed drops below 20. Same v0.9.1 cadence.
5. **Reset detection**. Same partial-state / different-device-id heuristics from web v0.9.1. Force-cleanup-once flag in `expo-secure-store` so the v0.17.0 deploy reaches a clean state automatically.
6. **Backup / restore**. Port the web's v0.15.0 `bip39` + `backup.ts`. Backup file is the same `.sbk` format. Share documents via the native share sheet.

## v0.18.0 — Push notifications
- Expo Notifications + APNs / FCM.
- Backend already has `push_subscriptions` (v0.6.0); reuse with a `device_type: 'mobile-expo'` discriminator.
- Quiet-hours preference.
- Encrypted push previews (server forwards ciphertext; SW / native decrypts).

## v0.19.0 — Media + voice notes
- Image / file upload (encrypted-blob path from web v0.11.0).
- Voice note recording with native audio APIs.
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
