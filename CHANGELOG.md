# Signalix Changelog

## v0.14.0 — 2026-06-08

### Theme

**Read receipts + delivery reliability.** Bulk read-receipt fan-out
when a chat opens, a periodic heartbeat + auto-drain outbox when the
WS reconnects, a sync-since-timestamp catch-up for missed receipts,
and the Message Info dialog that breaks down "Read by / Delivered to
/ Sent to" per participant. The state machine itself
(CREATED → SENT → DELIVERED → READ) was already in place since v0.1.0;
v0.14.0 fixes the gaps in propagation and surfaces.

### Added

#### API (`Signalix-api`)
- **`GET /messages/:messageId/recipients/status`** → `GetMessageRecipientsStatusResponse { messageId, statuses: MessageStatusDTO[] }`. Returns one row per chat participant (excluding the sender), backfilled with implicit `SENT` at the message's `created_at` for participants who never advanced past sent. Used by the Message Info dialog.
- **`GET /chats/:chatId/messages?since=<iso>`** — the existing endpoint now accepts a `since` cursor. When provided, the response also carries `statusUpdates: MessageStatusDTO[]` containing the per-recipient receipts that fired strictly after the cursor (bounded to 200). Drives the reconnect catch-up.

#### Contracts (`Signalix-contracts`)
- **`GetMessageRecipientsStatusResponse`**.
- **`GetMessagesRequest.since?`** + **`GetMessagesResponse.statusUpdates?`**.

#### Realtime + reliability (`Signalix-frontend`)
- **`WsClient.send()` returns `boolean`** so callers know whether the frame went out. `sendMessageSend` propagates it.
- **Periodic heartbeat** every 30s while the socket is OPEN; cleared on close + disconnect. Keeps idle proxies from dropping us and surfaces dead connections faster.
- **In-memory outbox.** `TempMessage` gains `queued?: boolean` + `outboxAttempts?: number`. When `dispatchSend` sees `send() === false`, it flags the temp as queued.
- **`drainOutbox()`** — fires on `server.authenticated` (every connect AND reconnect); walks every queued temp and re-emits via `dispatchSend` (reuses the encryption / fan-out pipeline unchanged).
- **`syncSinceLastEvent()`** — also fires on `server.authenticated`; for every chat with messages in memory, calls `getMessages({ since: max(createdAt) })`. Newly-arrived messages are dedup-inserted; status updates are replayed by picking the highest-ranked status (read > delivered > sent) per messageId.
- **`getMessageRecipientsStatus(messageId)`** api-client wrapper.

#### Message Info dialog (`Signalix-frontend`)
- **`MessageInfoModal.tsx`** *(new)*. Opens from the message actions menu (new "Info" item, sender-only). Direct chats: stacked Sent / Delivered / Read with timestamps. Group chats: bucketed "Read by N / Delivered to N / Sent to N" sections with avatar + name + per-participant timestamp.

### Fixed (regressions caught during v0.14.0 testing)

- **Issue 1 — Read tick was not blue.** `StatusIcon` rendered `text-white/70` in dark mode for the read state, indistinguishable from delivered. Now uses Apple `#007aff`/`#0a84ff` on transparent backgrounds and `#34c8ff` (Apple system-blue-light) on the blue `isMine` bubble. All 8 call sites pass `light` to pick the bubble-friendly cyan.
- **Issue 2 — Only the last unread message was marked read.** The MessageView effect found only the most recent unread row. Now iterates every unread message from another participant and fires `markRead` per message; a `markedReadRef: Set<string>` prevents re-firing on subsequent renders (the realtime layer never broadcasts MESSAGE_READ back to the originator, so our local `state` stays at 'delivered' even after we marked it).
- **Issue 3 — Unread badge counted 2 on a brand-new chat.** Race between `loadChats()` (server says `unreadCount: 1`) and the `MESSAGE_NEW` increment. Now the increment checks `chatTracked = s.chats.some(c => c.id === p.chatId)` — if the chat isn't in the sidebar yet, skip the +1; `loadChats` will populate the authoritative count.
- **Issue 4 — `MobileSearchOverlay` routed to `/chats/draft:<userId>` → "Loading…".** The dynamic `[chatId]` route detected the draft prefix and redirected, but flashed Loading first. Now mirrors `ChatSidebar.startNewChat`: existing direct chat → `/chats/<realId>`; otherwise `openDraftChat(user)` + `router.push('/chats')` (where `ChatsIndexPage` renders the in-memory draft via `currentDraft`).

### Not changed
- State machine, WS event names + payloads, message_status schema, E2EE pipeline, presence integration — all unchanged.
- The realtime layer's status-broadcast logic is unchanged; the bulk-read receipts arrive as N individual `server.message.read` events the existing handler already processes.
- No DB migration.

### Known limitations
- **Sync cursor is `max(createdAt)` of loaded messages.** If a chat has no messages locally (never opened), `syncSinceLastEvent` skips it; missed messages for that chat surface when the user opens it (existing pagination + the new `since` query). Acceptable for the common "I was on chat X when I disconnected" case.
- **Outbox is in-memory only.** Page reload before WS ACK loses the queued temp. Persistent outbox to IDB is on the v0.15.0+ shortlist.

### Roadmap signal
- **v0.15.0+** — persistent outbox (IDB), bulk read-receipt protocol (one event, list of message ids) to cut the WS chatter when opening a chat with many unread.

## v0.13.0 — 2026-06-08

### Theme

**Message search (E2EE-aware).** Global + in-chat search across
text, filenames, group names, and usernames — with a critical caveat
the v0.7.1 search predates: v0.10.0+ encrypted message bodies live
in per-recipient envelopes the server can't open. v0.13.0 keeps the
server's ILIKE for the metadata it CAN see (group titles, sender
usernames, legacy plaintext rows) and pairs it with a client-side
walk over the in-memory message store so decrypted bodies and
encrypted-attachment filenames are findable too. The UI merges both
result sets dedup-by-messageId. A new full-screen mobile overlay
replaces the cramped inline sidebar search on phones.

### Added

#### Server-side search extension (`Signalix-api`)
- **`MessagesService.searchMessages`** WHERE clause now matches **any of**: `messages.ciphertext` (legacy plaintext rows; empty for v0.10.0+ encrypted), `chats.title` (group titles), `sender.username`, `sender.display_name`. Previous behaviour was ciphertext-only.
- **`ChatsService.searchInChat`** same extension scoped to a single chat: ciphertext OR sender username OR sender display_name.
- Both keep their existing per-user-deletion + chat-deletion visibility cutoffs and keyset pagination.

#### Client-side local search (`Signalix-frontend`)
- **`src/lib/local-search.ts`** *(new)* — `searchLocalMessages(query, inputs, opts)` walks `chat.store.state.messages` + `state.chats`:
  - TEXT messages: matches against the already-decrypted body the store carries after `decryptStoredMessage`.
  - IMAGE / FILE / AUDIO: parses `MediaMetadataV1` JSON and matches `filename`.
  - Skips the `[Unable to decrypt message]` / `[Unable to decrypt attachment]` sentinels so they don't pollute results.
  - Emits records in the same `MessageSearchResultDTO` shape the server endpoint returns.
- **`searchLocalChatMessages`** — single-chat variant emitting `InChatSearchMatchDTO`.
- **`mergeSearchResults` / `mergeInChatResults`** — dedup by `messageId`, local entry wins (has the real decrypted plaintext where the server only has the empty sentinel).

#### UI integration (`Signalix-frontend`)
- **`ChatSidebar`** — `handleSearchChange` and `loadMoreMessageResults` now fan out server + local search in parallel and merge results.
- **`MessageView` in-chat search effect** — same merge. The existing X-of-Y counter, up/down nav, scroll-to, and `signalix-search-hit` highlight (already shipped in v0.7.1) work unchanged.
- **`MobileSearchOverlay.tsx`** *(new)* — full-screen modal triggered by tapping the sidebar search input on mobile. Own input + result list (People + Messages sections) + tap-to-open routing to `/chats/:id?m=:messageId`. Desktop is unchanged.

#### Tests (`Signalix-frontend`)
- **`lib/local-search.test.ts`** *(new)* — 8 cases: text body case-insensitive match, filename match inside `MediaMetadataV1`, miss controlled, decrypt-failure sentinel skipping, single-chat filter, newest-first sort, merge dedup with local priority, server-only emit. 29/29 frontend tests passing total.

### Fixed
- v0.7.1 search would return zero matches for v0.10.0+ encrypted message bodies (`messages.ciphertext = ''`). Client-side walk closes that blind spot for any message the user has actually loaded.

### Not changed
- The wire shapes (`MessageSearchResultDTO`, `InChatSearchMatchDTO`, request/response wrappers) are unchanged. No contract bump.
- WS protocol, fan-out, E2EE pipeline, media encryption, safety-number flow — all untouched.
- The realtime layer is a no-op for this release.

### Known limitations (intentional)
- **Local search only sees messages the user has loaded into the store.** Older history in chats they've never opened isn't indexed client-side. The server side covers what it can (titles, usernames, legacy plaintext).
- **No vector / semantic search.** ILIKE-only per spec.
- **No fuzzy matching** (typos, stems). Exact substring case-insensitive.
- **Server-side ILIKE is blind to encrypted bodies** by design — that's the whole point of E2EE; the client-side walk is the deliberate compensating mechanism.

### Roadmap signal
- **v0.14.0+** — PostgreSQL full-text index on `chats.title` + `users.username` for larger deployments; bulk-decrypt-and-index on the client so non-loaded history becomes searchable too.

## v0.12.0 — 2026-06-08

### Theme

**Safety number / device verification UI.** The fingerprint
foundation that shipped in v0.9.1 is now surfaced in the chat
profile: users can read the 12-group safety number, scan a QR code,
mark a peer as verified, and get an automatic "Security number
changed" warning when either side's identity key rotates. v0.9.1 +
v0.11.0 fan-out continue to drive the encryption itself — v0.12.0
just gives users the tools to confirm *who* they're talking to.

### Added

#### Verification state (`Signalix-frontend`)
- **`FingerprintRecord`** in `lib/crypto/db.ts` carries a verification snapshot: `verifiedAt`, `verifiedLocalIdentityKey`, `verifiedPeerIdentityKey`, `verifiedSafetyNumber`. The snapshot is frozen at the moment of the "Mark as verified" click so a later silent identity rotation can be detected by diff.
- **`deriveVerificationStatus(record, currentLocalKey, currentPeerKey)`** — pure function returning `'unknown' | 'unverified' | 'verified' | 'changed'`. `'changed'` fires when the current identity keys diverge from the verified snapshot (either side rotating triggers it).
- **`markPeerVerified` / `unmarkPeerVerified`** helpers + the upsert path in `cacheSafetyNumber` that preserves the snapshot when refreshing the "current view".

#### Service API (`Signalix-frontend`)
- **`SignalCryptoService.getPeerVerification(peerUserId)`** — single call that fetches the peer's current bundle (so a key rotation is detected immediately), validates the Ed25519 signature (v0.9.1 hardening still applies), refreshes the cached fingerprint, and returns `{ status, safetyNumber, localIdentityKey, peerIdentityKey, verifiedAt, verifiedSafetyNumber }`.
- **`SignalCryptoService.markPeerVerified(peerUserId)` / `unmarkPeerVerified(peerUserId)`** — wraps the IDB helpers.
- `CryptoService` interface + `MockCryptoService` updated accordingly.

#### UI (`Signalix-frontend`)
- **New `EncryptionPanel` inside `ContactProfileModal`** for direct chats:
  - "End-to-end encrypted beta" pill.
  - Safety number rendered as 2 rows of 6 × 5-digit groups, selectable.
  - QR code (160×160) carrying `signalix-safety:<number>` so two devices can compare visually.
  - Status badge: *Verified • <date>* / *Not verified* / *No longer verified*.
  - Button: *Mark as verified* (or *Unverify* / *Re-verify* depending on status).
  - Amber warning panel when status is `'changed'`: **"Security number changed — Confirm the new number with them before resuming sensitive conversations."**
  - Loading + retry states for the bundle fetch.
- New `qrcode` dependency (+ `@types/qrcode`).

#### Tests
- `fingerprints.test.ts` — 5 new cases on `deriveVerificationStatus` (unknown / unverified / verified / changed-by-peer / changed-by-local). 21/21 frontend tests passing total.

### Fixed
- The previously-orphan v0.9.1 fingerprint cache (computed but never displayed) finally has a user-facing surface.

### Not changed
- No DB migration. No new API endpoints — the verification state lives entirely in client IndexedDB.
- No contract changes. The realtime layer is untouched.
- Direct E2EE, group E2EE, media E2EE, recipient routing — all unchanged.

### Known limitations (intentional, per scope)
- **No production trust model.** Verification is local to each device's IDB. Wiping browser storage resets the verified flag for every peer.
- **No cloud key backup or account recovery.** Same model as the rest of the E2EE stack.
- **No group safety numbers.** v0.12.0 covers direct chats only — groups need a different ceremony (or Sender Keys, v0.13.0+).
- **No media-specific verification.** Attachment metadata flows through the same per-recipient envelope as text; verification status is per-peer, not per-message.

### Roadmap signal
- **v0.13.0+** — Sender Keys for groups; group safety numbers; encrypted push previews; signed-URL media access.

## v0.11.0 — 2026-06-08

### Theme

**Media / file / voice E2EE beta.** Image, file, and voice attachments
are now encrypted client-side before upload — the API and MinIO only
ever see opaque AES-256-GCM ciphertext. The per-attachment media key
rides inside the existing per-recipient envelope (same X3DH-style
fan-out that protects text since v0.10.x), so multi-device works
automatically and no new key-management surface was introduced.

### Added

#### Client-side file encryption (`Signalix-frontend`)
- **`src/lib/crypto/file-crypto.ts`** — `encryptFile(blob)` returns `{ ciphertext, key, iv }` using a freshly generated 32-byte AES-256-GCM key + 12-byte IV. `decryptFile(ciphertext, key, iv)` for the recipient side. Plus base64url wire encoders/decoders for the key + IV (`encodeKeyForWire`, `decodeKeyFromWire`, …) and the `MediaMetadataV1` type that describes the in-envelope JSON.
- **`src/lib/crypto/use-decrypted-blob-url.ts`** — `useDecryptedBlobUrl(info)` React hook that fetches the encrypted blob, decrypts it in memory, and exposes a one-shot `blob:` URL for `<img>` / `<audio>` to render against. Cleans up via `URL.revokeObjectURL` on unmount or when the input changes. Companion `downloadDecryptedAttachment` for file save-as flows.

#### Encrypt-then-upload entry points (`Signalix-frontend`)
- **`MessageInput.tsx`** — image / file / voice paths now run `encryptFile` → `uploadEncryptedBlob` → build `MediaMetadataV1` JSON (`{ v:1, url, k, iv, mime, size, filename?, duration? }`) → ship it via `onSend` as the message ciphertext. The store's per-recipient envelope encrypts the JSON, so the server never sees the URL ↔ key pair together.
- **`uploadEncryptedBlob(ciphertext)`** in `lib/api-client.ts` — `POST /api/v1/media/encrypted-blob`, returns `{ url, size }`.

#### Encrypted-blob upload endpoint (`Signalix-api`)
- **`POST /api/v1/media/encrypted-blob`** in `MediaController` — accepts opaque ciphertext bytes up to 25 MB. Object key format `encrypted/{userId}/{uuid}.bin` under the existing media bucket. MIME validation is skipped on purpose (the bytes are random-looking ciphertext); size cap matches the largest pre-v0.11.0 attachment kind (files).
- **`MessagesService.sendMessage`** lifted the `messageType === TEXT` guard on `recipients[]`. IMAGE / FILE / AUDIO can now be sent via the per-recipient envelope path.

#### Receive-side wiring (`Signalix-frontend`)
- **`chat.store.decryptStoredMessage`** no longer short-circuits on non-TEXT. For IMAGE / FILE / AUDIO the decrypted "plaintext" is the metadata JSON, which the renderers re-parse to grab the URL + key + IV.
- **`MessageView.tsx`** — new `ImageBubble` component (renders via `useDecryptedBlobUrl`), updated `VoiceBubbleSection` (passes the blob URL to `VoiceBubble`), updated `FileCard` (download path uses `downloadDecryptedAttachment` for encrypted files; legacy plaintext files still go through `/files/:id/download`). New `BrokenAttachment` tile for `[Unable to decrypt attachment]`.
- **`DECRYPT_FAILED_ATTACHMENT_PLACEHOLDER`** exported from `crypto.service` — used for the media sentinel; existing `[Unable to decrypt message]` continues to cover TEXT.

#### Tests
- `src/lib/crypto/file-crypto.test.ts` — 6 unit tests covering roundtrip, ciphertext != plaintext, fresh key/IV per call, tamper rejection, wrong-length key/IV rejection, base64url wire roundtrip. **16/16 frontend tests passing total.**

### Fixed
- Media bubbles that previously rendered the raw URL now decrypt to a blob URL — the server can never link "this image URL" to "Alice sent it to Bob" beyond the WS metadata it already sees.

### Not changed
- WS protocol (only the contents of `ciphertext` shifted from URL/legacy-JSON to the new metadata schema; no new events or DTO fields).
- Database schema. The MinIO bucket structure is additive (`encrypted/` prefix joins the existing `messages/` and `voice/` prefixes).
- Direct + group TEXT E2EE — unchanged.
- Legacy plaintext rows still render correctly (`parseImageInfo` / `parseFileInfo` / `parseVoiceInfo` fall back when no `v:1` envelope is detected).

### Known limitations (intentional)
- **No backup / recovery.** Lose the message envelope, lose access to that attachment forever — the media key only lives inside the per-recipient envelope.
- **Link previews, avatars, profile photos remain plaintext.** Scope-limited to message attachments.
- **No streaming decrypt.** The entire ciphertext is loaded into memory before AES-GCM open. Acceptable up to the 25 MB cap; revisit when raising it.
- **The encrypted blob is at a public MinIO URL.** Knowing the URL only grants access to ciphertext, which is useless without the media key, so this is intentional — but anyone who saw the URL via the wire (no one in practice, since the URL itself is inside the envelope) could replay-fetch. Future hardening could move to signed URLs.

### Roadmap signal
- **v0.12.0+** — Sender Keys for groups (drop the O(participants) cost on text + attachments alike), encrypted push previews, signed-URL media access, link-preview encryption.

## v0.10.1 — 2026-06-08

### Theme

**E2EE multi-device hygiene + realtime group surface.** v0.10.1 closes the
intermittent decrypt failures that surfaced after the v0.10.0 group
beta — almost all of them traced back to two interacting bugs:
**(a)** the sender's single-device assumption (picking `bundles[0]`)
silently locked out a recipient's second browser, and **(b)** the
v0.9.1 reset detection wiped the local IDB without telling the server,
leaving the bundle endpoint to keep handing out orphan one-time
pre-key ids the device no longer had. v0.10.1 fans direct E2EE out to
every recipient device (matching v0.10.0's group path), keys the
per-recipient payload map by `deviceId` instead of `userId` so each
device gets *its* envelope, and teaches the API to scrub a device's
prior pre-keys on identity change. A one-time forced-cleanup is added
to the client so existing deployments converge without a manual purge.

The realtime layer also gains its first non-message broadcast:
`server.chat.created` makes a newly-created group appear in every
participant's sidebar immediately, without a manual refresh.

### Added

#### Multi-device direct fan-out (`Signalix-frontend`, `Signalix-api`)
- **`SignalCryptoService.encryptForUserAllDevices(plaintext, recipientUserId)`** — fetches the recipient's full bundle list and encrypts the same plaintext separately for every device. Replaces the v0.9.x `bundles[0]` single-device assumption that broke Brave + Chrome on the same account.
- **`chat.store.dispatchSend` / `dispatchEdit`** for direct chats now uses the same per-recipient `recipients[]` shape as group sends. Direct + group are unified on the same fan-out plumbing.
- **`MessagesService.sendMessage` / `editMessage`** no longer reject `recipients[]` for direct chats — the `chatType !== GROUP` guard was lifted.

#### Per-device payload routing (`Signalix-api`, `Signalix-realtime`, `Signalix-contracts`)
- **`recipientPayloads` keyed by `deviceId`** (not `userId`). When a recipient has two devices (Brave + Chrome), the map now holds both entries instead of last-write-wins overwriting one. `RecipientEnvelopeDTO`'s comment updated.
- **`ChatsController.getMessages`** accepts the caller's `deviceId` from the JWT and forwards it to the service; **`ChatsService.getMessages`** JOINs `group_message_recipients` on `(recipient_user_id, recipient_device_id)` so a multi-device user no longer gets duplicate rows per message.
- **Realtime `event-router.onMessageSend` / `onMessageEdit`** look up `recipientPayloads[participantConn.deviceId]` per-connection, so each device receives its own envelope.

#### Server-side stale-key hygiene (`Signalix-api`)
- **`CryptoService.registerDeviceKeys`** detects an identity-key change versus the row already stored and, in the same transaction, **`DELETE`s** the device's prior `signed_pre_keys` + `pre_keys` before inserting the new material. Without this, the bundle endpoint kept handing out orphan one-time pre-key ids that the recipient's IDB no longer held → `Local one-time pre-key X not found`.
- API-side `Logger` lines on register / upload / bundle so the cleanup is observable in deploy logs.

#### Forced one-time cleanup (`Signalix-frontend`)
- **`localStorage['signalix-stale-prekey-cleanup-v1']`** — on the first init after the v0.10.1 deploy on each device, the client wipes its local identity / SPKs / pre-keys / fingerprints (**preserving the plaintext cache** so sender history survives) and re-registers. The server-side identity-change detection above then purges the orphan rows. Idempotent: subsequent inits no-op.

#### Decrypt-failure cache life cycle (`Signalix-frontend`)
- **`clearDecryptFailureCache()`** runs on every successful `cryptoService.init()`. A transient failure from a previous session (init race, the v0.9.0 envelope-drop bug, etc.) auto-heals on the next page load; a genuinely broken message re-caches for the current session.

#### MESSAGE_NEW flicker fix (`Signalix-frontend`)
- `chat.store` decrypts encrypted-text messages **before** inserting them into the store. The bubble used to render the raw envelope JSON for ~50ms until the async decrypt finished and replaced it; now it appears directly with plaintext. Browser notifications also use the decrypted plaintext as preview body (instead of the envelope JSON).

#### `server.chat.created` realtime broadcast (`Signalix-contracts`, `Signalix-api`, `Signalix-realtime`, `Signalix-frontend`)
- New `ClientEvent.CHAT_CREATED` + `ServerEvent.CHAT_CREATED` with `ClientChatCreatedPayload { chatId }` and `ServerChatCreatedPayload { chat: ChatDTO }`.
- New `GET /api/v1/chats/:chatId` → `{ chat: ChatDTO }`. Forbidden for non-participants, 404 if the chat doesn't exist. Used by the realtime layer to fetch the canonical DTO with the caller's JWT (auth check + DTO retrieval in one call).
- Realtime `onChatCreated`: re-fetches the canonical chat, seeds `cm.learnChatParticipants` so subsequent message sends route to every participant immediately, then fans `server.chat.created` out to every participant connection (skipping the originating device).
- Frontend `chat.store.createGroupChat` fires `wsClient.sendChatCreated({ chatId })` after the REST response; the WS handler dedupes by `chat.id` so the originator (who already inserted the chat from REST) doesn't double-insert.

#### Diagnostic logging (`Signalix-frontend`, `Signalix-api`)
- Frontend `CRYPTO_DEBUG_LOGS` runtime constant (defaults to dev-only). When flipped to `true` it surfaces the full crypto trail in production: `env probe` (deviceId, isBraveDetected), `encryptForUserAllDevices` (bundle ids), `preKey selected by sender`, `preKey consumed locally`, `decrypt attempt`, `decrypt success`/`decrypt failed` (with the entire local key inventory: `localSignedPreKeyIds`, `localUnconsumedPreKeyIds`, `localTotalPreKeyCount`, `spkLookup`, `preKeyLookup`, `preKeyAlreadyConsumed`, `reason`).
- AES-GCM auth-tag failures now throw with a descriptive reason instead of Web Crypto's opaque `OperationError`.
- API `CryptoService.registerDeviceKeys`, `uploadPreKeys`, `getKeyBundle` emit Nest `Logger` lines with the relevant key ids so the server log shows exactly which pre-key was claimed for each bundle hand-out.

### Fixed
- **Intermittent "Unable to decrypt message" on direct chats.** Combination of the multi-device fan-out, per-device payload routing, server-side stale-key wipe, and forced cleanup. Confirmed by side-by-side `decrypt success` / `decrypt failed` diagnostic logs.
- **Brave-only decrypt failure** when the user was logged into Chrome and Brave on the same account. Symmetric: Chrome-only failure if Brave registered second.
- **MESSAGE_NEW envelope JSON flash** between WS receipt and async decrypt.
- **Push / in-app notification preview** for encrypted text used to show the raw envelope JSON; now shows the decrypted plaintext (or the failure placeholder).
- **Group chat doesn't appear in participants' sidebar until refresh.** Closed by `server.chat.created`.

### Not changed
- WS protocol semantics, REST routes (only added — no breaking changes), DB schema (no new migrations).
- Direct E2EE single-device path still encrypts correctly; multi-device is additive.
- Realtime layer architecture (still in-memory routing cache, no Redis).

### Operational notes
- Deploy order: contracts → api → realtime → frontend. (Same as v0.10.0; contracts is build-time only.)
- After deploy each device runs the forced cleanup exactly once. Server logs will show a burst of `Identity key changed for device X; wiping stale SPKs + pre-keys server-side.` lines — that's expected for the first day or two as devices come online.
- No `Signalix-infra` changes — no new env vars, no new services, no schema migration.

## v0.10.0 — 2026-06-07

### Theme

**Group E2EE beta — text only.** v0.10.0 extends the direct-text E2EE from v0.9.x to group chats via **per-recipient encryption fan-out**: the sender runs the v0.9.x X3DH-style handshake N times (once per recipient) and ships N envelopes, one for each device that should be able to decrypt. The server stores each envelope in a new `group_message_recipients` table and the realtime layer delivers each recipient only their own copy.

> ⚠️ **Beta.** Per-recipient fan-out, **not** production-grade Sender Keys. The per-message cost is `O(participants)` for both encrypt and bandwidth — fine for small groups, not for thousands. Sender Keys is a v0.11.0+ goal. **Group media (images, files, voice notes) remain plaintext on the server in v0.10.0.** New joiners cannot decrypt history (intentional — they have no recipient row for older messages).

### Added

#### Server / database (`Signalix-api`)

- **Migration `V15__group_message_recipients.sql`** — one row per (message × recipient × device) for group encrypted text sends. Columns: `message_id`, `recipient_user_id`, `recipient_device_id`, `ciphertext`, `encryption_version`, `pre_key_id`, `signed_pre_key_id`, `created_at`. PK = `(message_id, recipient_user_id, recipient_device_id)`. Index on `(recipient_user_id, message_id)` for the history-fetch JOIN. `ON DELETE CASCADE` from `messages` so a delete-for-everyone cleans up the per-recipient rows.
- **`MessagesService.sendMessage` fan-out path.** When `recipients[]` is provided and the chat is a group and the message is TEXT, the service: validates each recipient is a participant (and is not the sender), writes one row into `messages` with empty ciphertext + `encryption_version=1`, inserts N rows in `group_message_recipients`, and returns `recipientPayloads: Record<UUID, RecipientEnvelopeDTO>` so the realtime layer can fan out per-recipient.
- **`MessagesService.editMessage` re-fan-out.** Same shape on edit: the service deletes the existing `group_message_recipients` rows and inserts the new set within the same transaction, then returns the per-recipient payload map. Direct E2EE edits also pick up envelope re-routing on the same DTO (`encryptionVersion`, `senderDeviceId`, `recipientDeviceId`, `preKeyId`, `signedPreKeyId`).
- **`ChatsService.getMessages`** now `LEFT JOIN`s `group_message_recipients` on `recipient_user_id = caller.id` and replaces the top-level `ciphertext` / envelope columns with the per-recipient row's values when present. Senders + non-recipients see the empty sentinel ciphertext (which the frontend's plaintext-cache + failure-cache turns into either the cached plaintext or `[Unable to decrypt message]`).
- **Push preview** for an empty top-level ciphertext (group encrypted send) becomes the generic `🔒 New encrypted message` instead of a blank body.

#### Realtime (`Signalix-realtime`)

- **Per-recipient broadcast** in `event-router.onMessageSend` and `onMessageEdit`. When the API response carries `recipientPayloads`, the router builds a personalized `server.message.new` / `server.message.edited` per participant: each recipient gets only the ciphertext + envelope encrypted to *their* device. Non-recipients (or recipients without a key bundle published) fall back to the top-level row, which for a group encrypted send is the empty sentinel.
- **`common/api-client.sendMessage`** signature widened with `recipients?: GroupRecipientPayloadDTO[]`.
- **`common/api-client.editMessage`** now takes a payload object (was `ciphertext` positional) so it can forward envelope fields + `recipients`.

#### Frontend (`Signalix-frontend`)

- **`chat.store.dispatchSend` group path.** For group + TEXT + non-draft sends, the store now: resolves participant userIds (excluding self), runs `cryptoService.encryptForRecipient` once per recipient in parallel, builds `recipients[]`, and ships the WS frame with empty top-level ciphertext + `recipients`. If **any** recipient encryption fails the whole send falls back to plaintext (no partial leak).
- **`chat.store.dispatchEdit`.** Symmetric path for edits. Re-encrypts the new plaintext per recipient and ships the recipients map. Sender-side plaintext cache is updated optimistically so a later refresh keeps showing the edit.
- **`MessageView` header pill.** Group chats now show the same `🔒 End-to-end encrypted beta` pill as direct chats, with a tooltip noting "media, files, and voice notes are not encrypted."

#### Contracts (`Signalix-contracts`)

- **New `GroupRecipientPayloadDTO`** in `api/message.contract.ts` (`recipientUserId`, `recipientDeviceId`, `ciphertext`, `encryptionVersion`, optional `preKeyId` + `signedPreKeyId`).
- **New `RecipientEnvelopeDTO`** for the API → realtime fan-out map.
- **Extended `SendMessageRequest`, `EditMessageRequest`, `ClientMessageSendPayload`, `ClientMessageEditPayload`** with `recipients?: GroupRecipientPayloadDTO[]` plus envelope-passthrough fields on edit.
- **Extended `SendMessageResponse`, `EditMessageResponse`** with `recipientPayloads?: Record<UUID, RecipientEnvelopeDTO>`.
- **Extended `ServerMessageEditedPayload`** with optional envelope fields so per-recipient edits ride on the existing event.

### Not changed

- Direct E2EE — every v0.9.x property carries over byte-for-byte. The direct send/edit paths short-circuit before the group fan-out resolver.
- Reactions, delete-for-me, delete-for-everyone, status updates, typing, presence — all unchanged.
- Image / file / voice messages, link previews on plaintext text, search (the search endpoint won't match the encrypted body — same as v0.9.x direct).
- WS event names. v0.10.0 reuses `client.message.send`, `server.message.new`, `client.message.edit`, `server.message.edited`; the new fields are optional and additive.

### Known limitations

- **Per-recipient cost.** Send latency and bandwidth scale linearly with participant count. Acceptable for small groups; Sender Keys is the path forward (v0.11.0+).
- **No multi-device fan-out yet.** Single-device assumption from v0.9.x carries over — the sender uses the first bundle per recipient.
- **New joiners don't decrypt history.** Intentional consequence of the fan-out: a member added after the fact has no recipient row in `group_message_recipients`, so old messages render as `[Unable to decrypt message]`.
- **Reply previews to a group encrypted message show an empty quote bubble.** The reply preview reads `messages.ciphertext` (the sentinel) and v0.10.0 does not extend it with the per-recipient JOIN. v0.11.0 will.
- **Group media, files, voice notes remain plaintext on the server.**
- **Push notification previews** for group encrypted sends show the generic `🔒 New encrypted message` rather than the message body.

### Roadmap signal

- **v0.11.0** — Sender Keys for groups (constant-cost per send), group media / file / voice encryption, encrypted push previews, per-recipient JOIN in reply previews.
- **v0.12.0+** — multi-device fan-out, safety-number verification UI surfacing the v0.9.1 fingerprint store.

## v0.9.1 — 2026-06-07

### Theme

**E2EE hardening.** v0.9.1 closes the worst footguns from the v0.9.0 beta. Direct text messages still flow the same way on the wire; the difference is that the server now refuses to publish a malformed bundle, the client refuses to encrypt to one, one-time pre-keys are actually consumed, a corrupt local crypto store auto-recovers, and a re-render that fails to decrypt no longer flickers between empty body and `"[Unable to decrypt message]"`.

> Scope is still direct text only. Groups, media, voice, push previews, multi-device fan-out, Double Ratchet, and the safety-number verification UI remain v0.10.0+ work.

### Added

#### Server-side signature verification (`Signalix-api`)

- **`crypto.service.registerDeviceKeys` and `rotateSignedPreKey`** now verify the Ed25519 signature on `signedPreKey.publicKey` against the device's `signingKey` *before* writing the row, using `node:crypto.webcrypto.subtle.verify('Ed25519', …)`. Failure → 400 `VALIDATION_ERROR` with a generic public message; the dev log records the keyId for diagnosis.
- **Wire-format byte-length checks.** `identityKey`, `signingKey`, `signedPreKey.publicKey`, every `preKey.publicKey` must be 32 bytes; `signedPreKey.signature` must be 64 bytes. Anything else is rejected up front rather than persisted as a corrupt row.
- **Rotate flow looks up the device's existing signing key** so a rotated SPK signature can't be re-signed by an attacker who only has the new SPK private key.

#### Client-side bundle validation (`Signalix-frontend`)

- **`signal.service.assertBundleIsValid`** runs before any ECDH derivation:
  - structural shape (`identityKey`, `signingKey`, `signedPreKey.{keyId, publicKey, signature}` present; `preKey` optional but well-typed),
  - base64url decodability,
  - exact byte lengths (32 / 32 / 32 / 64; 32 if `preKey` present),
  - Ed25519 signature check on `signedPreKey.publicKey` via `signingKey`.
- On failure `encryptForRecipient` throws — there is no plaintext fallback for a tampered bundle.

#### One-time pre-key consumption + auto top-up (`Signalix-frontend`)

- `decryptIncoming` flips `PreKeyRecord.consumed = true` after a successful decrypt that used `preKeyId`, then asynchronously checks the unconsumed count. If it dips below 20, the service generates fresh X25519 pairs back up to a target of ~100 and publishes them via `POST /crypto/devices/pre-keys`. Init-time top-up uses the same target.
- Pre-keys are kept locally even after consumption (so late envelopes that reference them can still decrypt during the transition window); the `consumed` flag is what drives the top-up math.

#### Device reset detection + banner (`Signalix-frontend`)

- `signal.service.init` detects three reset triggers: no local identity, identity bound to a different `deviceId`, or partial state (identity present but no signed pre-keys / no pre-keys). On any of these it wipes the four crypto-only IDB stores plus the plaintext cache plus the fingerprint cache, regenerates a fresh identity + signed pre-key + 100 pre-keys, republishes them, and sets `wasReset = true`.
- New `EncryptionResetBanner` component (mounted in `app/chats/layout.tsx`) reads `getCryptoStatus().wasReset` once init settles and shows a dismissible amber notice: **"Encryption keys were reset on this device."** Clicking *Got it* calls `acknowledgeCryptoReset()` so the banner doesn't reappear on re-render.

#### Decrypt failure cache (`Signalix-frontend`)

- `PlaintextCacheRecord` gains an optional `failed: boolean` (plus dev-only `failedReason`). `cacheDecryptFailure(messageId, chatId, reason?)` records the negative result; `isDecryptFailureCached(messageId)` short-circuits future attempts.
- `chat.store.decryptStoredMessage` checks the failure cache before attempting decrypt and writes to it on the catch path. History reloads no longer re-run a broken handshake on every render, and the UI no longer flickers between empty body and `"[Unable to decrypt message]"`.

#### Safety number foundation (`Signalix-frontend`, no UI yet)

- New `src/lib/crypto/fingerprints.ts` — computes the per-peer safety number as 12 groups of 5 decimal digits (5200 rounds of SHA-256 over the lexicographically-ordered identity-key pair, extended to 60 bytes via one more SHA-256 round, then chunked into 5-byte groups mod 10⁵).
- New `STORE_FINGERPRINTS` IndexedDB store (`peerUserId` keyed). `SignalCryptoService.getSafetyNumber(peerUserId)` returns the cached value or computes + caches one from a freshly validated bundle.
- v0.9.1 ships the foundation only — the verification UI lands in v0.10.0.

#### IndexedDB schema bump (`Signalix-frontend`)

- `CRYPTO_DB_VERSION` 1 → 2 — adds the `fingerprints` store. Existing `identity` / `signed-pre-keys` / `pre-keys` / `plaintext-cache` rows survive the upgrade unchanged.
- New helpers `idbDelete` and `idbClearStore` used by the reset path.

#### Tests (`Signalix-frontend`)

- Added `vitest` as a dev dependency and `test` / `test:watch` scripts.
- `src/lib/crypto/utils.test.ts` — base64url round-trip, `concatBytes`, `bytesToHex`, `verifyEd25519Signature` (positive, tampered-message, malformed-key).
- `src/lib/crypto/fingerprints.test.ts` — output format (12 × 5 digits), symmetry across argument order, sensitivity to a single bit flip.

### Fixed

- **Sender no longer encrypts to a forged or corrupted bundle.** Before v0.9.1 any 32-byte string at `signedPreKey.publicKey` would have been accepted; now the Ed25519 signature must verify or the send aborts.
- **Top-up math** now counts only unconsumed pre-keys instead of the raw row total. The previous code could leave the local pool effectively empty while the row count looked healthy.
- **Repeated decrypt attempts on history reload.** Each failed message used to re-run the handshake on every render of the message list; v0.9.1 caches the failure and short-circuits.

### Not changed

- WS protocol, contracts, REST routes — all identical to v0.9.0. Existing v0.9.0 clients keep talking to a v0.9.1 server (their bundles will be re-validated on next register / rotate; if they were ever publishing junk, the server now says so).
- Realtime service untouched.
- Database schema (`V14__crypto_foundation.sql`) untouched. The `consumed_at` server column was already there since v0.8.0; v0.9.1 just mirrors that state locally too.

### Known limitations (carried from v0.9.0)

- Single-device only; the sender picks the first bundle returned.
- No Double Ratchet — forward secrecy still bounded by signed-pre-key rotation.
- No encrypted push previews.
- No safety-number / key-verification UI (foundation only).
- Groups, images, files, voice notes remain plaintext on the server.

### Roadmap signal

- **v0.10.0** — Double Ratchet, multi-device fan-out, encrypted push previews, safety-number verification UI surfacing the v0.9.1 fingerprint store, automatic signed-pre-key rotation.
- **v0.11.0+** — group encryption (Sender Keys), media / file / voice encryption.

## v0.9.0 — 2026-06-07

### Fixed (post-initial-cut)

- **Realtime envelope forwarding (`Signalix-realtime`).** The first cut of v0.9.0 missed two lines in `ws/event-router.onMessageSend`:
  - The `api.sendMessage(...)` call did not extract the envelope fields (`encryptionVersion`, `senderDeviceId`, `recipientDeviceId`, `preKeyId`, `signedPreKeyId`) from the incoming WS payload, so the API persisted rows with `encryption_version = 0` and envelope columns `NULL`.
  - The construction of `ServerMessageNewPayload` did not spread the same fields from the API's returned `MessageDTO`, so recipients received the JSON envelope as raw ciphertext without the `encryptionVersion >= 1` flag that triggers the client decrypt path. The chat UI rendered `{"v":1,"c":"…","iv":"…","eph":"…"}` verbatim instead of decrypted plaintext.
- **`common/api-client.sendMessage` payload type widened** with the five optional envelope fields.
- **`signal.service.decryptIncoming` init-race protection (`Signalix-frontend`).** A `MESSAGE_NEW` arriving between login and the completion of the initial key publish used to hit empty IndexedDB and fall through to `[Unable to decrypt message]`. `decryptIncoming` now `await`s the service's in-flight `initPromise` before doing IDB lookups.
- **Dev diagnostics.** `signal.service.decryptIncoming` logs `[signalix-crypto] decrypt attempt` (with `version`, `signedPreKeyId`, `preKeyId`) and `[signalix-crypto] decrypt success`. `chat.store.decryptStoredMessage` logs `[signalix-crypto] decrypt failed` with the reason on the catch path. All log calls are gated on `NODE_ENV !== 'production'`.

### Theme

**Signal Protocol Beta for direct chats.** v0.9.0 turns the v0.8.0 foundation into a working — if simplified — end-to-end encryption layer for **direct text messages only**. Groups, images, files, voice notes, reactions, replies, forwards, edit, and delete all continue to flow as before; only the body of a direct TEXT message gets sealed before it leaves the sender's browser.

> ⚠️ **Beta — not production-grade.** No Double Ratchet, no signature verification yet, single-device assumption, no per-message forward secrecy beyond signed-pre-key rotation. **v0.10.0 hardens this**: multi-device fan-out, Double Ratchet, Ed25519 signature verification, one-time pre-key deletion after use, encrypted push previews, key-verification UX. v0.11.0+ extends encryption to groups and media.

### Added

#### Frontend Signal service (`Signalix-frontend`)

- **`src/lib/crypto/signal.service.ts`** — `SignalCryptoService implements CryptoService`. Real Web Crypto.
  - **Identity**: X25519 (ECDH) + Ed25519 (signing). Generated on first `init({ deviceId })`; published once via `POST /crypto/devices/keys`; persisted as live `CryptoKey` instances in IndexedDB.
  - **Signed pre-key**: X25519, signed with the device's Ed25519 key, published in the same initial call.
  - **One-time pre-keys**: 100 generated up front; auto top-up when the local pool drops below 20.
  - **Encrypt path**: ephemeral X25519 keypair → `ECDH(eph, recipientSPK) || ECDH(eph, recipientOTP)` → HKDF-SHA256 with `info="signalix-v1-direct-text"` → AES-256-GCM with 12-byte random IV. Envelope on the wire is `JSON.stringify({ v: 1, c, iv, eph })` (all values base64url) carried in `messages.ciphertext`. The 5 envelope columns from v0.8.0 carry the routing metadata.
  - **Decrypt path**: parses the envelope, looks up the local signed-pre-key + one-time pre-key by id, reverses the ECDH, derives the same AES key. Failure → `"[Unable to decrypt message]"` sentinel.
- **`src/lib/crypto/db.ts`** — IndexedDB schema `signalix-crypto-v1` with stores `identity`, `signed-pre-keys`, `pre-keys`, `plaintext-cache`. CryptoKey instances stored directly via structured clone.
- **`src/lib/crypto/plaintext-cache.ts`** — local plaintext cache so the sender can re-render their own outgoing text after a history reload.
- **`src/lib/crypto/utils.ts`** — base64url, X25519 / Ed25519 raw public-key importers, HKDF → AES-256-GCM helper. `Uint8Array<ArrayBuffer>` return types so strict-TS Web Crypto signatures accept them.
- **`crypto.service.ts` swap point** — picks `SignalCryptoService` by default, falls back to the v0.8.0 `MockCryptoService` when `NEXT_PUBLIC_E2EE_DEV_FALLBACK=true`.

#### `auth.store` (`Signalix-frontend`)

- Every successful auth path (hydrate / login / register / loginWithOAuth / silent refresh) now fires `cryptoService.init({ deviceId })` in the background. Errors logged with `[signalix-crypto] init failed`; never blocks auth.

#### `chat.store` integration (`Signalix-frontend`)

- **`sendMessage`** for direct + TEXT + non-draft messages encrypts via `cryptoService.encryptForRecipient` before WS send. Plaintext fallback on every failure mode so transitioning users never lose messages.
- **`MESSAGE_NEW` handler** carries the envelope fields through, inserts the encrypted body, then in a follow-up tick decrypts and replaces the ciphertext in the store.
- **`MESSAGE_SENT` handler** caches the sender's plaintext keyed by the freshly-assigned `messageId`.
- **`loadMessages`** runs every history page through `decryptStoredMessage` (cache lookup → live decrypt → sentinel) before insertion.

#### UX (`Signalix-frontend`)

- Direct-chat header shows a **🔒 End-to-end encrypted beta** pill next to the presence row. Hidden for drafts and groups.
- Messages whose ciphertext is `"[Unable to decrypt message]"` render with an italic + open-padlock treatment in muted color.

#### Build / infra (`Signalix-infra`, `Signalix-frontend`)

- **`NEXT_PUBLIC_E2EE_DEV_FALLBACK`** new build-arg / env var. Default `false`. Wired in `Signalix-frontend/Dockerfile` and `docker-compose.yml` `frontend.build.args`.
- **`Signalix-frontend/.env.example`** documents the flag.

### Not changed
- All v0.8.0 crypto endpoints, contracts, DB schema (V14) — unchanged. The v0.9.0 frontend simply starts using them.
- Group chats, image messages, file attachments, voice notes, reactions, replies, forwards, edit, delete, search, push — keep working exactly as v0.8.0.
- Realtime service untouched. Envelope fields ride on existing WS payloads as additive optional properties.

### Known limitations
- **No Double Ratchet** — single ephemeral keypair per message; forward secrecy bounded by signed-pre-key rotation.
- **No multi-device fan-out** — sender picks the first device bundle returned by the API.
- **No signature verification** on signed pre-keys server-side.
- **Pre-keys not deleted locally** after first use.
- **Sender history** on a brand-new browser shows `"[Unable to decrypt message]"` for messages sent from the previous device.
- **Push notification previews** still read the encrypted ciphertext.
- **No safety-number / key-verification UX.**
- **Groups, images, files, voice notes** remain plaintext on the server.

### Roadmap signal
- **v0.10.0** — Double Ratchet, multi-device fan-out, signed-pre-key signature verification, one-time pre-key deletion after use, encrypted push previews, key-verification UX, automatic signed-pre-key rotation.
- **v0.11.0+** — group encryption (Sender Keys), media / file / voice encryption (encrypt-then-upload), key change notifications.

## v0.8.0 — 2026-06-07

### Theme

**Encryption foundation.** This release prepares Signalix for a future Signal-Protocol-style E2EE rollout. **No messages are actually encrypted yet** — the server still receives and persists plaintext bodies in `messages.ciphertext`, and the frontend's crypto layer is a passthrough mock. **v0.9.0 will be the real E2EE beta.**

### Added

#### Crypto schema (`Signalix-api`)

- **Migration `V14__crypto_foundation.sql`**:
  - `device_identity_keys(device_id, registration_id, identity_key, signing_key, key_algorithm, registered_at, updated_at)` — one row per device with the long-term identity + signing public keys + Signal-style registration_id. Upsert on register.
  - `signed_pre_keys(device_id, key_id, public_key, signature, key_algorithm, created_at, rotated_at)` — rotated periodically; prior rows kept for late handshakes with `rotated_at` set. Partial index `WHERE rotated_at IS NULL`.
  - `pre_keys(device_id, key_id, public_key, key_algorithm, created_at, consumed_at)` — one-time bundle, atomically claimed by `GET /crypto/users/:userId/key-bundle` (`UPDATE ... WHERE (device_id, key_id) IN (SELECT ... FOR UPDATE SKIP LOCKED)`). Partial index on unconsumed rows.
  - `messages` ALTERs with five envelope columns: `encryption_version INTEGER NOT NULL DEFAULT 0`, `sender_device_id UUID`, `recipient_device_id UUID`, `pre_key_id INTEGER`, `signed_pre_key_id INTEGER`. Defaults preserve the v0.7.x plaintext flow.
- All key material is stored as raw `BYTEA`; the API encodes / decodes base64url on the wire.

#### Crypto API endpoints (`Signalix-api`)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/crypto/devices/keys` | JWT | Initial publish of identity / signing / signed-pre-key / first batch of one-time pre-keys. Idempotent. DeviceId always from JWT, never from body. |
| PATCH | `/api/v1/crypto/devices/keys/signed-pre-key` | JWT | Periodic rotation. Prior signed pre-key marked `rotated_at = NOW()` (kept for in-flight handshakes). |
| POST | `/api/v1/crypto/devices/keys/pre-keys` | JWT | Top up the one-time pool. Returns the new unconsumed count. |
| GET | `/api/v1/crypto/users/:userId/key-bundle` | JWT | Return every device bundle for the target user. Each call atomically claims one unconsumed pre-key per device (`FOR UPDATE SKIP LOCKED`); falls back to signed-pre-key-only when the pool is exhausted. |

- New `CryptoModule` (controller + service + DTOs). `class-validator` ensures every key/signature on the wire is base64url and within reasonable size bounds. The DB layer treats all material as opaque blobs — **no signature verification yet** (that lands with v0.9.0 once the protocol is wired).

#### Contracts (`Signalix-contracts`)

- New file `api/crypto.contract.ts` with `DeviceIdentityKeyDTO`, `PreKeyDTO`, `SignedPreKeyDTO`, `DeviceKeyBundleDTO`, `KeyBundleResponse`, `RegisterDeviceKeysRequest/Response`, `RotateSignedPreKeyRequest/Response`, `UploadPreKeysRequest/Response`. Re-exported from `api/index.ts`.
- `MessageDTO`, `SendMessageRequest`, `ClientMessageSendPayload`, `ServerMessageNewPayload` each gain five optional envelope fields (`encryptionVersion`, `senderDeviceId`, `recipientDeviceId`, `preKeyId`, `signedPreKeyId`). All optional; v0.7.x consumers continue to work unchanged.

#### Frontend crypto scaffolding (`Signalix-frontend`)

- New `src/lib/crypto/` directory:
  - **`crypto.types.ts`** — re-exports of contracts DTOs + the `CryptoService` interface + `EncryptedEnvelope` shape + `CryptoStatus` (used by debug / settings UI).
  - **`crypto.service.ts`** — exports a singleton `cryptoService: CryptoService` (currently the mock) plus `getCryptoStatus()`. Swap point: a single line change replaces the implementation when v0.9.0 ships.
  - **`crypto.mock.ts`** — `MockCryptoService`. `encryptForRecipient` returns `{ ciphertext: plaintext, encryptionVersion: 0 }`; `decryptIncoming` passes plaintext through but surfaces `[encrypted message — upgrade to view]` if it ever receives a non-zero version (lets future-versioned clients coexist without crashing the message list on a stale UI). Dev log: `[signalix-crypto] mock service ready — no E2EE active`.
- **Not integrated** — the service is NOT yet called from `chat.store.sendMessage` or the WS receive path. The scaffolding exists so v0.9.0's wiring is a focused change at known call sites.

### Read / write of envelope columns (`Signalix-api`)

- `SendMessageDto` widened with `encryptionVersion`, `senderDeviceId`, `recipientDeviceId`, `preKeyId`, `signedPreKeyId` (all `@IsOptional()`, validated with `class-validator`).
- `MessagesService.sendMessage` INSERTs the new columns; default `encryption_version = 0` keeps existing flows untouched.
- `ChatsService.getMessages` SELECTs the new columns and maps them onto `MessageDTO` only when non-default — plaintext rows stay clean in the JSON response.

### Not changed

- Direct chats, group chats, media, files, voice notes, reactions, replies, forwards, edit, delete, sidebar + in-chat search, push notifications — **all continue to work as v0.7.1**.
- Realtime service untouched. The new envelope fields ride on the existing WS payloads as additive optional properties; no new events.
- Infra: no new buckets, services, or env vars. Migration V14 is applied by the existing Flyway service.

### Roadmap signal

- **v0.9.0 — E2EE beta**: real X25519 / Ed25519 keypair generation in the browser, signed pre-key signature verification on publish, X3DH-style handshake on first message, Double Ratchet for ongoing sessions, multi-device fanout. Backend signature verification, key rotation policies, key-server abuse protections. Client-side IndexedDB store for session state.

## v0.7.1 — 2026-06-07

### Added

#### In-chat search (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`GET /api/v1/chats/:chatId/search?q=&limit=&cursor=`** — substring lookup scoped to a single chat the caller participates in. Same ILIKE machinery as the global endpoint but with one chat in scope. Matches `message_type IN ('text', 'file')` — FILE bodies are JSON-stringified blobs that include the filename, so substring-matching the raw `ciphertext` catches filename hits without needing a JSON cast. Default `limit` 100 (max 200) so the UI can render "X of Y" navigation across the entire match set. Cursor on `created_at DESC` for paging through huge match sets when needed.
- **New contracts** — `SearchInChatRequest`, `InChatSearchMatchDTO`, `SearchInChatResponse`.
- **`SearchInChatDto` (API)** — `class-validator`: `q` 2–200 chars, `limit` 1–200.
- **`Signalix-frontend/src/lib/api-client.ts → searchInChat(chatId, q, opts)`** — typed wrapper.
- **MessageView header morphs into a search bar** when the magnifier icon is tapped: ✕ close + pill input with magnifier glyph + match counter (`3 of 12`, `No matches`, `…` while loading) + ↑/↓ navigation. Same shell on desktop and mobile so the existing header-action layout doesn't reflow.
- **Keyboard** — `Enter` next, `Shift+Enter` previous, `Esc` close. The input is auto-focused when search opens.
- **Match navigation + highlighting** — `MessageBubble` gains three optional props: `searchTerm`, `isMatch`, `isActiveMatch`. Matched bubbles get a soft amber ring (`signalix-search-match`); the focused match gets a stronger blue ring with glow (`signalix-search-active`) plus an `useEffect`-driven `scrollIntoView({ block: 'center' })`. TEXT bubbles render every occurrence of the query inline via `<mark>` (`highlightMatches` helper) — case-insensitive, all occurrences. FILE bubbles get the ring only (filename is already substring-matched server-side).
- **Stale-response protection** — a `searchSeqRef` increments on every new query; responses for older sequence numbers are dropped. Same protection added to the sidebar global search.
- **CSS utilities** — `.signalix-search-match` and `.signalix-search-active` added to `globals.css` with light/dark variants. The existing `.signalix-search-hit` (transient ring from sidebar click-through) is untouched.

#### Sidebar global search — pagination scroll (`Signalix-frontend`)

- `ChatSidebar.handleResultsScroll` watches the results container; firing `loadMoreMessageResults()` when within 120 px of the bottom. `messageNextCursor` + `messageHasMore` + `loadingMore` state added; existing `searchMessages` helper already accepted `{ cursor }`. A "Load more" fallback button appears below the list when there's more to fetch.
- `activeQueryRef` tracks the current query so cursor-paged responses from older queries are discarded; the container also scrolls back to top on every new query.

#### Message search (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`GET /api/v1/messages/search?q=&limit=&cursor=`** — case-insensitive substring lookup over the caller's accessible TEXT messages. `q` is required (2–200 chars); `limit` defaults to 20 (max 50); `cursor` is the base64-ISO of the previous page's last row. Sort is `created_at DESC`.
- **SQL implementation** — single `messages` query with:
  - `m.ciphertext ILIKE $pattern ESCAPE '\\'` (input escaped to neutralise `%`, `_`, `\\` from the user).
  - `EXISTS chat_participants` for access control.
  - `NOT EXISTS message_deletions` (per-user deletion) and `NOT EXISTS chat_deletions` cutoff (`m.created_at <= cd.deleted_at`).
  - `LEFT JOIN LATERAL` for direct-chat label/avatar resolution — picks the other participant's display name + avatar in the same query rather than a follow-up round-trip.
  - `LEFT(m.ciphertext, 280)` keeps the response payload small.
  - Keyset cursor `m.created_at < $cursorTs::timestamptz`.
- **New contracts** — `SearchMessagesRequest`, `MessageSearchResultDTO`, `SearchMessagesResponse`. `MessageSearchResultDTO` is self-contained (chat label / avatar, sender info, snippet) so clients can render rows without an extra lookup against their cached chat list.
- **`SearchMessagesDto` (API)** — `class-validator` rules: `q` 2–200 chars, `limit` 1–50.
- **`Signalix-frontend/src/lib/api-client.ts → searchMessages(q, { limit?, cursor? })`** — typed wrapper.
- **Sidebar UX** — the existing search input now fires both `searchUsers` and `searchMessages` in parallel inside a single `useTransition`. Results panel renders two sections: **People** (existing user-search behaviour) and **Messages** (new). Snippet rendering inlines a `<mark>` with translucent blue background for the matched substring; if the match position is > 60 chars into the body, the snippet windows around it instead of always slicing from the start.
- **Click-through to message** — message-result click navigates to `/chats/<chatId>?m=<messageId>`. `MessageView` reads `m` via `useSearchParams`, finds the row via `data-message-id`, and `scrollIntoView({ block: 'center' })` + applies a transient `signalix-search-hit` ring (new utility in `globals.css`) that fades after ~2 s. The auto-scroll-to-bottom effect is skipped on the first render when targeting a specific message so we don't fight the highlight scroll.

### Known limitations (v0.7.1)

- Scroll-to-message only works when the target is in the loaded page (default 50 newest messages). Older messages: open the chat, scroll up manually.
- Sidebar messages section shows only the first 12 hits — pagination via `nextCursor` is exposed by the API but not yet wired into the UI.
- ILIKE without a trigram or FTS index — fine for current dataset sizes; future iteration can add `pg_trgm` or `tsvector` for sublinear search.
- TEXT messages only — image / file / voice payloads are URLs/JSON, not user content.

## v0.7.0 — 2026-06-07

### Added

#### Group profile + avatar + description (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **Migration `V13__chats_avatar_description.sql`** — adds two nullable columns to `chats`: `avatar_url TEXT` (public MinIO URL) and `description TEXT` (≤500 chars enforced by the API DTO). Direct chats leave both `NULL`.
- **`ChatDTO`** — surfaces optional `avatarUrl` and `description`. Returned by `getUserChats()` so list rendering and headers pick the new fields up without an extra fetch.
- **`POST /api/v1/chats/:chatId/avatar`** — `multipart/form-data` field `avatar`. JPEG / PNG / WebP, max 5 MB. Owner or admin only. Stored under `chats/{chatId}/{uuid}.{ext}` in the existing `signalix-avatars` MinIO bucket — no new bucket. Returns `{ chatId, avatarUrl }`. Replacing an avatar prunes the stale object in-band; failure to prune is logged but never propagated.
- **`DELETE /api/v1/chats/:chatId/avatar`** — owner / admin only; clears the column and deletes the underlying object.
- **`PATCH /api/v1/chats/:chatId`** — body widened from `{ title }` to `{ title?, description? }`. Owner/admin only. Empty body → 400. `description: null` (or empty string) clears the column. `UpdateGroupChatRequest` / `UpdateGroupChatResponse` are widened accordingly in contracts; existing callers passing only `title` continue to work.
- **Frontend group profile UI** — `GroupInfoModal` reworked into a full details surface: avatar with upload / remove menu over a camera badge (owner / admin), editable description with multiline textarea + Save/Cancel, refreshed per-member three-dot menu hosting "Transfer ownership" and "Remove from group", "Add Members" picker (existing), "Leave Group" (existing). Avatar surfaces also picked up by `ChatItem` (sidebar) and `MessageView` (header) so the new image propagates immediately after upload.

#### Transfer ownership (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`POST /api/v1/chats/:chatId/transfer-ownership`** body `{ newOwnerId }`.
- Owner-only. Runs as a single DB transaction: verifies the chat type, the requester is the current owner, the target is a member, then atomically demotes the requester to `admin`, promotes the target to `owner`, bumps `chats.updated_at`, and returns the full refreshed participant list.
- New contracts: `TransferGroupOwnershipRequest` `{ newOwnerId }`, `TransferGroupOwnershipResponse` `{ chatId, ownerId, previousOwnerId, participants }`.
- Store action `transferGroupOwnership(chatId, newOwnerId)` syncs participants into the cached chat after the API returns.

### Changed

- **`Signalix-api/src/chats/chats.service.ts`** — `assertManager(chatId, userId, action)` private helper centralises the owner/admin role check used by update / avatar endpoints; keeps error messages and behaviour consistent across surfaces. `ChatsModule` now imports `StorageModule` since group avatars share the user-avatar storage abstraction.
- **`updateGroupChat`** store action signature widened from `(chatId, title)` to `(chatId, { title?, description? })`. Existing `GroupInfoModal` rename flow updated.

### Not changed

- Realtime service untouched — group improvements are REST-driven and persisted by the API. No new events.
- Infra: no new buckets, services, or env vars. The new avatar key prefix lives inside the existing `signalix-avatars` bucket.

## v0.6.1 — 2026-06-07

### Added

#### Voice messages (`Signalix-frontend`, `Signalix-api`)

- **`POST /api/v1/media/voice`** — new endpoint on `MediaController`. Accepts a single audio blob under the `audio` multipart field. Allowed MIME types: `audio/webm`, `audio/ogg`, `audio/mp4`, `audio/aac`, `audio/x-m4a`, `audio/mpeg`, `audio/wav`, `audio/x-wav`. Max 10 MB. Object key `voice/{userId}/{uuid}.{ext}` in the existing `signalix-media` bucket (no new bucket).
- **`MediaService.uploadVoice()`** — mime allowlist + size check + `pickAudioExt()` helper to derive an extension from `mimetype` when the upload had no filename.
- **Composer mic + recording state** — `MessageInput.tsx` now hosts `startRecording`, `cancelRecording`, `finishRecordingAndSend` with full `MediaRecorder` lifecycle: `getUserMedia`, `MediaRecorder.isTypeSupported`-driven MIME selection (Opus on Chrome/Edge/Firefox, MP4/AAC on Safari), 250 ms timer with a 5-min hard cap, microphone teardown on unmount or cancel.
- **`VoiceBubble` player** — new component (`components/VoiceBubble.tsx`). Hidden `<audio preload="metadata">`, glass circular play/pause button, linear progress track, mm:ss timestamp. `loadedmetadata` + `durationchange` listeners override the recorder-reported duration with the true blob duration once available.
- **`uploadVoice(blob, filename)`** — frontend API client wrapper. Preserves `blob.type` so the server-side mime allowlist matches.
- **`MessageView.tsx` AUDIO routing** — `isAudio` branch, `parseVoiceInfo()` parses `JSON.stringify({ url, duration, size })` ciphertext, `VoiceBubbleSection` wraps `VoiceBubble` with the standard time/status footer.
- **Previews for AUDIO** — `🎙️ Voice message`:
  - Sidebar last-message preview (`ChatItem.tsx`).
  - Browser notification body (`chat.store.ts` MESSAGE_NEW handler).
  - Reply preview (`MessageView.getReplyPreviewText`).
  - Web Push body (`MessagesService.previewForPush()`).
- **MIC icon** added to `MessageInput.tsx`; the mic button swaps with the send button when the textarea is empty and no attachment is staged.

### Fixed

#### Voice send round-trip — `SendableMessageType` propagation (`Signalix-contracts`, `Signalix-api`, `Signalix-realtime`, `Signalix-frontend`)

- Pre-fix symptom: voice notes uploaded to MinIO (returned 201, file readable as `.webm`) but no chat row was ever created and the recipient never saw anything.
- Root cause: four independent type/value bottlenecks restricted `messageType` to `TEXT | IMAGE | FILE`:
  1. `ClientMessageSendPayload.messageType` (contracts, WS).
  2. `SendMessageRequest.messageType` (contracts, REST).
  3. `SendMessageDto.messageType` `@IsIn([TEXT, IMAGE, FILE])` + TS type (API). **This was the runtime breaker** — `class-validator` rejected AUDIO with a 400 even though the upload had succeeded.
  4. `sendMessage()` payload typing in `Signalix-realtime/src/common/api-client.ts`.
  5. `chat.store.sendMessage` did `as MessageType.TEXT | MessageType.IMAGE | MessageType.FILE` (TS-only cast).
- Fix: new exported type alias `SendableMessageType = TEXT | IMAGE | FILE | AUDIO` in `Signalix-contracts/src/enums/message-type.enum.ts`. All four bottlenecks now reference it. `SendMessageDto` adds `MessageType.AUDIO` to its `@IsIn(...)` array so runtime validation accepts the type. No DB migration needed — `V3__chat.sql`'s `CHECK (message_type IN ('text', 'image', 'video', 'audio', 'file'))` already permitted `'audio'`.

## v0.6.0 — 2026-06-06

### Added

#### Progressive Web App foundation (`Signalix-frontend`)

- **`public/manifest.webmanifest`** — name, short_name, start_url `/chats`, scope `/`, display `standalone`, theme `#007aff`, background `#0c0c12`, two SVG icons (purpose `any` + `maskable`), categories `social, communication`.
- **SVG icons** — `public/icon.svg` (standard 512×512), `public/icon-maskable.svg` (same glyph with safe-area padding), `public/apple-touch-icon.svg` (180×180 for iOS add-to-home-screen). Brand chat-bubble mark, white on iOS-blue.
- **Service worker** at `public/sw.js`. Minimal, no caching of API or WebSocket traffic. Handlers: `install` (skipWaiting + version log), `activate` (clients.claim), `fetch` (no-op pass-through that satisfies install criteria), `message` (SKIP_WAITING).
- **`ServiceWorkerRegistration` client component** — calls `navigator.serviceWorker.register('/sw.js', { scope: '/' })` on `window.load`. Surfaces both success and failure to the console in dev (previously silently swallowed).
- **`InstallPrompt` component** — captures `beforeinstallprompt`, shows a dismissible glass banner at the bottom of the viewport; dismissal persisted in `localStorage` under `signalix-install-dismissed`; auto-hides on `appinstalled`.
- **Next.js metadata** — `Metadata.manifest`, `Metadata.appleWebApp` (`capable`, `title`, `statusBarStyle: black-translucent`), `Metadata.icons` (svg + apple-touch), `Viewport.themeColor` per `prefers-color-scheme`.

#### Liquid Glass UI refresh (`Signalix-frontend`)

- Translucent surfaces with `backdrop-blur-xl/2xl` replace opaque cards across chat shell, sidebar, modals, profile, install banner, auth pages.
- Layered background gradients (warm white in light, deep graphite + faint blue tint in dark — no pure black). New utility classes `surface-glass`, `surface-glass-strong`, `shadow-glass`, `shadow-glass-sm` in `globals.css`.
- Soft motion: `hover:scale-[1.04]` / `active:scale-[0.98]` on rail items, `hover:scale-[1.005]` on chat rows, 200 ms transitions.
- Pill-shaped inputs (Apple Search / visionOS style), rounded-full `MessageInput` composer with glass border + focus glow.
- Graphite glass message bubbles (no purple): outgoing `bg-[#1d1d1f]/[0.85]` light / `bg-white/[0.10]` dark with white text; incoming `bg-white/65 dark:bg-white/[0.06]` glass. Bubble corner radius `24px` with `8px` tail asymmetry.
- Sidebar active state is now `bg-white/60 dark:bg-white/[0.08]` translucent — blue reserved for primary CTAs only (logo, send, install, submit).
- Dropdown menus + reaction popovers use frosted glass with hairline borders and `shadow-glass`.

#### Web Push notifications (`Signalix-api`, `Signalix-frontend`, `Signalix-infra`)

- **Migration `V12__push_subscriptions.sql`** — table `push_subscriptions(id, user_id, endpoint, p256dh, auth, created_at, updated_at)` with `UNIQUE(user_id, endpoint)` and `idx push_subscriptions_user_id`. Multi-device support: one row per (user × browser endpoint).
- **`PushModule`** in `Signalix-api` — `PushController` + `PushService` + `dto/subscribe.dto.ts`.
  - `GET /api/v1/push/public-key` (public) — returns `{ publicKey }` so the frontend can subscribe without baking the key as a build-arg.
  - `POST /api/v1/push/subscribe` (JWT) — upserts the device subscription via `ON CONFLICT (user_id, endpoint) DO UPDATE`.
  - `DELETE /api/v1/push/unsubscribe` (JWT) — removes the subscription for the caller.
- **`web-push` npm package** added. `PushService.onModuleInit` configures VAPID once at boot. `sendToUser(userId, payload)` POSTs the payload via `webpush.sendNotification` to every device on file for the user; subscriptions returning **404** or **410** are pruned automatically.
- **Dispatch on send** — `MessagesService.sendMessage()` issues a fire-and-forget push to every other participant whose `presence.status IS NULL OR != 'online'`. Per-message body uses `previewForPush()`: `📷 Photo`, `📎 {filename}`, or the trimmed text. Push failure never blocks the response.
- **VAPID env vars** — `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` added to `ConfigService`, `.env.example`, and `Signalix-infra/env/api.env.example`. Empty keys disable push entirely (the service logs a warn and turns into a no-op).
- **Frontend push client** — `src/lib/push.ts`:
  - `getPushStatus()` returns `{ supported, permission, subscribed }`.
  - `ensureRegistration()` idempotently registers `/sw.js` and `await navigator.serviceWorker.ready` before subscribing — avoids the "subscription tied to a SW the browser later evicted" failure mode that surfaced as `Pruned stale push subscription` in API logs.
  - `enablePush()` requests permission, tears down any prior subscription (browser + server) for a clean re-subscribe, fetches the VAPID key, calls `PushManager.subscribe`, and POSTs to the API. Dev logs every step.
  - `disablePush()` tears down browser + server, idempotent.
- **`PushSettingsCard` in `/settings/profile`** — surfaces permission status (`Not supported` / `Blocked` / `Off` / `On`), Enable / Disable buttons, and any error returned by the flow.
- **`sw.js` push + notificationclick handlers** — `push` event parses the JSON payload and calls `self.registration.showNotification(title, options)` with title (sender name), body (preview), icon (avatar), badge (app icon), tag (`signalix-chat:{chatId}` to coalesce). `notificationclick` finds an open Signalix client and `client.focus() + postMessage({ type: 'NAVIGATE', path })`, falling back to `clients.openWindow(targetPath)`. The frontend's `ServiceWorkerRegistration` listens for `NAVIGATE` and calls `router.push(path)` to keep the SPA experience.

### Fixed

#### Profile page could not scroll (`Signalix-frontend`)

- Body is locked at `100dvh; overflow:hidden` for the chats shell. The profile page's `min-h-screen` wrapper got clipped — the Sign Out button at the bottom was unreachable.
- Wrapper changed to `h-full overflow-y-auto` so the profile content scrolls within the locked viewport. Sticky nav bar still pinned at the top of that scrolling container.
- Bottom padding switched to `calc(env(safe-area-inset-bottom) + 2.5rem)` so the last option clears the iOS home indicator and browser chrome.

#### Duplicate profile entry in the left rail (`Signalix-frontend`)

- The desktop `IconRail` had both a Settings gear and a bottom avatar — both opening `/settings/profile`. Removed Settings gear. Bottom-rail trio is now Theme → Logout → Avatar/Profile, all sharing the same 40×40 rounded-2xl glass shell and motion. The avatar button uses the real user image (`Avatar` component with initials fallback).
- New Logout button (red accent on hover) calls `auth.store.logout()` (disconnects WS + clears persisted session) + `router.replace('/login')`.
- Normalized SVG sizes: moon/sun/monitor/group/compose were using `w-4.5 h-4.5` — invalid Tailwind utilities that silently fell back to the SVG's intrinsic width, making them render visibly larger. All bottom-rail and sidebar header icons now use `w-5 h-5 stroke-[1.6]`.

#### Service worker registration was silent on failure (`Signalix-frontend`)

- `ServiceWorkerRegistration` used to `.catch(() => {})` on `register()`, leaving DevTools showing "no SW registered" without any console clue. Replaced with explicit dev-mode `console.info` on success (scope, active/installing/waiting state) and `console.error` on failure. Added matching `console.info` to the SW's `install`, `activate`, and `push` events.

## v0.2.0 — 2026-06-04

### Added

#### Google OAuth login (`Signalix-api`, `Signalix-frontend`)

- No new migration — `auth_providers`, `users`, and `device_sessions` already support all required columns (`provider_email`, `avatar_url`, nullable `password_hash`; `CHECK` constraint already allows `'google'`).
- **`ConfigService`** — added `frontendUrl`, `googleClientId`, `googleClientSecret`, `googleCallbackUrl` getters. `FRONTEND_URL` previously read as `process.env` in `forgotPassword` is now routed through the service.
- **`GET /api/v1/auth/google`** — redirects the browser to Google's OAuth consent screen (`openid email profile` scopes).
- **`GET /api/v1/auth/google/callback`** — receives the authorization code from Google:
  1. Exchanges code for an access token via `https://oauth2.googleapis.com/token`.
  2. Fetches the user profile from `https://openidconnect.googleapis.com/v1/userinfo`.
  3. **Provider exists** → sign in existing user.
  4. **Email matches existing user, provider missing** → link Google provider to that user, then sign in.
  5. **No existing user** → create user (username derived + sanitized from email prefix, `is_verified = true`, `avatar_url` from Google), create `auth_providers` row, then sign in.
  6. Applies the same browser device-slot reuse logic as local login (revoke old session, reuse device, enforce device cap).
  7. Redirects to `FRONTEND_URL/oauth/callback?<AuthSessionDTO params>` on success, or `FRONTEND_URL/login?error=oauth_cancelled|oauth_failed` on error.
- **No Passport dependency** — OAuth2 code exchange is done with plain `fetch` to Google's endpoints.
- **`loginWithOAuth(session)`** added to `auth.store.ts` — stores session, connects WebSocket, sets store state.
- **`/oauth/callback`** page — reads `AuthSessionDTO` from query params, calls `loginWithOAuth`, redirects to `/chats`. Wrapped in `Suspense` (Next.js 15 requirement for `useSearchParams`).
- **`/login`** page — "Continue with Google" button (`<a href="${API_BASE}/api/v1/auth/google">`), divider, and inline OAuth error display (reads `?error` from URL via `Suspense`-wrapped `useSearchParams`).
- **`.env.example`** — added `FRONTEND_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`.

#### Resend email integration for password reset (`Signalix-api`, `Signalix-infra`)

- **`resend` npm package** added to `Signalix-api` dependencies.
- **`src/email/email.module.ts`** — new `EmailModule`; exports `EmailService`; imported by `AuthModule`.
- **`src/email/email.service.ts`** — new `EmailService`:
  - Initialises the Resend client from `RESEND_API_KEY`. If the key is absent (empty), the service skips sending and logs the reset URL to the console so local dev can still test the flow.
  - `sendPasswordReset(to, resetUrl)` — sends a transactional email with subject "Reset your Signalix password", a clean HTML body with an inline "Reset Password" button, a 15-minute expiry notice, and a safe-to-ignore footer.
  - On Resend delivery failure: logs the error server-side; in non-production environments also logs the reset URL as a fallback.
- **`forgotPassword` in `auth.service.ts`** — replaced the `console.log` TODO with `this.email.sendPasswordReset(email, resetUrl)`. Token hashing, expiry, and idempotent one-token-per-user behaviour are unchanged.
- **`ConfigService`** — added `resendApiKey` (defaults to `''`) and `emailFrom` (defaults to `Signalix <onboarding@resend.dev>`).
- **`.env.example`** and **`Signalix-infra/env/api.env.example`** — added `RESEND_API_KEY` and `EMAIL_FROM` with local-dev guidance.

#### Apple OAuth login (`Signalix-api`, `Signalix-frontend`)

- No new migration — `auth_providers` `CHECK` constraint already includes `'apple'`; existing schema supports all required columns.
- **`ConfigService`** — added `appleClientId`, `appleTeamId`, `appleKeyId`, `applePrivateKey`, `appleCallbackUrl` getters.
- **`GET /api/v1/auth/apple`** — redirects the browser to Apple's sign-in consent screen (`name email` scopes, `response_mode: form_post`).
- **`POST /api/v1/auth/apple/callback`** — Apple uses `response_mode=form_post`, so the browser POSTs to the callback (not a GET redirect). Receives `code`, optional `user` JSON, and optional `error`:
  1. Builds a short-lived (5 min) ES256 JWT client secret signed with the `.p8` private key.
  2. Exchanges the authorization code for tokens at `https://appleid.apple.com/auth/token`.
  3. Decodes the `id_token` (JWT payload, no re-verification needed — returned directly by Apple).
  4. Extracts the stable `sub` claim (Apple user ID) and email (may be a private relay address).
  5. If email is absent in the ID token, falls back to the `user` JSON field sent **only on the first authorization**.
  6. **Provider exists** (`sub` match) → sign in existing user.
  7. **Email matches existing user, provider missing** → link Apple provider to that user, then sign in.
  8. **No existing user** → create user (username derived from email prefix, `display_name` from `user.name` if present, `is_verified = true`), create `auth_providers` row, then sign in.
  9. Applies the same browser device-slot reuse logic as local login.
  10. Redirects to `FRONTEND_URL/oauth/callback?<AuthSessionDTO params>` on success, or `FRONTEND_URL/login?error=oauth_cancelled|oauth_failed` on error.
- **No Passport dependency** — ES256 client secret signed via `JwtService.sign()` with `algorithm: 'ES256'` override; token exchange done with plain `fetch`.
- **`APPLE_CALLBACK_URL` must be HTTPS** — Apple does not permit plain HTTP redirect URIs. Use a tunnel (e.g. ngrok) for local development.
- **"Continue with Apple" button** added to `/login` page.
- **`.env.example`** — added `APPLE_CLIENT_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY`, `APPLE_CALLBACK_URL`.

#### Email verification (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`V6__email_verification.sql`** — new Flyway migration. Creates `email_verification_tokens(id, user_id, token_hash, expires_at, used_at, created_at)`. `UNIQUE` index on `token_hash`; index on `user_id`.
- **`VerifyEmailRequest`**, **`VerifyEmailResponse`**, **`ResendVerificationRequest`**, **`ResendVerificationResponse`** added to `@signalix/contracts` (`auth.contract.ts`).
- **`INVALID_VERIFICATION_TOKEN`** added to `ErrorCode` enum.
- **`POST /api/v1/auth/verify-email`** — public endpoint. Accepts `{ token }`. Hashes the raw token, looks up the record, validates expiry and `used_at`. Sets `users.is_verified = true` and stamps `used_at`. Returns `400 INVALID_VERIFICATION_TOKEN` if invalid or expired.
- **`POST /api/v1/auth/resend-verification`** — public endpoint. Accepts `{ email }`. Always returns success (prevents enumeration). If a local, unverified user is found: deletes any existing token, inserts a new one (24-hour TTL), and sends a verification email via `EmailService`.
- **`register()` in `auth.service.ts`** — after the transaction completes, fires `sendVerificationEmail` as void (email failure does not roll back registration). New local users start with `is_verified = false`.
- **`loginWithGoogle` / `loginWithGitHub` / `loginWithApple`** — when linking an OAuth provider to an existing unverified local user, sets `is_verified = true` as part of the link `UPDATE`.
- **`EmailService.sendEmailVerification(to, verificationUrl)`** — added to `email.service.ts`; same Resend-with-fallback pattern as `sendPasswordReset`. Subject: "Verify your Signalix email address". HTML body includes an "Verify Email" button and a 24-hour expiry notice.
- **`/verify-email`** page — reads `?token` from query params, calls `POST /auth/verify-email` on mount. Shows a green checkmark and "Email verified" on success; red X and "Verification link is invalid or has expired" on error. Wrapped in `Suspense` (Next.js 15 requirement).
- **Profile page verification badge** — email row in `/settings/profile` now shows a green "Verified" pill or amber "Unverified" pill alongside the address.
- **Resend button in profile** — shown below the account info card when `!user.isVerified && providers.includes('local')`. Calls `POST /auth/resend-verification`; shows inline confirmation on success; disabled during request.
- Token security: 32-byte `randomBytes(32).toString('hex')` raw token; only the SHA-256 hash is stored; raw token is never exposed in API responses or logs (only embedded in the email link).

#### GitHub OAuth login (`Signalix-api`, `Signalix-frontend`)

- No new migration — `auth_providers` `CHECK` constraint already includes `'github'`; `users` and `device_sessions` already support all required columns.
- **`ConfigService`** — added `githubClientId`, `githubClientSecret`, `githubCallbackUrl` getters.
- **`GET /api/v1/auth/github`** — redirects the browser to GitHub's OAuth consent screen (`read:user user:email` scopes).
- **`GET /api/v1/auth/github/callback`** — receives the authorization code from GitHub:
  1. Exchanges code for an access token via `https://github.com/login/oauth/access_token`.
  2. Fetches the user profile from `https://api.github.com/user`.
  3. If the profile email is null (private GitHub email), fetches the primary verified email via `https://api.github.com/user/emails`.
  4. **Provider exists** → sign in existing user.
  5. **Email matches existing user, provider missing** → link GitHub provider to that user, then sign in.
  6. **No existing user** → create user (username derived from GitHub login, `is_verified = true`, `avatar_url` from GitHub), create `auth_providers` row, then sign in.
  7. Applies the same browser device-slot reuse logic as local login (revoke old session, reuse device, enforce device cap).
  8. Redirects to `FRONTEND_URL/oauth/callback?<AuthSessionDTO params>` on success, or `FRONTEND_URL/login?error=oauth_cancelled|oauth_failed` on error.
- **No Passport dependency** — OAuth2 code exchange is done with plain `fetch` to GitHub's endpoints.
- **"Continue with GitHub" button** added to `/login` page alongside the Google button.
- **`.env.example`** — added `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_CALLBACK_URL`.

#### Password reset (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`V5__password_reset.sql`** — new Flyway migration. Creates `password_reset_tokens(id, user_id, token_hash, expires_at, used_at, created_at)`. Indexed on `user_id`. Token hash is `UNIQUE` to prevent hash collisions at the DB level.
- **`ForgotPasswordRequest`**, **`ForgotPasswordResponse`**, **`ResetPasswordRequest`**, **`ResetPasswordResponse`** added to `@signalix/contracts` (`auth.contract.ts`).
- **`INVALID_RESET_TOKEN`** added to `ErrorCode` enum.
- **`POST /api/v1/auth/forgot-password`** — public endpoint.
  - Accepts `{ email }`. Always responds with the same message (prevents email enumeration).
  - Generates a 32-byte random token, stores its SHA-256 hash with a 15-minute TTL.
  - Deletes any existing unused tokens for the same user (one active token per user).
  - Logs the reset URL to the API console (`FRONTEND_URL/reset-password?token=…`). Production would replace this with transactional email.
- **`POST /api/v1/auth/reset-password`** — public endpoint.
  - Accepts `{ token, newPassword }`. Validates token (not used, not expired).
  - Updates `password_hash` with a fresh bcrypt hash (cost 12).
  - Marks token as `used_at = NOW()` (one-time use).
  - Revokes **all** `device_sessions` for the user so every device must re-authenticate.
  - Returns `400 INVALID_RESET_TOKEN` if the token is invalid or expired.
- **`forgotPassword(email)`** and **`resetPassword(token, newPassword)`** added to frontend `api-client.ts`.
- **`/forgot-password`** page — email form; shows a neutral confirmation message on submit regardless of whether the email exists.
- **`/reset-password`** page — reads `?token` from query params; password + confirm fields; redirects to `/login` on success; shows error on invalid/expired token.
- **"Forgot password?" link** added to `/login` page next to the password label.

#### Delete for me (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`V4__message_deletions.sql`** — new Flyway migration. Creates `message_deletions(message_id, user_id, deleted_at)` with a composite primary key and cascade deletes. Indexed on `user_id`.
- **`DeleteMessageForMeRequest`** and **`DeleteMessageForMeResponse`** added to `@signalix/contracts` (`message.contract.ts`).
- **`POST /api/v1/messages/:messageId/delete-for-me`** — new authenticated endpoint.
  - Verifies the message exists and the caller is a chat participant.
  - Inserts into `message_deletions`; idempotent (`ON CONFLICT DO UPDATE` no-op returns the original `deleted_at`).
  - Does **not** delete from `messages` — other participants are unaffected.
- **`GET /api/v1/chats/:chatId/messages`** — updated to exclude messages the requesting user has deleted for themselves (`NOT EXISTS` subquery against `message_deletions`).
- **`deleteMessageForMe(messageId)`** added to frontend `api-client.ts`.
- **`deleteMessageForMe(chatId, messageId)`** action added to `chat.store.ts` — calls API, then filters the message from local state.
- **Trash icon** added to each message bubble in `MessageView`. Appears on hover, placed outside the bubble (left for received, right for sent). Disabled during in-flight request.

#### Partial username search (`Signalix-contracts`, `Signalix-api`, `Signalix-frontend`)

- **`GET /api/v1/users/search?q=<query>&limit=<n>`** — new authenticated endpoint.
  - Case-insensitive contains search against `username` column (`ILIKE %query%`).
  - Excludes the authenticated caller from results.
  - `limit` defaults to 10, maximum 25.
  - Returns `ApiResponse<UserSearchResponse>` where `UserSearchResponse = { users: PublicUserDTO[] }`.
- **`UserSearchRequest`** and **`UserSearchResponse`** added to `@signalix/contracts` (`user.contract.ts`).
- **`searchUsers(q, limit?)`** added to frontend `api-client.ts`.
- **Chat sidebar search** updated to use the new partial search endpoint and display a scrollable list of matching users. Clicking any result navigates to an existing chat or opens a new conversation composer.

### Unchanged

- `GET /api/v1/users/lookup/:username` (exact username lookup) remains fully functional.

---

## v0.1.1 — cleanup

- Removed obsolete `version: "3.9"` field from `Signalix-infra/docker-compose.yml`.
- Fixed `Signalix-api/.env.example` `PORT` from `3000` → `4000` to match Docker Compose env.
- Updated `Signalix-infra/scripts/up.sh` to check for `env/frontend.env` and print frontend URL.
- Rewrote all five `README.md` files with full local setup, Docker setup, env reference, endpoints, and v0.1 limitations.
- Added `Signalix-frontend` service to `Signalix-infra/docker-compose.yml`.

---

## v0.1.0 — initial MVP

### Signalix-contracts
- API contracts: auth, users, chats, messages, presence.
- WebSocket contracts: `ClientEvent`, `ServerEvent`, all payload interfaces.
- Shared enums: `MessageStatus`, `MessageLifecycleState`, `MessageType`, `PresenceStatus`, `ChatType`, `ParticipantRole`, `AuthProvider`, `DeviceType`.
- Error contracts: `ApiResponse<T>`, `ErrorCode`, `WsError`.
- Protocol: message lifecycle state machine, realtime routing rules.

### Signalix-api
- `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`.
- `GET /chats`, `GET /chats/:chatId/messages`.
- `POST /messages/send`, `POST /messages/:messageId/status`.
- `GET /users/lookup/:username`.
- `GET /presence`, `POST /presence/status`.
- PostgreSQL via `pg` pool, Flyway migrations (V1–V3), JWT (access 1 h / refresh 30 d).

### Signalix-realtime
- WebSocket server on port 5000.
- JWT authentication on connect (30 s timeout).
- In-memory connection routing (`chatId → Set<userId>`).
- `client.message.send` → persist via API → `server.message.sent` + `server.message.new` to recipients.
- `client.message.delivered` / `client.message.read` → status update + broadcast.
- `server.user.online` / `server.user.offline` presence broadcasts.
- Heartbeat / heartbeat-ack.
- Auto-reconnect on socket close.

### Signalix-frontend
- Register page, login page.
- Auth state with access/refresh token, silent 401 refresh.
- Chat list, message history (API), message send via WebSocket.
- Delivered/read status icons.
- Online/offline presence indicator.
- Exact username lookup → new conversation composer.

### Signalix-infra
- Docker Compose: `postgres`, `flyway`, `api`, `realtime`, `frontend`.
- Env examples for all services.
- Helper scripts: `up.sh`, `down.sh`, `logs.sh`, `migrate.sh`.
