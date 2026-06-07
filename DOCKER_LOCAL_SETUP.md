# Signalix Docker Local Setup

## Goal

Run the Signalix v0.1 system locally using Docker Compose.

Services:

```txt
frontend
api
realtime
postgres
flyway
```

Optional later:

```txt
redis
pgadmin
nginx
minio
monitoring
```

## Local Ports

```txt
Frontend:  http://localhost:3000
API:       http://localhost:4000
Realtime:  ws://localhost:5000
Postgres:  localhost:5432
```

## Repository Relationship

```txt
workspace/
├── docs/
├── Signalix-contracts/
├── Signalix-api/
├── Signalix-realtime/
├── Signalix-frontend/
└── Signalix-infra/
```

`Signalix-infra/docker-compose.yml` can reference sibling repositories using relative paths.

## Environment Files

Never commit real `.env` files.

Each repo should include:

```txt
.env.example
```

Local real environment files can be created manually:

```txt
Signalix-api/.env
Signalix-realtime/.env
Signalix-frontend/.env
```

`.gitignore` must include:

```txt
.env
.env.local
.env.*.local
```

## Example API Environment

```txt
NODE_ENV=development
PORT=4000
DATABASE_URL=postgres://signalix:signalix_password@postgres:5432/signalix_dev
JWT_SECRET=replace_me_locally
JWT_ACCESS_EXPIRES_IN=1h
JWT_REFRESH_EXPIRES_IN=30d
```

## Example Realtime Environment

```txt
NODE_ENV=development
PORT=5000
JWT_SECRET=replace_me_locally
API_BASE_URL=http://api:4000
```

## Example Frontend Environment

```txt
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_WS_URL=ws://localhost:5000
```

## Example Postgres Environment

```txt
POSTGRES_USER=signalix
POSTGRES_PASSWORD=signalix_password
POSTGRES_DB=signalix_dev
```

## Flyway Environment

```txt
FLYWAY_URL=jdbc:postgresql://postgres:5432/signalix_dev
FLYWAY_USER=signalix
FLYWAY_PASSWORD=signalix_password
FLYWAY_LOCATIONS=filesystem:/flyway/sql
```

## Startup Order

```txt
postgres -> flyway -> api -> realtime -> frontend
```

## Local Development Commands

Recommended scripts inside `Signalix-infra/scripts`:

```bash
./scripts/up.sh
./scripts/down.sh
./scripts/logs.sh
./scripts/migrate.sh
```

## v0.1 Docker Compose Service List

```yaml
services:
  postgres:
    image: postgres:16

  flyway:
    image: flyway/flyway:latest
    depends_on:
      - postgres

  api:
    build: ../Signalix-api
    depends_on:
      - flyway

  realtime:
    build: ../Signalix-realtime
    depends_on:
      - api

  frontend:
    build: ../Signalix-frontend
    depends_on:
      - api
      - realtime
```

## Migration Strategy

DEV:

```txt
Docker startup -> Flyway auto
```

STAGING:

```txt
Pipeline -> Flyway -> Deploy
```

PROD:

```txt
Pipeline -> Flyway -> Approval -> Deploy
```
