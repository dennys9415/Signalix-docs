# Claude Instructions for Signalix

You are generating code for Signalix, a professional multi-repo real-time messaging system inspired by WhatsApp and Signal.

## Core Rule

Signalix is a multi-repo system. Work on one repository at a time unless explicitly instructed otherwise.

## Active v0.1 Repositories

```txt
Signalix-contracts
Signalix-api
Signalix-realtime
Signalix-frontend
Signalix-infra
```

Do not create future repositories unless explicitly requested.

## Absolute Rules

1. Never invent contracts locally.
2. Always use `Signalix-contracts` as the source of truth.
3. Never duplicate DTOs inside API, frontend, or realtime if they belong in contracts.
4. Never modify shared enums without a version update.
5. Never remove fields from shared contracts.
6. All API responses must use `ApiResponse<T>`.
7. All WebSocket event names and payloads must exist in `Signalix-contracts`.
8. Do not implement real E2EE in v0.1.
9. Do not claim Signal Protocol is implemented in v0.1.
10. Do not create OAuth flows unless specifically requested.
11. Do not create password reset unless specifically requested for v0.2.
12. Do not introduce Prisma. Use direct SQL only.
13. Do not add unnecessary services or frameworks.
14. Do not create empty chats.
15. Do not implement group chats in v0.1.
16. If unsure, stop and ask before generating architecture-changing code.

## Technology Decisions

API:
- NestJS
- TypeScript
- PostgreSQL
- Direct SQL queries using `pg`
- JWT authentication
- Flyway migrations

Realtime:
- Node.js
- TypeScript
- `ws`
- JWT verification

Frontend:
- Next.js
- React
- TypeScript
- Tailwind CSS
- Zustand

Infrastructure:
- Docker Compose for local development
- PostgreSQL container
- Flyway for migrations
- `.env.example` committed
- `.env` ignored

## Branch and Release Strategy

Branches:

```txt
dev
staging
production
```

Tags:

```txt
v0.1.0
v0.2.0
v1.0.0
```

## v0.1 Scope

Build only:
- Register with email, username, and password
- Login with email or username and password
- JWT authentication
- Access token valid for 1 hour
- Refresh token valid for 30 days
- Maximum 5 active devices per user
- Exact username lookup only
- Direct 1-to-1 chats
- One direct chat only between two users
- Chat created only when first message is sent
- Text messages stored as `ciphertext`
- PostgreSQL message persistence
- API history loading
- WebSocket realtime updates
- Message statuses: `sent`, `delivered`, `read`
- Online/offline global presence
- Docker Compose local setup
- Shared contracts

Do not build in v0.1:
- OAuth
- Password reset
- Media uploads
- Real E2EE
- Push notifications
- Group chats
- Mobile app
- Advanced multi-device sync
- Kafka/NATS
- Redis Pub/Sub
- Kubernetes
- Monitoring stack

## Roadmap

v0.2:
- OAuth: Google, GitHub, Apple
- Password reset
- `delete_for_me`
- Partial username search

v0.3:
- `delete_for_everyone`
- Edit messages

Future:
- Signal Protocol
- `Signalix-crypto-sdk`
- Media
- Notifications
- Monitoring
- Distributed realtime scaling

## Message Rules

- Use `ciphertext`, not `content`.
- v0.1 does not implement real encryption.
- `READ` always wins.
- Message status must not revert.
- v0.1 has no edit and no delete.
- Soft delete field `deleted_at` exists for future use.
