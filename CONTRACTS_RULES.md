# Signalix Contracts Rules

## Purpose

`Signalix-contracts` is the single source of truth for communication between repositories.

It defines:
- API request/response shapes
- WebSocket event names
- WebSocket payloads
- Shared enums
- Error codes
- Shared response wrappers
- Protocol constants
- Version metadata

It does not define:
- Business logic
- Database query logic
- Framework-specific implementation
- Runtime services

## Dependency Direction

```txt
Signalix-contracts
  -> Signalix-api
  -> Signalix-realtime
  -> Signalix-frontend
```

## Required Shared API Response

```ts
export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: ApiError;
}

export interface ApiError {
  code: ErrorCode;
  message: string;
  context?: Record<string, unknown>;
}
```

## Message Types

```ts
export enum MessageType {
  TEXT = "text",
  IMAGE = "image",
  VIDEO = "video",
  AUDIO = "audio",
  FILE = "file"
}
```

v0.1 implements only `TEXT`.

## Message Status

```ts
export enum MessageStatus {
  SENT = "sent",
  DELIVERED = "delivered",
  READ = "read"
}
```

v0.1 implements all three:
- sent
- delivered
- read

Rules:
- `READ` always wins.
- Message status must never revert.
- API/database remains source of truth.

## Message Lifecycle

```txt
CREATED -> SENT -> DELIVERED -> READ
           ↘ FAILED
```

## WebSocket Events

Client events:
- `client.authenticate`
- `client.message.send`
- `client.message.delivered`
- `client.message.read`
- `client.typing.start`
- `client.typing.stop`
- `client.presence.update`
- `client.heartbeat`

Server events:
- `server.authenticated`
- `server.message.new`
- `server.message.sent`
- `server.message.delivered`
- `server.message.read`
- `server.typing.start`
- `server.typing.stop`
- `server.user.online`
- `server.user.offline`
- `server.error`
- `server.heartbeat.ack`

Typing can exist in contracts but does not need to be fully implemented in v0.1.

## v0.1 Message Payloads

Use `ciphertext`, not `content`.

```ts
export interface ClientMessageSendPayload {
  chatId?: string;
  recipientUsername?: string;
  ciphertext: string;
  messageType: MessageType.TEXT;
  tempId?: string;
}

export interface ServerMessageNewPayload {
  messageId: string;
  chatId: string;
  senderId: string;
  ciphertext: string;
  messageType: MessageType;
  timestamp: string;
}
```

Important:

```txt
v0.1 uses the field name ciphertext for future compatibility.
Real E2EE is not implemented in v0.1.
```

## Breaking Change Rules

Breaking changes include:
- Removing fields
- Renaming fields
- Changing enum values
- Removing enum values
- Renaming WebSocket events
- Changing payload structure
- Changing API response wrappers

Safe changes include:
- Adding optional fields
- Adding new enum values when old values remain valid
- Adding new events
- Adding new DTOs

## Versioning

Use semantic versioning:

```txt
MAJOR.MINOR.PATCH
```

Examples:
- `v0.1.0` initial MVP contracts
- `v0.2.0` OAuth/password reset/delete_for_me contracts added
- `v0.3.0` edit/delete_for_everyone contracts added
- `v1.0.0` production-ready stable contracts
