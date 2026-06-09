# Signalix v0.16.0 — Release Notes

**Release date:** 2026-06-09
**Previous version:** v0.15.0

---

## Overview

v0.16.0 introduces the **mobile foundation MVP** — a brand-new
repository (`Signalix-mobile`) that ships a React Native + Expo SDK
54 application for Android. The mobile client mirrors the web's auth,
chat list, direct + group conversation, and settings flows on top of
the existing REST + WebSocket surface — **zero server-side changes**.

The five existing repositories (`Signalix-api`,
`Signalix-contracts`, `Signalix-realtime`, `Signalix-frontend`,
`Signalix-infra`) are byte-identical to v0.15.0 apart from the
lockstep version bump and a small `.npmignore` fix in `contracts`
required to make the package usable from the new mobile app.

What ships:

- A standalone `Signalix-mobile` Expo app with auth, chat list,
  conversation screen, settings, bottom-tab navigation, secure
  session storage, realtime + REST clients, and three new
  documentation files in `docs/`.
- Expo SDK 54 (React Native 0.81, React 19.1) + Metro 0.83 with a
  carefully scoped `metro.config.js` that pulls `@signalix/contracts`
  from the sibling repo without copying.
- `.npmignore` fix in `Signalix-contracts` so `dist/protocol/` is no
  longer accidentally excluded from the npm pack.

What does **not** ship:

- E2EE on mobile — the cryptographic stack ports in v0.17.0 via
  `@stablelib/*`. v0.16.0 is **plaintext only** on mobile.
  Web↔web E2EE between web clients is unaffected.
- Push notifications, media uploads, voice notes, voice/video calls,
  group management UI, iOS production builds. All are tracked in
  `docs/MOBILE_ROADMAP.md`.

---

## Why a separate repository

The mobile app sits alongside the existing five repos under the same
`proyect/` workspace and the same lockstep release tag (`v0.16.0`),
but it lives in its own git repository
(`github.com/dennys9415/Signalix-mobile`) for the same reasons each
other Signalix repo is independent:

- Independent CI / build cadence (Expo + EAS pipeline vs Docker +
  Flyway pipelines).
- Distinct release surface — the mobile artifact is an APK / AAB /
  TestFlight build, not a container image.
- Native-only tooling (Metro, Expo Go, Android Studio) that doesn't
  belong in the rest of the workspace.

The mobile repo follows the same conventions as the others: branch
`dev`, tag `v<X.Y.Z>`, `CLAUDE.md` for assistant rules, `@signalix/contracts`
via the `file:` protocol.

---

## Architecture snapshot

```
Signalix-mobile  ──HTTP──►  Signalix-api  ──SQL──►  PostgreSQL
                 ──WS────►  Signalix-realtime  ──HTTP──►  Signalix-api
```

```
Signalix-mobile/src/
  App.tsx
  index.js                              — registerRootComponent(App)
  metro.config.js                       — watchFolders + extraNodeModules → ../Signalix-contracts
  app.json                              — extra.apiUrl + extra.wsUrl per emulator
  babel.config.js                       — babel-preset-expo
  screens/
    LoginScreen.tsx
    RegisterScreen.tsx
    ChatsScreen.tsx
    ConversationScreen.tsx
    SettingsScreen.tsx
  navigation/
    RootNavigator.tsx                   — AuthStack | AppStack (Tabs: Chats/Settings + Conversation)
  store/
    auth.store.ts                       — zustand: session/hydrated/loading/error
    chat.store.ts                       — zustand: chats/messages/unread/presence + WS handlers
  services/
    api-client.ts                       — REST wrapper + 401-refresh-retry
    ws-client.ts                        — WebSocket: 30s heartbeat + 3s reconnect
    secure-store.ts                     — expo-secure-store (session) + AsyncStorage (cache)
  lib/
    config.ts                           — Constants.expoConfig.extra → apiUrl + wsUrl
    theme.ts                            — colors / radius / spacing tokens
    format.ts                           — formatChatTime helper
```

---

## Plaintext degradation (the v0.16.0 trade-off)

The web E2EE stack uses Web Crypto (`crypto.subtle`) + IndexedDB. Both
APIs are absent on React Native. Three options were on the table:

1. **Port the crypto stack now.** Requires replacing every Web Crypto
   call with `@stablelib/*` (X25519, Ed25519, AES-GCM, HKDF) and every
   IndexedDB read/write with `expo-secure-store` + `AsyncStorage`.
   Two-to-three weeks of work plus a careful round-trip test matrix
   against the web. Out of scope for v0.16.0.
2. **Ship without mobile at all.** Pushes the foundation to v0.17.0
   and bundles it with the E2EE port — the wrong order, since the
   mobile UI surface needs to exist before the crypto layer can be
   plugged into it.
3. **Ship plaintext-only and degrade.** Mobile sends and receives
   `ciphertext: <plain UTF-8>` against the existing endpoints. The
   server doesn't care — it never reads `ciphertext`. Web clients
   that receive plaintext from a mobile peer see plaintext (no
   decrypt attempt). Mobile clients that receive an encrypted
   envelope from a web peer see `[encrypted]` placeholder text.

Option 3 was chosen because:

- It gets a working mobile client into testers' hands now.
- It keeps web↔web E2EE intact — encrypted conversations between two
  web clients still ride the v0.14.x envelope path.
- The degradation is **per-conversation**, not global. A web user
  who never opens a chat with a mobile peer never loses E2EE.
- It documents an explicit, time-boxed trade-off — v0.17.0 closes
  the gap.

The degradation matrix:

| Sender → Recipient | v0.16.0 behaviour            |
|---|---|
| web ↔ web          | E2EE (same as v0.15.0)       |
| mobile ↔ mobile    | Plaintext                    |
| web → mobile       | Plaintext (web sender skips E2EE if peer is mobile-only) |
| mobile → web       | Plaintext (web recipient renders as-is) |

This table is also in `docs/MOBILE_ARCHITECTURE.md` so the rule
travels with the architecture doc.

---

## The contracts `.npmignore` fix

While integrating `@signalix/contracts` into the mobile app, the
following error surfaced at Metro bundle time:

```
Unable to resolve module ./protocol from
  node_modules/@signalix/contracts/dist/index.js
```

`dist/index.js` does `__exportStar(require("./protocol"), exports)`,
so the missing `dist/protocol/` directory meant nothing
re-exported. The directory exists on disk and `npm run build`
generates it correctly — but it was being **excluded from the npm
pack**.

The old `.npmignore`:

```
src
rules
protocol
schema
tools
versions
*.log
```

These patterns match the named folder at **any depth** (gitignore /
npmignore globbing semantics). So `protocol` matched both the
top-level `protocol/` (which should be excluded) and
`dist/protocol/` (which must ship). The fix anchors each pattern to
the repository root:

```
/src
/rules
/protocol
/schema
/tools
/versions
*.log
```

Effect: `npm pack` now includes the full `dist/` tree, and the
mobile app (plus any future consumer that resolves the package as a
real `node_modules` install rather than via `extraNodeModules`)
gets the complete TypeScript surface.

---

## Metro configuration for the local file: dependency

The mobile app pulls contracts via `"@signalix/contracts":
"file:../Signalix-contracts"`. Two pieces of Metro setup were
required:

1. **`watchFolders`** — Metro 0.83 only transforms files inside the
   project root or its `watchFolders`. Adding the sibling contracts
   directory means Metro re-bundles when contracts dist changes.
2. **`resolver.nodeModulesPaths`** + **`resolver.extraNodeModules`**
   — Metro 0.83's strict resolver doesn't follow symlinks
   reliably and doesn't auto-detect the file: target. Mapping
   `@signalix/contracts` → `../Signalix-contracts` via
   `extraNodeModules` resolves the import directly to the dist build,
   independent of how npm chose to copy / link the dependency.

The full config (in `Signalix-mobile/metro.config.js`):

```js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const projectRoot = __dirname;
const contractsRoot = path.resolve(projectRoot, '..', 'Signalix-contracts');

const config = getDefaultConfig(projectRoot);

config.watchFolders = [
  ...(config.watchFolders || []),
  contractsRoot,
];

config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(contractsRoot, 'node_modules'),
];

config.resolver.extraNodeModules = {
  ...(config.resolver.extraNodeModules || {}),
  '@signalix/contracts': contractsRoot,
};

config.resolver.disableHierarchicalLookup = false;

module.exports = config;
```

This pattern is captured in `docs/MOBILE_SETUP.md` so the next person
who scaffolds a workspace doesn't re-discover it.

---

## What changed per repository

| Repo               | Change in v0.16.0                                                       |
|---|---|
| Signalix-api       | Version bump + README changelog entry. No code, schema, or env change.  |
| Signalix-contracts | Version bump + README changelog entry. `.npmignore` fix.                |
| Signalix-realtime  | Version bump + README changelog entry. No code or event change.         |
| Signalix-frontend  | Version bump + README changelog entry. No code change.                  |
| Signalix-infra     | Version bump + README changelog entry. No service or env change.        |
| Signalix-mobile    | **New repository.** Initial scaffold + screens + stores + services.    |
| Signalix-docs      | Mirrored from `proyect/docs/` — picks up `MOBILE_*.md` + this file.    |

---

## Upgrade notes

For the existing five repos, v0.16.0 is a no-op upgrade. There is no
new migration, no new env var, no new service. `git pull && git
checkout v0.16.0` is sufficient.

For the new mobile repo:

```bash
cd Signalix-contracts && npm install && npm run build
cd ../Signalix-mobile
npm install
npm run start
# press `a` to launch Android emulator (requires Expo Go SDK 54)
```

See `docs/MOBILE_SETUP.md` for the full setup walkthrough including
the `10.0.2.2` emulator-to-host networking trick.

---

## Looking ahead — v0.17.0

E2EE port to mobile is the next theme. Scope:

- Replace web's `crypto.subtle` calls with a thin adapter; mobile
  backend uses `@stablelib/x25519`, `@stablelib/ed25519`,
  `@stablelib/aes-gcm`, `@stablelib/hkdf` + `@stablelib/sha256`.
- Replace IndexedDB with `expo-secure-store` (private keys) +
  `AsyncStorage` (pre-key pool, fingerprints, plaintext cache).
- Port v0.15.0 key backup / restore (`.sbk` files) to mobile via
  the native share sheet.
- Identical wire format. Identical envelope JSON. Identical HKDF
  info strings. Round-trip tested against the web client.

See `docs/MOBILE_ROADMAP.md` for the full roadmap through v0.22.0.
