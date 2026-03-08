# postgres

Optional add-on stack for running PostgreSQL on the VPS, with automated daily backups and offsite sync to [Cloudflare R2](https://developers.cloudflare.com/r2/).

This is intentionally separate from the core infrastructure in [vps-infra](../README.md) — not every service needs a database, and when one does, it often makes sense to deploy a dedicated instance per project instead. Copy this folder to your VPS and deploy it when you need shared or standalone Postgres.

---

## Services

### db (PostgreSQL 17)
A PostgreSQL 17 instance using the lightweight Alpine image. Credentials are loaded from `.env` and data is persisted in `./postgres-data/`. The container is attached to the external `web` network so any app stack on the same VPS can reach it by hostname `db`.

### backup (postgres-backup-local)
[`prodrigestivill/postgres-backup-local`](https://github.com/prodrigestivill/docker-postgres-backup-local) runs `pg_dump` on a schedule and saves compressed backup files to `./backups/`. It is configured to:
- Dump daily at 2am
- Keep the last 7 daily backups
- Keep the last 4 weekly backups
- Keep the last 6 monthly backups

Old backups outside those windows are deleted automatically, keeping disk usage bounded.

### backup-sync (rclone)
[`rclone`](https://rclone.org/) syncs the `./backups/` directory to a Cloudflare R2 bucket once per hour. It runs immediately on startup and then every 60 minutes. Since rclone only transfers new or changed files, hourly syncs are cheap. This means your offsite copy is at most one hour behind the local backup.

---

## Backup flow

```
[PostgreSQL]
     ↓  pg_dump at 2am daily
[backup] → ./backups/  (local, rolling retention)
                ↓  rclone sync every hour
         [Cloudflare R2]  (offsite, persistent)
```

---

## Prerequisites

1. The core [vps-infra](../README.md) stack must be running (the `web` Docker network must exist).
2. A free [Cloudflare account](https://dash.cloudflare.com/) with R2 enabled.

### Setting up Cloudflare R2

1. Go to **R2 Object Storage** in the Cloudflare dashboard and create a bucket.
2. Go to **R2 → Manage R2 API tokens** and create a token with **Object Read & Write** permissions scoped to your bucket.
3. Copy the **Access Key ID**, **Secret Access Key**, and your **Account ID** (shown in the R2 overview page).
4. Your R2 endpoint will be: `https://<account_id>.r2.cloudflarestorage.com`

Cloudflare R2 includes 10GB of free storage and charges no egress fees.

---

## Configuration

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `POSTGRES_USER` | Database superuser username |
| `POSTGRES_PASSWORD` | Database superuser password |
| `POSTGRES_DB` | Default database name to create on first run |
| `R2_ACCESS_KEY_ID` | R2 API token Access Key ID |
| `R2_SECRET_ACCESS_KEY` | R2 API token Secret Access Key |
| `R2_ENDPOINT` | R2 endpoint URL (`https://<account_id>.r2.cloudflarestorage.com`) |
| `R2_BUCKET` | Name of the R2 bucket to sync backups into |

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

### View backup logs

```bash
docker compose logs -f backup
docker compose logs -f backup-sync
```

### Trigger a manual backup

```bash
docker compose exec backup /backup.sh
```

### List local backups

```bash
ls -lh backups/
```

---

## Restoring from a backup

Backups are plain SQL dumps compressed with gzip (`.sql.gz`). To restore:

**From a local backup file:**

```bash
gunzip -c backups/last/yourdb-latest.sql.gz | \
  docker compose exec -T db psql -U $POSTGRES_USER $POSTGRES_DB
```

**From R2** (if the VPS is gone and you're starting fresh):

1. Deploy this stack on the new VPS and start it: `docker compose up -d db`
2. Install rclone locally or on the new VPS and configure it with the same R2 credentials
3. Download the backup: `rclone copy r2:your-bucket/postgres/last/yourdb-latest.sql.gz .`
4. Restore it:
   ```bash
   gunzip -c yourdb-latest.sql.gz | \
     docker compose exec -T db psql -U $POSTGRES_USER $POSTGRES_DB
   ```

---

## Connecting from another stack

Any other Docker Compose stack on the same VPS can reach this database by attaching to the `web` network and using `db` as the hostname:

```yaml
services:
  app:
    image: your-app
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    networks:
      - web

networks:
  web:
    external: true
```

> **Note:** The database is not exposed on any host port — it is only reachable from within the `web` Docker network. This is intentional; never publish the Postgres port publicly.

---

## Directory structure

```
.
├── compose.yml        # Stack definition
├── .env               # Your local secrets (gitignored)
├── .env.example       # Template — commit this
├── .gitignore
├── backups/           # Local backup files (gitignored)
└── postgres-data/     # Persisted database files (gitignored)
```
