# Signalix v0.14.0 — Release Notes

**Release date:** 2026-06-08
**Previous version:** v0.13.0

---

## Overview

v0.14.0 lights up the read-receipt + delivery-reliability story that
v0.1.0 left half-built. The state machine itself
(`CREATED → SENT → DELIVERED → READ`) already worked end to end; what
this release adds is **propagation, fan-out, and the surfaces that
let users actually see status**. Plus four regressions caught during
the cut and fixed before tag.

What ships:

- New `GET /messages/:id/recipients/status` and a sync-since-timestamp
  cursor on `GET /chats/:id/messages` so the frontend can fetch per-
  recipient breakdowns and catch up on receipts missed during a WS
  disconnect.
- WS client: 30s heartbeat, `send()` returns boolean so the chat
  store knows when to queue, in-memory outbox auto-drains on
  `server.authenticated`, sync runs alongside.
- `MessageInfoModal` accessible from the message actions menu — direct
  chats show Sent / Delivered / Read times; groups show "Read by N /
  Delivered to N / Sent to N" with timestamps per participant.
- Read tick is **blue** again (cyan on the blue isMine bubble), and
  **every** unread message gets marked when the chat opens, not just
  the last.

What does **not** ship:

- No persistent (IDB-backed) outbox yet — refresh before WS ACK loses
  the queued message. v0.15.0+ work.
- No bulk read-receipt protocol — opening a chat with 50 unread fires
  50 WS events. Idempotent server-side but noisy on the wire.
- No new DB migration.

---

## How a read receipt flows now

```
Receiver opens chat
  ↓ MessageView mount effect (chat.id changes)
  ↓ markedReadRef = new Set()
  ↓
MessageView messages-effect (runs on every messages change)
  ↓ toMark = messages.filter(m => 'id' in m && m.senderId !== me && m.state !== 'read' && !markedReadRef.has(m.id))
  ↓ for each m in toMark:
  ↓   chat.store.markRead(chat.id, m.id)
  ↓     wsClient.sendMessageRead({ messageId, chatId })   ← N events, one per unread
  ↓   markedReadRef.add(m.id)
  ↓
Realtime onMessageStatus
  ↓ api.updateMessageStatus(messageId, READ)     ← UPSERT in message_status
  ↓ broadcast server.message.read to OTHER participants   ← skip the originator
  ↓
Sender chat.store handler
  ↓ ServerEvent.MESSAGE_READ → set m.state = 'read' for that messageId
  ↓ MessageView re-render
  ↓ StatusIcon sees state='read' → renders the cyan/blue double-check
```

---

## How reconnect recovery works

```
WS dropped (proxy timeout, network blip, …)
  ↓ ws.onclose → reconnectTimer set for 3s
  ↓
WS reopens, sends client.authenticate
  ↓ server.authenticated arrives
  ↓ chat.store handler:
  ↓   drainOutbox()
  ↓     for each TempMessage flagged queued:
  ↓       outboxAttempts++; queued = false
  ↓       dispatchSend(...)   ← reuses encryption + fan-out pipeline
  ↓   syncSinceLastEvent()
  ↓     for each chat with messages in memory:
  ↓       since = max(createdAt) in that chat
  ↓       res = api.getMessages(chatId, { since })
  ↓       insert any new messages (dedup by id)
  ↓       for each status update: pick max-rank per messageId, apply to local state
```

The periodic heartbeat keeps the connection alive between user
actions so the reconnect path is only triggered by actual network
events, not idle proxy timeouts.

---

## How Message Info works

Long-press / right-click on a message you sent → "Info" in the menu.

- **Direct chat**: Sent / Delivered / Read times stacked, with em-dash
  for fields the recipient hasn't reached yet.
- **Group chat**: server returns one row per participant (excluding
  you), backfilled with implicit `SENT` at `created_at`. The dialog
  buckets them into Read by / Delivered to / Sent to, each with the
  per-recipient timestamp and avatar.

The endpoint is `GET /messages/:messageId/recipients/status` and is
authorized by chat participation.

---

## Migration

- **No DB migration.**
- **No protocol break.** The new endpoint is additive; existing
  clients keep working. `GetMessagesRequest.since` and
  `GetMessagesResponse.statusUpdates` are optional.
- **No infra changes.** Rebuild + redeploy `api`, `realtime`, and
  `frontend` images.

---

## Validation

- `npm run typecheck` — clean across all four code packages.
- `npm run build` — clean.
- `npm test` (frontend) — **29/29 vitest tests pass**.

**Manual scenarios** (mirrors the spec's bug list):

1. Alice sends 3 messages to Bob while Bob's chat is closed. Bob
   opens it. Alice sees double-check **blue** on all 3, not just the
   last.
2. Send a single message to a brand-new chat. Receiver's unread
   badge reads **1** (not 2).
3. Open the app on half-screen / mobile, tap a user in search. URL
   stays at `/chats` — no `/chats/draft:<userId>` "Loading…" flash.
4. Long-press a sent message → "Info" → Message Info dialog opens
   with the right times / per-recipient list.
5. Drop the WS (toggle airplane mode for 10s, restore). Any messages
   sent while disconnected go out as soon as the socket comes back.
   Any receipts that fired during the drop arrive via the sync query.

---

## Files of interest

**API**
- `Signalix-api/src/messages/messages.controller.ts` — `GET /:messageId/recipients/status`.
- `Signalix-api/src/messages/messages.service.ts` — `getMessageRecipientStatuses`.
- `Signalix-api/src/chats/chats.service.ts` — extended `getMessages` with `since` + `statusUpdates`.
- `Signalix-api/src/chats/dto/get-messages.dto.ts` — `since` field.

**Contracts**
- `Signalix-contracts/src/api/message.contract.ts` — `GetMessageRecipientsStatusResponse`, extended `GetMessagesRequest` + `GetMessagesResponse`.

**Frontend**
- `Signalix-frontend/src/lib/ws-client.ts` — `send()` returns boolean, `isConnected()`, periodic heartbeat, cleanup.
- `Signalix-frontend/src/store/chat.store.ts` — `drainOutbox`, `syncSinceLastEvent`, `TempMessage.queued`, `MESSAGE_NEW` chat-tracked guard, `AUTHENTICATED` handler.
- `Signalix-frontend/src/components/MessageView.tsx` — mark-all-unread effect with `markedReadRef`, "Info" menu item, `MessageInfoModal` mount.
- `Signalix-frontend/src/components/MessageInfoModal.tsx` *(new)*.
- `Signalix-frontend/src/components/StatusIcon.tsx` — cyan/blue read tick.
- `Signalix-frontend/src/components/MobileSearchOverlay.tsx` — `openUserResult` no longer routes to `/chats/draft:*`.
- `Signalix-frontend/src/lib/api-client.ts` — `getMessages({ since })`, `getMessageRecipientsStatus`.

---

## What's next — v0.15.0 preview

- Persistent outbox in IndexedDB so messages survive page reload
  before WS ACK.
- Bulk read-receipt protocol — single WS event with a list of message
  ids, to keep the wire quiet when opening chats with many unread.
- Per-recipient delivery hints driven off the presence map (skip
  retry for users we know are definitely offline; trigger immediate
  retry when they come online).
