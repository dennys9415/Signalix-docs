# Signalix MVP Scope

## Version

```txt
v0.1.0
```

## Goal

Build a local Docker-based real-time direct messaging MVP with authentication, PostgreSQL persistence, WebSocket communication, online/offline presence, Flyway migrations, and shared TypeScript contracts.

## In Scope for v0.1

### Authentication

- Register with email, username, and password.
- Login with email or username and password.
- Password hashing.
- JWT issuing.
- Access token valid for 1 hour.
- Refresh token valid for 30 days.
- Basic session tracking.
- Maximum 5 active devices per user.

### OAuth Direction

OAuth is v0.2 scope.

Providers:
- Google
- GitHub
- Apple

### Password Reset

Password reset is v0.2 scope.

### Users

- User creation.
- Unique email.
- Unique username.
- Display name.
- Basic profile fields.
- Exact username lookup only.

Future:
- Partial username search in v0.2.
- Old username becomes immediately available after username change.

### Chats

- Create direct 1-to-1 chat only when the first message is sent.
- Only one direct chat may exist between two users.
- Empty chats do not exist.
- List user chats.
- Retrieve chat messages through API.

### Messages

- Send text message.
- Store message in PostgreSQL.
- Use `ciphertext` as the message field name.
- Return message through API contract.
- Broadcast message through realtime service.
- Show message in frontend.
- Use UUID + created_at for message identity/order.
- Implement statuses:
  - sent
  - delivered
  - read

Important:

```txt
v0.1 uses ciphertext as a future-compatible field name.
Real E2EE is not implemented in v0.1.
```

### Edit/Delete

v0.1:

```txt
no edit
no delete
```

v0.2:

```txt
delete_for_me
```

v0.3:

```txt
delete_for_everyone
edit message
```

DB should prepare soft delete with:

```txt
deleted_at
```

### Realtime

- WebSocket server using `ws`.
- Authenticate socket connection using JWT.
- Online/offline global presence.
- Message broadcast.
- Delivery/read receipt realtime events.
- API loads message history.
- WebSocket handles realtime only.
- Offline recovery through API pull on reconnect.

### Infrastructure

- Docker Compose local setup.
- PostgreSQL container.
- Flyway migration service.
- API container.
- Realtime container.
- Frontend container.
- `.env.example` files only.
- `.env` ignored.

## Out of Scope for v0.1

- Real Signal Protocol encryption.
- End-to-end encryption claims.
- OAuth.
- Password reset.
- Media uploads.
- Push notifications.
- Group chats.
- Mobile apps.
- Advanced multi-device sync.
- MFA/2FA.
- Device trust.
- Redis Pub/Sub.
- Kafka/NATS.
- Kubernetes.
- Production cloud deployment.
- Monitoring stack.
- CDN.
- Streaming.

## MVP Success Criteria

The MVP is successful when:

1. A user can register.
2. A user can login using email or username and password.
3. A user receives an access token and refresh token.
4. The frontend connects to the realtime service using the JWT.
5. A user can find another user by exact username.
6. A user can send a first message to create a direct chat.
7. Only one direct chat exists between two users.
8. A user can send a text message.
9. The message is saved in PostgreSQL as `ciphertext`.
10. The other connected user receives the message in realtime.
11. Message statuses support sent/delivered/read.
12. Online/offline status appears in the UI.
13. Message history loads through API.
14. Offline recovery works through API pull on reconnect.
15. The full stack runs locally through Docker Compose.
