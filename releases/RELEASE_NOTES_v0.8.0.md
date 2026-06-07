# Signalix v0.8.0 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.7.1

---

## Overview

v0.8.0 lays down the **encryption foundation** for Signalix. It is *not* an E2EE release.

What ships:

- Database tables for device identity, signed pre-keys, and one-time pre-keys.
- Four REST endpoints for publishing / rotating / uploading / fetching keys.
- Optional envelope columns on `messages` and matching optional fields on the message contracts.
- A frontend `lib/crypto/` abstraction with a mock implementation that passes plaintext through.

What does **not** ship:

- Real Signal Protocol.
- Any actual encryption of message bodies.
- Signature verification of submitted signed pre-keys.
- Wiring of the crypto service into the send / receive code path.

The server still receives plaintext in `messages.ciphertext`. **v0.9.0 is the planned E2EE beta.**

---

## Why scaffold ahead of the protocol

E2EE is a cross-cutting change: schema, contracts, API, frontend, key management UX, multi-device handling, and recovery flows all need to land together. Trying to ship them all in one release means a long, risky changelist; trying to bolt them on one-by-one means breaking the wire format multiple times.

v0.8.0 takes the **low-risk slice** that future iterations need:

- **Schema** — adds the tables and the message envelope columns so v0.9.0 doesn't need a coordinated migration alongside a code freeze.
- **Wire format** — the optional `encryptionVersion`, `senderDeviceId`, `recipientDeviceId`, `preKeyId`, `signedPreKeyId` fields land in the contracts now. v0.7.x clients ignore them; future clients can populate them without a contract break.
- **Abstraction** — `cryptoService` is a singleton today, mocked. v0.9.0 will replace one line of import to swap in the real implementation.

No callers depend on encryption being real. No user-facing change in v0.8.0.

---

## Database

### Migration `V14__crypto_foundation.sql`

| Table / Column | Purpose |
|---|---|
| `device_identity_keys(device_id PK, registration_id, identity_key, signing_key, key_algorithm, registered_at, updated_at)` | Per-device long-term identity. Upsert on register. `registration_id` is the Signal-style 32-bit session disambiguator. |
| `signed_pre_keys(device_id, key_id, public_key, signature, key_algorithm, created_at, rotated_at)` PK `(device_id, key_id)` | Rotated periodically. Old rows kept with `rotated_at` set for handshakes that referenced them. Partial index `WHERE rotated_at IS NULL`. |
| `pre_keys(device_id, key_id, public_key, key_algorithm, created_at, consumed_at)` PK `(device_id, key_id)` | One-time pre-keys. Atomically consumed via `UPDATE ... WHERE (device_id, key_id) IN (SELECT ... FOR UPDATE SKIP LOCKED)`. Partial index on unconsumed. |
| `messages.encryption_version INTEGER NOT NULL DEFAULT 0` | 0 == plaintext. 1+ reserved for future protocol versions. |
| `messages.sender_device_id UUID` | Reserved for v0.9.0. |
| `messages.recipient_device_id UUID` | Reserved for v0.9.0. |
| `messages.pre_key_id INTEGER` | Reserved for v0.9.0. |
| `messages.signed_pre_key_id INTEGER` | Reserved for v0.9.0. |

All key material is stored as raw `BYTEA` and exchanged as `base64url` on the wire.

---

## API endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/crypto/devices/keys` | JWT | Initial publish: identity + signing + signed pre-key + initial one-time pre-key batch. Idempotent. DeviceId from JWT. |
| PATCH | `/api/v1/crypto/devices/keys/signed-pre-key` | JWT | Periodic rotation. Marks the prior signed pre-key as `rotated_at = NOW()`. |
| POST | `/api/v1/crypto/devices/keys/pre-keys` | JWT | Top-up of one-time pre-keys. Returns new unconsumed count. |
| GET | `/api/v1/crypto/users/:userId/key-bundle` | JWT | Returns every device bundle for the target user. Each call atomically consumes one one-time pre-key per device; falls back to signed-pre-key-only when the pool is exhausted. |

`CryptoModule` owns the storage layer; `class-validator` enforces base64url + size bounds on every key blob.

**No signature verification yet.** v0.9.0 will validate `signedPreKey.signature` against `signingKey` before persisting.

---

## Contracts

New file `api/crypto.contract.ts`:

- `DeviceIdentityKeyDTO`, `PreKeyDTO`, `SignedPreKeyDTO`, `DeviceKeyBundleDTO`, `KeyBundleResponse`.
- `RegisterDeviceKeysRequest`/`Response`, `RotateSignedPreKeyRequest`/`Response`, `UploadPreKeysRequest`/`Response`.
- `KeyAlgorithm` string-literal union (currently `'x25519'`; pinning a value here is the future-proof handle if we add post-quantum algorithms).

`MessageDTO`, `SendMessageRequest`, `ClientMessageSendPayload`, `ServerMessageNewPayload` each gain five optional envelope fields. v0.7.x clients ignore them; the server persists them when provided.

---

## Frontend

New `src/lib/crypto/`:

- `crypto.types.ts` — `CryptoService` interface, `EncryptedEnvelope`, `CryptoStatus`, plus re-exports of contracts crypto DTOs.
- `crypto.service.ts` — exports singleton `cryptoService` + `getCryptoStatus()`. **Swap point** for v0.9.0.
- `crypto.mock.ts` — `MockCryptoService`. `encryptForRecipient(plaintext)` returns `{ ciphertext: plaintext, encryptionVersion: 0 }`. `decryptIncoming` returns plaintext, or `[encrypted message — upgrade to view]` if a non-zero version slips in from a future client.

The service is **not** integrated into the send / receive pipeline yet — `chat.store.sendMessage` and the WS handlers still operate on plaintext. That wiring is the v0.9.0 task.

---

## Migrations

| Migration | Purpose |
|---|---|
| `V14__crypto_foundation.sql` | Identity + signed + one-time key tables, plus 5 envelope columns on `messages`. Scaffolding only. |

---

## New environment variables

None.

---

## Validation

```bash
cd Signalix-contracts && npm run typecheck && npm run build   # ✓
cd Signalix-api       && npm run typecheck && npm run build   # ✓
cd Signalix-realtime  && npm run typecheck                    # ✓ (unchanged)
cd Signalix-frontend  && npm run typecheck && npm run build   # ✓
```

Manual checks against the local Docker Compose stack:

1. Stack starts; Flyway applies V14 automatically. Existing chats / messages remain readable; sending text works exactly as v0.7.1.
2. From the browser console with a logged-in session, call the new endpoints with fake base64url blobs — all four return 200 / 201; the bundle endpoint claims one pre-key on each call.
3. `getCryptoStatus()` from the frontend console returns `{ e2eeActive: false, implementation: 'mock (plaintext passthrough — v0.8.0 foundation)' }`.
4. Voice notes, files, images, reactions, replies, forwards, edit, delete, search, push notifications — every prior feature still works.

---

## Compatibility

- v0.7.x clients keep working against a v0.8.0 server (the new fields are optional on the wire, and the server defaults `encryption_version = 0`).
- A v0.8.0 client running against a v0.7.x server **must not** call the new crypto endpoints — they'll 404. The frontend doesn't call them in v0.8.0, so this is fine in practice.

---

## Known limitations

- **NO actual E2EE.** The header is loud about this on purpose. Anyone reading the changelog or pulling the release should know that messages remain plaintext on the server.
- **No signature verification.** The server accepts whatever signed-pre-key signature the client posts; verifying that the signature is well-formed against `signingKey` is a v0.9.0 task.
- **No automatic key rotation.** Devices have to call PATCH `/crypto/devices/keys/signed-pre-key` themselves; there's no server-side reminder yet.
- **Single key algorithm.** Only `x25519` is currently surfaced; the column is present so future algorithms can land additively.
- **Mock service in production builds.** If you ship a v0.8.0 build to production users, the encryption banner shows `e2eeActive: false`. Don't claim production-grade encryption.
