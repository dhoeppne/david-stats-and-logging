# Promtail install — Hetzner (dns1, dns2)

Promtail runs as a binary + systemd unit. No Docker required.

> **Run as root.** Steps 1–3 write to `/usr/local/bin`, `/etc/promtail`, and
> `/etc/systemd/system`. Become root with `sudo -i` first, or prefix the
> privileged commands with `sudo` as shown. (Permission-denied errors there
> mean you're not root.)

## 1. Download the binary

Pick the latest 3.x release that matches your Loki version (currently `3.2.1`).

```sh
PROMTAIL_VERSION=3.2.1
curl -L -o /tmp/promtail.zip \
  "https://github.com/grafana/loki/releases/download/v${PROMTAIL_VERSION}/promtail-linux-amd64.zip"
unzip /tmp/promtail.zip -d /tmp
sudo install -m 0755 /tmp/promtail-linux-amd64 /usr/local/bin/promtail
rm -rf /tmp/promtail.zip /tmp/promtail-linux-amd64
```

## 2. Drop the config

```sh
sudo mkdir -p /etc/promtail /var/lib/promtail
sudo cp promtail-config.yml /etc/promtail/promtail-config.yml
# Edit /etc/promtail/promtail-config.yml: set the `host:` label under
# the technitium scrape job to dns1 or dns2.
```

## 3. Install + enable the systemd unit

```sh
sudo cp promtail.service /etc/systemd/system/promtail.service
sudo systemctl daemon-reload
sudo systemctl enable --now promtail
sudo systemctl status promtail
```

## 4. Verify

```sh
journalctl -u promtail -f
```

In Grafana → Explore → Loki, run:

```
{host="dns1"}
```

You should see recent journald lines.
