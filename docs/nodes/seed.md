# Running a seed node

A seed node serves the chain to new peers without mining. Use cases:

- Running infrastructure (block explorers, indexers, dApps) without
  reward incentives
- Geographic redundancy (more peer-discovery surface area)
- Validation-only audit nodes

## Config

Nearly identical to a miner; the difference is `--relay-only` or
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

The node syncs the chain, accepts inbound P2P, and gossips blocks and
txs, but never builds blocks. No `--mine`, no entropy fetch.

## Why run one

- **dApp operators** benefit from a local node so user requests do not
  cross the open internet for every chain read.
- **Indexer operators** (explorer, analytics, alerting) gain reads
  served from a local SQLite.
- **Network redundancy** for the testnet without the operational
  complexity of mining.

A seed node consumes approximately 1/4 the CPU of a miner with an
identical disk footprint.

## Becoming a public seed

To be discoverable by other miners as a bootstrap option, publish the
node's `host:port` so other configs can pick it up:

1. Add the address to `TESTNET_SEED_NODES` in `core/constants.py` (PRs
   welcome).
2. Add an `A`/`AAAA` record under one of the
   `TESTNET_DNS_SEED_HOSTS` domains.
3. Or announce the address in Discord, Twitter, or the README; most
   operators add bootstrap nodes manually.

Inbound P2P must be reachable: firewall open on port 18333, no NAT
double-translation, no aggressive ISP filtering.

## Validation-only mode

For a node that downloads and validates every block without serving
anything (audit-only):

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
serves nothing back. Suitable for verifying chain integrity from cold.
