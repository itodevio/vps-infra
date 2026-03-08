# vps-infra

Core infrastructure stack for a VPS, managed with Docker Compose. Includes a reverse proxy with automatic TLS, automatic container updates, uptime monitoring, and a full observability stack (logs + metrics).

## Services

### Traefik (reverse proxy)
[Traefik v3.1](https://doc.traefik.io/traefik/) handles all incoming HTTP/HTTPS traffic. It:
- Listens on ports 80 and 443
- Automatically redirects all HTTP traffic to HTTPS
- Obtains and renews TLS certificates from Let's Encrypt via the TLS challenge
- Routes requests to other containers based on their Docker labels
- Exposes its own dashboard at the host defined by `TRAEFIK_DASHBOARD_HOST`, protected by HTTP basic auth
- Has `exposedbydefault=false`, meaning containers are **not** routed unless they explicitly opt in with `traefik.enable=true`

### Uptime Kuma (uptime monitoring)
[Uptime Kuma](https://github.com/louislam/uptime-kuma) is a self-hosted monitoring tool that tracks whether your services are up and alerts you when they go down. It is exposed via Traefik at the host defined by `UPTIME_KUMA_HOST`. On first visit, you create an admin account through the web UI.

From the UI you can add monitors (HTTP, TCP, DNS, Docker containers, etc.) and configure notification channels. **Telegram** is the recommended notification method — it's free, natively supported, and takes about 2 minutes to set up via a Telegram bot.

### Watchtower
[Watchtower](https://containrrr.dev/watchtower/) monitors running containers and automatically pulls and redeploys updated images. It is configured to:
- Only update containers that have the `com.centurylinklabs.watchtower.enable=true` label (`--label-enable`)
- Check for updates every 30 seconds
- Use rolling restarts to minimise downtime
- Read private registry credentials from `~/.docker/config.json`

---

## Observability

The following services form a zero-config observability stack. Once running, every new container you deploy is **automatically monitored** — no labels or per-service configuration required.

### Promtail (log collector)
[Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/) watches the Docker socket and automatically tails the logs of every running container. It labels each log stream with `container`, `service`, `project`, and `stream`, then ships everything to Loki.

### Loki (log storage)
[Loki](https://grafana.com/oss/loki/) stores and indexes all container logs on the local filesystem. Logs are queryable in Grafana using LogQL — filter by service name, search for error strings, tail live output, etc.

### cAdvisor (container metrics)
[cAdvisor](https://github.com/google/cadvisor) scrapes per-container CPU, memory, network, and disk I/O metrics in real time. Auto-discovers all running containers with no configuration.

### node-exporter (host metrics)
[node-exporter](https://github.com/prometheus/node-exporter) exposes host-level metrics — total CPU usage, total RAM, disk space, network throughput. Pairing this with cAdvisor in Grafana lets you see a container's RAM as a percentage of total VPS memory.

### VictoriaMetrics (metrics storage)
[VictoriaMetrics](https://victoriametrics.com/) scrapes cAdvisor and node-exporter on a 15-second interval and stores metrics with a 3-month retention period. It is Prometheus-compatible, so standard PromQL queries and Grafana dashboards work out of the box.

### Grafana (visualisation)
[Grafana](https://grafana.com/) is the unified UI for both logs and metrics. Both Loki and VictoriaMetrics are pre-configured as datasources automatically on first start — no manual setup needed. Access it at the host defined by `GRAFANA_HOST`, log in with username `admin` and the password from `GRAFANA_ADMIN_PASSWORD`.

**Recommended dashboards to import** (use the ID in Grafana → Dashboards → Import):
- `1860` — Node Exporter Full (host metrics)
- `14282` — cAdvisor (per-container metrics)

> **Note on resources:** the observability stack uses approximately 500–700MB of RAM. This is negligible on a 4GB+ VPS but worth knowing on smaller instances.

---

## Network

All services share an **external** Docker network called `web`. This network must exist before starting the stack, and all other app stacks on the same VPS should also attach to it so Traefik can reach them.

Create it once:

```bash
docker network create web
```

---

## Prerequisites

1. **Follow the [vps-guide](https://github.com/itodevio/vps-guide)** — covers initial server setup (firewall, SSH hardening, Docker installation, etc.) and should be completed before setting up this stack.
2. A domain with DNS A records pointing to your server's IP for any hostname you want to use.

---

## Configuration

### 1. Environment file (this repo)

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `ACME_EMAIL` | Email address for Let's Encrypt certificate notifications |
| `TRAEFIK_DASHBOARD_HOST` | Hostname for the Traefik dashboard (e.g. `dashboard.yourdomain.com`) |
| `TRAEFIK_DASHBOARD_AUTH` | HTTP basic auth credentials in htpasswd format (see below) |
| `UPTIME_KUMA_HOST` | Hostname for Uptime Kuma (e.g. `status.yourdomain.com`) |
| `GRAFANA_HOST` | Hostname for Grafana (e.g. `grafana.yourdomain.com`) |
| `GRAFANA_ADMIN_PASSWORD` | Password for the Grafana `admin` user |

`.env` is gitignored and will never be committed.

#### Generating a dashboard password

Use `htpasswd` (from the `apache2-utils` package) to generate the value for `TRAEFIK_DASHBOARD_AUTH`:

```bash
htpasswd -nb admin yourpassword
```

This outputs something like:

```
admin:$apr1$xxxx$xxxxxxxxxxxxxxxxxxxxxxxx
```

Paste the full output as the value of `TRAEFIK_DASHBOARD_AUTH` in your `.env`. Dollar signs do **not** need to be escaped in the `.env` file.

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

### View logs

```bash
# All services
docker compose logs -f

# Single service
docker compose logs -f traefik
```

### Restart a single service

```bash
docker compose restart traefik
```

---

## Exposing other services via Traefik

Any other Docker Compose stack on the same server can be routed through Traefik by:

1. Attaching to the `web` network
2. Adding the appropriate Traefik labels

Minimal example:

```yaml
services:
  app:
    image: your-app
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`app.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=myresolver"
      # optional: enable watchtower auto-updates for this container
      - "com.centurylinklabs.watchtower.enable=true"

networks:
  web:
    external: true
```

Traefik will automatically detect the container, request a TLS certificate for the hostname, and start routing traffic — no restart required.

---

## Directory structure

```
.
├── compose.yml          # Main stack definition
├── .env                 # Your local secrets (gitignored)
├── .env.example         # Template — commit this
├── .gitignore
├── config/              # Service configuration files (committed)
│   ├── loki.yml
│   ├── promtail.yml
│   ├── victoriametrics-scrape.yml
│   └── grafana/provisioning/datasources/datasources.yml
├── letsencrypt/         # Persisted ACME certificates (gitignored)
├── uptime-kuma-data/    # Persisted Uptime Kuma data (gitignored)
├── loki-data/           # Persisted Loki logs (gitignored)
├── victoriametrics-data/ # Persisted metrics (gitignored)
└── grafana-data/        # Persisted Grafana data (gitignored)
```
