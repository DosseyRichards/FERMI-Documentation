# Running a seed node

A seed node serves the chain to new peers but doesn't mine. Useful for:

- Running infrastructure (block explorers, indexers, dApps) without
  reward incentives
- Geographic redundancy (more peer-discovery surface area)
- Validation-only audit nodes

## Config

Almost identical to a miner; the difference is `--relay-only` or
`[mining].enabled = false`:

```toml
[node]
data_dir = "/var/lib/waveledger-testnet"
port     = 18333
host     = "0.0.0.0"
testnet  = true
relay_only = true                                # don't mine

[mining]
enabled = false                                  # explicit

[discovery]
enable_dns_seeds       = true
enable_hardcoded_seeds = true
bootstrap_nodes        = ["seed.waveledger.net:18333"]

[dashboard]
enabled = true
port    = 8080
host    = "127.0.0.1"

[messenger]
enabled = false

[security]
require_auth = true
```

## Run

```bash
python3 node.py --testnet --relay-only --config ~/seed.toml
```

The node syncs the chain, accepts inbound P2P, gossips blocks + txs,
but never builds blocks itself. No `--mine`, no entropy fetch.

## Why run one

- **You operate a dApp** and want a local node so user requests don't
  cross the open internet for every chain read.
- **You operate an indexer** (explorer, analytics, alerting) and want
  reads off a local SQLite.
- **You want network redundancy** for the testnet without committing
  to the operational complexity of mining.

A seed node uses ~1/4 the CPU of a miner and the same disk footprint.

## Becoming a public seed

If you want to be discoverable by other miners as a bootstrap option,
publish your `host:port` somewhere their config can pick up:

1. Add your address to `TESTNET_SEED_NODES` in `core/constants.py` (PR
   welcome).
2. Add an `A`/`AAAA` record under one of the
   `TESTNET_DNS_SEED_HOSTS` domains.
3. Or just announce your address in Discord / Twitter / the README;
   most operators will add bootstrap nodes manually anyway.

Inbound P2P must be reachable: firewall open on port 18333, no NAT
double-translation, no aggressive ISP filtering.

## Validation-only mode

If you want a node that downloads + validates every block but never
serves anything (audit-only):

```toml
[node]
host = "127.0.0.1"                               # localhost only
max_peers = 5
```

```bash
python3 node.py --testnet --relay-only --no-mdns --no-upnp --no-seeds \
    --bootstrap seed.waveledger.net:18333 \
    --config ~/audit.toml
```

The node connects only to the seed, syncs, validates every block, and
serves nothing back. Useful for verifying chain integrity from cold.
