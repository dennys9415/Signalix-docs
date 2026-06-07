# Signalix Architecture

## Project Summary

Signalix is a real-time messaging web application inspired by WhatsApp and Signal. The first version starts as a professional direct chat MVP with a clean multi-repo architecture, local Docker-based development, direct PostgreSQL persistence, WebSocket communication, and shared TypeScript contracts.

The long-term vision is to support secure messaging, multi-device sessions, OAuth providers, media, notifications, and eventual Signal Protocol-style end-to-end encryption.

v0.1 must remain focused and avoid overengineering.

## Active v0.1 Repositories

```txt
Signalix-contracts
Signalix-api
Signalix-realtime
Signalix-frontend
Signalix-infra
```

Future repositories should not be created during v0.1 unless explicitly approved:

```txt
Signalix-auth
Signalix-notifications
Signalix-media
Signalix-crypto-sdk
Signalix-monitoring
```

## Repository Responsibilities

### Signalix-contracts

The single source of truth for shared communication structures.

Owns:
- API DTOs
- API response types
- WebSocket event names
- WebSocket payloads
- Shared enums
- Error codes
- Protocol constants
- Versioning rules

Must not contain:
- Business logic
- Database queries
- Runtime service code
- Framework-specific implementation

### Signalix-api

The central backend API.

Technology:
- NestJS
- TypeScript
- PostgreSQL
- Direct SQL queries
- `pg`
- JWT authentication
- Flyway migrations

Owns:
- Registration
- Login by email or username and password
- JWT issuing
- Refresh token sessions
- Users
- Auth providers table
- Devices
- Device sessions
- Chats
- Messages
- Message persistence
- Message status persistence
- Exact username lookup
- REST API validation

Does not own:
- Persistent WebSocket connections
- Frontend UI
- Infrastructure orchestration
- Media processing in v0.1
- Real E2EE in v0.1

### Signalix-realtime

The WebSocket service.

Technology:
- Node.js
- TypeScript
- `ws`
- JWT verification

Owns:
- WebSocket connections
- JWT-based socket authentication
- Online/offline presence
- Message delivery events
- Read receipt events
- Basic direct chat realtime events
- Typing indicators later

In v0.1, the realtime service must not duplicate API business logic. It should use contracts from `Signalix-contracts`.

### Signalix-frontend

The web client.

Technology:
- Next.js
- React
- TypeScript
- Tailwind CSS
- Zustand

Owns:
- Register UI
- Login UI
- Direct chat UI
- Exact username lookup UI
- Message list rendering
- Message input
- WebSocket client
- Local UI state
- Online/offline indicators

### Signalix-infra

Local infrastructure and future deployment configuration.

Owns:
- Docker Compose
- Local service orchestration
- PostgreSQL container
- Flyway migration container or startup step
- Optional database admin container
- Environment templates
- Local setup documentation
- Future CI/CD and deployment configs

Must not contain business logic.

## Runtime Flow

### Login Flow

```txt
Frontend
  -> API login endpoint
  -> API validates email/username + password
  -> API creates or updates device/session
  -> API returns access token + refresh token
  -> Frontend stores auth state
  -> Frontend connects to realtime service with JWT
```

JWT rules:

```txt
Access token: 1 hour
Refresh token: 30 days
```

### OAuth Flow

OAuth is part of v0.2, not v0.1.

Supported providers planned:
- Google
- GitHub
- Apple

### WebSocket Flow

```txt
Frontend opens WebSocket connection
  -> sends JWT during connection/auth handshake
  -> realtime verifies JWT
  -> realtime marks user online
  -> frontend receives realtime events
  -> API remains source of truth for message persistence
```

## Message History and Realtime Rule

Signalix uses:

```txt
API for history
WebSocket for realtime only
```

Offline recovery for v0.1:

```txt
Client reconnects
  -> frontend pulls missed messages from API
  -> frontend resumes realtime updates through WS
```

## v0.1 Message Model

v0.1 does not implement real end-to-end encryption.

However, Signalix uses the field name:

```txt
ciphertext
```

from v0.1 to prepare the system for future Signal Protocol compatibility.

Important:

```txt
v0.1 stores message content in a field named ciphertext for future compatibility.
Real E2EE is not implemented yet.
Do not market v0.1 as encrypted chat.
Do not claim Signal Protocol support until the crypto layer exists.
```

## Direct Chat Rules

- v0.1 supports only direct 1-to-1 chats.
- Group chats are out of scope for v0.1.
- Only one direct chat may exist between two users.
- Empty chats do not exist.
- A chat is created only when the first message is sent.

## Username Rules

- `username` is unique.
- v0.1 supports exact username lookup only.
- If a user changes username in the future, the old username becomes immediately available.
- Partial username search is planned for v0.2.

## Message State Rules

Message statuses:

```txt
sent
delivered
read
```

Rules:
- v0.1 implements all three statuses.
- `READ` always wins.
- Message status must never revert.
- Realtime transports events.
- API/database remains source of truth.

## Deletion/Edit Rules

```txt
v0.1: no edit, no delete
v0.2: delete_for_me
v0.3: delete_for_everyone + edit
```

The DB prepares for soft delete using:

```txt
deleted_at
```

## Device Rules

- v0.1 prepares device/session architecture.
- Maximum active devices per user: 5.
- Advanced multi-device Signal Protocol behavior is future scope.

## Scaling Direction

v0.1:
- Docker Compose local
- PostgreSQL local
- Single API instance
- Single realtime instance
- No Redis required yet

Future:
- Cloud staging
- Cloud production
- Redis Pub/Sub for distributed realtime
- Queue-based notifications
- Object storage for media
- Monitoring dashboards
- Kubernetes or managed containers

## Non-Negotiable Architecture Rule

```txt
contracts -> api -> realtime -> frontend
```

No service may invent duplicate DTOs, enums, or WebSocket payloads locally.
