# Signalix v0.10.0 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.9.1

---

## Overview

v0.10.0 is the **Group E2EE beta**. The v0.9.x direct-text encryption now covers group text messages too, via **per-recipient encryption fan-out**: the sender runs the X3DH-style handshake once per recipient device and ships N envelopes, one per recipient. The server stores each envelope in a new table; the realtime layer delivers each recipient only their own copy. This is **not** production-grade Sender Keys — Sender Keys is v0.11.0+.

What ships:

- **Per-recipient fan-out** for group TEXT messages: sender encrypts N times → server persists per-recipient rows → realtime delivers per-recipient `server.message.new`.
- **Edit re-fan-out**: editing a group encrypted text message re-encrypts per recipient and replaces the per-recipient rows in a single transaction.
- **API + contracts**: `GroupRecipientPayloadDTO`, extended `SendMessageRequest` / `EditMessageRequest` / WS payloads, new `RecipientEnvelopeDTO` for the API → realtime hand-off.
- **Database migration `V15__group_message_recipients.sql`** — one row per (message × recipient × device).
- **UX**: 🔒 *End-to-end encrypted beta* pill in group chat headers with a tooltip noting media/files/voice remain unencrypted.

What does **not** ship:

- Sender Keys.
- Multi-device fan-out (single-device assumption from v0.9.x carries over).
- Group media / files / voice E2EE.
- Encrypted push previews.
- Safety-number verification UI (foundation already in v0.9.1).
- Reply preview decryption for quotes of group encrypted messages (UI shows empty quote bubble).

---

## How a group encrypted message flows

```
sender (Alice) — group {Alice, Bob, Charlie}
──────────────
plaintext "hello group"
  ↓ chat.store.sendMessage
  ↓ tempMsg.ciphertext = "hello group"          (local optimistic UI)
  ↓ dispatchSend → resolveGroupRecipientIds → [Bob, Charlie]
  ↓ encryptForGroup(plaintext, [Bob, Charlie])
  ↓   await Promise.all(
  ↓     cryptoService.encryptForRecipient(plaintext, {recipientUserId: Bob}),
  ↓     cryptoService.encryptForRecipient(plaintext, {recipientUserId: Charlie}),
  ↓   )
  ↓   → [
  ↓       {recipientDeviceId: Bob.dev, ciphertext: env1, encryptionVersion: 1, ...},
  ↓       {recipientDeviceId: Charlie.dev, ciphertext: env2, encryptionVersion: 1, ...},
  ↓     ]
  ↓ wsClient.sendMessageSend({ ciphertext: '', recipients: [...] })
  ↓
realtime
  ↓ onMessageSend → api.sendMessage(...)        (forwards recipients[])
  ↓
api
  ↓ MessagesService.sendMessage
  ↓   assertRecipientsAreParticipants
  ↓   INSERT messages (ciphertext='', encryption_version=1, sender_device_id=Alice.dev)
  ↓   INSERT group_message_recipients × 2 (Bob row, Charlie row)
  ↓   returns { message, recipientPayloads: { Bob.id: …, Charlie.id: … } }
  ↓
realtime
  ↓ per participant in chat:
  ↓   if recipientPayloads has them → personalized server.message.new
  ↓   else (= sender's other devices) → top-level (empty) — sender uses local cache
  ↓
recipient (Bob)
  ↓ chat.store.MESSAGE_NEW handler
  ↓ decryptStoredMessage(m) → decryptIncoming(envelope)
  ↓ ECDH + HKDF + AES-GCM open → "hello group"
  ↓ cachePlaintext(m.id, m.chatId, "hello group")
```

### Refresh path

- **Recipient refresh** — `GET /chats/:chatId/messages` joins `group_message_recipients` on `recipient_user_id = caller.id`, returns the per-recipient ciphertext + envelope. Frontend `decryptStoredMessage` finds the plaintext-cache hit (set on first decrypt) or runs `decryptIncoming` again.
- **Sender refresh** — no recipient row exists for Alice. Server returns empty top-level ciphertext + `encryption_version=1`. Frontend `lookupPlaintext` hits the cache populated on `server.message.sent`; if the cache was cleared, Alice sees `[Unable to decrypt message]` (documented limitation, same as v0.9.x direct).
- **New joiner** — no recipient row for any message that predates their join. Old messages render as `[Unable to decrypt message]`. This is the intended v0.10.0 property.

---

## Migration

- **Run `V15__group_message_recipients.sql`** before deploying the new api / realtime / frontend images. The new table is empty on first apply; nothing else is touched.
- **No protocol break.** A v0.9.x client talking to a v0.10.0 server still works for direct chats. A v0.9.x client sending into a group will send plaintext as before — the v0.10.0 server has no special handling for that case and stores plaintext, same as today.
- **No infra changes.** Same containers, ports, env vars as v0.9.x.
- **Frontend IDB unchanged.** v0.9.1's schema v2 covers v0.10.0.

---

## Operational notes

- After deploy, watch the `MessagesService` logger for `VALIDATION_ERROR` rejections from the new fan-out path — these are the signal that some client is shipping malformed `recipients[]`.
- The `recipientPayloads` map in the API response is internal to the realtime layer. Clients never see it directly.
- **Plaintext fallback.** If `cryptoService.encryptForRecipient` throws for *any* recipient in a group, the whole send drops to plaintext (encryption_version=0). This avoids leaking the body to the recipients we did succeed for while leaving others stuck on `[Unable to decrypt]`.

---

## Validation

- `npm run typecheck` — clean across `Signalix-contracts`, `Signalix-api`, `Signalix-realtime`, `Signalix-frontend`.
- `npm run build` — clean in all four repos.
- `npm test` (frontend) — 10/10 vitest unit tests still pass.

**Manual scenario** (matches the spec):

1. Create group `{Alice, Bob, Charlie}`.
2. Alice sends "hello group" → DB `messages.ciphertext = ''`, two rows in `group_message_recipients`.
3. Bob sees "hello group". Charlie sees "hello group". Alice sees "hello group" from local cache.
4. Refresh Bob and Charlie → still "hello group" (history join returns their envelope; decrypt + cache).
5. Add Dave to the group → Dave loads history, all old rows show `[Unable to decrypt message]` (no recipient row for Dave). New messages sent after the add include Dave in `recipients[]`.
6. Direct chats still work (verified by `resolveDirectRecipient` short-circuit + 10/10 unit tests on the crypto layer).

---

## Files of interest

**Contracts**

- `Signalix-contracts/src/api/message.contract.ts` — `GroupRecipientPayloadDTO`, `RecipientEnvelopeDTO`, request/response extensions.
- `Signalix-contracts/src/websocket/payloads.contract.ts` — `recipients?` on send/edit; envelope fields on edit + edited payloads.

**API**

- `Signalix-api/migrations/V15__group_message_recipients.sql` *(new)*.
- `Signalix-api/src/messages/messages.service.ts` — `sendMessage` + `editMessage` fan-out, `assertRecipientsAreParticipants`, push-preview fallback.
- `Signalix-api/src/messages/dto/send-message.dto.ts` *(extended)*, `dto/edit-message.dto.ts` *(extended)*.
- `Signalix-api/src/chats/chats.service.ts` — `getMessages` per-recipient JOIN + override mapping.
- `Signalix-api/src/messages/messages.controller.ts` — `edit` controller now forwards the full DTO.

**Realtime**

- `Signalix-realtime/src/ws/event-router.ts` — per-recipient broadcast in `onMessageSend` + `onMessageEdit`.
- `Signalix-realtime/src/common/api-client.ts` — `sendMessage` + `editMessage` payload-shape changes.

**Frontend**

- `Signalix-frontend/src/store/chat.store.ts` — `resolveGroupRecipientIds`, `encryptForGroup`, `dispatchSend` group path, `dispatchEdit`, `cachePlaintext` on edit.
- `Signalix-frontend/src/components/MessageView.tsx` — group chat header pill.

---

## What's next — v0.11.0 preview

- **Sender Keys for groups** — replace `O(participants)` per-send with a per-chat group key + ratchet. Drops bandwidth + encrypt cost to `O(1)`.
- **Group media / file / voice encryption** — encrypt-then-upload via the same per-recipient envelope (or Sender Keys, depending on order of operations).
- **Encrypted push previews** — the server forwards opaque ciphertext to push; the service worker decrypts client-side.
- **Reply preview JOIN** so quotes of group encrypted messages decrypt correctly for the viewer.
