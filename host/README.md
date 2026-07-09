# Host configuration (hephaestus)

These are **host-level** settings that aren't part of any compose file but are
required for the observability stack. Checked in here so the setup is reproducible.

## Loki Docker log driver (log intake)

Every container's stdout is shipped to Loki by the **Loki Docker log driver**,
set as the daemon default in [`daemon.json`](daemon.json). This is why logs from
*all* stacks appear in Loki/Grafana with `container_name` / `compose_service` /
`host` labels — no per-app code needed.

Install / apply (hephaestus runs **snap** Docker):

```bash
# 1. Install the plugin (already installed here)
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions

# 2. Put daemon.json in place (snap path) and restart the daemon
sudo cp daemon.json /var/snap/docker/current/config/daemon.json
sudo snap restart docker      # brief blip; containers with restart:unless-stopped return
```

Notes:
- `mode: non-blocking` means a Loki outage drops logs instead of blocking containers.
- The default driver only affects **newly created** containers; recreate a stack
  (`docker compose up -d --force-recreate`) to move existing containers onto it.
- Keep Loki itself off the loki driver to avoid a feedback loop (it logs fine to
  the default json-file until you pin it).
