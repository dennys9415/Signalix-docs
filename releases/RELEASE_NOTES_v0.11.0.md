# Signalix v0.11.0 — Release Notes

**Release date:** 2026-06-08
**Previous version:** v0.10.1

---

## Overview

v0.11.0 closes the last "the server can see X" gap on the message
pipeline: **image, file, and voice attachments are now encrypted
client-side before upload**. The API and MinIO only store AES-256-GCM
ciphertext under an `encrypted/{userId}/{uuid}.bin` prefix; the
per-attachment media key rides inside the existing per-recipient
envelope (same X3DH-style fan-out that protects text since v0.10.x), so
multi-device hand-off and group fan-out work automatically without any
new key-management surface.

What ships:

- Client-side AES-256-GCM file encryption with a fresh per-attachment key.
- New `POST /api/v1/media/encrypted-blob` endpoint that stores opaque ciphertext (no MIME validation, 25 MB cap).
- Encrypted attachment metadata schema (`MediaMetadataV1 = { v:1, url, k, iv, mime, size, filename?, duration? }`) — the JSON is what the sender encrypts and the recipient decrypts; the wire ciphertext is the existing per-recipient envelope.
- Receive-side rendering via a `useDecryptedBlobUrl` hook that fetches + decrypts the blob and produces a `blob:` URL for `<img>` / `<audio>` / file download.
- `[Unable to decrypt attachment]` sentinel + `BrokenAttachment` tile when decrypt fails.
- 6 new vitest unit tests on the file-encryption helpers; 16/16 frontend tests passing.

What does **not** ship:

- No backup / recovery for media keys (same model as text).
- Link previews, avatars, and profile photos remain plaintext.
- No streaming decrypt — the whole blob is decrypted in memory.
- No signed-URL access control — encrypted blobs are at public MinIO URLs (knowing the URL only gets you ciphertext).

---

## End-to-end flow

```
Alice (sender)
──────────────
pick image / file / voice
  ↓ MessageInput
  ↓ encryptFile(blob)              → { ciphertext, key, iv }
  ↓ uploadEncryptedBlob(ciphertext) → { url, size }       (POST /media/encrypted-blob)
  ↓ build MediaMetadataV1 JSON     = { v:1, url, k, iv, mime, size, [filename], [duration] }
  ↓ onSend(metadataJson, …, MessageType.IMAGE | FILE | AUDIO)
  ↓
chat.store.dispatchSend
  ↓ isEncryptable = TEXT | IMAGE | FILE | AUDIO   ← v0.11.0 widened
  ↓ encryptForUserAllDevices(metadataJson, recipientUserId)
  ↓ → recipients[] (one envelope per device)
  ↓ wsClient.sendMessageSend({ ciphertext:'', recipients, messageType })
  ↓
api.MessagesService.sendMessage
  ↓ recipients[] accepted for any TEXT/IMAGE/FILE/AUDIO   ← v0.11.0 widened guard
  ↓ INSERT messages (ciphertext='', encryption_version=1, message_type)
  ↓ INSERT group_message_recipients × N
  ↓ recipientPayloads keyed by deviceId
  ↓
realtime.event-router.onMessageSend
  ↓ per participant device → server.message.new with that device's envelope
  ↓
Bob (recipient)
───────────────
chat.store MESSAGE_NEW
  ↓ decryptStoredMessage()                    ← v0.11.0 no longer skips non-TEXT
  ↓   decryptIncoming(envelope) → metadataJson
  ↓   ciphertext for the stored row = metadataJson
  ↓
MessageView render
  ↓ parseImageInfo / parseFileInfo / parseVoiceInfo → { url, k, iv, mime, … }
  ↓ useDecryptedBlobUrl({ url, k, iv, mime })
  ↓   fetch(url) → AES-GCM open with (k, iv) → blob: URL
  ↓ <img src={blobUrl} />  or  <audio src={blobUrl} />  or  Download → saveAs
```

---

## Migration

- **No DB migration.**
- **No protocol break.** All wire additions are additive; v0.10.x clients still talk to a v0.11.0 server (they just can't send/receive encrypted attachments).
- **Legacy plaintext rows keep working.** `parseImageInfo` / `parseFileInfo` / `parseVoiceInfo` detect the `v:1` envelope and fall back to the legacy shapes (raw URL for image, `{url,name,size}` for file, `{url,duration,size}` for audio). `useDecryptedBlobUrl` returns the raw URL when no key material is present, so legacy `<img src>` semantics are preserved.

---

## Operational notes

- Deploy order: contracts → api → realtime → frontend.
- MinIO admins: the new object key prefix is `encrypted/{userId}/{uuid}.bin` under the existing media bucket. No bucket-level policy change required.
- Plaintext upload endpoints (`/media/upload`, `/media/voice`, `/files/upload`) remain available for any legacy callers / debugging. New attachments from v0.11.0+ clients always go through `/media/encrypted-blob`.
- Existing plaintext attachments in MinIO are not affected and continue to render via the legacy code path.

---

## Validation

- `npm run typecheck` — clean across all four code packages.
- `npm run build` — clean.
- `npm test` (frontend) — 16/16 vitest tests pass (utils 7, fingerprints 3, file-crypto 6).

**Manual scenario:**

1. Send an image from Alice. Inspect MinIO — the object under `encrypted/...` is not a viewable image; downloading + opening it yields garbage.
2. Bob receives the message — sees the decrypted image render normally.
3. Send a file (PDF, ZIP, etc) — Bob downloads it; the file opens correctly.
4. Send a voice note — Bob plays it back.
5. Direct + group chat both work.
6. Text E2EE unchanged.
7. Alice's own history of attachments survives a refresh because the plaintext cache stores the decrypted metadata JSON.

---

## Files of interest

**Frontend**
- `src/lib/crypto/file-crypto.ts` *(new)*
- `src/lib/crypto/file-crypto.test.ts` *(new)*
- `src/lib/crypto/use-decrypted-blob-url.ts` *(new)*
- `src/lib/crypto/signal.service.ts` — `DECRYPT_FAILED_ATTACHMENT_PLACEHOLDER`
- `src/lib/crypto/crypto.service.ts` — re-export
- `src/lib/api-client.ts` — `uploadEncryptedBlob`
- `src/components/MessageInput.tsx` — encrypt-then-upload paths
- `src/components/MessageView.tsx` — `ImageBubble`, `BrokenAttachment`, `parseImageInfo`, updated `parseFileInfo` / `parseVoiceInfo`, updated `FileCard` / `VoiceBubbleSection`
- `src/store/chat.store.ts` — `isEncryptable` widened, `decryptStoredMessage` covers non-TEXT, attachment-specific failure placeholder

**API**
- `src/media/media.controller.ts` — `POST /media/encrypted-blob`
- `src/media/media.service.ts` — `uploadEncryptedBlob`
- `src/messages/messages.service.ts` — lifted TEXT-only guard on `recipients[]`

---

## What's next — v0.12.0 preview

- Sender Keys for groups (drop O(participants) cost on text + media alike).
- Encrypted push notification previews (SW-side decrypt).
- Signed-URL access control for encrypted blobs.
- Link-preview encryption.
- Avatar / profile photo encryption.
