# Running a node

There are three node *roles* in WaveLedger. You can run all three on
one machine, or split them across machines (recommended for production
for failure-domain separation).

| Role | Purpose | Inbound? | Persistent state? |
|---|---|---|---|
| [**Miner**](miner.md) | Builds blocks, mines them, propagates to peers | P2P only | Chain (SQLite) |
| [**Seed**](seed.md) | Maintains chain state, serves it to new peers, doesn't mine | P2P + optional RPC | Chain (SQLite) |
| [**Entropy aggregator**](entropy.md) | Pulls entropy from upstream sources, serves it to miners | HTTP `/api/random` | In-memory pool only |

A "full node" usually means miner + seed in one process. The dApp's
chat node is exactly that.

## Quick start

| You want to... | Read... |
|---|---|
| ...spin up the whole testnet on Fly in 15 minutes | [Self-hosting on Fly.io](fly.md) |
| ...run a miner on a VPS you already have | [Self-hosting on a VPS](vps.md) |
| ...join the public testnet from a laptop or hobbyist box | [Running a miner](miner.md) |
| ...mirror chain state without mining | [Running a seed node](seed.md) |
| ...provide entropy to your own miners | [Running the entropy aggregator](entropy.md) |
| ...understand every config knob | [Config reference](config.md) |
| ...keep it running | [Operational runbook](runbook.md) |
