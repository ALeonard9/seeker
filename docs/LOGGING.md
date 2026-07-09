# Logging & monitoring — how to query it

The homelab ships **every container's logs** to Loki automatically (the Loki
Docker log driver is the daemon default on hephaestus), and **metrics** for the
host, every container, and the Proxmox node into Prometheus. Grafana ties both
together.

- **Grafana**: `https://grafana.aleonard.us` (or `http://hephaestus.aleonard.us:3000`)
  - Dashboard **Homelab / Stack Health** — host CPU/mem/disk, per-container CPU/mem,
    restarts, Proxmox node + guests.
  - Dashboard **Homelab / Logs** — log volume, error volume, and a live log view with
    a container filter.
- **Loki**: `http://hephaestus.aleonard.us:3100`  ·  **Prometheus**: internal (`gaia_prometheus_prod:9090`).

## The log labels (keep queries simple)

Every log line carries these Loki labels (added by the Docker driver, low-cardinality):

| Label | Example | Meaning |
|-------|---------|---------|
| `host` | `hephaestus` | The Docker host |
| `container_name` | `orion_web_prod` | The container |
| `compose_service` | `orion_web` | The compose service |
| `compose_project` | `orion` | The compose project/stack |
| `service_name` | `aleonard-us-api` | Set by apps that push directly (e.g. the API) |

**Rule of thumb:** filter by label first (`{...}`), then grep the line with `|=`/`|~`.
Keep labels for *identity* (who), keep everything else *in the log line* (what).

## LogQL cheat sheet

```logql
# Everything from one container
{container_name="orion_web_prod"}

# Everything on the host, only errors (case-insensitive)
{host="hephaestus"} |~ "(?i)error|exception|traceback|fatal"

# One stack (all its containers)
{compose_project="orion"}

# The new API, only ERROR level
{container_name="phoenix_api_prod"} |= "ERROR"

# Error rate per container over 5m (for a graph / alert)
sum by (container_name) (rate({host="hephaestus"} |~ "(?i)error" [5m]))

# Count of a specific string in the last hour
sum(count_over_time({container_name="phoenix_web_prod"} |= "500" [1h]))

# Structured logs: parse JSON or logfmt, then filter by a field
{container_name="phoenix_api_prod"} | json | level="ERROR"
{compose_service="orion_web"} | logfmt | status>=500
```

## Writing app logs so they're easy to query

1. **One event per line.** No multi-line stack traces split across lines where you can help it.
2. **Include a level token** (`INFO`/`WARN`/`ERROR`) so `|~ "(?i)error"` and `| json | level=…` work.
3. **Prefer structured** (JSON or `key=value` logfmt) for anything you'll filter on
   (status, user, duration) — then `| json` / `| logfmt` exposes them as fields.
4. **Don't put high-cardinality values in Loki labels** (no user ids / request ids as
   labels) — put them in the line and extract at query time.
5. Let the Docker driver own identity labels (`container_name`, `compose_service`);
   apps don't need to set those.

## Metrics quick reference (Prometheus)

```promql
# host
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)          # CPU %
(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes) * 100      # Mem %
# containers
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m]))      # CPU cores
sum by (name) (changes(container_start_time_seconds{name!=""}[1h]))        # restarts
# proxmox (gaia)
pve_up                                                                     # node/guest up
pve_cpu_usage_ratio{id=~"node/.*"}                                         # node CPU
```

## Health alerting

Grafana unified alerting can watch these and notify the existing Telegram channel
(the Watchtower bot). Suggested starter alerts:
- **Container silent** — `absent_over_time({container_name="orion_web_prod"}[10m])` (no logs = likely down).
- **Error spike** — error-rate query above > N for 5m.
- **Host disk** — host disk % > 90 for 15m.
- **Proxmox node down** — `pve_up{id=~"node/.*"} == 0`.
Wire the Telegram contact point (bot token + chat id) once and route these to it.
