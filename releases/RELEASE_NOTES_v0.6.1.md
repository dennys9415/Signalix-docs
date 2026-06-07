# Signalix v0.6.1 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.6.0

---

## Overview

v0.6.1 is a focused feature release: **voice messages**. Users can now record a voice note from the composer, see a recording state, cancel or send, and play back through an inline `VoiceBubble` player in the chat. A small but load-bearing fix to the type system in `Signalix-contracts` made the round-trip work end-to-end — without it, voice notes uploaded to MinIO successfully but no chat row was ever created.

No breaking changes. No migration. No new bucket.

---

## New Features

### Voice messages

**Recording (composer)** — `MessageInput.tsx`:

- Mic button replaces the send button while the textarea is empty and no attachment is staged.
- Tapping the mic asks for microphone permission via `navigator.mediaDevices.getUserMedia({ audio: true })`. Permission errors (`NotAllowedError`, `NotFoundError`) surface above the input.
- The input row morphs into a recording bar: pulsing red dot + elapsed timer (mm:ss) + Cancel (X) + Send (arrow).
- MediaRecorder MIME is auto-selected via `MediaRecorder.isTypeSupported()`:
  - Chrome / Edge / Firefox → `audio/webm;codecs=opus`
  - Safari → `audio/mp4` (AAC)
- 5-minute hard cap; the timer auto-sends when it hits the cap.
- Cancel discards the take and releases the microphone tracks. The component's `useEffect` cleanup also releases the mic on unmount (chat changed, logout, navigation).

**Upload + send** — `lib/api-client.ts → uploadVoice(blob, filename)`:

- Wraps the captured blob in a `File`, preserving `blob.type` so the API mime allowlist matches.
- POSTs `multipart/form-data` to the new `POST /api/v1/media/voice` endpoint.
- On success, the composer emits a `MessageType.AUDIO` message whose `ciphertext` is `JSON.stringify({ url, duration, size })` — mirroring the `MessageType.FILE` shape.

**Player (chat bubble)** — `components/VoiceBubble.tsx`:

- Hidden `<audio preload="metadata">` drives playback.
- Glass circular play/pause button, linear progress track, mm:ss timestamp.
- `loadedmetadata` + `durationchange` listeners override the recorder-reported duration with the file's real duration once available.
- Outgoing-bubble variant: white-on-graphite. Incoming variant: blue accent on light glass.

**Routing inside the bubble** — `components/MessageView.tsx`:

- New `isAudio` flag, `parseVoiceInfo()` helper for the `ciphertext` JSON, `VoiceBubbleSection` wraps `VoiceBubble` with the standard time/status footer.
- AUDIO bubbles also get the `overflow-hidden` no-padding container that IMAGE / FILE already use.

**API endpoint** — `MediaService.uploadVoice()` + `POST /api/v1/media/voice`:

- Multipart field name: `audio`.
- Allowed MIME types: `audio/webm`, `audio/ogg`, `audio/mp4`, `audio/aac`, `audio/x-m4a`, `audio/mpeg`, `audio/wav`, `audio/x-wav`.
- Max 10 MB.
- Object key `voice/{userId}/{uuid}.{ext}` in the existing `signalix-media` bucket (no new bucket, no infra change).
- Returns `{ voiceUrl }`.

**Previews** — `🎙️ Voice message` everywhere AUDIO can appear:

- Sidebar last-message preview (`ChatItem.tsx`).
- Browser notification body (`chat.store.ts` MESSAGE_NEW handler).
- Reply preview chip in the composer (`MessageView.getReplyPreviewText`).
- Web Push body (`MessagesService.previewForPush()`).

---

## Fixes

### Voice send round-trip — `SendableMessageType` propagation

**Symptom (pre-fix):** voice notes uploaded to MinIO successfully (the API returned 201 and the `.webm` file was readable from the public URL), but no chat row was ever created and the recipient never received anything.

**Root cause:** four independent type/value bottlenecks restricted `messageType` to `TEXT | IMAGE | FILE`:

1. `ClientMessageSendPayload.messageType` (contracts, WebSocket).
2. `SendMessageRequest.messageType` (contracts, REST).
3. `SendMessageDto.messageType` with `@IsIn([TEXT, IMAGE, FILE])` + TS type (API). **This was the runtime breaker** — `class-validator` rejected AUDIO with a 400 even though the upload had succeeded, and the WS sender layer never surfaced the error.
4. `sendMessage()` payload typing in `Signalix-realtime/src/common/api-client.ts`.
5. `chat.store.sendMessage` did `(messageType ?? TEXT) as MessageType.TEXT | MessageType.IMAGE | MessageType.FILE` (TS-only cast — runtime passed AUDIO through, but the API DTO rejected it downstream).

**Fix:** new exported type alias `SendableMessageType` in `Signalix-contracts`:

```ts
// Signalix-contracts/src/enums/message-type.enum.ts
export type SendableMessageType =
  | MessageType.TEXT
  | MessageType.IMAGE
  | MessageType.FILE
  | MessageType.AUDIO;
```

All four bottlenecks now reference it. `SendMessageDto` adds `MessageType.AUDIO` to its `@IsIn(...)` array so runtime validation accepts the type.

**No DB migration needed** — `V3__chat.sql`'s `CHECK (message_type IN ('text', 'image', 'video', 'audio', 'file'))` already permitted `'audio'` since v0.1. Likewise `MessageDTO.messageType` and `ServerMessageNewPayload.messageType` already use the full `MessageType` enum, so persistence and broadcast were never the bottleneck.

---

## Migrations

None.

---

## New environment variables

None.

---

## Validation

```bash
cd Signalix-contracts && npm run typecheck && npm run build   # ✓
cd Signalix-api       && npm run typecheck && npm run build   # ✓
cd Signalix-realtime  && npm run typecheck && npm run build   # ✓
cd Signalix-frontend  && npm run typecheck && npm run build   # ✓
```

Manual checks performed against the local Docker Compose stack:

1. Login. Composer shows the mic button when the textarea is empty.
2. Tap mic → permission prompt → granted → recording bar with timer counting up.
3. Cancel: bar disappears, mic LED in the browser turns off.
4. Record again, send: bubble appears with play/pause + duration.
5. Recipient sees the same VoiceBubble. Sidebar last-message preview reads `🎙️ Voice message`.
6. Recipient offline: Web Push notification with body `🎙️ Voice message`.
7. Reload — REST history reload renders AUDIO bubbles correctly.
8. Text, image, file messages all continue to work.
9. Safari (audio/mp4), Firefox (webm), Chrome (webm/opus) all round-trip.

---

## Known limitations

- No waveform rendering, no scrubbing (seek-to-position), no playback speed.
- 5-minute hard cap on a single recording.
- iOS requires a user gesture before mic capture works (browser policy — already handled by our `tap → getUserMedia` flow).
- The recorder estimates duration from `Date.now()` deltas; the player overrides it with the `<audio>`'s real duration once metadata loads.
- No transcription, no voice-to-text, no voice effects — explicit non-goals for v0.6.1.
