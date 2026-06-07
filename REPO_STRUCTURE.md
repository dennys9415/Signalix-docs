# Signalix Repository Structure

Signalix starts as a multi-repo system from day one.

Recommended local workspace:

```txt
workspace/
├── docs/
├── Signalix-contracts/
├── Signalix-api/
├── Signalix-realtime/
├── Signalix-frontend/
└── Signalix-infra/
```

## Active Repositories for v0.1

```txt
Signalix-contracts
Signalix-api
Signalix-realtime
Signalix-frontend
Signalix-infra
```

## docs

Workspace-level documentation.

```txt
docs/
├── ARCHITECTURE.md
├── CLAUDE.md
├── CONTRACTS_RULES.md
├── DATABASE_SCHEMA.md
├── DOCKER_LOCAL_SETUP.md
├── MVP_SCOPE.md
└── REPO_STRUCTURE.md
```

This is not required to be a separate git repo yet.

## Signalix-api

```txt
Signalix-api/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/
│   ├── db/
│   ├── auth/
│   ├── users/
│   ├── devices/
│   ├── sessions/
│   ├── chats/
│   ├── messages/
│   └── common/
├── migrations/
│   ├── V1__init.sql
│   ├── V2__auth.sql
│   └── V3__chat.sql
├── scripts/
│   └── migrate.sh
├── Dockerfile
├── package.json
├── tsconfig.json
├── .env.example
├── .gitignore
├── CLAUDE.md
└── README.md
```

## Signalix-realtime

```txt
Signalix-realtime/
├── src/
│   ├── server.ts
│   ├── config/
│   ├── auth/
│   ├── ws/
│   │   ├── connection-manager.ts
│   │   ├── event-router.ts
│   │   └── room-manager.ts
│   ├── presence/
│   └── common/
├── Dockerfile
├── package.json
├── tsconfig.json
├── .env.example
├── .gitignore
├── CLAUDE.md
└── README.md
```

## Signalix-frontend

```txt
Signalix-frontend/
├── src/
│   ├── app/
│   ├── components/
│   ├── features/
│   │   ├── auth/
│   │   ├── chat/
│   │   └── presence/
│   ├── lib/
│   │   ├── api-client.ts
│   │   ├── ws-client.ts
│   │   └── auth.ts
│   ├── store/
│   └── types/
├── public/
├── Dockerfile
├── package.json
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── .env.example
├── .gitignore
├── CLAUDE.md
└── README.md
```

## Signalix-infra

```txt
Signalix-infra/
├── docker-compose.yml
├── env/
│   ├── api.env.example
│   ├── frontend.env.example
│   ├── realtime.env.example
│   ├── postgres.env.example
│   └── flyway.env.example
├── postgres/
│   └── init/
├── docs/
│   └── local-setup.md
├── scripts/
│   ├── up.sh
│   ├── down.sh
│   ├── logs.sh
│   └── migrate.sh
├── CLAUDE.md
└── README.md
```

Future repositories:

```txt
Signalix-auth
Signalix-notifications
Signalix-media
Signalix-crypto-sdk
Signalix-monitoring
```
