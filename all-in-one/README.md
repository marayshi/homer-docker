# Homer 10 — Production SIP Capture

HOMER 10 stack for capturing SIP traffic from Kamailio via HEP/UDP, storing in ClickHouse, and visualizing in Grafana.

Based on [sipcapture/homer-docker](https://github.com/sipcapture/homer-docker). Stripped down for production — no demo services, secrets externalized, configurable via `.env`.

## Architecture

```
Kamailio SBC ──HEP/UDP──▶ heplify-server ──Loki push──▶ qryn ──▶ ClickHouse
                                  │                        │
                             metrics:9096             Loki/Tempo/Prom API
                                  │                        │
                               Vector ──prom write──▶   Grafana :3000
                                  │
                             node-exporter
```

### Services (6 containers)

| Service | Role | Port |
|---------|------|------|
| **clickhouse-server** | Time-series database | internal only |
| **qryn** | Polyglot API (Loki/Tempo/Prometheus) | internal only |
| **heplify-server** | HEP receiver (SIP capture) | `${HEP_LISTEN}` (default `127.0.0.1:9060/udp`) |
| **grafana** | Dashboards & UI | `${GRAFANA_LISTEN}` (default `0.0.0.0:3000`) |
| **vector** | Metrics scraping & forwarding | internal only |
| **node-exporter** | Host system metrics | internal only |

## Quick Start

### 1. Configure

```bash
cp .env.example .env
nano .env
```

Set your passwords and network config in `.env`:

```env
# Grafana
ADMIN_USER=admin
ADMIN_PASSWORD=<strong-password>
GRAFANA_ROOT_URL=http://<your-server-ip>:3000
GRAFANA_LISTEN=0.0.0.0:3000

# ClickHouse (must match across both vars)
CLICKHOUSE_USER=qryn
CLICKHOUSE_PASSWORD=<strong-password>
CLICKHOUSE_AUTH=qryn:<strong-password>

# HEP listener
# 127.0.0.1:9060 — Kamailio on same host (localhost)
# 0.0.0.0:9060   — Kamailio on a remote server
HEP_LISTEN=127.0.0.1:9060
```

### 2. Start

```bash
docker compose pull
docker compose up -d
docker compose ps    # all 6 services should be running
```

### 3. Access Grafana

Open `http://<your-server-ip>:3000` and login with the credentials from `.env`.

Test datasources: **Settings → Data Sources → Test** each (Loki, Prometheus, Tempo).

### 4. Configure Kamailio to send HEP

Add to your Kamailio configuration (`kamailio.cfg`):

```kamailio
# ---- HEP / Homer tracing ----
loadmodule "siptrace.so"

# If Kamailio is on the same host, use 127.0.0.1
# If remote, use the Homer server's IP
modparam("siptrace", "duplicate_uri", "sip:127.0.0.1:9060;transport=udp")
modparam("siptrace", "hep_mode_on", 1)
modparam("siptrace", "trace_to_database", 0)
modparam("siptrace", "hep_version", 3)
modparam("siptrace", "hep_capture_id", 100)
modparam("siptrace", "trace_on", 1)
```

Then call `sip_trace()` in your routing logic:

```kamailio
request_route {
    sip_trace();
    # ... rest of your routing logic
}

onreply_route {
    sip_trace();
}

failure_route {
    sip_trace();
}
```

Restart Kamailio:

```bash
systemctl restart kamailio
```

Make a test SIP call and check the **SIP Overview** dashboard in Grafana.

## Alerting (Slack)

Grafana handles alerting natively — **no AlertManager needed**. Grafana's unified alerting can send directly to Slack, email, PagerDuty, webhooks, etc.

Setup via Grafana UI:

1. **Alerting → Contact points → Add contact point**
2. Select **Slack**, paste your [Incoming Webhook URL](https://api.slack.com/messaging/webhooks), set the channel
3. **Notification policies** → set the default contact point to your Slack
4. **Alert rules** → create rules against Loki/Prometheus datasources (e.g., SIP error rate > threshold)

## Dashboards

Pre-provisioned Grafana dashboards:

- **SIP Overview** — ASR, NER, call metrics
- **Call Flow** — SIP ladder/flow visualization
- **CDR Search** — Call Detail Records
- **SIP KPIs / Stats / Error Rates** — Detailed SIP analytics
- **SIP Method Responses** — Method-level breakdown
- **SIP Call Register** — Registration tracking
- **QOS RTCP / XRTP / Horaclifix** — Voice quality metrics
- **HEP Stats** — HEP protocol statistics
- **Host Overview** — System metrics (node-exporter)

## Data Retention

Configured to **30 days** via qryn `LABELS_DAYS` and `SAMPLES_DAYS` environment variables. ClickHouse automatically purges data older than 30 days.

## Maintenance

```bash
# View logs
docker compose logs -f heplify-server
docker compose logs -f qryn

# Restart a service
docker compose restart heplify-server

# Update images
docker compose pull && docker compose up -d

# Check disk usage
docker system df
du -sh /var/lib/docker/volumes/all-in-one_clickhouse_data/
```

## Configuration Reference

All settings are in `.env` (see `.env.example` for docs):

| Variable | Default | Description |
|----------|---------|-------------|
| `ADMIN_USER` | `admin` | Grafana admin username |
| `ADMIN_PASSWORD` | `admin` | Grafana admin password |
| `GRAFANA_ROOT_URL` | `http://localhost:3000` | Public URL for Grafana (used in alert links) |
| `GRAFANA_LISTEN` | `0.0.0.0:3000` | Grafana bind address |
| `CLICKHOUSE_USER` | `qryn` | ClickHouse username |
| `CLICKHOUSE_PASSWORD` | `changeme` | ClickHouse password |
| `CLICKHOUSE_AUTH` | `qryn:changeme` | ClickHouse auth string for qryn |
| `HEP_LISTEN` | `127.0.0.1:9060` | HEP UDP bind address |

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Service definitions (6 containers) |
| `.env` | Production secrets (not in git) |
| `.env.example` | Template for `.env` |
| `vector/vector.toml` | Metrics scraping config |
| `clickhouse/opentelemetry_zipkin.sql` | ClickHouse init (trace export) |
| `grafana/provisioning/` | Grafana datasources, dashboards, alerts |
