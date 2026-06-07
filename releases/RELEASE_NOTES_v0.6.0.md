# Signalix v0.6.0 — Release Notes

**Release date:** 2026-06-06
**Previous version:** v0.5.0

---

## Overview

v0.6.0 ships three threads of work in parallel:

1. **Progressive Web App foundation** — Signalix is installable from Chrome / Edge and runs as a standalone window.
2. **Liquid Glass / visionOS-inspired UI refresh** — translucent surfaces, soft motion, graphite-glass message bubbles, pill-shaped inputs. No functional changes.
3. **Web Push notifications** — recipients get OS-level notifications when a message arrives and they're offline. Multi-device. Click-through opens the right chat.

No breaking changes to the WebSocket protocol or existing REST endpoints. One additive migration (`V12__push_subscriptions.sql`). The realtime service is untouched.

---

## New Features

### Progressive Web App

Signalix now satisfies Chrome/Edge install criteria and registers a minimal service worker.

| Piece | Where |
|---|---|
| Manifest | `public/manifest.webmanifest` — name, short_name, start_url `/chats`, scope `/`, display `standalone`, two SVG icons (purpose `any` + `maskable`), `apple-touch-icon` |
| Icons | `public/icon.svg`, `public/icon-maskable.svg`, `public/apple-touch-icon.svg` (brand chat-bubble mark, white on iOS-blue) |
| Service worker | `public/sw.js` — `install` (skipWaiting), `activate` (clients.claim), `fetch` (no-op pass-through), `message` (SKIP_WAITING). No caching of API or WebSocket traffic. |
| Registration | `components/ServiceWorkerRegistration.tsx` — calls `register('/sw.js', { scope: '/' })` on `window.load`. Surfaces success + failure to the console in dev. |
| Install banner | `components/InstallPrompt.tsx` — captures `beforeinstallprompt`, dismissible, persists dismissal in `localStorage`, auto-hides on `appinstalled`. |

Next.js metadata wired up: `Metadata.manifest`, `Metadata.appleWebApp` (`capable`, `title`, `statusBarStyle`), `Metadata.icons`, `Viewport.themeColor` per `prefers-color-scheme`.

### Liquid Glass UI refresh

Visual-only pass across the chat shell, sidebar, modals, profile, install banner, and auth pages.

- Translucent surfaces with `backdrop-blur-xl/2xl` replace opaque cards.
- Layered background gradients (warm white in light, deep graphite + faint blue tint in dark — no pure black).
- New utility classes in `globals.css`: `surface-glass`, `surface-glass-strong`, `shadow-glass`, `shadow-glass-sm`.
- Pill-shaped inputs (Apple Search / visionOS style). Composer is a rounded-full glass pill with focus glow.
- Graphite glass message bubbles (no purple): outgoing dark with white text; incoming light glass. Corner radius `24px` with `8px` tail asymmetry.
- Sidebar active state is `bg-white/60 dark:bg-white/[0.08]` — blue reserved for primary CTAs only (logo, send, install, submit).
- Soft motion: `hover:scale-[1.04]` / `active:scale-[0.98]` on rail items, `hover:scale-[1.005]` on chat rows, 200 ms transitions.

### Web Push notifications

**Migration `V12__push_subscriptions.sql`** — table `push_subscriptions(id, user_id, endpoint, p256dh, auth, created_at, updated_at)` with `UNIQUE(user_id, endpoint)`. One row per (user × browser endpoint).

**`PushModule`** (`Signalix-api`):

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/push/public-key` | public | Returns the VAPID public key so the frontend can subscribe without a build-arg |
| POST | `/api/v1/push/subscribe` | JWT | Upserts the device subscription |
| DELETE | `/api/v1/push/unsubscribe` | JWT | Removes the subscription |

**Dispatch** lives in `MessagesService.sendMessage()` — after the transaction, it queries `chat_participants LEFT JOIN presence` and fires a push to every other participant whose `presence.status IS NULL OR != 'online'`. Body is rendered by `previewForPush()` (`📷 Photo`, `📎 {filename}`, or the trimmed text). Fire-and-forget; failure never blocks the response.

Subscriptions returning **404** or **410** are pruned automatically by `PushService.sendToUser()`.

**Frontend client** (`src/lib/push.ts`):

- `getPushStatus()` → `{ supported, permission, subscribed }`.
- `ensureRegistration()` idempotently registers `/sw.js` and `await navigator.serviceWorker.ready` before subscribing.
- `enablePush()` requests permission, tears down any prior subscription (browser + server) for a clean re-subscribe, fetches the VAPID key, calls `PushManager.subscribe`, POSTs to the API. Dev logs every step.
- `disablePush()` tears down browser + server. Idempotent.

**`PushSettingsCard`** in `/settings/profile` shows `Not supported` / `Blocked` / `Off` / `On` and Enable / Disable buttons.

**Service worker push + notificationclick handlers** — `push` parses the JSON payload and calls `self.registration.showNotification(title, options)` with title, body, icon (sender avatar), badge, tag (`signalix-chat:{chatId}` to coalesce). `notificationclick` focuses an existing Signalix window and asks it to `router.push('/chats/:chatId')` via `postMessage`, falling back to `clients.openWindow`.

**New env vars (API + infra):** `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`. Generated once with `npx web-push generate-vapid-keys`. Leaving them empty disables push entirely (no error, just a startup warn).

---

## Fixes

### Profile page could not scroll

Body is locked at `100dvh; overflow:hidden` for the chats shell. The profile page's `min-h-screen` wrapper got clipped — the Sign Out button at the bottom was unreachable. Wrapper changed to `h-full overflow-y-auto` so content scrolls within the locked viewport. Sticky nav still pinned. Bottom padding now `calc(env(safe-area-inset-bottom) + 2.5rem)` so the last option clears the iOS home indicator.

### Duplicate profile entry in the left rail

The desktop `IconRail` had both a Settings gear and a bottom avatar — both opening `/settings/profile`. Removed the gear. Bottom trio is now Theme → Logout → Avatar/Profile, sharing the same 40×40 rounded-2xl glass shell and motion. The avatar button uses the real user image (`Avatar` component with initials fallback). New Logout button (red accent on hover) calls `auth.store.logout()` (disconnects WS + clears persisted session) + `router.replace('/login')`.

### Service worker registered silently on failure

`ServiceWorkerRegistration` used to `.catch(() => {})` on `register()`, leaving DevTools showing "no SW registered" without any console clue. Now explicit `console.info` on success (scope, active/installing/waiting state) and `console.error` on failure. SW `install`, `activate`, and `push` events also log to the SW's own console pane.

### Invalid Tailwind `w-4.5` classes made theme/group icons render larger

Moon, sun, monitor, group, and compose SVGs used `w-4.5 h-4.5` — not a valid Tailwind utility — and silently fell back to the SVG's intrinsic width, making them visibly larger than the rest of the rail. All bottom-rail and sidebar header glyphs normalized to `w-5 h-5 stroke-[1.6]`.

---

## Migrations

| Migration | Purpose |
|---|---|
| `V12__push_subscriptions.sql` | Web Push device subscriptions table (one row per user × endpoint) |

---

## New environment variables

### `Signalix-api` (and `Signalix-infra/env/api.env.example`)

| Variable | Required | Description |
|---|---|---|
| `VAPID_PUBLIC_KEY` | Push | Generate with `npx web-push generate-vapid-keys`. Empty disables push. |
| `VAPID_PRIVATE_KEY` | Push | Pair to the public key. Keep secret. |
| `VAPID_SUBJECT` | no | `mailto:` or HTTPS URL identifying the app owner. Default `mailto:admin@signalix.local` |

The frontend fetches the VAPID public key at runtime via `GET /api/v1/push/public-key` — no `NEXT_PUBLIC_*` build-arg, no rebuild required to enable push.

---

## Validation

```bash
cd Signalix-contracts && npm run typecheck   # ✓ (no contract changes for v0.6.0)
cd Signalix-api       && npm run typecheck && npm run build   # ✓
cd Signalix-realtime  && npm run typecheck && npm run build   # ✓ (unchanged)
cd Signalix-frontend  && npm run typecheck && npm run build   # ✓
```

Manual checks performed against the local Docker Compose stack:

1. Install Signalix from Chrome / Edge address-bar prompt → standalone window opens at `/chats`.
2. DevTools → Application → Manifest shows two icons, theme, scope. Service Worker `signalix-pwa-v3` activated.
3. `/settings/profile` → Notifications → Enable. Permission granted, subscription saved, status pill flips to `On`.
4. Close Signalix tab. From another user, send a message. OS notification appears with sender name + body. Click → focuses Signalix (or opens new window) at the right chat.
5. Disable from the same card → subscription removed both browser-side and server-side.

---

## Known limitations

- **iOS Safari ≤ 16.3** does not deliver Web Push for non-installed PWAs. Users must add Signalix to the home screen first.
- **No per-chat mute, no group mute, no sound customization, no in-app banner suppression while the chat is open** — slated for a later iteration.
- The "offline" check uses `presence` only. If a recipient has a tab open but their `presence.status` is stale (e.g. WS dropped without an offline event), they may receive a push even though their browser would have shown an in-app notification too. Acceptable tradeoff while realtime stays untouched.
- The PWA service worker does NOT cache static assets or messages — installed Signalix behaves as a thin shell over the network. Offline page + asset precache is a later iteration.
- SVG icons only. Some app stores and older browsers prefer PNG fallbacks; not generated yet.
