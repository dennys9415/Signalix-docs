# Signalix v0.2.0 — Release Notes

**Release date:** 2026-06-04
**Previous version:** v0.1.1

---

## Overview

v0.2.0 is the first feature release after the initial MVP. It adds social login, account security flows (password reset, email verification), a transactional email pipeline, improvements to user discovery, per-user message deletion, and a profile page. Several runtime stability bugs found during v0.1 use are also resolved.

No breaking changes to the WebSocket protocol or existing REST endpoints.

---

## New Features

### OAuth Authentication

Users can now sign in or register using third-party identity providers instead of (or in addition to) a local password. All three providers follow the same account-linking model:

- **Existing provider match** — signs in the existing account.
- **Email matches a local account** — links the new provider to that account and signs in.
- **No match** — creates a new account automatically with `is_verified = true`.

| Provider | Start | Callback | Notes |
|---|---|---|---|
| Google | `GET /api/v1/auth/google` | `GET /api/v1/auth/google/callback` | `openid email profile` scopes |
| GitHub | `GET /api/v1/auth/github` | `GET /api/v1/auth/github/callback` | Falls back to `GET /user/emails` when GitHub profile email is set to private |
| Apple | `GET /api/v1/auth/apple` | `POST /api/v1/auth/apple/callback` | Uses `response_mode: form_post`; `user` JSON only on first authorization; stable identifier is `sub` claim |

No Passport.js dependency — OAuth2 code exchange is handled with plain `fetch`. The Apple client secret is a short-lived (5 min) ES256 JWT signed with the `.p8` private key via `@nestjs/jwt`.

The frontend gains "Continue with Google", "Continue with GitHub", and "Continue with Apple" buttons on the `/login` page. A new `/oauth/callback` page reads the session from the redirect URL and hydrates the auth store.

**Required env vars (new):** `FRONTEND_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_CALLBACK_URL`, `APPLE_CLIENT_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY`, `APPLE_CALLBACK_URL`

---

### Password Reset

A full forgot-password / reset-password flow backed by Flyway migration `V5__password_reset.sql`.

- `POST /api/v1/auth/forgot-password` — accepts `{ email }`; always returns the same success response (prevents email enumeration); generates a 32-byte random token, stores its SHA-256 hash with a 15-minute TTL (one active token per user).
- `POST /api/v1/auth/reset-password` — accepts `{ token, newPassword }`; validates the token (not used, not expired); updates the bcrypt password hash (cost 12); revokes **all** device sessions for the user; marks the token as used.
- Frontend pages: `/forgot-password` (email form, neutral confirmation) and `/reset-password` (password + confirm fields, error on invalid/expired token).
- Token security: raw token travels only in the email link; only the SHA-256 hash is stored in the database.

**New error code:** `INVALID_RESET_TOKEN`

---

### Email Verification

New `email_verification_tokens` table (`V6__email_verification.sql`). Local-password registrations now create an unverified account and send a verification email immediately after the transaction commits. Email failure does not roll back registration.

- `POST /api/v1/auth/verify-email` — accepts `{ token }`; marks `users.is_verified = true`; token is one-time use with a 24-hour TTL.
- `POST /api/v1/auth/resend-verification` — accepts `{ email }`; always returns success (prevents enumeration); replaces any existing unverified token and sends a fresh email.
- OAuth registration and OAuth account-linking both set `is_verified = true` directly — no email required.
- Frontend: `/verify-email` page shows a success checkmark or an error state. The profile page shows a "Verified" (green) or "Unverified" (amber) badge next to the email address. Unverified local-auth users see a resend button that confirms delivery inline.

**New error code:** `INVALID_VERIFICATION_TOKEN`

---

### Transactional Email (Resend)

A new `EmailService` (`src/email/email.service.ts`) wraps the [Resend](https://resend.com) SDK.

- `sendPasswordReset(to, resetUrl)` — subject: "Reset your Signalix password"; HTML body with an indigo "Reset Password" button and 15-minute expiry notice.
- `sendEmailVerification(to, verificationUrl)` — subject: "Verify your Signalix email address"; HTML body with a "Verify Email" button and 24-hour expiry notice.

**Local development:** leave `RESEND_API_KEY` empty. The service skips sending and prints the URL to the API console so the full flow can be tested without real email delivery.

**Required env vars (new):** `RESEND_API_KEY` (optional, empty = console fallback), `EMAIL_FROM` (default `Signalix <onboarding@resend.dev>`)

---

### Partial Username Search

`GET /api/v1/users/search?q=<query>&limit=<n>` — case-insensitive `ILIKE %query%` search against the `username` column. Excludes the authenticated caller. `limit` defaults to 10, maximum 25.

The sidebar search field now calls this endpoint on each keystroke (debounced) instead of requiring an exact match. Results appear as a scrollable list; clicking any user opens an existing chat or starts a new conversation.

---

### Delete for Me

`POST /api/v1/messages/:messageId/delete-for-me` — soft-deletes a message for the requesting user only. Stored in `message_deletions` (migration `V4__message_deletions.sql`). The message is not deleted from the `messages` table and remains visible to other participants.

`GET /api/v1/chats/:chatId/messages` now excludes rows the requesting user has deleted (using a `NOT EXISTS` subquery). The operation is idempotent.

The frontend adds a trash icon that appears on hover for each message bubble (left side for received, right side for sent). The icon is disabled while the request is in flight. On success the message is removed from local state immediately.

---

### User Profile Page

A new read-only profile page at `/settings/profile`:

- **Initials avatar** — derived from `displayName ?? username`, shown as a large circle.
- **Name block** — `displayName` as the primary heading, `@username` as secondary.
- **Account info card** — email address (with Verified / Unverified badge), user ID.
- **Email verification section** — amber warning card with resend button, shown only for unverified local-auth users.
- **Connected accounts** — lists all four providers (Local / Google / GitHub / Apple) with "Connected" or "Not connected" status.
- **Log out** button.

Accessible from the gear icon in the chat sidebar. Backed by a new authenticated endpoint `GET /api/v1/users/me` that returns `UserProfileResponse { user: UserDTO; providers: string[] }`.

---

## Improvements

### Display Name Throughout the UI

`display_name` is now the primary identity surface everywhere a user appears. `@username` is shown as a secondary line. Affected surfaces:

- Chat list rows — display name + `@username`
- User search results — display name + `@username`
- New conversation composer header — display name + Online / Offline
- Active chat header — display name + Online / Offline

### Sidebar Profile Strip

The top of the chat sidebar now shows the signed-in user's initials avatar, display name, and `@username`. Clicking the strip navigates to `/settings/profile`. This replaces the plain "Logout" button that previously lived in the sidebar.

### Initials Avatar

Used consistently on the profile page, the sidebar strip, and wherever a user avatar would otherwise be empty. Derived from the first character of `displayName ?? username`, uppercased, on an indigo background.

---

## Bug Fixes

### New Chat Realtime Visibility

**Problem:** When user B sent the very first message in a conversation, user A received `server.message.new` but the chat never appeared in their sidebar — the message was silently dropped because the `chatId` was not yet in the store.

**Fix:** The `MESSAGE_NEW` handler in `chat.store.ts` now detects an unknown `chatId` before updating state and calls `loadChats()` to refresh the sidebar immediately. No contracts or realtime protocol changes were needed.

---

### Auth Persistence After Reload

**Problem:** Reloading the page while authenticated caused a flash of the unauthenticated state and occasionally redirected to `/login` before the store could hydrate from `localStorage`.

**Fix:** The Zustand auth store now correctly hydrates its `hydrated` flag synchronously on mount. Auth guards wait for `hydrated === true` before making any routing decisions.

---

### Message Deduplication

**Problem:** Under certain reconnect conditions, `server.message.new` could fire twice for the same message, causing duplicates in the message list.

**Fix:** The `MESSAGE_NEW` handler checks whether a message with the same `messageId` already exists in the store before inserting. If found, the event is ignored.

---

### Chat Refresh Stability

**Problem:** Navigation between chat routes caused `loadChats` and the WebSocket `initWsHandler` to re-fire on every render, producing redundant API requests and occasional connection races.

**Fix:** A `useRef` guard keyed on `session.userId` in `chats/layout.tsx` ensures both functions are called exactly once per authenticated session, not on every route change.

---

## Database Migrations

| Migration | Added in |
|---|---|
| `V4__message_deletions.sql` | v0.2.0 |
| `V5__password_reset.sql` | v0.2.0 |
| `V6__email_verification.sql` | v0.2.0 |

All three run automatically via Flyway on startup in Docker Compose. For manual Flyway runs see `Signalix-api/README.md`.

---

## API Changes

### New endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/auth/forgot-password` | public | Request password reset email |
| POST | `/api/v1/auth/reset-password` | public | Consume reset token, update password |
| POST | `/api/v1/auth/verify-email` | public | Consume verification token |
| POST | `/api/v1/auth/resend-verification` | public | Re-send verification email |
| GET | `/api/v1/auth/google` | public | Initiate Google OAuth |
| GET | `/api/v1/auth/google/callback` | public | Handle Google OAuth code |
| GET | `/api/v1/auth/github` | public | Initiate GitHub OAuth |
| GET | `/api/v1/auth/github/callback` | public | Handle GitHub OAuth code |
| GET | `/api/v1/auth/apple` | public | Initiate Apple OAuth |
| POST | `/api/v1/auth/apple/callback` | public | Handle Apple form_post callback |
| GET | `/api/v1/users/me` | JWT | Authenticated user profile + providers |
| GET | `/api/v1/users/search` | JWT | Partial username search (`?q=&limit=`) |
| POST | `/api/v1/messages/:id/delete-for-me` | JWT | Soft-delete a message for the caller |

### Modified endpoints

| Method | Path | Change |
|---|---|---|
| GET | `/api/v1/chats/:chatId/messages` | Excludes messages the requesting user has deleted for themselves |

### No breaking changes

All v0.1 endpoints remain at the same paths and return the same shapes. Existing clients do not need to be updated to continue working.

---

## Contracts Changes (`@signalix/contracts`)

New types added to `src/api/auth.contract.ts`:
- `ForgotPasswordRequest / ForgotPasswordResponse`
- `ResetPasswordRequest / ResetPasswordResponse`
- `VerifyEmailRequest / VerifyEmailResponse`
- `ResendVerificationRequest / ResendVerificationResponse`

New types added to `src/api/user.contract.ts`:
- `UserProfileResponse { user: UserDTO; providers: string[] }`
- `UserSearchRequest / UserSearchResponse`

New types added to `src/api/message.contract.ts`:
- `DeleteMessageForMeRequest / DeleteMessageForMeResponse`

New error codes in `src/errors/error-code.enum.ts`:
- `INVALID_RESET_TOKEN`
- `INVALID_VERIFICATION_TOKEN`

---

## Upgrade Instructions

### From v0.1.x

1. **Pull all five repos** and rebuild contracts:
   ```bash
   cd Signalix-contracts && npm install && npm run build
   ```

2. **Update env files** — new variables required in `Signalix-api/.env` (or `Signalix-infra/env/api.env`):
   ```
   FRONTEND_URL=http://localhost:3000
   RESEND_API_KEY=          # leave empty for console fallback
   EMAIL_FROM=Signalix <onboarding@resend.dev>
   ```
   OAuth vars are optional — leave them empty to disable the respective provider buttons.

3. **Run Flyway migrations** — migrations V4–V6 are new. In Docker Compose they run automatically. For manual Flyway:
   ```bash
   flyway -url=jdbc:postgresql://localhost:5432/signalix \
          -user=signalix -password=signalix \
          -locations=filesystem:./Signalix-api/migrations \
          migrate
   ```

4. **Rebuild and restart** all services.

---

## Known Limitations

| Limitation | Notes |
|---|---|
| Unverified accounts can still log in | `is_verified = false` does not block local login in v0.2. Policy enforcement is planned. |
| Apple OAuth requires HTTPS | Apple rejects plain `http://` redirect URIs. Use ngrok or a similar tunnel for local testing. |
| Single-instance realtime | The WebSocket server uses in-memory routing. Horizontal scaling requires Redis Pub/Sub (planned). |
| No offline message queue | Messages sent while disconnected are lost. The client reconnects after 3 s but does not retry the send. |
| `ciphertext` is plain text | Signal Protocol / E2EE is not implemented. The field name anticipates a future encryption layer. |
| No group chats | `ChatType.DIRECT` only. |
| No media attachments | Text messages only. |
| `NEXT_PUBLIC_*` URLs are build-time constants | Changing the API or WS URL requires rebuilding the frontend image. |

---

## Next: v0.3.0 (Planned)

| Feature | Description |
|---|---|
| Message editing | `PUT /api/v1/messages/:id` — edit own messages within a time window; broadcast `server.message.edited` |
| Delete for everyone | `POST /api/v1/messages/:id/delete-for-everyone` — removes the message for all participants |
| Typing indicators | Route `client.typing.start` / `client.typing.stop` events already defined in contracts |
| Login block for unverified email | Optional policy: reject local login when `is_verified = false` |
| Redis Pub/Sub | Enable horizontal realtime scaling; decouple presence and routing from in-memory state |
| Nginx + TLS | Reverse proxy in `Signalix-infra` with automated certificate management |
| Health check endpoints | `GET /api/v1/health` and `/health` on the realtime server; `service_healthy` dependencies in Docker Compose |

---

## Repository Links

| Repository | Purpose |
|---|---|
| `Signalix-contracts` | Shared types, enums, WebSocket event contracts |
| `Signalix-api` | NestJS REST API, Flyway migrations, email service |
| `Signalix-realtime` | Node.js WebSocket server |
| `Signalix-frontend` | Next.js 15 chat client |
| `Signalix-infra` | Docker Compose local development stack |
