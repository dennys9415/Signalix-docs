# Signalix Changelog

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
