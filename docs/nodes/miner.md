# Running a miner

A miner produces new blocks. To run one you need:

- A wallet (the miner creates one on first boot if you don't bring your own)
- Access to a registered [entropy source](../concepts/entropy.md)
- A reachable [seed node](seed.md) to bootstrap the chain from (or be one)
- 1+ CPU core, ~1 GB RAM, ~10 GB disk

## Install

Clone the repo:

```bash
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git ~/waveledger
cd ~/waveledger
pip install -r deploy/requirements-seed.txt
```

## Minimal config (testnet)

`~/waveledger.toml`:

```toml
[node]
data_dir   = "/var/lib/waveledger-testnet"
port       = 18333
host       = "0.0.0.0"
max_peers  = 25
testnet    = true
relay_only = false

[mining]
enabled       = true
miner_address = ""                              # auto-create if blank
qrng_host     = "entropy.waveledger.net"        # the public testnet aggregator
qrng_port     = 443

[discovery]
enable_mdns           = false                   # off on internet-exposed nodes
enable_upnp           = false
enable_dns_seeds      = true
enable_hardcoded_seeds= true
bootstrap_nodes       = ["seed.waveledger.net:18333"]

[dashboard]
enabled = true
port    = 8080
host    = "127.0.0.1"

[messenger]
enabled = false                                  # this is a miner, not a dApp host

[security]
require_auth = true
```

## Run

```bash
mkdir -p /var/lib/waveledger-testnet
python3 node.py --testnet --mine --config ~/waveledger.toml
```

Expected first-boot output (abridged):

```
Auto-created miner wallet
WaveLedger QRNG & Mining ASM BETA [TESTNET]
  Node ID:    489d0c3879454833...
  P2P Port:   18333
  Height:     0
  Mining:     ON
  Miner:      dc66f82c048f35144599737ed54ab702
  QRNG:       FIPS (no fallback)
  Sync:       idle
  Storage:    SQLite (WAL)
  Auth:       write-only
  Press Ctrl+C to stop
```

The miner connects to `seed.waveledger.net:18333`, syncs the chain to
the current tip, then starts producing blocks.

## CLI flags

Most things live in the config file; a few are command-line only or
override the config:

| Flag | Notes |
|---|---|
| `--mine` | Required to enable mining (overrides `[mining].enabled = false`) |
| `--testnet` | Use testnet genesis + network magic + port 18333 |
| `--miner-address <hex>` | Use a specific address instead of auto-creating |
| `--data-dir <path>` | Override `[node].data_dir` |
| `--config <path>` | Path to config toml |
| `--bootstrap host:port [host:port ...]` | Override `[discovery].bootstrap_nodes` |
| `--port <int>` | Override `[node].port` |
| `--no-mdns` | Disable LAN service discovery |
| `--no-upnp` | Disable UPnP NAT punchthrough |
| `--no-seeds` | Don't use DNS / hardcoded seed lists |
| `--no-dashboard` | Don't start the dashboard server |
| `--no-messenger` | Don't start the messenger server |
| `--dashboard-port <int>` | Port for the local dashboard (default 8080) |
| `--messenger-port <int>` | Port for the messenger (default 8081) |
| `--require-auth` | Require Bearer API key for all dashboard reads, not just writes |
| `--relay-only` | Don't mine, just maintain state + serve peers (= run as a seed) |

## Pointing at a different entropy source

The testnet's public aggregator (`entropy.waveledger.net`) is the
default. To bring your own:

1. Run your own [aggregator](entropy.md) somewhere reachable.
2. Set `[mining].qrng_host` + `qrng_port` to your aggregator's
   hostname + port.
3. The source ID exposed by your aggregator (the `source` field in
   its `/api/health` response) must be in the chain's trust list,
   or your blocks will be rejected. The trust list is governance-controlled.

In v1 the trust list is hardcoded; in v2 it'll be a Fourier contract
modifiable by the BDFL via Timelock.

## Bring your own miner address

If you have an existing wallet you want to receive rewards into:

```bash
python3 node.py --testnet --mine --miner-address $YOUR_ADDR --config ~/waveledger.toml
```

The node will refuse to start if it doesn't have the private key for
`$YOUR_ADDR`. To register a wallet you control, drop its JSON into
`{data_dir}/wallets/` (see [`wallet_cli.py`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/wallet_cli.py)).

## Systemd

Recommended for long-running production miners:

```ini
[Unit]
Description=WaveLedger Testnet Miner
After=network-online.target

[Service]
Type=simple
User=waveledger
Group=waveledger
WorkingDirectory=/opt/waveledger
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /opt/waveledger/node.py --testnet --mine \
    --config /opt/waveledger/deploy/testnet/config-miner.toml \
    --data-dir /var/lib/waveledger-testnet
Restart=on-failure
RestartSec=10s

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/waveledger-testnet /opt/waveledger

[Install]
WantedBy=multi-user.target
```

Save as `/etc/systemd/system/waveledger-miner.service`,
`systemctl daemon-reload && systemctl enable --now waveledger-miner`.
Logs via `journalctl -u waveledger-miner -f`.

## Health checks

| Check | How |
|---|---|
| Mining is alive | `curl http://127.0.0.1:8080/api/mining` — `running: true` |
| Synced | `curl http://127.0.0.1:8080/api/status` — `sync_state: "synced"` |
| Peers connected | `curl http://127.0.0.1:8080/api/peers` — `peer_count > 0` |
| Entropy reachable | `curl http://127.0.0.1:8080/api/qrng` — `reachable: true` |
| Recent block | `curl http://127.0.0.1:8080/api/chain` — `tip_timestamp` < 30s ago |

A monitoring loop polling these every 30s catches most failure modes.

## Common issues

| Symptom | Likely cause |
|---|---|
| `QRNG: not reachable` | `qrng_host` wrong, or aggregator is down |
| No peers connecting | Inbound port not open at firewall / NAT; UPnP off and no port-forward |
| Mining starts then halts | Difficulty spiral (see [Blocks](../concepts/blocks.md#difficulty-adjustment)); reset chain or wait |
| Block age growing | Mempool stuck — check entropy is alive; check tx propagation |
| `insufficient WAVE for deploy fee` (from playground) | Miner wallet has no balance yet; wait for first coinbase |
