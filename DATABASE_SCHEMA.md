# Signalix Database Schema

## Database

PostgreSQL.

Use direct SQL queries. Do not use Prisma.

Migrations:
- Use Flyway.
- Local DEV may run Flyway automatically during Docker startup.
- STAGING runs Flyway in the pipeline before deploy.
- PROD runs Flyway in the pipeline with approval before deploy.

## Required PostgreSQL Extension

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

## v0.1 Tables

### users

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  password_hash TEXT,
  display_name VARCHAR(100) NOT NULL,
  avatar_url TEXT,
  bio TEXT,
  is_verified BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW()
);
```

### auth_providers

```sql
CREATE TABLE auth_providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider VARCHAR(50) NOT NULL,
  provider_user_id VARCHAR(255) NOT NULL,
  provider_email VARCHAR(255),
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  UNIQUE(provider, provider_user_id),
  CHECK (provider IN ('local', 'google', 'github', 'apple'))
);
```

v0.1 uses `local`. OAuth providers are v0.2.

### devices

```sql
CREATE TABLE devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  device_name TEXT,
  device_type VARCHAR(20) NOT NULL DEFAULT 'web',
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  last_seen TIMESTAMP WITHOUT TIME ZONE,
  CHECK (device_type IN ('web', 'desktop', 'mobile'))
);
```

Maximum active devices per user: 5.

### device_sessions

```sql
CREATE TABLE device_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  device_id UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
  refresh_token_hash TEXT NOT NULL,
  ip VARCHAR(100),
  user_agent TEXT,
  last_seen TIMESTAMP WITHOUT TIME ZONE,
  revoked BOOLEAN NOT NULL DEFAULT FALSE,
  expires_at TIMESTAMP WITHOUT TIME ZONE NOT NULL,
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW()
);
```

Token rules:
- Access token: 1 hour
- Refresh token: 30 days

### chats

```sql
CREATE TABLE chats (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(20) NOT NULL DEFAULT 'direct',
  title VARCHAR(255),
  created_by UUID NOT NULL REFERENCES users(id),
  direct_pair_key TEXT UNIQUE,
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  CHECK (type IN ('direct', 'group'))
);
```

v0.1 rules:
- Only direct chats are supported.
- Empty chats do not exist.
- A chat is created only when the first message is sent.
- Only one direct chat may exist between two users.
- `direct_pair_key` should be deterministic from sorted user UUIDs.

### chat_participants

```sql
CREATE TABLE chat_participants (
  chat_id UUID NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(20) NOT NULL DEFAULT 'member',
  joined_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  PRIMARY KEY (chat_id, user_id),
  CHECK (role IN ('owner', 'admin', 'member'))
);
```

### messages

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id UUID NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  sender_id UUID NOT NULL REFERENCES users(id),
  ciphertext TEXT NOT NULL,
  message_type VARCHAR(20) NOT NULL DEFAULT 'text',
  reply_to UUID REFERENCES messages(id),
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  edited_at TIMESTAMP WITHOUT TIME ZONE,
  deleted_at TIMESTAMP WITHOUT TIME ZONE,
  CHECK (message_type IN ('text', 'image', 'video', 'audio', 'file'))
);
```

Important:
- v0.1 uses `ciphertext` for future compatibility.
- Real E2EE is not implemented in v0.1.
- Message identity/order rule: UUID + created_at.

### message_status

```sql
CREATE TABLE message_status (
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status VARCHAR(20) NOT NULL,
  timestamp TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  PRIMARY KEY (message_id, user_id),
  CHECK (status IN ('sent', 'delivered', 'read'))
);
```

Rules:
- v0.1 implements sent/delivered/read.
- `READ` always wins.
- Application logic must prevent downgrade.

### presence

```sql
CREATE TABLE presence (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  device_id UUID REFERENCES devices(id) ON DELETE SET NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'offline',
  last_seen TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  CHECK (status IN ('online', 'offline', 'away'))
);
```

v0.1 uses global presence: online/offline.

## Tables added since v0.1

These were applied as additive migrations between v0.2 and v0.6. See `Signalix-api/migrations/` for the exact DDL.

| Migration | Table / Change | Purpose |
|---|---|---|
| `V4__message_deletions.sql` | `message_deletions(message_id, user_id, deleted_at)` | Per-user soft delete for messages (delete-for-me) |
| `V5__password_reset.sql` | `password_reset_tokens` | Password reset token store (32-byte random, SHA-256 hashed, 15-min TTL) |
| `V6__email_verification.sql` | `email_verification_tokens` | Email verification tokens (32-byte random, SHA-256 hashed, 24-h TTL) |
| `V7__chat_deletions.sql` | `chat_deletions(chat_id, user_id, deleted_at)` | Per-user chat visibility cutoff (delete-chat-for-me) |
| `V8__message_reactions.sql` | `message_reactions(message_id, user_id, emoji, created_at)` | Emoji reactions, one per (message × user) |
| `V9__message_reply_forward.sql` | `messages.reply_to_message_id`, `messages.is_forwarded` | Reply chain + forward marker on the existing `messages` table |
| `V10__link_preview.sql` | `messages.link_preview` (JSONB) | Cached Open Graph / Twitter preview for the first URL in a text message |
| `V11__read_state.sql` | `chat_read_state(chat_id, user_id, last_read_message_id, ...)` | Persistent unread counters per (chat × user) |
| `V12__push_subscriptions.sql` | `push_subscriptions(id, user_id, endpoint, p256dh, auth, created_at, updated_at)` with `UNIQUE(user_id, endpoint)` and `idx_user_id` | Web Push device subscriptions added in v0.6.0. One row per (user × browser endpoint), so a single user can receive pushes on every device they've opted in from. Subscriptions returning 404/410 from the push service are pruned in-band by `PushService.sendToUser()`. |
| `V13__chats_avatar_description.sql` | `chats.avatar_url TEXT`, `chats.description TEXT` | Group avatar (public MinIO URL under `chats/{chatId}/...`) and editable plain-text description (≤500 chars, owner/admin editable). Both nullable; direct chats leave them `NULL`. Added in v0.7.0. |
| `V14__crypto_foundation.sql` | `device_identity_keys`, `signed_pre_keys`, `pre_keys` tables + 5 columns on `messages` (`encryption_version`, `sender_device_id`, `recipient_device_id`, `pre_key_id`, `signed_pre_key_id`) | Scaffolding for the Signal-Protocol-style E2EE layer added in v0.8.0; **populated and used in v0.9.0+** for direct text messages between v0.9.0 clients. v0.9.1 added server-side Ed25519 verification on insert into `signed_pre_keys` (no schema change), so any new row in that table is now guaranteed to carry a signature that verifies against the device's `signing_key`. |

No schema change was needed for **voice messages** in v0.6.1 — `V3__chat.sql`'s `CHECK (message_type IN ('text', 'image', 'video', 'audio', 'file'))` already permitted `'audio'`. Voice notes are stored as a `messages` row of type `audio` whose `ciphertext` carries `{ url, duration, size }` (mirroring how `file` works).

### v0.8.0 crypto-foundation tables in detail

```sql
CREATE TABLE device_identity_keys (
  device_id        UUID PRIMARY KEY REFERENCES devices(id) ON DELETE CASCADE,
  registration_id  INTEGER NOT NULL,           -- Signal-style session disambiguator
  identity_key     BYTEA NOT NULL,              -- X25519 public, 32 bytes raw
  signing_key      BYTEA NOT NULL,              -- Ed25519 public, 32 bytes raw
  key_algorithm    TEXT NOT NULL DEFAULT 'x25519',
  registered_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE signed_pre_keys (
  device_id        UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
  key_id           INTEGER NOT NULL,
  public_key       BYTEA NOT NULL,              -- X25519, 32 bytes raw
  signature        BYTEA NOT NULL,              -- Ed25519 sig over public_key, 64 bytes raw
  key_algorithm    TEXT NOT NULL DEFAULT 'x25519',
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  rotated_at       TIMESTAMPTZ,
  PRIMARY KEY (device_id, key_id)
);
-- Partial index: cheap "current signed pre-key" lookups.
CREATE INDEX idx_signed_pre_keys_current
  ON signed_pre_keys (device_id, created_at DESC)
  WHERE rotated_at IS NULL;

CREATE TABLE pre_keys (
  device_id        UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
  key_id           INTEGER NOT NULL,
  public_key       BYTEA NOT NULL,              -- X25519, 32 bytes raw
  key_algorithm    TEXT NOT NULL DEFAULT 'x25519',
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  consumed_at      TIMESTAMPTZ,
  PRIMARY KEY (device_id, key_id)
);
-- Partial index: speeds up the atomic `FOR UPDATE SKIP LOCKED` claim.
CREATE INDEX idx_pre_keys_unconsumed
  ON pre_keys (device_id, key_id)
  WHERE consumed_at IS NULL;

ALTER TABLE messages
  ADD COLUMN encryption_version  INTEGER NOT NULL DEFAULT 0,
  ADD COLUMN sender_device_id    UUID REFERENCES devices(id) ON DELETE SET NULL,
  ADD COLUMN recipient_device_id UUID REFERENCES devices(id) ON DELETE SET NULL,
  ADD COLUMN pre_key_id          INTEGER,
  ADD COLUMN signed_pre_key_id   INTEGER;
```

Notes:
- `encryption_version = 0` is the plaintext default — preserved on every pre-v0.8.0 message and every v0.8.0 send.
- The `FK ... ON DELETE SET NULL` on the message-level device columns is deliberate: removing a device shouldn't void history, just disconnect the envelope metadata.
- `signed_pre_keys` and `pre_keys` are owned by `devices`; `ON DELETE CASCADE` removes a device's keys when the device row goes away.

## Future Tables

Do not implement unless explicitly requested.

### message_deletions

Reference DDL for `delete_for_me` (now superseded by the actual `V4` migration).

```sql
CREATE TABLE message_deletions (
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  deleted_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  PRIMARY KEY (message_id, user_id)
);
```

### device_keys

For future Signal Protocol-style architecture.

### pre_keys

For future Signal Protocol-style session initialization.

### media_files

For future image/video/file support.

### push_tokens

For future push notification support.

## Required Indexes

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_auth_providers_user_id ON auth_providers(user_id);
CREATE INDEX idx_devices_user_id ON devices(user_id);
CREATE INDEX idx_device_sessions_user_id ON device_sessions(user_id);
CREATE INDEX idx_device_sessions_device_id ON device_sessions(device_id);
CREATE INDEX idx_chats_created_by ON chats(created_by);
CREATE INDEX idx_chat_participants_user_id ON chat_participants(user_id);
CREATE INDEX idx_messages_chat_id_created_at ON messages(chat_id, created_at);
CREATE INDEX idx_messages_sender_id ON messages(sender_id);
CREATE INDEX idx_message_status_user_id ON message_status(user_id);
CREATE INDEX idx_presence_status ON presence(status);
```

## Flyway File Structure

Inside `Signalix-api`:

```txt
migrations/
  V1__init.sql
  V2__auth.sql
  V3__chat.sql
```
