# Signalix v0.7.0 — Release Notes

**Release date:** 2026-06-07
**Previous version:** v0.6.1

---

## Overview

v0.7.0 focuses on **group improvements**. Groups now have a proper profile surface: avatar, description, member roster with role badges, owner-managed membership, and ownership transfer. No new bucket, no new env var, no realtime change.

One additive migration: `V13__chats_avatar_description.sql`.

---

## New Features

### Group avatar

- **`POST /api/v1/chats/:chatId/avatar`** — multipart `audio` ⟶ correction: `avatar` field. JPEG / PNG / WebP, max 5 MB. Owner or admin only.
- **`DELETE /api/v1/chats/:chatId/avatar`** — owner / admin only.
- Stored under `chats/{chatId}/{uuid}.{ext}` in the existing **`signalix-avatars`** MinIO bucket. Reuses the same public-read policy as user avatars. Stale objects are pruned in-band when replaced.
- Returned in `ChatDTO.avatarUrl` from `GET /api/v1/chats` and surfaces in:
  - `ChatItem` (sidebar row)
  - `MessageView` header
  - `ForwardModal` chat picker
  - `GroupInfoModal` hero
- **Initials fallback** preserved everywhere when `avatarUrl` is empty.

### Group description

- **`chats.description`** column (V13). Plain text, max 500 chars (enforced by API DTO).
- **`PATCH /api/v1/chats/:chatId`** body widened from `{ title }` to `{ title?, description? }`. Owner / admin only. At least one field required.
- `description: null` (or empty string) clears the column.
- Editable in `GroupInfoModal` via an Add / Edit affordance under a "Description" section with multiline textarea + Save/Cancel.

### Member management (refresh of an existing surface)

- Adding members + removing members + self-leave were already in place since v0.5.0.
- v0.7.0 reworks the **per-member action menu** into a three-dot dropdown that hosts both "Transfer ownership" and "Remove from group" actions, so the owner-only Transfer action lives next to the existing Remove action instead of needing a separate UI affordance.

### Transfer ownership

- **`POST /api/v1/chats/:chatId/transfer-ownership`** body `{ newOwnerId }`. Owner only.
- Runs as a single DB transaction:
  1. Verifies the chat is a group.
  2. Verifies the requester is the current owner.
  3. Verifies the target is already a group member.
  4. Demotes the requester to `admin`.
  5. Promotes the target to `owner`.
  6. Bumps `chats.updated_at`.
  7. Returns the full refreshed participant list so clients can refresh role chips without a second round-trip.
- New contract types: `TransferGroupOwnershipRequest`, `TransferGroupOwnershipResponse`.

---

## Migrations

| Migration | Purpose |
|---|---|
| `V13__chats_avatar_description.sql` | Adds `chats.avatar_url TEXT` and `chats.description TEXT`. Both nullable; direct chats leave them `NULL`. |

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

1. As owner, create a group with 2+ other members.
2. Open the group info modal. Upload a photo via the camera badge on the avatar — see the new image populate sidebar row + message header + modal instantly.
3. Add a description, save. Refresh the page — description persists.
4. Tap a non-owner member's three-dot menu → "Transfer ownership". Old owner becomes admin; new owner gets the Owner pill.
5. As the new owner, edit description / change avatar / kick a member.
6. As a regular member, the avatar badge + description Edit + Transfer entry are all hidden (no API call possible).
7. Direct chats: no avatar badge, no description section (the modal isn't rendered for them — gated by `chat.type !== ChatType.GROUP`).

---

## Known limitations

- **No invite links** — explicit non-goal for v0.7.0. Members are added only via the existing user search + add flow.
- **No advanced roles** beyond owner / admin / member. (`ChatType` and `ParticipantRole` enums unchanged.)
- **No public groups, no channels, no calls** — explicit non-goals.
- **No realtime broadcast** of avatar / description changes. Other clients pick up the change on next chat list refresh or page reload. A future iteration can add a `chat.updated` event over WebSocket.
- **Avatar mime allowlist** matches user avatars (JPEG, PNG, WebP). GIFs are not allowed for group avatars even though they are allowed for image messages.
- **No audit log** of role changes — ownership transfer is silent at the protocol level.
