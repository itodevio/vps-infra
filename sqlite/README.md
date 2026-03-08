# sqlite

Template for deploying an app that uses SQLite, with continuous replication to [Cloudflare R2](https://developers.cloudflare.com/r2/) via [Litestream](https://litestream.io/).

Unlike PostgreSQL, SQLite is a file-based database embedded directly in the app process — there's no server to dump from on a schedule. Litestream solves this by running as a sidecar container that monitors the SQLite WAL (Write-Ahead Log) and streams changes to R2 continuously. The offsite copy is typically only seconds behind the live database.

---

## How it works

Litestream hooks into SQLite's WAL mechanism. Every time your app writes to the database, those changes are streamed to R2 almost immediately. This means:

- **No scheduled jobs** — replication is continuous, not periodic
- **Near real-time** — R2 is seconds behind the live database, not hours
- **Non-intrusive** — your app doesn't know Litestream exists; it just writes to a file as normal
- **Point-in-time restore** — Litestream stores snapshots and WAL segments, so you can restore to any point in time, not just the last backup

```
[app container] → writes to /data/app.db (shared volume)
                          ↓  WAL streaming (continuous)
               [litestream sidecar] → Cloudflare R2
```

---

## This is a template

Unlike the `postgres/` stack which is a standalone deployable service, this folder is a **template** — SQLite is embedded in your app, so there's no generic database container to deploy on its own. The `compose.yml` here shows the pattern to follow when building your app's stack.

**To use it:**

1. Copy this folder to your project (or VPS)
2. Replace the `app` service in `compose.yml` with your actual app image and config
3. Make sure your app writes its SQLite database to `/data/app.db` inside the container
4. Mount the `sqlite-data` volume at `/data` in your app service
5. Fill in `.env` and deploy

If your app uses a different database filename or path, update `litestream.yml` accordingly — the `path` field under `dbs` must point to the actual `.db` file inside the container.

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
| `APP_HOST` | Public hostname for your app (used by Traefik) |
| `R2_ACCESS_KEY_ID` | R2 API token Access Key ID |
| `R2_SECRET_ACCESS_KEY` | R2 API token Secret Access Key |
| `R2_ENDPOINT` | R2 endpoint URL (`https://<account_id>.r2.cloudflarestorage.com`) |
| `R2_BUCKET` | Name of the R2 bucket to replicate into |

If you have additional app-specific environment variables, add them to `.env` and reference them in the `app` service under `environment:`.

---

## Usage

### Start the stack

```bash
docker compose up -d
```

Litestream begins replicating immediately on startup. If the R2 bucket already contains a previous replica (e.g. after a restore), Litestream will pick up from where it left off.

### View replication logs

```bash
docker compose logs -f litestream
```

You should see periodic snapshot and WAL segment uploads confirming replication is active.

### Stop the stack

```bash
docker compose down
```

---

## Restoring from R2

Use this when setting up on a new VPS after losing the original, or to recover from data corruption.

**1. Start only the Litestream container in restore mode:**

```bash
docker run --rm \
  -v sqlite-data:/data \
  -e R2_ACCESS_KEY_ID=your_key \
  -e R2_SECRET_ACCESS_KEY=your_secret \
  litestream/litestream restore \
    -o /data/app.db \
    -config /dev/stdin <<EOF
dbs:
  - path: /data/app.db
    replicas:
      - type: s3
        bucket: your-bucket
        path: db
        endpoint: https://<account_id>.r2.cloudflarestorage.com
        access-key-id: \$R2_ACCESS_KEY_ID
        secret-access-key: \$R2_SECRET_ACCESS_KEY
        force-path-style: true
EOF
```

Or more simply, if you have this folder set up with `.env` already populated:

```bash
docker compose run --rm litestream restore -o /data/app.db
```

**2. Then start the full stack normally:**

```bash
docker compose up -d
```

Litestream will restore the database to its most recent state and then resume replication.

---

## Multiple databases

If your app uses more than one SQLite file, add entries to `litestream.yml`:

```yaml
dbs:
  - path: /data/app.db
    replicas:
      - type: s3
        bucket: $R2_BUCKET
        path: db
        endpoint: $R2_ENDPOINT
        access-key-id: $R2_ACCESS_KEY_ID
        secret-access-key: $R2_SECRET_ACCESS_KEY
        force-path-style: true

  - path: /data/cache.db
    replicas:
      - type: s3
        bucket: $R2_BUCKET
        path: cache
        endpoint: $R2_ENDPOINT
        access-key-id: $R2_ACCESS_KEY_ID
        secret-access-key: $R2_SECRET_ACCESS_KEY
        force-path-style: true
```

---

## Directory structure

```
.
├── compose.yml        # Stack template — edit the app service before deploying
├── litestream.yml     # Litestream replication config
├── .env               # Your local secrets (gitignored)
└── .env.example       # Template — commit this
```
