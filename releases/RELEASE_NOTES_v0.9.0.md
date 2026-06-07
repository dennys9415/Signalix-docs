# Signalix v0.9.0 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.8.0

---

## Overview

v0.9.0 is the **Signal Protocol Beta** release for Signalix. It turns the v0.8.0 crypto foundation into a working end-to-end encryption layer — for **direct text messages only**.

What ships:

- Real X25519 + Ed25519 key generation in the browser via Web Crypto.
- Encrypted direct text messages between v0.9.0+ clients (X25519 ECDH + HKDF-SHA256 + AES-256-GCM under a `JSON.stringify({ v: 1, c, iv, eph })` envelope).
- IndexedDB-backed key storage and sender plaintext cache.
- Auto-init on every successful login, top-up of one-time pre-keys when the pool runs low.
- UX: 🔒 "End-to-end encrypted beta" pill in direct chat headers; `"[Unable to decrypt message]"` rendering for any bubble that can't be opened.
- Plaintext fallback on every failure mode so transitioning users never lose messages.

What does **not** ship:

- Double Ratchet (no per-message forward secrecy beyond signed-pre-key rotation cadence).
- Multi-device fan-out (the sender picks the first bundle returned).
- Server-side signed-pre-key signature verification (the API still stores opaque blobs).
- Local pre-key deletion after first use.
- Encrypted push notification previews.
- Group encryption, media/file/voice encryption, key-verification UX (safety numbers).

These are explicit v0.10.0 / v0.11.0 work items. **v0.9.0 is not production-grade E2EE** and the in-product banner says so.

---

## How a direct text message flows

```
sender                                                     recipient
──────                                                     ─────────
plaintext "hello"
   ↓ chat.store.sendMessage
   ↓ tempMsg.ciphertext = "hello"  (local UI)
   ↓ dispatchSend → resolveDirectRecipient → recipientUserId
   ↓ cryptoService.encryptForRecipient(plaintext, { recipientUserId })
   ↓     GET /api/v1/crypto/users/:userId/key-bundle
   ↓     pick first bundle = { identityKey, signedPreKey, preKey? }
   ↓     gen ephPair (X25519)
   ↓     IKM = ECDH(eph, SPK) [|| ECDH(eph, OTP)]
   ↓     key = HKDF-SHA256(IKM, info="signalix-v1-direct-text")
   ↓     iv  = random 12 bytes
   ↓     ct  = AES-256-GCM(key, iv, plaintext)
   ↓     envelope = JSON.stringify({ v: 1, c: b64u(ct), iv: b64u(iv), eph: b64u(ephPub) })
   ↓ WS client.message.send {
   ↓     ciphertext: envelope,
   ↓     encryptionVersion: 1,
   ↓     senderDeviceId, recipientDeviceId, preKeyId?, signedPreKeyId
   ↓ }
   ↓
   ↓                                                             API
   ↓                                                             ──
   ↓                                                             persists row
   ↓                                                             messages.ciphertext = envelope (opaque to us)
   ↓                                                             messages.encryption_version = 1
   ↓                                                             … envelope columns set …
   ↓                                                             emits MESSAGE_SENT to sender, MESSAGE_NEW to recipient
   ↓
sender                                                     recipient
MESSAGE_SENT { messageId, ... }                            MESSAGE_NEW { ciphertext=envelope, encryptionVersion=1, ... }
cachePlaintext(messageId, "hello")                          ↓ insert raw envelope into store
                                                            ↓ async: decryptIncoming(envelope)
                                                            ↓     parse JSON → { v, c, iv, eph }
                                                            ↓     look up local SPK private (by signedPreKeyId)
                                                            ↓     look up local OTP private (by preKeyId, if any)
                                                            ↓     IKM = ECDH(ephPub, SPK_priv) [|| ECDH(ephPub, OTP_priv)]
                                                            ↓     same HKDF → same AES key
                                                            ↓     decrypt → plaintext "hello"
                                                            ↓ cachePlaintext(messageId, "hello")
                                                            ↓ replace ciphertext with plaintext in store
                                                            ↓ bubble re-renders
```

History reload (`loadMessages`) walks every returned message through the same decrypt path, with the local plaintext cache short-circuiting messages we already opened (or that we sent ourselves and cached on send).

---

## Fixed during release

A first cut of v0.9.0 shipped with two missed lines in `Signalix-realtime/src/ws/event-router.onMessageSend`:

1. The `api.sendMessage(...)` call did not extract `payload.{encryptionVersion, senderDeviceId, recipientDeviceId, preKeyId, signedPreKeyId}` from the incoming WS frame. The API DTO accepted them since v0.8.0, but the WS hop was dropping them — so the API persisted rows with `encryption_version = 0` and the envelope columns NULL.
2. The construction of `ServerMessageNewPayload` was not spreading the same fields from the API's returned `MessageDTO`. So even if the columns had been persisted, the recipient never received the `encryptionVersion >= 1` flag over WS. The client decrypt path didn't trigger, and the chat UI rendered the raw JSON envelope `{"v":1,"c":"…","iv":"…","eph":"…"}` verbatim.

Both lines now spread the envelope fields end-to-end. The wire format is unchanged — these fields were already defined on `ClientMessageSendPayload` and `ServerMessageNewPayload` since v0.8.0; only the realtime forwarding was missing.

Companion frontend changes:

- `signal.service.decryptIncoming` now `await`s the in-flight `initPromise` before reading IndexedDB. A `MESSAGE_NEW` arriving between login and the completion of the initial key publish used to hit empty IDB and render `[Unable to decrypt message]`.
- Dev-only diagnostic logs: `[signalix-crypto] decrypt attempt` / `success` / `failed` with the reason. Gated on `NODE_ENV !== 'production'`.

## What changed

### Frontend

| Piece | Where |
|---|---|
| `SignalCryptoService` | `src/lib/crypto/signal.service.ts` |
| IndexedDB schema (`signalix-crypto-v1`) | `src/lib/crypto/db.ts` |
| Plaintext cache | `src/lib/crypto/plaintext-cache.ts` |
| base64url / X25519 / HKDF helpers | `src/lib/crypto/utils.ts` |
| Swap point (mock vs signal) | `src/lib/crypto/crypto.service.ts` |
| API wrappers (`registerDeviceKeys`, `rotateSignedPreKey`, `uploadPreKeys`, `getKeyBundle`) | `src/lib/api-client.ts` |
| Init on login | `src/store/auth.store.ts` |
| Encrypt on send, decrypt on receive/load, cache plaintext on `MESSAGE_SENT` | `src/store/chat.store.ts` |
| Lock pill + failure-bubble rendering | `src/components/MessageView.tsx` |

### Backend

No code changes. The v0.8.0 endpoints serve v0.9.0 clients as-is. The 5 envelope columns on `messages` from V14 now carry real values.

### Contracts

No code changes. The optional envelope fields added in v0.8.0 are now populated by v0.9.0 clients.

### Infra

`NEXT_PUBLIC_E2EE_DEV_FALLBACK` build-arg added to `docker-compose.yml` `frontend.build.args` and to `Signalix-frontend/Dockerfile`. Default `false`.

---

## Env vars

| Name | Default | Purpose |
|---|---|---|
| `NEXT_PUBLIC_E2EE_DEV_FALLBACK` | `false` | When `true`, the frontend uses the v0.8.0 plaintext-passthrough mock instead of the real Signal service. Useful for local dev against unpublished peers. |

---

## Validation

```bash
cd Signalix-contracts && npm run typecheck && npm run build   # ✓ (no contract changes)
cd Signalix-api       && npm run typecheck && npm run build   # ✓ (no code changes)
cd Signalix-realtime  && npm run typecheck                    # ✓ (unchanged)
cd Signalix-frontend  && npm run typecheck && npm run build   # ✓
```

Manual checks against the local Docker Compose stack:

1. Stack up; flyway applies V14 (already in v0.8.0).
2. Login as user A. DevTools → IndexedDB → `signalix-crypto-v1` → `identity` should contain the freshly-generated record.
3. Network tab: `POST /api/v1/crypto/devices/keys` returns 201 with `{ deviceId, preKeyCount: 100 }`.
4. Open a direct chat with user B (who has also logged in once). Send a TEXT message.
5. SQL: `SELECT encryption_version, sender_device_id, recipient_device_id, pre_key_id, signed_pre_key_id, LEFT(ciphertext, 80) FROM messages WHERE chat_id = '<id>' ORDER BY created_at DESC LIMIT 1;`. `encryption_version` should be `1`; `ciphertext` should start with `{"v":1,"c":"...` (the JSON envelope).
6. User B's session: the message renders as plaintext. Sender sees the same plaintext locally (sourced from the in-memory tempMsg, then cached to IndexedDB on MESSAGE_SENT).
7. Refresh user B's tab. History reload decrypts every prior message from this chat.
8. Refresh user A's tab. Sent messages still readable (loaded from IndexedDB cache).
9. Send an **image / file / voice note** in the same chat — it flows unencrypted (per scope). Backend sees the media URL / metadata as before.
10. Group chat: open one, send a text. Backend sees plaintext (groups stay unencrypted in v0.9.0).
11. Set `NEXT_PUBLIC_E2EE_DEV_FALLBACK=true`, rebuild. `cryptoService.isReady()` from the frontend console returns `true` but `getCryptoStatus().e2eeActive` returns `false` and outgoing messages use `encryption_version: 0` again.
12. Provoke decrypt failure (e.g. clear IndexedDB while keeping the auth session) and reload an old chat. Affected bubbles render as `"[Unable to decrypt message]"` in italic + open padlock.

---

## Known limitations

(Lifted into the README intro disclaimer and the in-product banner.)

- **No Double Ratchet.** Single ephemeral keypair per message. Forward secrecy is bounded by signed-pre-key rotation cadence; v0.10.0 adds chain keys.
- **No multi-device fan-out.** A recipient with multiple devices only receives the message on the first device the bundle endpoint returns.
- **No signature verification.** The server accepts whatever Ed25519 signature the client posts on the signed pre-key; clients don't verify it either. v0.10.0 fixes both ends.
- **Pre-keys are not deleted locally** after first use — history reload still decrypts.
- **Sender history on a fresh device** can't display previously-sent messages (the plaintext cache lives in the original device's IndexedDB). v0.10.0 adds sender-side archival.
- **Push notification previews** still read the encrypted ciphertext blob. Recipients see useless previews — v0.10.0 routes encrypted messages through a generic "New encrypted message" body.
- **No safety-number UX** — users can't verify they're talking to the right device.
- **Groups, images, files, voice notes, reactions metadata, edited bodies** all remain plaintext on the server.

---

## Compatibility

- **v0.7.x / v0.8.0 clients ↔ v0.9.0 server** — works. Old clients ignore envelope fields; their outgoing messages stay plaintext (`encryption_version=0`).
- **v0.9.0 client ↔ v0.8.0 peer** — partial: v0.9.0 tries to encrypt and the peer has no published key bundle, so the send falls back to plaintext. As soon as the peer logs in on v0.9.0, the next message becomes encrypted.
- **Mixed-version groups** — irrelevant; groups stay plaintext until v0.11.0.

---

## Roadmap signal

- **v0.10.0** — Double Ratchet, multi-device fan-out, server-side signed-pre-key signature verification, one-time pre-key deletion after first use, encrypted push previews, safety-number UX, automatic signed-pre-key rotation timer.
- **v0.11.0+** — group encryption via Sender Keys, media/file/voice encryption (encrypt-then-upload-to-MinIO), key change notifications, key transparency.
