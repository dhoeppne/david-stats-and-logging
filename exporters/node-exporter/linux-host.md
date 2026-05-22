# node_exporter on a Linux host (NPM box, dns1, dns2)

Install as a binary + systemd unit. No Docker required.

> **Run as root.** Every step here writes to root-owned locations
> (`/etc/passwd`, `/usr/local/bin`, `/etc/systemd/system`). Either become
> root once with `sudo -i` and run the commands as written, or prefix each
> command with `sudo`. If you see `useradd: Permission denied` /
> `cannot lock /etc/passwd`, that's the symptom — you're not root.

## 1. Create a system user

```sh
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
```

## 2. Download the binary

```sh
NE_VERSION=1.8.2
ARCH=linux-amd64
cd /tmp
curl -L -o ne.tar.gz \
  "https://github.com/prometheus/node_exporter/releases/download/v${NE_VERSION}/node_exporter-${NE_VERSION}.${ARCH}.tar.gz"
tar -xzf ne.tar.gz
sudo install -m 0755 node_exporter-${NE_VERSION}.${ARCH}/node_exporter /usr/local/bin/node_exporter
rm -rf ne.tar.gz node_exporter-${NE_VERSION}.${ARCH}
```

## 3. systemd unit

Create `/etc/systemd/system/node_exporter.service` (e.g.
`sudo nano /etc/systemd/system/node_exporter.service`) with:

```ini
[Unit]
Description=Prometheus node_exporter
After=network-online.target
Wants=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 4. Enable + start

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s http://localhost:9100/metrics | head
```

## 5. Firewall

Allow inbound `9100/tcp` from the NAS:

- **NPM box (LAN)**: from `10.0.0.17` (NAS LAN IP).
- **dns1 / dns2 (Hetzner)**: from the NAS's Tailscale IP (e.g. `100.64.0.10/32`).
  If using `ufw`:

  ```sh
  sudo ufw allow from 100.105.48.123 to any port 9100 proto tcp
  ```

  Or rely on the Tailscale ACL to gate `:9100`.

## 6. Tailscale Prometheus endpoint (Hetzner + NPM)

Each host's tailscaled also exposes its own Prometheus metrics. Enable it
once per host:

```sh
sudo mkdir -p /etc/systemd/system/tailscaled.service.d
sudo tee /etc/systemd/system/tailscaled.service.d/prometheus.conf >/dev/null <<'EOF'
[Service]
Environment=TS_TAILSCALED_EXTRA_ARGS=--prom-listen-addr=:5252
EOF
sudo systemctl daemon-reload
sudo systemctl restart tailscaled
curl -s http://localhost:5252/metrics | head
```

The exact flag name (`--prom-listen-addr` vs `--prometheus-listen-addr`)
varies across Tailscale versions — check `tailscaled --help` if the
above fails and adjust.

On Unraid, set the equivalent in the Tailscale plugin's "Extra Args"
field rather than via systemd.
