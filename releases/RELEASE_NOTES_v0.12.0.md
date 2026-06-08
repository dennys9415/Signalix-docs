# Signalix v0.12.0 — Release Notes

**Release date:** 2026-06-08
**Previous version:** v0.11.0

---

## Overview

v0.12.0 lights up the safety-number / device-verification UI that
the v0.9.1 fingerprint foundation was preparing for. Users can now
open any direct chat's profile, read a 12-group safety number, scan
a QR code, mark the peer as verified, and get an automatic warning
if either side's identity key rotates after the verification was
granted. No new API endpoints, no new DB rows, no contract changes —
the entire feature lives in the existing `fingerprints` IndexedDB
store and the `ContactProfileModal` component.

What ships:

- Per-peer verification snapshot in `FingerprintRecord` (`verifiedAt`,
  `verifiedLocalIdentityKey`, `verifiedPeerIdentityKey`,
  `verifiedSafetyNumber`).
- `deriveVerificationStatus` pure function that returns
  `'unknown' | 'unverified' | 'verified' | 'changed'`.
- `SignalCryptoService.getPeerVerification` / `markPeerVerified` /
  `unmarkPeerVerified`.
- `EncryptionPanel` inside `ContactProfileModal` with safety number
  + QR code + status badge + verify/unverify button + amber warning
  when the key changes.
- `qrcode` dependency (+ types).
- 5 new vitest cases — 21/21 total.

What does **not** ship:

- No production trust model — verification is local to each device's IDB.
- No cloud backup / account recovery for verification state.
- No group safety numbers — direct chats only.
- No media-specific verification.

---

## How verification flows

```
User opens chat profile
  ↓ ContactProfileModal mounts
  ↓ refresh()
  ↓   cryptoService.getPeerVerification(peerUserId)
  ↓     getKeyBundle(peerUserId)         → fresh peer identity key
  ↓     assertBundleIsValid(bundle)      → Ed25519 sig check (v0.9.1)
  ↓     maybeCacheSafetyNumber(...)      → recompute + upsert (preserves snapshot)
  ↓     lookupSafetyNumber(...)          → read updated record
  ↓     deriveVerificationStatus(record, currentLocal, currentPeer)
  ↓       • no record           → 'unknown'
  ↓       • no verifiedAt       → 'unverified'
  ↓       • snapshot == current → 'verified'
  ↓       • snapshot != current → 'changed'   ← warning banner
  ↓ render EncryptionPanel

User clicks "Mark as verified"
  ↓ cryptoService.markPeerVerified(peerUserId)
  ↓   read existing record (must already have a fingerprint)
  ↓   write { ...record, verifiedAt: now, verifiedLocalIdentityKey: record.localIdentityKey, verifiedPeerIdentityKey: record.peerIdentityKey, verifiedSafetyNumber: record.safetyNumber }
  ↓ refresh() → status is now 'verified'

Peer rotates identity (e.g. v0.9.1 reset detection on their device)
  ↓ next time we computeSafetyNumber:
  ↓   record.peerIdentityKey  is updated to the NEW peer identity (current view)
  ↓   record.verifiedPeerIdentityKey stays at the OLD peer identity (frozen snapshot)
  ↓ deriveVerificationStatus → 'changed'
  ↓ UI: amber "Security number changed" panel + badge "No longer verified"
  ↓ button now reads "Re-verify"
```

---

## Migration

- **No DB migration.** Schema unchanged.
- **No protocol break.** All wire shapes are identical to v0.11.0.
- **IDB schema unchanged.** The `fingerprints` store was added in
  v0.9.1 (`CRYPTO_DB_VERSION = 2`). New verification fields are
  optional on `FingerprintRecord`; existing rows without them just
  return `'unverified'` from `deriveVerificationStatus`.
- **`qrcode` is the only new dependency.** No bundle bloat — ~16 KB
  gzipped on top of the existing JS.

---

## Operational notes

- Deploy order: contracts → api → realtime → frontend. Only the
  frontend has real changes for v0.12.0; the other repos move in
  lockstep purely for tag alignment.
- No env var changes. No infra changes.

---

## Validation

- `npm run typecheck` — clean across all four code packages.
- `npm run build` — clean.
- `npm test` (frontend) — **21/21 vitest tests pass** (utils 7,
  fingerprints 8, file-crypto 6).

**Manual scenario** (matches the spec):

1. Alice and Bob open each other's chat profile. Both see the same
   12-group safety number (`computeSafetyNumber` orders the identity
   key pair lexicographically so the output is symmetric — already
   covered by the v0.9.1 test).
2. Alice marks Bob as verified. Badge turns green, button switches
   to "Unverify".
3. Alice refreshes the page. Status persists (verification snapshot
   is in IndexedDB).
4. Bob clears his browser data → his `init()` regenerates identity
   keys → server-side wipe (v0.10.1 cleanup) drops his old SPKs +
   pre-keys.
5. Alice opens Bob's profile again. `getKeyBundle` returns Bob's new
   identity key. `deriveVerificationStatus` compares snapshot (old)
   vs current (new) → returns `'changed'`. Alice sees the amber
   warning and "No longer verified" badge.
6. Alice talks to Bob via another channel, confirms the new safety
   number, clicks "Re-verify" → snapshot updates to the new keys →
   badge green again.

---

## Files of interest

- `Signalix-frontend/src/lib/crypto/db.ts` — extended `FingerprintRecord`.
- `Signalix-frontend/src/lib/crypto/fingerprints.ts` — `deriveVerificationStatus`, `markPeerVerified`, `unmarkPeerVerified`, snapshot-preserving `cacheSafetyNumber`.
- `Signalix-frontend/src/lib/crypto/fingerprints.test.ts` — 5 new cases.
- `Signalix-frontend/src/lib/crypto/signal.service.ts` — `getPeerVerification`, `markPeerVerified`, `unmarkPeerVerified` on the service.
- `Signalix-frontend/src/lib/crypto/crypto.types.ts` — interface additions.
- `Signalix-frontend/src/lib/crypto/crypto.mock.ts` — mock additions.
- `Signalix-frontend/src/components/ContactProfileModal.tsx` — `EncryptionPanel`, `SafetyNumberDisplay`, `QRCodePanel`, `StatusBadge`.
- `Signalix-frontend/package.json` — `qrcode` + `@types/qrcode`.

---

## What's next — v0.13.0 preview

- Sender Keys for groups (drop the O(participants) cost on text + media).
- Group safety numbers — multi-member ceremony.
- Encrypted push previews (SW-side decrypt).
- Signed-URL media access.
- Link-preview encryption.
- Avatar / profile photo encryption.
