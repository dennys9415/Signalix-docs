# Signalix v0.9.1 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.9.0

---

## Overview

v0.9.1 is the **E2EE hardening** release. The v0.9.0 beta worked end-to-end but had several footguns: the server would accept any 32-byte string as a signed pre-key, the client would happily encrypt to a corrupted bundle, one-time pre-keys were never actually consumed, and a transient IndexedDB problem could brick decryption permanently. v0.9.1 fixes all of those.

What ships:

- **Server-side Ed25519 verification** of `signedPreKey.signature` on register and rotate (`Signalix-api`), plus exact byte-length checks on every key material field.
- **Client-side bundle validation** — `assertBundleIsValid` runs before any ECDH derivation and includes the Ed25519 signature check. A tampered bundle throws instead of producing unreadable ciphertext.
- **One-time pre-key consumption** — `PreKeyRecord.consumed` is flipped after a successful decrypt; the unconsumed pool is what drives top-up math; the pool is restocked to ~100 when it drops below 20.
- **Device reset auto-recovery** — corrupt or partial local crypto state triggers a wipe + regenerate + republish, with a one-time amber banner: *"Encryption keys were reset on this device."*
- **Decrypt failure cache** — `PlaintextCacheRecord.failed` short-circuits re-attempts, so history reloads don't re-run broken handshakes or flicker the placeholder.
- **Safety number foundation** — new `fingerprints.ts` computes the per-peer 12-group safety number and caches it in a new IndexedDB store. **No UI yet** — that's v0.10.0.
- **Tests** — `vitest` added; 10 unit tests covering base64url, Ed25519 verify, and safety number behavior.

What does **not** ship:

- Multi-device fan-out, Double Ratchet, encrypted push previews — still v0.10.0.
- Safety-number verification UI — only the cryptographic foundation.
- Group / media / voice E2EE — v0.11.0+.

---

## What changed end-to-end

### Send path (sender browser)

```
plaintext "hello"
   ↓ chat.store.sendMessage
   ↓ cryptoService.encryptForRecipient(plaintext, { recipientUserId })
   ↓     GET /api/v1/crypto/users/:userId/key-bundle
   ↓     assertBundleIsValid(bundle)          ← NEW v0.9.1
   ↓       • shape & base64url decodability
   ↓       • byte lengths (32/32/32/64, optional preKey 32)
   ↓       • Ed25519 verify(signingKey, signature, signedPreKey.publicKey)
   ↓     maybeCacheSafetyNumber(peerUserId)   ← NEW v0.9.1 (best-effort)
   ↓     ephPair = generate X25519
   ↓     IKM = ECDH(eph, SPK) [|| ECDH(eph, OTP)]
   ↓     key = HKDF-SHA256(IKM, info="signalix-v1-direct-text")
   ↓     ct  = AES-256-GCM(key, iv, plaintext)
```

### Receive path (recipient browser)

```
WS server.message.new {ciphertext, encryptionVersion=1, signedPreKeyId, preKeyId?}
   ↓ chat.store handler
   ↓ decryptStoredMessage(m)
   ↓   lookupPlaintext(m.id) ?               → use cache (unchanged from v0.9.0)
   ↓   isDecryptFailureCached(m.id) ?        ← NEW v0.9.1 — render placeholder, no re-attempt
   ↓   await cryptoService.decryptIncoming(envelope)
   ↓     parse envelope; IDB lookups SPK + preKey by id
   ↓     ECDH + HKDF + AES-GCM open
   ↓     if (preKey) mark preKey.consumed = true ← NEW v0.9.1
   ↓     void maybeTopUpAfterConsume()       ← NEW v0.9.1 (replenish ≤20 → ~100)
   ↓   on success: cachePlaintext
   ↓   on failure: cacheDecryptFailure        ← NEW v0.9.1
```

### Server publish path (`Signalix-api`)

```
POST /api/v1/crypto/devices/keys              POST /api/v1/crypto/devices/signed-pre-key/rotate
  ↓ DTO + class-validator                       ↓ DTO + class-validator
  ↓ assertSize(...) on every key                ↓ lookup signing_key from device_identity_keys
  ↓ assertSignedPreKeySignature(...)            ↓ assertSize + assertSignedPreKeySignature
  ↓   Ed25519 verify via node:crypto.webcrypto  ↓
  ↓ INSERT rows in transaction                  ↓ UPDATE old SPK rotated_at; INSERT new SPK
```

A 400 `VALIDATION_ERROR` now means "we caught your bundle before it polluted the device tables" instead of "rows persisted; downstream peers will start seeing decryption failures next week."

---

## Migration

- **No database migration.** Schema is identical to v0.9.0 (`V14__crypto_foundation.sql`).
- **No protocol change.** WS payloads and REST shapes match v0.9.0.
- **Frontend IDB schema bumps to v2.** Existing rows survive; the upgrade only adds the `fingerprints` store.
- **v0.9.0 clients still work against a v0.9.1 server**, as long as their bundles were well-formed. A client that was publishing junk will start getting `400 VALIDATION_ERROR` from `/crypto/devices/keys` and `/crypto/devices/signed-pre-key/rotate` — which is the intended behavior.

---

## Operational notes

- **Watch your error logs** for `CryptoService` Logger output the first time v0.9.1 reaches a populated environment. Any pre-existing client that was publishing malformed material will now surface here.
- **The reset banner is the only user-visible regression vector.** If you see it on every login, something else is corrupting the IDB; investigate before assuming "the user wiped browser data."
- **No infra changes.** Same Docker images, env vars, ports as v0.9.0. `Signalix-infra/docker-compose.yml` is untouched.

---

## Validation

- `npm run typecheck` — clean across `Signalix-api`, `Signalix-frontend`.
- `npm run build` — clean in both repos.
- `npm test` (frontend) — 10/10 vitest unit tests pass:
  - base64url round-trip + empty input
  - `concatBytes` order/length
  - `bytesToHex` zero-padding
  - `verifyEd25519Signature` accept / reject-tampered / reject-malformed-key
  - safety number format (12 × 5 digits)
  - safety number symmetry across argument order
  - safety number sensitivity to a single bit flip

---

## Files of interest

**Server**

- `Signalix-api/src/crypto/crypto.service.ts` — `assertSignedPreKeySignature`, `assertSize`, byte-length and Ed25519 checks on register + rotate.

**Frontend**

- `Signalix-frontend/src/lib/crypto/signal.service.ts` — bundle validation, reset detection + wipe path, one-time pre-key consumption, top-up math against unconsumed count, `getSafetyNumber`.
- `Signalix-frontend/src/lib/crypto/utils.ts` — `verifyEd25519Signature`, `bytesToHex`.
- `Signalix-frontend/src/lib/crypto/db.ts` — schema bump to v2, `STORE_FINGERPRINTS`, `idbDelete`, `idbClearStore`, `FingerprintRecord`, `PlaintextCacheRecord.failed`.
- `Signalix-frontend/src/lib/crypto/plaintext-cache.ts` — `cacheDecryptFailure`, `isDecryptFailureCached`.
- `Signalix-frontend/src/lib/crypto/fingerprints.ts` *(new)* — `computeSafetyNumber`, `cacheSafetyNumber`, `lookupSafetyNumber`.
- `Signalix-frontend/src/lib/crypto/crypto.service.ts` — `acknowledgeCryptoReset`, `wasReset` on `CryptoStatus`.
- `Signalix-frontend/src/components/EncryptionResetBanner.tsx` *(new)* — the dismissible amber banner.
- `Signalix-frontend/src/app/chats/layout.tsx` — banner mount.
- `Signalix-frontend/src/store/chat.store.ts` — failure-cache short-circuit in `decryptStoredMessage`.
- `Signalix-frontend/vitest.config.ts` *(new)*, `src/lib/crypto/*.test.ts` *(new)*.

---

## What's next — v0.10.0 preview

- Surface the safety number in the UI (chat profile screen → "Verify safety number" sheet).
- Multi-device fan-out: encrypt one envelope per recipient device.
- Double Ratchet between conversation pairs for per-message forward secrecy.
- Encrypted push notification previews (server passes ciphertext + envelope through to the push payload; the SW unwraps locally).
- Automatic signed pre-key rotation on a schedule.
