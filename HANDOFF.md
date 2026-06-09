# Signalix — Project Handoff

**As of:** v0.16.0 (released 2026-06-09).
**Owner:** Luis Muñoz (`luismunoz@lcubestudios.io`).
**Repos:** 7 (api, contracts, realtime, frontend, infra, mobile, docs).

This document is the single entry point for anyone (human or AI) picking the project up after v0.16.0. Read this first, then drill into the topic-specific docs.

---

## 1. Workspace layout

All 7 repos sit side by side under `proyect/`:

```
proyect/
  docs/                  — single source of truth for cross-repo docs
    CHANGELOG.md
    ARCHITECTURE.md
    DATABASE_SCHEMA.md
    DOCKER_LOCAL_SETUP.md
    MOBILE_ARCHITECTURE.md
    MOBILE_SETUP.md
    MOBILE_ROADMAP.md
    MOBILE_TESTING.md          (manual QA checklist)
    HANDOFF.md                 (this file)
    releases/RELEASE_NOTES_v*.md
  Signalix-api/          — NestJS REST API           — branch dev
  Signalix-contracts/    — shared TS DTOs / events   — branch dev
  Signalix-realtime/     — WebSocket server          — branch dev
  Signalix-frontend/     — Next.js web client        — branch dev
  Signalix-infra/        — Docker Compose stack      — branch dev
  Signalix-mobile/       — React Native + Expo app   — branches main + dev
  Signalix-docs/         — public mirror of docs/    — branch main
```

The `docs/` folder under `proyect/` is the **editable source**. `Signalix-docs/` is a mirror — populated by `rsync -av --delete proyect/docs/ proyect/Signalix-docs/` at every release. Do **not** edit `Signalix-docs/` directly.

---

## 2. State at v0.16.0

| Repo | v0.16.0 status | Notes |
|---|---|---|
| Signalix-api | No code change | Lockstep version bump only. |
| Signalix-contracts | `.npmignore` fix | Anchored `/src`, `/protocol`, etc. so `dist/protocol/` ships in the npm pack. No TS surface change. |
| Signalix-realtime | No code change | Lockstep version bump only. |
| Signalix-frontend | No code change | Lockstep version bump only. Web↔web E2EE intact. |
| Signalix-infra | No code change | Lockstep version bump only. |
| Signalix-mobile | **New repo, initial scaffold** | Expo SDK 54 + RN 0.81 + React 19.1. Plaintext-only. |
| Signalix-docs | Mirrored | Picks up MOBILE_* + RELEASE_NOTES_v0.16.0.md. |

**Tag `v0.16.0`** points at HEAD on every repo's release branch. All 7 are pushed to GitHub under `dennys9415/Signalix-*`.

### Critical caveats

- **Mobile is plaintext-only.** Web↔web E2EE is unchanged. Web↔mobile and mobile↔mobile drop to plaintext. The degradation matrix is in `MOBILE_ARCHITECTURE.md` and `RELEASE_NOTES_v0.16.0.md`.
- **Mobile is Android-only in practice.** iOS code paths exist (Expo manages both) but iOS testing / signing / TestFlight is out of scope for v0.16.0.
- **The mobile app has not been QA'd end-to-end yet** — only the bundle compile + typecheck + expo-doctor were validated. Manual emulator testing is `MOBILE_TESTING.md`. Treat anything not on that checklist as untested.

---

## 3. Recently completed work (last 3 releases)

Read these release notes for context — they cover everything from the last quarter:

- `docs/releases/RELEASE_NOTES_v0.14.0.md` — Per-recipient read receipts, reconnect sync (`GET /chats/:id/messages?since=`).
- `docs/releases/RELEASE_NOTES_v0.15.0.md` — Key backup / device recovery (`.sbk` files, 12-word phrase, PBKDF2 600k + AES-256-GCM).
- `docs/releases/RELEASE_NOTES_v0.16.0.md` — Mobile foundation MVP.

For releases before v0.14.0, see the chronological `docs/CHANGELOG.md`.

---

## 4. What's next — v0.17.0 (E2EE port to mobile)

**Theme:** match the web's v0.10.x+ E2EE so direct + group text round-trips end-to-end between mobile and web. Drop the v0.16.0 plaintext degradation.

The full breakdown with effort estimates is in `docs/MOBILE_ROADMAP.md` → "v0.17.0" section. High level:

1. Extract a `CryptoBackend` interface from `Signalix-frontend/src/lib/crypto/`. Web stays on Web Crypto. Mobile uses `@stablelib/*`.
2. Extract a `KeyStore` interface. Web stays on IndexedDB. Mobile uses `expo-secure-store` (private keys) + `AsyncStorage` (pool / cache / fingerprints).
3. Port device registration, pre-key pool top-up, reset detection.
4. Port `bip39` + `backup.ts` — same `.sbk` format, cross-platform backups.
5. Round-trip test the full matrix: web↔web, web↔mobile, mobile↔web, mobile↔mobile, group with mixed peers, edit envelope re-route, reconnect sync.

**Estimated effort:** 12-15 days for one engineer, sequentially.

---

## 5. Release workflow (mandatory checklist)

Every lockstep release follows the same 9 steps. **Do not skip the user-confirmation gate** — see "Release conventions" memory rule.

1. Implementation work in whichever repo(s) drive the theme.
2. `npm run typecheck` clean in every touched repo. Web: also `npm run build`. Mobile: also `npx expo-doctor` and `npx expo export --platform android` to confirm the bundle compiles.
3. **Ask the user via AskUserQuestion** whether to proceed with the release workflow OR wait for them to test first OR add more features. **Do not skip this gate**.
4. On confirmation: bump `package.json` version in all 5 npm-managed repos (api, contracts, realtime, frontend, mobile). Infra has no `version` field.
5. Update each repo's `README.md`:
   - Bump the `**Version: v<X.Y.Z>**` line.
   - Add a new `## v<X.Y.Z> changelog — <Theme> (<repo> no-op)` section above the previous one. If the repo had real changes, the section names what changed; otherwise it's a "Not changed" no-op note.
6. Update `proyect/docs/CHANGELOG.md` (add `## v<X.Y.Z> — <date>` block at the top with theme + added + fixed + not-changed sections).
7. Create `proyect/docs/releases/RELEASE_NOTES_v<X.Y.Z>.md` (full prose release notes — see existing files as templates).
8. `rsync -av --delete --exclude .git --exclude .DS_Store proyect/docs/ proyect/Signalix-docs/` to mirror docs.
9. Per repo: stage changed files, commit with message `Signalix v<X.Y.Z>: <Theme>` (HEREDOC for multi-line + `Co-Authored-By` if Claude wrote it), tag `v<X.Y.Z>`, push branch + tag.

Mobile-specific notes for step 9:
- Mobile lives on `main` AND `dev` (created during v0.16.0). Push BOTH branches + the tag.
- Mobile commits should add `package-lock.json` even if it's noisy — Metro 0.83's resolver depends on the exact npm layout.

---

## 6. Local development quick reference

### Web stack (full Docker)
```bash
cd Signalix-infra && ./scripts/up.sh
# api: http://localhost:4000, realtime: ws://localhost:5000, frontend: http://localhost:3000
```

### Mobile (against running web stack)
```bash
# Build contracts dist first (mobile depends on it via file: protocol)
cd Signalix-contracts && npm install && npm run build

cd ../Signalix-mobile
npm install
npm run start          # press `a` for Android emulator (Expo Go SDK 54)
```

Mobile uses `http://10.0.2.2:4000` / `ws://10.0.2.2:5000` to reach the host's localhost from inside the Android emulator. iOS simulator uses `http://localhost:*` directly. Override via `app.json` → `extra.apiUrl` / `extra.wsUrl` if pointing at a remote backend.

### Per-repo typecheck
```bash
(cd Signalix-api && npm run typecheck)
(cd Signalix-contracts && npm run typecheck)
(cd Signalix-realtime && npm run typecheck)
(cd Signalix-frontend && npm run typecheck)
(cd Signalix-mobile && npm run typecheck)
```

---

## 7. Conventions and rules

Both reflected in the per-repo `CLAUDE.md` files (read them before generating code):

- **`@signalix/contracts` is the single source of truth.** Never invent local DTOs / enums / event names. If you need a new one, add it to contracts first.
- **API responses use `ApiResponse<T>`.** Never raw payloads.
- **Messages use `ciphertext`, not `content`.** Even when the content is plaintext (v0.1 web, v0.16.0 mobile).
- **`READ` always wins over `DELIVERED`/`SENT`.** Status must never revert.
- **Web frontend has no browser-API substitutes baked in.** Use Web Crypto + IndexedDB + Service Workers directly.
- **Mobile has no browser APIs.** Use `expo-secure-store`, `AsyncStorage`, native WebSocket. Don't import `indexedDB`, `localStorage`, `crypto.subtle`, `document`, or `window`.
- **Direct SQL via `pg`. No Prisma, no ORM.**

---

## 8. Files most worth reading before changing anything

For a new contributor or new AI session, in priority order:

1. `docs/HANDOFF.md` (this file).
2. `docs/CHANGELOG.md` — what shipped when, with theme summaries.
3. `docs/ARCHITECTURE.md` — system-level diagram + responsibility split.
4. `docs/MOBILE_ARCHITECTURE.md` — only relevant for mobile work.
5. `Signalix-mobile/CLAUDE.md` — mobile assistant rules (absolute, do not violate).
6. `Signalix-frontend/src/lib/crypto/` — the entire folder, before touching mobile E2EE.
7. `docs/releases/RELEASE_NOTES_v0.10.0.md` — the original E2EE design doc; still authoritative.

---

## 9. Known limitations and tech debt

- **Mobile package-lock symlinking quirk.** `npm install` creates a symlink for the `file:` dep by default, but Metro 0.83 doesn't follow it reliably. Workaround in `Signalix-mobile/metro.config.js` (extraNodeModules + watchFolders). If a future Metro release fixes symlink following, the config can be simplified.
- **Per-repo `package-lock.json` versions drift from `package.json` version.** Pre-existing pattern — the locks have stale `version` fields from earlier releases. Not load-bearing, but cosmetic.
- **`docs/releases/PROJECT_CONTEXT.md` is stale.** Refers to v0.5.0 stabilization work. Kept as historical artifact, not authoritative.
- **No automated tests run in CI.** All validation is manual (`npm run typecheck` + manual QA). A test-runner setup is unscheduled tech debt.
- **`docs/CLAUDE.md` mentions only v0.1 repos.** That file is severely out of date and references rules ("do not implement group chats in v0.1") that no longer apply. The per-repo `CLAUDE.md` files inside each `Signalix-*` directory are the authoritative current rules. Plan to deprecate or rewrite `docs/CLAUDE.md` in a future cleanup.

---

## 10. How to continue this project with a new AI session

When opening a fresh Claude / GPT / Cursor session, paste the prompt in `docs/HANDOFF_PROMPT.md` (if it exists) or use the one at the end of this file. The prompt:

1. Tells the AI where the workspace is.
2. Tells the AI which docs to read first (in order).
3. Tells the AI the absolute rules.
4. Tells the AI to ask before running the release workflow.

The shortest possible bootstrap prompt:

> I'm continuing the Signalix project. The workspace is at `/Users/kira/SingularityBox/Signalix/proyect/`. Read `docs/HANDOFF.md` first, then ask me what you should work on. Never run the release workflow (bumps + docs + rsync + commit/tag/push) without explicit confirmation — I need a checkpoint to test before each release.

The full version, with role definition and pre-loaded context, is in section "Continuation prompt" below.

---

## 11. Continuation prompt (copy-paste verbatim into a new AI session)

```
You are continuing work on Signalix, a multi-repo encrypted messaging system. The current release is v0.16.0 (Mobile Foundation MVP, 2026-06-09).

WORKSPACE
The workspace is at /Users/kira/SingularityBox/Signalix/proyect/ with 7 repos side by side: Signalix-api, Signalix-contracts, Signalix-realtime, Signalix-frontend, Signalix-infra, Signalix-mobile, Signalix-docs.

READ FIRST (in order, before doing anything)
1. docs/HANDOFF.md — the project entry point. Covers state, conventions, next theme, release workflow, known limitations.
2. docs/CHANGELOG.md (top 200 lines) — what shipped recently and why.
3. docs/MOBILE_ROADMAP.md — what's next on mobile (v0.17.0 E2EE port is the active theme).
4. The CLAUDE.md inside whichever repo you're touching — the absolute rules for that surface.

ABSOLUTE RULES
- @signalix/contracts is the single source of truth for DTOs, enums, and WS event names. Never invent local duplicates in api / realtime / frontend / mobile.
- All API responses use ApiResponse<T>.
- Messages use the field `ciphertext`, not `content` — even when the content is plaintext.
- READ status always wins; never revert to DELIVERED or SENT.
- Mobile has no browser APIs: no indexedDB, no localStorage, no crypto.subtle, no document, no window. Use expo-secure-store, AsyncStorage, native WebSocket.
- Direct SQL via pg in the API. No Prisma, no ORM.

RELEASE WORKFLOW IS GATED
Never run the lockstep release workflow (version bumps + README updates + CHANGELOG + rsync docs + commit + tag + push to 7 repos) without explicit user confirmation. The user needs a checkpoint to test before each release. Ask via the AskUserQuestion tool when implementation work is complete.

CURRENT STATE
- v0.16.0 just shipped. Signalix-mobile is brand new, plaintext-only, scaffolding complete but not yet QA'd end-to-end. The 5 existing repos are byte-identical to v0.15.0 apart from version bumps and a contracts .npmignore fix.
- Next theme is v0.17.0: port E2EE to mobile via @stablelib/* + expo-secure-store + AsyncStorage. Estimated 12-15 days. See docs/MOBILE_ROADMAP.md for the full work breakdown.

START BY
Reading docs/HANDOFF.md end to end, then asking me what to work on. Do not start implementation work without confirming scope.
```

Save the prompt above into `docs/HANDOFF_PROMPT.md` if you want a clean copy-paste source.
