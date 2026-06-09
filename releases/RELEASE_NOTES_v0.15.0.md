# Signalix v0.15.0 — Release Notes

**Release date:** 2026-06-09
**Previous version:** v0.14.0

---

## Overview

v0.15.0 closes a long-standing E2EE gap: until now, wiping your
browser's IndexedDB or moving to a new device meant losing access to
every previously-encrypted message — the identity keys, signed
pre-keys, one-time pre-keys, peer fingerprints, and plaintext cache
were all device-local with no recovery path. v0.15.0 lets you export
all of that state into a single AES-256-GCM ciphertext file (`.sbk`)
encrypted with a 12-word recovery phrase that **only you** see. The
server never touches the phrase or the plaintext keys.

Restore the file + phrase on a new device or after a wipe, and
previously-encrypted chats decrypt again. Sender history survives
because the plaintext cache is part of the backup; recipient history
survives because the identity / SPKs / pre-keys ride along.

What ships:

- 12-word recovery phrase drawn from a 256-word curated English
  wordlist (96 bits of base entropy + 600 000 PBKDF2 iterations).
- `createBackup(phrase)` / `restoreBackup(file, phrase)` /
  `deleteCryptoDb()` in `lib/crypto/backup.ts`.
- `Settings → Security` page with Create, Restore, and Wipe flows.
- 7 new vitest cases (`bip39.test.ts`); 36/36 total passing.

What does **not** ship:

- No server-side backup storage — download-only by design.
- No full BIP39 wordlist (256 curated words instead of 2048).
- No production trust model — the phrase is the only recovery factor.
- No new API endpoints, no DB migration, no IDB schema bump.

---

## How a backup is built

```
User taps "Create backup"
  ↓ bip39.generateRecoveryPhrase()
  ↓   12 random bytes from crypto.getRandomValues
  ↓   each byte → wordlist[i]      ("ocean lamp river zebra train horse stone mountain forest deep silent voice")
  ↓
  ↓ backup.createBackup(phrase)
  ↓   collectPayload():
  ↓     identity         (exportKey 'jwk' on identityPair + signingPair)
  ↓     signed-pre-keys  (exportKey 'jwk' on each pair + signature)
  ↓     pre-keys         (exportKey 'jwk' on each pair + consumed flag)
  ↓     plaintext-cache  (raw rows; the sender uses these to re-render their own history)
  ↓     fingerprints     (peer safety numbers + verification snapshots)
  ↓   deriveKey(phrase, salt):
  ↓     PBKDF2-SHA-256, 600k iterations, fresh 16-byte salt
  ↓     → 256-bit AES-GCM key
  ↓   encrypt(JSON, key, fresh 12-byte IV)
  ↓   assemble: SLXBKP01 | v1 | salt | iv | ciphertext
  ↓
UI shows the phrase in a 12-cell grid; triggers download of
  signalix-backup-<date>.sbk
```

The user is expected to copy the phrase by hand (the panel offers a
clipboard button as a convenience, but the canonical UX is "write it
down on paper"). The phrase is never sent to the server, never
written to localStorage, never logged.

---

## How a restore works

```
User picks a .sbk file + enters their recovery phrase
  ↓ backup.restoreBackup(file, phrase)
  ↓   parseHeader: magic SLXBKP01 + version byte    ← reject random files early
  ↓   read salt, iv, ciphertext
  ↓   deriveKey(phrase, salt)
  ↓   AES-GCM decrypt → JSON payload                ← wrong phrase ≡ auth tag mismatch
  ↓   importPayload:
  ↓     importKey 'jwk' for every CryptoKey
  ↓     buffer-to-base64-url for the SPK signature blob
  ↓   wipe the five crypto IDB stores                ← only after every import succeeds
  ↓   idbPutMany for every restored row
  ↓ returns RestoreSummary { deviceId, exportedAt, signedPreKeyCount, preKeyCount, plaintextCacheCount, fingerprintCount }
  ↓
UI shows the green success card + Reload-to-apply button.
The next page load picks up the restored identity through the
existing v0.9.1 init path.
```

A bad phrase manifests as an AES-GCM `OperationError` (auth tag
mismatch) which we rewrap as a generic "Wrong recovery phrase or
corrupted backup" so timing oracles don't tell an attacker which
specific step failed.

---

## Migration

- **No DB migration.**
- **No protocol break.**
- **IDB schema version stays at 2** (the new backup module only
  reads + writes existing stores; no new stores).
- **Backup file format is versioned** — the schema byte (currently 1)
  is reserved for future evolution; older clients reject newer files
  with `Unsupported backup schema version N`.

---

## Operational notes

- Deploy order: contracts → api → realtime → frontend. Only the
  frontend has real changes; the other repos move in lockstep for
  tag alignment.
- No env var changes. No infra changes.
- The new page lives at `/settings/security`. Existing
  `/settings/profile` has been updated to link to it.
- The backup file MIME on the wire is `application/octet-stream`
  with the `.sbk` extension. No magic in the browser — just bytes.

---

## Validation

- `npm run typecheck` — clean across all four code packages.
- `npm run build` — clean.
- `npm test` (frontend) — **36/36 vitest tests pass** (utils 7,
  fingerprints 8, file-crypto 6, local-search 8, bip39 7 ← new).

**Manual scenario** (matches the spec validation list):

1. Open Settings → Security. Tap "Create backup". A 12-word phrase
   appears and a `signalix-backup-YYYY-MM-DD.sbk` downloads.
2. Write the phrase down. Tap "I've written it down".
3. DevTools → Application → IndexedDB → delete `signalix-crypto-v1`.
4. Reload — the v0.9.1 reset detection fires (which is expected:
   the wipe wiped local state).
5. Open Settings → Security → Restore backup. Pick the `.sbk` file
   and paste the phrase. Tap Restore.
6. Green summary card shows the restored counts. Tap Reload to apply.
7. Open any direct chat that had E2EE history. Messages render
   plaintext — sender's own messages from before the wipe survive
   thanks to the plaintext cache; received messages decrypt with
   the restored SPK + pre-key pairs.

---

## Files of interest

**Frontend**
- `Signalix-frontend/src/lib/crypto/bip39.ts` *(new)* — wordlist + phrase helpers.
- `Signalix-frontend/src/lib/crypto/bip39.test.ts` *(new)* — 7 vitest cases.
- `Signalix-frontend/src/lib/crypto/backup.ts` *(new)* — `createBackup`, `restoreBackup`, `deleteCryptoDb`, file format encode/decode, JWK import/export plumbing.
- `Signalix-frontend/src/app/settings/security/page.tsx` *(new)* — UI.
- `Signalix-frontend/src/app/settings/profile/page.tsx` — added link to /settings/security.

**Other repos** — version bumps only (no functional changes).

---

## What's next — v0.16.0 preview

- Optional server-side encrypted backup blob (opt-in; server stores
  opaque bytes, recovery still requires the phrase).
- Migration to the full BIP39 2048-word list for cross-app phrase
  compatibility.
- "Backup nearing X days old" reminders surfaced in the chat
  sidebar when the user hasn't backed up since their last key
  rotation.
- WebAuthn-protected recovery as a second-factor option (the
  authenticator unlocks a stored phrase).
