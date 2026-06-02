# Running a node

WaveLedger defines three node *roles*. All three can run on a single
machine, or be split across machines (recommended for production to
achieve failure-domain separation).

| Role | Purpose | Inbound? | Persistent state? |
|---|---|---|---|
| [**Miner**](miner.md) | Builds blocks, mines them, propagates to peers | P2P only | Chain (SQLite) |
| [**Seed**](seed.md) | Maintains chain state, serves it to new peers, doesn't mine | P2P + optional RPC | Chain (SQLite) |
| [**Entropy aggregator**](entropy.md) | Pulls entropy from upstream sources, serves it to miners | HTTP `/api/random` | In-memory pool only |

A "full node" denotes a combined miner + seed in one process. The
dApp's chat node runs in this configuration.

## Quick start

| Goal | Document |
|---|---|
| Deploy the full testnet on Fly in 15 minutes | [Self-hosting on Fly.io](fly.md) |
| Run a miner on an existing VPS | [Self-hosting on a VPS](vps.md) |
| Join the public testnet from a laptop or hobbyist host | [Running a miner](miner.md) |
| Mirror chain state without mining | [Running a seed node](seed.md) |
| Provide entropy to a private set of miners | [Running the entropy aggregator](entropy.md) |
| Reference every configuration option | [Config reference](config.md) |
| Operate the node in production | [Operational runbook](runbook.md) |
