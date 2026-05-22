# Home observability — Prometheus + Loki + Grafana on Unraid

GitOps observability stack for an Unraid NAS, scraping the home LAN and a
Hetzner-hosted Tailscale mesh. Everything is configured from files in this
repo — the Grafana UI is **not** the source of truth.

```
                ┌─────────────────────────────────────────────┐
                │             NAS (Unraid, Docker)            │
                │                                             │
                │  ┌──────────┐  ┌──────┐  ┌──────────┐       │
                │  │Prometheus│──│ Loki │  │ Grafana  │       │
                │  └─────┬────┘  └──┬───┘  └──────┬───┘       │
                │        │          │             │            │
                │        ├──── node_exporter (host net)         │
                │        ├──── blackbox_exporter                │
                │        ├──── unraid_api plugin                │
                │        │   Promtail (Docker socket → Loki)    │
                └────────┼─────────────────────────────────────┘
                         │
            ┌────────────┴───────────────┐
            │                            │
   LAN (192.168.1.0/24)         Tailscale (100.64.0.0/10)
            │                            │
        NPM box                  dns1.dbh.fyi
   ├─ node_exporter:9100        ├─ node_exporter:9100
   ├─ nginx-exporter:9113       ├─ tailscaled prom :5252
   └─ tailscaled prom :5252     ├─ technitium :5380/metrics
                                └─ promtail (binary + systemd) → Loki
                                dns2.dbh.fyi (same as dns1)
```

## Prerequisites

- Unraid with Docker enabled and the **Compose Manager** Community App.
- `appdata` share on the **cache pool** (the standard Unraid default).
- A `loki` user share configured to write to the **HDD array** — Loki
  chunks land here.
- Tailscale on every host (NAS, NPM, dns1, dns2).
- NPM already terminates HTTPS for `grafana.<PUBLIC_DOMAIN>` and
  `prometheus.<PUBLIC_DOMAIN>` and proxies to the NAS on `:3000` / `:9090`
  respectively. This repo doesn't touch NPM's proxy config.

## First-time deploy

1. **Clone the repo onto the NAS.** A sensible location:

   ```sh
   mkdir -p /mnt/user/appdata/observability
   cd /mnt/user/appdata/observability
   git clone <repo-url> .
   ```

2. **Create the array share for Loki chunks** (Unraid UI → Shares → Add).
   Use cache: No. Allocation method: any. Mount point ends up at
   `/mnt/user/loki/`.

3. **Fill in your secrets.**

   ```sh
   cp .env.example .env
   $EDITOR .env       # set PUBLIC_DOMAIN, GRAFANA_ADMIN_PASSWORD, ALERT_WEBHOOK_URL
   ```

4. **Fill in IPs.** Edit these files with your actual addresses:
   - `prometheus/targets/node.yml` — NAS LAN IP, NPM LAN IP, dns1/dns2 Tailscale IPs.
   - `prometheus/targets/technitium.yml` — Tailscale IPs of dns1/dns2.
   - `prometheus/targets/tailscale.yml` — Tailscale IPs of every host.
   - `prometheus/prometheus.yml` — `unraid_api` and `nginx` job target IPs.
   - `promtail/hetzner/promtail-config.yml` — NAS Tailscale IP in the
     `clients[].url`.

5. **Bring up the stack.**

   ```sh
   docker compose up -d
   docker compose ps
   ```

6. **Open Grafana** at `https://grafana.<PUBLIC_DOMAIN>/`. Log in as
   `admin` / your `GRAFANA_ADMIN_PASSWORD`. You should see five
   dashboards already provisioned: Node Overview, Unraid, DNS Health,
   Service Uptime, Log Explorer.

7. **Deploy exporters on the other hosts:**
   - **NPM box**: see [exporters/node-exporter/linux-host.md](exporters/node-exporter/linux-host.md)
     and [exporters/nginx-prometheus-exporter/](exporters/nginx-prometheus-exporter/).
   - **dns1, dns2**: see [exporters/node-exporter/linux-host.md](exporters/node-exporter/linux-host.md)
     and [promtail/hetzner/install.md](promtail/hetzner/install.md).
     Enable Technitium's `/metrics` endpoint in its admin UI (Settings
     → Web Service → Enable Prometheus metrics).

8. **Verify** at `http://nas:9090/targets` — every target should be `UP`.

## Day-to-day workflows

### Add a new Prometheus scrape target

Edit `prometheus/targets/apps.yml`:

```yaml
- targets: ["10.0.0.39:8080"]
  labels:
    app: my-new-service
    host: arcane
```

Then:

```sh
git add prometheus/targets/apps.yml && git commit -m "scrape my-new-service"
git push
# On the NAS:
git pull
curl -X POST http://localhost:9090/-/reload
```

Prometheus picks up the new target without a restart. Confirm at
`http://nas:9090/targets`.

### Add or update a Grafana dashboard

1. Edit (or add) a JSON file under `grafana/dashboards/`.
2. Commit, push, `git pull` on the NAS.
3. Grafana's file provider rescans every 30 seconds — no restart needed.

The dashboards are provisioned with `editable: false`, so saves from the
UI are blocked. To iterate on a dashboard, edit the JSON locally; or,
temporarily flip `editable: true` in
`grafana/provisioning/dashboards/dashboards.yml`, restart Grafana, make
changes in the UI, export JSON, paste it back into the repo, then flip
`editable: false` again.

### Add a Promtail log source (NAS)

Docker containers are auto-discovered. For non-Docker logs on the NAS,
add a `static_configs` block to `promtail/nas/promtail-config.yml`,
commit, `git pull`, and restart Promtail:

```sh
docker compose restart promtail
```

### Add a Promtail log source (Hetzner)

Edit `promtail/hetzner/promtail-config.yml`, then on dns1/dns2:

```sh
scp promtail/hetzner/promtail-config.yml dns1:/etc/promtail/
ssh dns1 systemctl restart promtail
```

### Change the alert webhook

Edit `ALERT_WEBHOOK_URL` in `.env`, then:

```sh
docker compose up -d grafana    # picks up the new env var
```

## Alert rules

Defined in `grafana/provisioning/alerting/rules.yml`:

| UID                            | Fires when                                                                  |
|--------------------------------|------------------------------------------------------------------------------|
| `alert-blackbox-probe-failed`  | `probe_success == 0` for > 2 minutes                                         |
| `alert-cert-expiring-soon`     | TLS cert expires in < 14 days                                                |
| `alert-node-down`              | `up{job="node"} == 0` for > 2 minutes                                        |
| `alert-disk-usage-high`        | filesystem > 85% full for > 10 minutes                                       |
| `alert-unraid-disk-temp-high`  | any array disk over 45 °C for > 10 minutes                                    |

All route to a single `default-webhook` contact point sourcing the URL
from `$ALERT_WEBHOOK_URL`.

## Storage layout on Unraid

| Path                                | Volume                       | Why                                              |
|-------------------------------------|------------------------------|--------------------------------------------------|
| `/mnt/user/appdata/prometheus/data` | Cache (SSD)                  | TSDB does many small random writes; HDD murders it |
| `/mnt/user/appdata/grafana`         | Cache (SSD)                  | SQLite + small file writes                       |
| `/mnt/user/appdata/loki/index`      | Cache (SSD)                  | Index lookups want low latency                   |
| `/mnt/user/appdata/loki/wal`        | Cache (SSD)                  | Sync writes                                      |
| `/mnt/user/loki/chunks`             | Array (HDD)                  | Sequential, append-only, large                    |
| `/mnt/user/loki/compactor`          | Array (HDD)                  | Long-running compaction working dir              |
| `/mnt/user/appdata/promtail`        | Cache (SSD)                  | positions.yaml (tiny)                            |

Loki retention is 30 days (`retention_period: 720h`); the compactor runs
inside the same container and enforces it.

## Known gaps and things to confirm on first install

1. **Unraid API plugin metric names.** The `unraid_api` scrape job uses
   `localhost:9999` by default and the dashboard / alert use names like
   `unraid_disk_temperature_celsius`. Confirm against
   `curl http://nas:9999/metrics` after plugin install and adjust:
   - `prometheus/prometheus.yml` (`unraid_api` job target/port)
   - `grafana/dashboards/unraid.json` (metric names in `expr`)
   - `grafana/provisioning/alerting/rules.yml` (`alert-unraid-disk-temp-high`)
2. **Tailscale Prometheus flag.** `--prom-listen-addr` is the current
   spelling on recent Tailscale; older versions used different names.
   See [exporters/node-exporter/linux-host.md](exporters/node-exporter/linux-host.md)
   step 6 and adjust if needed.
3. **Technitium `/metrics` path.** Present from Technitium 11.x. If your
   build doesn't expose it, upgrade or remove the `technitium` job
   from `prometheus/prometheus.yml`.
4. **NPM stub_status.** Only exposes connection counts (active, accepts,
   handled, requests, reading, writing, waiting) — not per-vhost stats.
   Swapping NPM's nginx for one with the VTS module is out of scope.

## Repository layout

```
.
├── README.md
├── docker-compose.yml
├── .env.example
├── prometheus/
│   ├── prometheus.yml
│   ├── targets/           ← file_sd: edit these to add targets
│   └── rules/             ← recording rules
├── blackbox/
│   └── blackbox.yml
├── loki/
│   └── loki-config.yml
├── promtail/
│   ├── nas/promtail-config.yml
│   └── hetzner/
│       ├── promtail-config.yml
│       ├── promtail.service
│       └── install.md
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/datasources.yml
│   │   ├── dashboards/dashboards.yml
│   │   └── alerting/
│   │       ├── contact-points.yml
│   │       ├── notification-policies.yml
│   │       └── rules.yml
│   └── dashboards/
│       ├── node-overview.json        (community #1860, vendored)
│       ├── service-uptime.json       (community #7587, vendored)
│       ├── unraid.json               (custom)
│       ├── dns-health.json           (custom)
│       └── log-explorer.json         (custom)
└── exporters/
    ├── nginx-prometheus-exporter/
    │   ├── docker-compose.yml
    │   └── stub_status.conf
    └── node-exporter/
        ├── unraid.md
        └── linux-host.md
```
