# Self-hosting on a VPS

The bare-VPS path uses systemd + Caddy on Ubuntu 22.04+. Lower-cost
than Fly (approximately $19/month for the full 3-node setup) at the
cost of operating systemd directly.

## Topology

| VPS | Role | Inbound ports |
|---|---|---|
| 1 | Entropy aggregator | 443 (via Caddy) |
| 2 | dApp + seed + miner #1 | 443 (chat UI), 18333 (P2P) |
| 3 | Miner #2 | 18333 (P2P) — optional public, or fully private |

Three $6/month droplets host the full testnet. The dApp node should be
the largest (consider $12/month) since it runs the most services.

## Entropy VPS

```bash
ssh root@entropy.example.com
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git /opt/waveledger
sudo bash /opt/waveledger/deploy/testnet/setup_entropy_vps.sh

# Put TLS in front:
ENTROPY_HOST=entropy.example.com \
  sudo -E bash /opt/waveledger/deploy/testnet/setup_entropy_proxy.sh
```

The setup script:

1. Installs `python3`, `pip`, `ufw`, `curl`, `jq`
2. Creates the `waveledger` system user
3. Clones the repo into `/opt/waveledger`
4. Opens port 8420 on UFW
5. Drops a systemd unit `waveledger-entropy.service` and starts it

The proxy script puts Caddy on 443 with auto-issued Let's Encrypt
certs, reverse-proxying to `127.0.0.1:8420`.

## dApp + seed + miner VPS

```bash
ssh root@chat.example.com
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git /opt/waveledger

# Install deps
sudo apt-get update && sudo apt-get install -y python3 python3-pip git ufw curl jq caddy
sudo pip3 install --break-system-packages -r /opt/waveledger/deploy/requirements-seed.txt

# Service user + dirs
sudo useradd --system --create-home --shell /bin/bash waveledger || true
sudo mkdir -p /var/lib/waveledger-testnet
sudo chown waveledger:waveledger /var/lib/waveledger-testnet /opt/waveledger

# Firewall
sudo ufw allow OpenSSH && sudo ufw allow 18333/tcp && sudo ufw allow 80/tcp \
       && sudo ufw allow 443/tcp && yes | sudo ufw enable

# Drop the systemd unit + miner config (see deploy/testnet/ for templates)
sudo cp /opt/waveledger/deploy/testnet/config-testnet.toml \
        /opt/waveledger/deploy/testnet/config-chat.toml

# Caddyfile
sudo tee /etc/caddy/Caddyfile <<CADDY
chat.example.com {
    encode gzip
    reverse_proxy 127.0.0.1:8081
}
CADDY

sudo systemctl daemon-reload
sudo systemctl enable --now waveledger-chat.service caddy
```

Set the admin password before first start:

```bash
sudo systemctl edit waveledger-chat.service
# Add under [Service]:
# Environment=WAVELEDGER_ADMIN_USER=admin
# Environment=WAVELEDGER_ADMIN_PASSWORD=your-strong-password
sudo systemctl restart waveledger-chat.service
```

## Miner #2 VPS

```bash
ssh root@miner2.example.com
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git /opt/waveledger
ENTROPY_HOST=entropy.example.com ENTROPY_PORT=443 \
  sudo -E bash /opt/waveledger/deploy/testnet/setup_miner_vps.sh
```

The script installs deps, opens port 18333, writes a
`config-miner.toml` that points at your entropy host, and drops a
`waveledger-testnet-miner.service` systemd unit.

## DNS

Same structure as the [Fly path](fly.md#dns), substituting VPS IPs:

| Type | Name | Value |
|---|---|---|
| A | `chat` | `<chat-vps-public-ipv4>` |
| AAAA | `chat` | `<chat-vps-public-ipv6>` |
| A | `entropy` | `<entropy-vps-public-ipv4>` |
| AAAA | `entropy` | `<entropy-vps-public-ipv6>` |
| A | `seed` | `<chat-vps-public-ipv4>` (same box) |
| AAAA | `seed` | `<chat-vps-public-ipv6>` |

## Smoke test

```bash
curl -sI https://chat.example.com | head -3
curl -s  https://entropy.example.com/api/health | jq .pool
nc -zv   seed.example.com 18333
```

## Operations

| Task | Command |
|---|---|
| Tail logs | `journalctl -u waveledger-chat -f` |
| Restart | `sudo systemctl restart waveledger-chat` |
| Reset chain | `sudo systemctl stop waveledger-chat && sudo rm -rf /var/lib/waveledger-testnet && sudo systemctl start waveledger-chat` |
| Update | `cd /opt/waveledger && sudo -u waveledger git pull && sudo systemctl restart waveledger-chat` |

For the full operational reference see [Runbook](runbook.md).
