# node_exporter on Unraid

The NAS's own node_exporter is already defined in the top-level
`docker-compose.yml` as the `node_exporter` service. It uses **host
networking** and bind-mounts `/proc`, `/sys`, and `/` so it can see real
host metrics rather than container-namespaced ones.

If you'd rather run it via the Unraid UI (Docker tab → Add Container)
instead of the Compose stack, use these settings:

| Field             | Value                                                              |
|-------------------|--------------------------------------------------------------------|
| Repository        | `quay.io/prometheus/node-exporter:v1.8.2`                          |
| Network Type      | `host`                                                             |
| Privileged        | off                                                                |
| Extra Parameters  | `--pid=host`                                                       |
| Post Arguments    | see below                                                          |

Post arguments (the command the container runs):

```
--path.rootfs=/host
--path.procfs=/host/proc
--path.sysfs=/host/sys
--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($|/)
```

Bind mounts (Add path):

| Container path | Host path | Mode               |
|----------------|-----------|--------------------|
| `/host/proc`   | `/proc`   | Read Only / rslave |
| `/host/sys`    | `/sys`    | Read Only / rslave |
| `/host`        | `/`       | Read Only / rslave |

Verify after start:

```sh
curl -s http://localhost:9100/metrics | head
```

You should see `node_cpu_seconds_total{...}` lines.

## Unraid API Prometheus plugin

Install via Community Applications: search for the plugin that exposes
Prometheus metrics for Unraid (the popular options are
`unraid-api` (forked-from ElectricBrainUK) and `unraid_exporter`).
Whichever you choose:

1. Note the port and `/metrics` path it exposes (commonly `:9999/metrics`).
2. Edit `prometheus/prometheus.yml` → `unraid_api` job → update `targets`.
3. Inspect `curl http://localhost:9999/metrics` once installed and align
   the metric names referenced in `grafana/dashboards/unraid.json` and
   `grafana/provisioning/alerting/rules.yml` (UnraidDiskTempHigh rule).
