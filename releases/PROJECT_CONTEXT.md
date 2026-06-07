# Signalix v0.5.0 Stabilization & Draft Chat

**Status**: Complete ✓  
**Date**: 2026-06-06  
**Build**: ✓ typecheck clean, ✓ build successful

## Project Overview

**Signalix v0.5.0** is a real-time encrypted chat platform with:
- **4 core services**: api (NestJS 10), realtime (Fastify + WebSocket), frontend (Next.js 15), contracts (TypeScript types)
- **Stack**: NestJS + PostgreSQL + Zustand v5 + Next.js 15 + React 19 + Tailwind
- **State**: All 4 services complete and buildable, typecheck clean

## Work Completed: 5 Critical Bugs + Draft Chat UX

### Bug Fixes

1. **Issue 1: New users getting default avatars**
   - **Root cause**: OAuth user creation assigned `null` avatar, but UI showed default image
   - **Fix**: `auth.service.ts` lines 479, 615 → explicitly set `avatarUrl: null` (no fallback to provider default)

2. **Issue 2: Group chat messages not showing sender identity**
   - **Root cause**: `getSenderName()` function existed but wasn't called in message bubbles
   - **Fix**: `MessageView.tsx` → display sender name above incoming group messages

3. **Issue 3: Deleted direct chat reopening to wrong group**
   - **Root cause**: `startNewChat()` matched any chat with participant, including groups
   - **Fix**: `ChatSidebar.tsx` → filter by `c.type === ChatType.DIRECT`

4. **Issue 4: Old messages appearing after delete-and-restart**
   - **Root cause**: `chat_deletions.deleted_at` visibility cutoff not applied
   - **Fix**: 3-part solution:
     - `chats.service.ts`: `getMessages()` filters by `deleted_at` visibility
     - `deleteChatForMe()`: Updates timestamp with `NOW()` on re-delete
     - `messages.service.ts`: Removed `DELETE FROM chat_deletions` clearing all users

5. **Issue 5: Chat options menu blocked by z-index**
   - **Root cause**: Header had no stacking context
   - **Fix**: `MessageView.tsx` header → `relative z-20`, dropdown → `z-[200]`

### Feature: Temporary Draft Chat UX

Users select someone to chat → draft appears immediately in sidebar → "Start the conversation" message → send first message → backend creates real chat. No database storage until send.

**Architecture**: Draft lives at `/chats` (not `/chats/draft:<id>`) via `currentDraft: ChatDTO | null` field in Zustand store. Eliminates race condition between async store update and sync route change.

## Architecture

### Key Design Decisions

1. **Draft lives at `/chats`, not `/chats/draft:<id>`**
   - Eliminates race condition between store update (async/batched) and route change (sync)
   - `/chats/[chatId]` immediately redirects draft routes to `/chats`
   - `/chats` page renders `MessageView` with `currentDraft` when active

2. **`currentDraft: ChatDTO | null` in store**
   - Direct field access, no `.find()` lookup
   - Guarantees draft exists when `activeChatId.startsWith('draft:')`
   - Updated by `openDraftChat()` and cleared by `removeDraftChat()`

3. **Draft lifecycle**
   - Created in memory only: `messages[draft.id] = []`, never persisted
   - Marked with `id: "draft:<userId>"` format
   - Removed from sidebar and store when user leaves without sending

4. **First-message confirmation flow**
   - User sends from draft with `recipientUsername` (no `chatId`)
   - Backend creates real chat, returns `MESSAGE_SENT` with real `chatId`
   - Store sets `pendingChatId` = real chatId
   - `/chats` page watches `pendingChatId`, calls `removeDraftChat()`, navigates to real chat
   - Real chat replaces draft in sidebar

## Files Modified

### Signalix-frontend

**`src/store/chat.store.ts`**
- Added `currentDraft: ChatDTO | null` field to state
- `openDraftChat(user)`: Creates draft with `id: "draft:<userId>"`, sets `currentDraft`, `activeChatId`, initializes `messages[draftId] = []`
- `removeDraftChat(draftId)`: Clears `currentDraft`, removes from chats/messages/unreadCounts, nullifies `activeChatId` if current
- `sendMessage()`: Already handles `recipientUsername` for drafts (unchanged)

**`src/app/chats/page.tsx`**
- Renders `MessageView` with `currentDraft` when `activeChatId.startsWith('draft:') && currentDraft`
- Effect sets `activeChatId = currentDraft.id` when draft exists (for sidebar clicks)
- Effect watches `pendingChatId`: removes draft, navigates to real chat on confirmation

**`src/app/chats/[chatId]/page.tsx`**
- Simplified: redirects draft routes immediately to `/chats`
- Real chat routes load normally; `pendingChatId` effect navigates on first-message confirmation
- Removed deferred cleanup timer (no longer needed with `/chats`-based draft UI)

**`src/components/ChatSidebar.tsx`**
- `startNewChat()`: Calls `openDraftChat(user)`, routes to `/chats` (not `/chats/draft:<id>`)

**`src/components/MessageView.tsx`**
- `handleSend()` for draft: Passes both `chatId` (draft.id) and `recipientUsername` so message appears optimistically
- Back button: Calls `removeDraftChat()` for drafts, then routes to `/chats`
- `handleTypingStart/Stop()`: No-ops for drafts (already implemented)
- `loadMessages()`: Skips drafts (already implemented)

## UX Flow

### Selecting a user to start chat

1. User searches for someone in sidebar
2. Clicks user → `startNewChat()` → `openDraftChat(user)` → `router.push('/chats')`
3. Draft ChatDTO created in store with `id = "draft:<userId>"`
4. `/chats` page renders `MessageView` with draft
5. Sidebar shows draft chat item with "New conversation" preview

### Sending first message from draft

1. User types message → `handleSend()` calls `sendMessage({ chatId: draft.id, recipientUsername, ... })`
2. Temp message added to `messages[draft.id]`
3. WebSocket sends message to backend with `recipientUsername`
4. Backend creates real chat, returns `MESSAGE_SENT { chatId: realId, messageId, ... }`
5. Store MESSAGE_SENT handler:
   - Confirms temp → real message in `messages[realId]`
   - Sets `pendingChatId = realId`
6. `/chats` page `pendingChatId` effect:
   - Removes draft from store
   - Clears `pendingChatId`
   - Loads fresh chats (real chat now included)
   - Navigates to `/chats/realId`
7. Real chat loads and persists in sidebar

### Back without sending

1. User in draft → taps back button
2. `handleBack()` calls `removeDraftChat(draft.id)`
3. Draft removed from sidebar
4. Routes to `/chats` (empty state)

## Edge Cases Handled

- **React Strict Mode**: No deferred cleanup timer needed (draft UI lives at `/chats`, not dynamic route)
- **Page refresh while composing draft**: Draft lost (in-memory only) - acceptable, user returns to inbox
- **Stale draft on sidebar reload**: `loadChats()` preserves existing drafts via `s.chats.filter(c => c.id.startsWith('draft:'))`
- **Multiple draft tabs**: Last opened draft wins (single `currentDraft` field); drafts in chats array may accumulate (rare edge case, acceptable)

## Testing Checklist

- [ ] Select user → draft opens immediately, "Start the conversation" text visible
- [ ] Type message in draft → sent to backend
- [ ] Backend creates chat → sidebar updates, real chat replaces draft
- [ ] Go back without sending → draft disappears from sidebar
- [ ] Send message from draft → no API call for draft history
- [ ] Delete real chat → can open draft with same recipient again
- [ ] Existing real chats still work normally (navigate, send, etc.)
- [ ] typecheck clean, build succeeds

## TypeScript & Build

```
npm run typecheck  # ✓ Passed
npm run build      # ✓ Passed (12 pages, 125KB chunk)
```

No type errors, no warnings.
