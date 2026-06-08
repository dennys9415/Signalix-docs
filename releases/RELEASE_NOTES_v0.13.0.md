# Signalix v0.13.0 — Release Notes

**Release date:** 2026-06-08
**Previous version:** v0.12.0

---

## Overview

v0.13.0 modernizes message search for the E2EE era. The v0.7.1
search backend (`/messages/search` + `/chats/:id/search` ILIKE on
`messages.ciphertext`) was built before encryption shipped — for
v0.10.0+ encrypted bodies the row stores `''` so server-side ILIKE
matched nothing. v0.13.0 solves this two ways:

1. **Server side** — extends the WHERE clause to also match the
   chat's group title and the sender's username / display name, so a
   query like `team-lunches` or `@jen` still resolves to the right
   chats even when the body is encrypted.
2. **Client side** — adds a parallel walk over `chat.store`'s
   in-memory message array. The store carries decrypted plaintext
   after `decryptStoredMessage` runs, so encrypted bodies become
   searchable without sending any plaintext back to the server.

A `mergeSearchResults` helper dedupes both result sets by
`messageId`. On mobile a new full-screen overlay replaces the
inline sidebar input so the chat list doesn't compete with the
search results for viewport space.

What ships:

- Server `searchMessages` / `searchInChat` WHERE extension.
- Client-side `searchLocalMessages` / `searchLocalChatMessages` over
  the in-memory store; includes filename matching inside
  `MediaMetadataV1` JSON for encrypted attachments.
- `mergeSearchResults` / `mergeInChatResults` dedup helpers.
- `MobileSearchOverlay` full-screen mobile search.
- 8 new vitest tests; 29/29 frontend total.

What does **not** ship:

- No full-text / vector search (per spec — ILIKE only).
- No bulk client-side decrypt-and-index for unloaded history (only
  what's already in the store gets searched locally).
- No new contracts or DB migrations.

---

## How the merged search works

```
User types "report" in the sidebar
  ↓ ChatSidebar.handleSearchChange("report")
  ↓ Promise.all([
  ↓   searchUsers("report"),
  ↓   searchMessages("report"),       ← server: matches users.username,
  ↓                                       users.display_name,
  ↓                                       chats.title (groups),
  ↓                                       messages.ciphertext (legacy plaintext)
  ↓ ])
  ↓ + searchLocalMessages("report", {chats, messagesByChat, currentUserId})
  ↓     walks state.messages — for each non-deleted, non-sentinel msg:
  ↓       TEXT  → m.ciphertext (decrypted body in the store)
  ↓       FILE  → JSON.parse(m.ciphertext).filename
  ↓       IMAGE → JSON.parse(m.ciphertext).filename (if any)
  ↓       AUDIO → JSON.parse(m.ciphertext).filename (if any)
  ↓
  ↓ mergeSearchResults(serverHits, localHits)
  ↓   • dedup by messageId
  ↓   • local entry wins (has real plaintext; server has '' for E2EE rows)
  ↓   • sort by createdAt DESC
  ↓
UI renders merged list
User taps a result
  ↓ router.push(/chats/:chatId?m=:messageId)
MessageView mounts
  ↓ useSearchParams() reads "?m="
  ↓ queries [data-message-id="…"] on the rendered bubbles
  ↓ scrollIntoView()
  ↓ applies .signalix-search-hit (2.2s transient highlight)
```

---

## Migration

- **No DB migration.**
- **No protocol break.** All wire shapes are identical to v0.12.0.
- **No contract bump** — `Signalix-contracts` is a no-op release.
- The realtime layer is untouched.

---

## Operational notes

- Deploy order: contracts → api → realtime → frontend.
- No env vars, no infra changes.

---

## Validation

- `npm run typecheck` — clean across all four code packages.
- `npm run build` — clean.
- `npm test` (frontend) — **29/29 vitest tests pass** (utils 7,
  fingerprints 8, file-crypto 6, local-search 8 ← new).

**Manual scenario** (matches the spec):

1. **Search keyword** — type a substring of a message you previously
   sent or received in an E2EE chat. The match appears in the result
   list (from the local walk, since the server can't see the
   encrypted body).
2. **Open result** — tap. The browser navigates to
   `/chats/:chatId?m=:messageId`.
3. **Scroll to message** — `MessageView` scrolls the bubble into
   view automatically.
4. **Highlight match** — the bubble gets the `signalix-search-hit`
   CSS class for ~2.2 seconds.
5. **Group / username** — typing a group name or a sender's
   username surfaces matches from the server side (covers chats
   the user hasn't loaded yet).
6. **Mobile** — tap the search bar in the sidebar; the
   `MobileSearchOverlay` takes over the screen with its own input
   and results.

---

## Files of interest

**API**
- `Signalix-api/src/messages/messages.service.ts` — `searchMessages` WHERE extension.
- `Signalix-api/src/chats/chats.service.ts` — `searchInChat` WHERE extension.

**Frontend**
- `Signalix-frontend/src/lib/local-search.ts` *(new)* — `searchLocalMessages`, `searchLocalChatMessages`, `mergeSearchResults`, `mergeInChatResults`.
- `Signalix-frontend/src/lib/local-search.test.ts` *(new)* — 8 vitest cases.
- `Signalix-frontend/src/components/MobileSearchOverlay.tsx` *(new)* — full-screen mobile search.
- `Signalix-frontend/src/components/ChatSidebar.tsx` — server + local merge in `handleSearchChange` / `loadMoreMessageResults`; mobile-only transparent button over the search input opens the overlay.
- `Signalix-frontend/src/components/MessageView.tsx` — server + local merge in the in-chat search effect.

---

## What's next — v0.14.0 preview

- Bulk decrypt-and-index on the client so chats the user hasn't
  opened in this session also become searchable locally.
- PostgreSQL full-text index on `chats.title` + `users.username` for
  larger deployments.
- Optional: client-side fuzzy matching (Levenshtein on display name
  + group title).
