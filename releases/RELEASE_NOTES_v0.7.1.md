# Signalix v0.7.1 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.7.0

---

## Overview

v0.7.1 ships **two layers of message search**:

1. **Global** — type a phrase in the sidebar; Signalix surfaces matching messages across every chat the caller belongs to. Click → jump to the bubble with a transient highlight. **New in this update:** the result list is now infinite-scroll.
2. **In-chat** — the magnifier icon in the chat header (previously disabled) is now wired. Tapping it morphs the entire header into a search bar with `X of Y` counter and ↑/↓ navigation between matches. iMessage / Telegram / WhatsApp pattern. Matched bubbles get an amber ring; the focused match gets a stronger blue ring + auto-scroll.

No migration. No new bucket. No realtime change.

---

## New Features

### Message search

**Endpoint** — `GET /api/v1/messages/search?q=<phrase>&limit=<n>&cursor=<base64-iso>`:

| Param | Required | Notes |
|---|---|---|
| `q` | yes | 2–200 chars. Matched case-insensitively (`ILIKE`). LIKE wildcards (`%`, `_`, `\`) in the user input are escaped before being interpolated. |
| `limit` | no | Default 20, max 50. |
| `cursor` | no | Base64-encoded ISO timestamp from a previous page's `nextCursor`. Keyset on `created_at DESC`. |

**Filters applied at the SQL layer:**

- Only TEXT messages (`m.message_type = 'text'`). Image / file / voice ciphertexts are URLs or JSON, not user content.
- Caller must be a participant of the chat (`EXISTS chat_participants`).
- Message not globally deleted (`m.deleted_at IS NULL`).
- Not per-user deleted (`NOT EXISTS message_deletions`).
- Respects the chat-level visibility cutoff from `delete-chat-for-me` (`NOT EXISTS chat_deletions` for messages at or before the cutoff).

**Response shape** — `MessageSearchResultDTO[]`:

```ts
interface MessageSearchResultDTO {
  messageId: UUID;
  chatId: UUID;
  chatType: ChatType;
  chatLabel: string;        // group title; OR other participant's display name (direct chat)
  chatAvatarUrl?: string;
  senderId: UUID;
  senderName: string;
  senderAvatarUrl?: string;
  ciphertext: string;       // server-truncated to 280 chars
  createdAt: ISODateString;
}
```

A `LEFT JOIN LATERAL` resolves the direct-chat "other participant" inline so direct-chat hits already include the right label and avatar — no follow-up round-trip needed.

### In-chat search

**Endpoint** — `GET /api/v1/chats/:chatId/search?q=<phrase>&limit=<n>&cursor=<base64-iso>`:

| Param | Required | Notes |
|---|---|---|
| `q` | yes | 2–200 chars. Case-insensitive ILIKE. LIKE wildcards in user input are escaped. |
| `limit` | no | Default 100, max 200 — in-chat UX needs the whole match set upfront for `X of Y` navigation. |
| `cursor` | no | Base64-encoded ISO timestamp for follow-up pages. Same keyset pattern as the global endpoint. |

**Filters applied at the SQL layer:**

- `message_type IN ('text', 'file')` — TEXT bodies and FILE filenames (the JSON-stringified payload is substring-matched so embedded filenames are searchable without a JSON cast).
- Caller must be a participant (`EXISTS chat_participants`).
- `m.deleted_at IS NULL`.
- `NOT EXISTS message_deletions` per-user.
- `NOT EXISTS chat_deletions` cutoff.

**Response** — `InChatSearchMatchDTO[]`:

```ts
interface InChatSearchMatchDTO {
  messageId: UUID;
  senderId: UUID;
  senderName: string;
  ciphertext: string;       // server-truncated to 280 chars
  messageType: MessageType;
  createdAt: ISODateString;
}
```

**Header UI** — when the magnifier icon is tapped:

- Header content morphs into: `✕` close · pill input with magnifier glyph · `X of Y` counter · ↑/↓ navigation.
- Same shell on desktop and mobile — the existing header layout doesn't reflow.
- Input is auto-focused; debounce 220 ms before firing the API call.
- `Enter` → next match; `Shift+Enter` → previous; `Esc` → close.

**Per-bubble highlighting:**

- `MessageBubble` gains three optional props: `searchTerm`, `isMatch`, `isActiveMatch`.
- Matched rows get a soft amber ring (`signalix-search-match` in `globals.css`).
- The focused row gets a stronger blue ring + glow (`signalix-search-active`) AND an `scrollIntoView({ block: 'center' })` from an `useEffect` on `isActiveMatch`.
- TEXT bubble bodies render every occurrence of the query inline as `<mark>` (`highlightMatches` helper). Case-insensitive, all occurrences. FILE bubbles get the ring only.
- Drafts skip search entirely — no real `chatId` to query.

### Sidebar global search — pagination scroll (new in this update)

- The Messages section in the sidebar search panel is now infinite-scroll.
- `ChatSidebar.handleResultsScroll` watches the scrollable container; when within 120 px of the bottom, it fires `loadMoreMessageResults()` against `searchMessages(q, { limit: 12, cursor })`.
- State: `messageNextCursor`, `messageHasMore`, `loadingMore`. A "Load more" fallback button appears below the list for keyboard / accessibility use.
- `activeQueryRef` tracks the current query so cursor-paged responses from older queries are discarded if the user typed something newer mid-flight.
- Container scrolls back to top on every new query.

### Sidebar UX

- The existing **Search** input now fires both `searchUsers` and `searchMessages` in parallel inside a single `useTransition`.
- Results panel renders two sections: **People** (existing behaviour) and **Messages** (new). Sections only render when they have hits.
- Snippet rendering inlines a `<mark>` with a translucent blue background over the matched substring. If the match position is far into the body, the snippet windows around the match (`…` prefix / suffix) instead of always slicing from the start.
- Time of the matched message is shown in the result row, mobile-friendly format (`formatChatTime`).

### Click-through to message

- Selecting a message result navigates to `/chats/<chatId>?m=<messageId>`.
- `MessageView`:
  - Reads `m` via `useSearchParams`.
  - Looks up the row via a new `data-message-id` attribute on each bubble wrapper.
  - `scrollIntoView({ behavior: 'smooth', block: 'center' })`.
  - Applies a transient `signalix-search-hit` class (defined in `globals.css`) that fades after ~2 s — a translucent blue ring with a soft glow on dark.
  - Skips the auto-scroll-to-bottom effect for that first render so the highlight scroll wins.

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
cd Signalix-realtime  && npm run typecheck                    # ✓ (unchanged)
cd Signalix-frontend  && npm run typecheck && npm run build   # ✓
```

Manual checks against the local Docker Compose stack:

1. Login. In sidebar Search, type a phrase you previously sent in some chat. People + Messages sections appear.
2. Click a message result. App navigates to the right chat and the matching bubble glows for ~2 seconds before settling.
3. Clear search → both sections empty out, chats list returns.
4. Search for `%` or `_` literals — the server escapes them so `%` matches the literal `%` character in messages.
5. As a non-member of a chat, search for a phrase that only appears in that chat → no result (the `EXISTS chat_participants` filter does the work).
6. Delete a message for yourself, then search for its text → no result for you (kept visible for other participants).
7. Delete a chat for yourself, then re-message the same participant and search a phrase from before the deletion → only matches after the cutoff appear.

---

## Known limitations

- **Global scroll-to-message** still only works when the target is in the loaded page (default 50 newest messages). Older messages: open the chat and scroll up. Auto-load-until-found is a planned follow-up.
- **In-chat search fetches 100 matches at a time.** The API exposes `nextCursor` for chats with more hits than that; wiring the navigation to pull additional pages on demand is a later iteration. The current "X of Y" reflects the loaded batch.
- **Inline `<mark>` highlight only on TEXT bubbles.** FILE/voice bubbles get the row-level ring only — the embedded filename isn't highlighted inside the FileCard yet.
- **ILIKE without index** — `messages.ciphertext ILIKE '%foo%'` does a sequential scan. Fine for current dataset sizes; a future `pg_trgm` or `tsvector` index can make it sublinear.
- **Search does NOT cover** voice transcription, image OCR, semantic / vector search, or AI search. Explicit non-goals.
