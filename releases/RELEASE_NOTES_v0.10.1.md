# Signalix v0.10.1 — Release Notes

**Release date:** 2026-06-08
**Previous version:** v0.10.0

---

## Overview

v0.10.1 is a hardening release on top of v0.10.0. It closes the intermittent direct-chat decrypt failures that surfaced once the group beta was running in Docker (root cause: single-device sender assumption + orphan server-side pre-keys from prior IDB wipes), and lands the first non-message realtime broadcast — `server.chat.created` — so newly-created groups appear in every participant's sidebar without a manual refresh.

What ships:

- **Multi-device direct E2EE** — the sender encrypts to *every* device the recipient has published, not just `bundles[0]`. Brave + Chrome on the same account both decrypt.
- **Per-device payload routing** — `recipientPayloads` is keyed by `deviceId` (not `userId`); `getMessages` JOIN filters by `(user, device)` so multi-device users don't see duplicated rows.
- **Server-side stale-key wipe** — `registerDeviceKeys` detects an identity change and `DELETE`s the device's prior `signed_pre_keys` + `pre_keys` in the same transaction, so `getKeyBundle` can never hand out a key id the recipient's IDB no longer has.
- **Forced one-time client cleanup** — a `localStorage` flag triggers wipe + re-register on the first init after deploy, so existing deployments converge without manual DB surgery.
- **Decrypt-failure cache cleared on init** — transient failures auto-heal on next page load.
- **MESSAGE_NEW flicker fix** — encrypted-text bubbles render plaintext directly instead of flashing the envelope JSON for ~50ms.
- **`server.chat.created` realtime event** — new group appears immediately in Bob & Charlie's sidebar.
- **Consolidated decrypt diagnostics** — single dev-only log line per attempt with the full local key inventory; flippable to production via a runtime constant.

What does **not** ship:

- No new DB migration (schema unchanged since V15).
- No Sender Keys yet (still per-recipient fan-out — Sender Keys is v0.11.0+).
- No encrypted media / files / voice.
- No safety-number verification UI (foundation present since v0.9.1, UI is v0.11.0+).
- No infra changes — no new env vars, no new build args.

---

## How a direct message flows now

```
sender (Alice, 1 device) → recipient (Bob, 2 devices: Chrome + Brave)
──────────────────────────────────────────────────────────────────────
plaintext "hi"
  ↓ chat.store.dispatchSend
  ↓ resolveDirectRecipient → Bob
  ↓ encryptForUser(Bob)
  ↓   getKeyBundle(Bob) → [{deviceId: BobChrome, …}, {deviceId: BobBrave, …}]
  ↓   encryptToBundle(plaintext, BobChrome)  → envelope_A (recipientDeviceId=BobChrome)
  ↓   encryptToBundle(plaintext, BobBrave)   → envelope_B (recipientDeviceId=BobBrave)
  ↓ wsClient.sendMessageSend({ ciphertext:'', recipients:[A, B] })
  ↓
api.MessagesService.sendMessage
  ↓ INSERT messages (ciphertext='', encryption_version=1, sender_device_id=Alice)
  ↓ INSERT group_message_recipients × 2 (BobChrome row, BobBrave row)
  ↓ returns { recipientPayloads: { BobChromeDeviceId: …, BobBraveDeviceId: … } }
  ↓
realtime.event-router.onMessageSend
  ↓ for each participant connection:
  ↓   override = recipientPayloads[ participantConn.deviceId ]
  ↓   send server.message.new with override's ciphertext + envelope
  ↓     → BobChrome gets envelope_A
  ↓     → BobBrave  gets envelope_B
```

### Refresh path (per-device JOIN)

```sql
LEFT JOIN group_message_recipients gmr
  ON gmr.message_id = m.id
 AND gmr.recipient_user_id = <jwt.sub>
 AND gmr.recipient_device_id = <jwt.deviceId>
```

Each device sees only its own envelope. Sender sees the empty sentinel and resolves to plaintext via the local cache (unchanged from v0.10.0).

---

## How a new group appears in real time

```
Alice (creator)
  ↓ POST /api/v1/chats/group { title, memberIds: [Bob, Charlie] }
  ◄  { chat: ChatDTO }                  ← REST response, Alice inserts locally
  ↓ wsClient.sendChatCreated({ chatId })
  ↓
realtime.event-router.onChatCreated
  ↓ api.getChatById(chatId, AliceJWT)   ← auth check + DTO retrieval in one call
  ↓ cm.learnChatParticipants(…)         ← seed routing cache
  ↓ broadcast server.chat.created to every participant connection
  ↓     (skip Alice's originating device — she already has it)
  ↓
Bob, Charlie
  ↓ ServerEvent.CHAT_CREATED handler
  ↓ chats.some(c => c.id === payload.chat.id) ? skip : prepend
  ↓ sidebar updates instantly, no refresh
```

---

## Migration

- **No DB migration.** V15 is the latest; v0.10.1 doesn't touch the schema.
- **No protocol break.** All wire additions are optional. v0.10.0 clients still talk to a v0.10.1 server (they just won't trigger the forced cleanup or receive `server.chat.created`).
- **Forced client cleanup is automatic.** First page load after the new frontend bundle deploys, each browser wipes its identity / SPKs / pre-keys / fingerprints, re-registers, and sets `localStorage['signalix-stale-prekey-cleanup-v1'] = 'done'`. The reset banner (v0.9.1) appears once; sender history is preserved (plaintext cache untouched).
- **Server log expectations**: a burst of `Identity key changed for device X; wiping stale SPKs + pre-keys server-side.` lines for the first 24-48h as devices come online. Healthy noise — that's the cleanup in action.

---

## Operational notes

- Deploy order: `Signalix-contracts` → `Signalix-api` → `Signalix-realtime` → `Signalix-frontend`. Contracts is build-time; the others pull the new types.
- Set `CRYPTO_DEBUG_LOGS = true` in `Signalix-frontend/src/lib/crypto/signal.service.ts` + `Signalix-frontend/src/store/chat.store.ts` to surface the full crypto trail in prod when chasing a regression. Default is dev-only. Next.js inlines `process.env.NODE_ENV` so a raw env check would DCE the logs in prod — routing through the runtime constant is how you flip it without editing every call site.
- No infra changes. `Signalix-infra` is at v0.10.0 still.

---

## Validation

- `npm run typecheck` — clean across `Signalix-contracts`, `Signalix-api`, `Signalix-realtime`, `Signalix-frontend`.
- `npm run build` — clean in all four repos.
- `npm test` (frontend) — 10/10 vitest unit tests pass.

**Manual scenario:**

1. Open the app in Chrome and Brave on the same account — both browsers complete the one-time forced cleanup (visible in console + server logs).
2. Send a direct message from a third user (Alice) to that account.
3. Both Chrome and Brave receive `server.message.new` with their own per-device envelope and decrypt. No `[Unable to decrypt message]` for new messages.
4. Alice creates a group with Bob and Charlie via the UI.
5. Bob and Charlie's sidebars update immediately without refresh.
6. Send a message into the group — every participant device receives + decrypts.

---

## Files of interest

**Contracts**
- `Signalix-contracts/src/websocket/client-event.enum.ts` — `CHAT_CREATED`.
- `Signalix-contracts/src/websocket/server-event.enum.ts` — `CHAT_CREATED`.
- `Signalix-contracts/src/websocket/payloads.contract.ts` — `ClientChatCreatedPayload`, `ServerChatCreatedPayload`; `RecipientEnvelopeDTO` keying clarified to deviceId.

**API**
- `Signalix-api/src/crypto/crypto.service.ts` — reset-aware `registerDeviceKeys` with identity-change `DELETE`; Logger lines on register / upload / bundle.
- `Signalix-api/src/messages/messages.service.ts` — `recipientPayloads` keyed by `recipientDeviceId`; lifted `chatType !== GROUP` guard.
- `Signalix-api/src/chats/chats.controller.ts` — `GET /chats/:chatId`.
- `Signalix-api/src/chats/chats.service.ts` — `getChatById`; `getMessages` accepts `deviceId` + JOIN filters by it.

**Realtime**
- `Signalix-realtime/src/ws/event-router.ts` — `onChatCreated`; per-device `recipientPayloads` lookup in `onMessageSend` + `onMessageEdit`.
- `Signalix-realtime/src/common/api-client.ts` — `getChatById`.

**Frontend**
- `Signalix-frontend/src/lib/crypto/signal.service.ts` — `encryptForUserAllDevices`, `encryptToBundle`, forced cleanup gate, `wipeStaleCryptoState`, env probe + decrypt-attempt logging.
- `Signalix-frontend/src/lib/crypto/plaintext-cache.ts` — `clearDecryptFailureCache`.
- `Signalix-frontend/src/store/chat.store.ts` — direct fan-out via `encryptForUser`; MESSAGE_NEW decrypt-before-insert; `ServerEvent.CHAT_CREATED` handler; `sendChatCreated` on group creation.
- `Signalix-frontend/src/lib/ws-client.ts` — `sendChatCreated`.

---

## What's next — v0.11.0 preview

- **Sender Keys** for groups (constant-cost per send).
- **Group media / file / voice encryption.**
- **Encrypted push previews** (SW-side decrypt).
- **Safety-number verification UI.**
- **Multi-device sync** — pending invites, cross-device read state.
