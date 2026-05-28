# Config reference

WaveLedger reads configuration from a TOML file passed via
`--config <path>`. Anything not in the file falls back to either CLI
flags or compile-time defaults from `core/constants.py`.

The full schema is below. All sections optional; common defaults are
sensible.

## `[node]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `data_dir` | string | `.waveledger` | Where chain.db + identity + wallets live |
| `port` | int | 8333 (mainnet) / 18333 (testnet) | P2P listen port |
| `host` | string | `0.0.0.0` | P2P bind address |
| `max_peers` | int | 25 | Cap on concurrent peer connections |
| `testnet` | bool | false | Use testnet genesis + magic + port |
| `relay_only` | bool | false | Run as a relay/seed (no mining) |

## `[mining]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | false | Master switch. `--mine` CLI flag also enables. |
| `miner_address` | string | `""` | Hex address to receive coinbase; auto-create wallet if blank |
| `qrng_host` | string | `127.0.0.1` | Entropy aggregator hostname |
| `qrng_port` | int | 8420 | Entropy aggregator port |

## `[discovery]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `enable_mdns` | bool | true | LAN service discovery |
| `enable_upnp` | bool | false | NAT punchthrough |
| `enable_dns_seeds` | bool | true | Resolve hardcoded DNS seed hosts |
| `enable_hardcoded_seeds` | bool | true | Hardcoded peer addresses |
| `bootstrap_nodes` | list[string] | `[]` | `host:port` strings; tried first |

## `[dashboard]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | true | Run the monitoring server |
| `port` | int | 8080 | Bind port |
| `host` | string | `0.0.0.0` | Bind host. Use `127.0.0.1` for loopback-only. |

## `[messenger]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | true | Run the chat dApp server |
| `port` | int | 8081 | Bind port |
| `host` | string | `0.0.0.0` | Bind host |

## `[security]`

| Key | Type | Default | Notes |
|---|---|---|---|
| `require_auth` | bool | false | If true, GET endpoints on the dashboard also require a Bearer key |

## Env vars

A handful of settings are pulled from env vars (cannot be in the TOML):

| Env var | Used by | Default |
|---|---|---|
| `WAVELEDGER_ADMIN_USER` | Messenger admin auth | `admin` |
| `WAVELEDGER_ADMIN_PASSWORD` | Messenger admin auth | `change-me-via-env` (insecure!) |
| `PYTHONUNBUFFERED` | systemd / Docker | recommended `1` |
| `WAVELEDGER_ROLE` | Fly container entrypoint | `entropy` \| `chat` \| `miner` |

## Examples

### Public testnet miner

```toml
[node]
data_dir   = "/var/lib/waveledger-testnet"
port       = 18333
host       = "0.0.0.0"
testnet    = true

[mining]
enabled   = true
qrng_host = "entropy.waveledger.net"
qrng_port = 443

[discovery]
enable_mdns           = false
enable_upnp           = false
enable_dns_seeds      = true
enable_hardcoded_seeds= true
bootstrap_nodes       = ["seed.waveledger.net:18333"]

[dashboard]
enabled = true
port    = 8080
host    = "127.0.0.1"

[messenger]
enabled = false

[security]
require_auth = true
```

### Public testnet seed (no mining)

```toml
[node]
data_dir   = "/var/lib/waveledger-testnet"
port       = 18333
host       = "0.0.0.0"
testnet    = true
relay_only = true

[mining]
enabled = false

[discovery]
bootstrap_nodes       = ["seed.waveledger.net:18333"]
enable_dns_seeds      = true
enable_hardcoded_seeds= true

[dashboard]
enabled = true
host    = "127.0.0.1"

[messenger]
enabled = false

[security]
require_auth = true
```

### Local dev (everything on one box, mDNS for LAN miners)

```toml
[node]
data_dir = ".waveledger-dev"
port     = 18333
testnet  = true

[mining]
enabled = true

[discovery]
enable_mdns           = true
enable_upnp           = true
bootstrap_nodes       = []

[dashboard]
enabled = true
host    = "0.0.0.0"           # expose locally for browsing

[messenger]
enabled = true
host    = "0.0.0.0"

[security]
require_auth = false           # ok for local dev only
```
