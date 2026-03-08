# redis

Optional add-on stack for running Redis on the VPS. Useful as a shared cache, session store, job queue, or pub/sub broker across multiple app stacks.

Like the other add-ons, this is intentionally separate from the core infrastructure — deploy it only when a service needs it.

---

## Service

### redis (Redis 7)
A Redis 7 instance using the lightweight Alpine image. It is configured to:
- Require password authentication (`--requirepass`) — Redis has no auth by default, so this is essential
- Persist data to disk using AOF (Append-Only File) with `everysec` fsync — a good balance between durability and performance. In the worst case (crash between fsyncs) you lose at most one second of writes.
- Store data in `./redis-data/`
- Only be accessible within the `web` Docker network — no host port is published

---

## Prerequisites

The core [vps-infra](../README.md) stack must be running (the `web` Docker network must exist).

---

## Configuration

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `REDIS_PASSWORD` | Password required for all Redis connections |

`.env` is gitignored and will never be committed.

---

## Usage

### Start the stack

```bash
docker compose up -d
```

### Stop the stack

```bash
docker compose down
```

### Connect via redis-cli

```bash
docker compose exec redis redis-cli -a $REDIS_PASSWORD
```

### View logs

```bash
docker compose logs -f redis
```

---

## Connecting from another stack

Any Docker Compose stack on the same VPS can reach Redis by attaching to the `web` network and using `redis` as the hostname:

```yaml
services:
  app:
    image: your-app-image
    environment:
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
    networks:
      - web

networks:
  web:
    external: true
```

The connection string format is `redis://:<password>@redis:6379` — note the colon before the password, which is the standard format when there is no username (Redis < 6 has no user concept; Redis 6+ defaults to the `default` user).

---

## A note on backups

Redis is most commonly used as a cache or session store, where losing data on a crash is acceptable. For that use case, the AOF persistence configured here is more than sufficient — it survives container restarts and redeployments.

If you are using Redis as a primary data store (e.g. for job queues where losing jobs would be a problem), consider adding an RDB snapshot as an additional safety net:

```yaml
command: >
  redis-server
  --requirepass ${REDIS_PASSWORD}
  --appendonly yes
  --appendfsync everysec
  --save 60 1
  --save 300 100
```

This takes an RDB snapshot every 60 seconds if at least 1 key changed, and every 5 minutes if at least 100 keys changed.

---

## Directory structure

```
.
├── compose.yml      # Stack definition
├── .env             # Your local secrets (gitignored)
├── .env.example     # Template — commit this
├── .gitignore
└── redis-data/      # Persisted data (gitignored)
```
