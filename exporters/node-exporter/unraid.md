# node_exporter on Unraid

The NAS's own node_exporter is already defined in the top-level
`docker-compose.yml` as the `node_exporter` service. It uses **host
networking** and bind-mounts `/proc`, `/sys`, and `/` so it can see real
host metrics rather than container-namespaced ones.

> **Mount propagation note.** The Compose file mounts these read-only with
> the *default* (private) propagation — **not** `rslave`. Unraid mounts `/`
> as a private mount, and Docker refuses to create a slave bind mount from a
> private source, failing with:
>
> ```
> Error response from daemon: path / is mounted on / but it is not a shared or slave mount
> ```
>
> Plain `:ro` avoids this and CPU/RAM/network metrics work unchanged. The
> only thing you lose is node_exporter's view of *nested* filesystems
> (`/mnt/user`, `/mnt/cache`, individual array disks) — those are separate
> mount points that aren't recursively visible without `rslave`. On this
> stack that's fine: array disk usage, health, and temperatures come from
> the **Unraid API plugin** (`unraid_api` scrape job), not node_exporter.
>
> **If you do want node_exporter to see every array/cache filesystem**, make
> the host root a shared mount *before* starting the container, then restore
> the `rslave` flags in `docker-compose.yml`:
>
> ```sh
> mount --make-rshared /
> ```
>
> This isn't persistent across reboots. To make it stick on Unraid, add the
> same line to `/boot/config/go` (which runs at every boot). Only do this if
> you specifically need per-filesystem metrics from node_exporter.

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

| Container path | Host path | Mode      |
|----------------|-----------|-----------|
| `/host/proc`   | `/proc`   | Read Only |
| `/host/sys`    | `/sys`    | Read Only |
| `/host`        | `/`       | Read Only |

(Use Read Only, **not** the slave/rslave propagation — see the mount
propagation note above for why.)

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
