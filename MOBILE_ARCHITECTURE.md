# Mobile Architecture — Signalix-mobile

**Applies to:** Signalix-mobile v0.16.0 (foundation MVP).

## Stack

- **Expo SDK 51** (managed workflow) — no `expo prebuild` required for v0.16.0; native modules are limited to those bundled with Expo Go (`expo-secure-store`, `@react-native-async-storage/async-storage`).
- **React Native 0.74**
- **TypeScript** strict
- **React Navigation** — `@react-navigation/native-stack` + `@react-navigation/bottom-tabs`
- **Zustand v5** for state (mirrors the web architecture)
- **`@signalix/contracts`** via `file:` protocol so the mobile app reuses the same DTOs, enums, and WS event names the web + API + realtime have always used

## Repository layout

```
Signalix-mobile/
  App.tsx                       — entry: GestureHandlerRootView + SafeAreaProvider + StatusBar + RootNavigator
  app.json                      — Expo config (apiUrl/wsUrl in `extra`)
  tsconfig.json                 — strict; path alias `~/*` → `src/*`
  package.json                  — deps
  babel.config.js               — `babel-preset-expo`
  src/
    screens/                    — top-level routes
      LoginScreen.tsx
      RegisterScreen.tsx
      ChatsScreen.tsx           — chat list (FlatList + RefreshControl)
      ConversationScreen.tsx    — per-chat history + composer + auto-scroll
      SettingsScreen.tsx        — username/email/version/logout
    navigation/
      RootNavigator.tsx         — AuthStack | AppStack (Tabs + Conversation)
    components/                 — reserved for v0.17.0 shared bits
    store/
      auth.store.ts             — session + login/register/logout, secure-store hydration
      chat.store.ts             — chats list, messages map, unread, presence, WS handlers
    services/
      api-client.ts             — REST wrapper, 401-refresh-retry, typed via @signalix/contracts
      ws-client.ts              — WebSocket client, 30s heartbeat, 3s reconnect
      secure-store.ts           — expo-secure-store (session) + AsyncStorage (non-sensitive cache)
    hooks/                      — reserved
    lib/
      config.ts                 — apiUrl + wsUrl from Constants.expoConfig.extra
      theme.ts                  — colors / radius / spacing tokens
      format.ts                 — formatChatTime
    types/                      — local-only types (DTOs come from @signalix/contracts)
```

## Auth flow

```
App boot
  ↓ RootNavigator → useAuthStore.hydrate()
  ↓ secure-store.loadSession()
  ↓
  ┌── session = null ──→ AuthStack (Login / Register)
  │
  └── session present ──→ wsClient.connect(accessToken)
                            chat.store.initWsHandler()
                            chat.store.loadChats()
                            AppStack (Tabs)

Login / Register
  ↓ api.login() / api.register()
  ↓ secure-store.saveSession(session)
  ↓ wsClient.connect(session.accessToken)
  ↓ navigation switches to AppStack (driven by `session` change in useAuthStore)

Logout
  ↓ wsClient.disconnect()
  ↓ api.logout() (best-effort)
  ↓ secure-store.clearSession()
  ↓ navigation reverts to AuthStack
```

Token refresh is transparent: any 401 from `api.*` triggers `POST /auth/refresh` and a single retry. Concurrent calls share the same in-flight refresh promise. If the refresh fails, the session is cleared and the user is bounced to `Login`.

## WebSocket flow

The native `WebSocket` global is provided by React Native — same code as the web client minus the browser-only bits.

```
auth.store.login()              api.login()
auth.store.hydrate()    ──┐
                          ├──→ wsClient.connect(accessToken)
                          │     openSocket()
                          │       send(client.authenticate)
                          │       setInterval(heartbeat, 30s)
                          │       onmessage → handler(event, payload)
                          │       onclose   → retry in 3s
                          │
chat.store.initWsHandler() ──→ wsClient.setHandler((event, payload) => …)
                                  server.message.sent       → promote temp
                                  server.message.new        → append + unread + auto-deliver/read
                                  server.message.delivered  → state advance
                                  server.message.read       → state advance
                                  server.chat.created       → prepend to list
                                  server.user.online/offline → presence map
```

The mobile app does not implement the outbox or sync-since-cursor paths from the web's v0.14.0. If a message fails to send because the WS is closed, the temp is flagged `queued` and the user sees "queued" inline; on the next reconnect the user can re-tap Send. A proper drain-on-authenticated path lands with v0.17.0.

## Crypto flow

**v0.16.0 is plaintext-only on mobile.** The mobile app does NOT register an identity / signing keypair, does not maintain a pre-key pool, does not publish to `/crypto/devices/keys`, and does not call into the E2EE fan-out path. Outgoing messages are sent without `recipients[]`; the server stores plaintext in `messages.ciphertext`.

**Effect on existing E2EE users**:
- Web ↔ web direct text: unchanged, still E2EE.
- Web ↔ mobile or mobile ↔ web: **falls back to plaintext** because the sender's `getKeyBundle` for the mobile-only user (or for the mobile device of a multi-device user) returns no bundle, and `dispatchSend`'s `encryptForUser` failure path drops to plaintext (existing v0.10.0 behaviour).
- Mobile ↔ mobile: plaintext.
- Web ↔ web in a group that has a mobile member: the group fan-out catches one missing recipient and drops the whole send to plaintext (same v0.10.0 behaviour).

This trade-off is intentional for v0.16.0 — the alternative is a full port of the X25519 / Ed25519 / AES-GCM / HKDF stack to React Native, which depends on either `react-native-quick-crypto` (native bridge, complicates the Expo managed workflow) or pure-JS libs like `@stablelib/*` (slower, larger bundle). v0.17.0 takes that on as its single theme.

The mobile UI shows the v0.16.0 fallback explicitly in **Settings → About → E2EE**: *"Web-only in v0.16.0 (port lands v0.17.0)"*.

## Storage

- **Sensitive (encrypted by the OS keychain):** `expo-secure-store`. Used only for the auth session blob (access + refresh tokens). The 2 KB SecureStore item limit is comfortable for the session payload (~600 bytes).
- **Non-sensitive cache:** `AsyncStorage`. Used by future cache code (draft text per chat, last-seen scroll position). v0.16.0 doesn't write to AsyncStorage yet; the wrapper in `services/secure-store.ts` is in place for v0.17.0+.

**Forbidden APIs:** `IndexedDB`, `localStorage`, `crypto.subtle`, `document`, `window` — these are browser-only. Importing them breaks the app at module-evaluation time on RN.

## Realtime guarantees

Same as the web v0.14.0 stack:
- 30 s heartbeat keeps idle TCP alive through carrier-grade NAT proxies.
- 3 s reconnect on any unexpected close.
- Unread count is server-canonical on chat list refresh (RefreshControl pull-to-refresh forces `api.getChats()`).
- Read receipts: the mobile MarkChatRead flow on Conversation mount fires `client.message.read` for every unread message from another participant — same per-message broadcast pattern v0.14.0 uses on web.
