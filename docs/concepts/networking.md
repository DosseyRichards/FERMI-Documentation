# Networking

## Wire protocol

WaveLedger speaks a custom framed JSON protocol over TCP:

```
+--------+--------+----------+
| MAGIC  | LENGTH | PAYLOAD  |
| 4 byte | 4 byte | LENGTH B |
+--------+--------+----------+
```

| Field | Value |
|---|---|
| MAGIC | `b'WAVE'` (mainnet), `b'TWAV'` (testnet) |
| LENGTH | uint32, big-endian, payload size in bytes |
| PAYLOAD | UTF-8 JSON envelope |

Envelopes are signed (ML-DSA-87 by the node's identity key) and contain
a `type` + `payload`. The full message catalog lives in
`network/protocol.py`.

## Peer lifecycle

1. **Inbound or outbound TCP connect** to port 18333 (testnet) /
   8333 (mainnet — reserved).
2. **VERSION** exchange — protocol version, network magic, best
   height, node identity public key.
3. **VERACK** — both sides acknowledge.
4. **GETHEADERS / HEADERS / GETDATA / BLOCK** — sync (see below).
5. **PING / PONG** — every 30s, drop peer after 90s of no pong.

## Peer discovery

A node finds peers via four mechanisms:

| Mechanism | When | Source |
|---|---|---|
| **Bootstrap nodes** | Always tried first | `bootstrap_nodes` in config |
| **DNS seeds** | If bootstrap is empty | `TESTNET_DNS_SEED_HOSTS` const |
| **Hardcoded seeds** | If DNS fails | `TESTNET_SEED_NODES` const |
| **mDNS** | LAN only, opt-in | Multicast service discovery |
| **PEX (peer exchange)** | After first connection | Gossip from known peers |

UPnP is supported but off by default — when enabled, the node tries to
open the inbound P2P port through your home router automatically.

## Initial Block Download (IBD)

When a node starts behind on the chain:

1. Pick the peer with the highest reported `best_height`.
2. Request 2,000-header batches via `GETHEADERS` until you reach their
   tip.
3. Verify each header's PoW + difficulty.
4. Once headers are fully downloaded, request 500-block batches via
   `GETDATA` (full bodies).
5. Apply blocks in order, running `validate_received_block` on each.

During IBD the miner is paused. After IBD the chain is "synced" and
mining resumes.

Headers-first means a malicious peer can't waste bandwidth feeding you
bad blocks — you reject bad headers cheaply before requesting bodies.

## Transaction propagation

When a tx enters the mempool (either user-submitted via JSON-RPC or
received from a peer), the receiving node broadcasts an `INV` of type
`TX` with the tx_id to all its peers. Peers that don't already have
the tx respond with `GETDATA`, the originator sends the full tx, peers
validate + add to their own mempool + re-broadcast.

This is the standard "Bitcoin-style" gossip — flooding with INV
deduplication.

## Block propagation

When a block is mined or received:

1. Local node: `add_validated_block(block)`.
2. Send `INV` of type `BLOCK` to all peers (the new block's hash).
3. Peers that don't have it send `GETDATA`.
4. Originator sends the full block.
5. Peers validate via `validate_received_block`, add, re-propagate.

If two peers race to send you the same block, you fetch from whichever
GETDATA returns first and ignore the second.

## Fork resolution

See [Blocks and consensus → Fork resolution](blocks.md#fork-resolution).
The node keeps competing chains in memory; whichever extends first and
crosses the cumulative-work threshold becomes canonical. Reorgs run
through `reorg_to(new_tip)`, which:

1. Rewinds the local chain to the fork point, undoing balance/nonce
   updates.
2. Re-runs forward along the new branch.
3. Returns the orphaned chain's non-coinbase txs to the mempool.

Reorgs deeper than `MAX_REORG_DEPTH` (100 blocks) are rejected — the
node needs to be re-synced manually.

## Network parameters

| Name | Value | Notes |
|---|---|---|
| Mainnet network magic | `b'WAVE'` | |
| Testnet network magic | `b'TWAV'` | |
| Default mainnet port | 8333 | (reserved; not running) |
| Default testnet port | 18333 | |
| Genesis timestamp (mainnet) | `1735000000.0` (reserved) | |
| Genesis timestamp (testnet) | `1800000000.0` | Distinct chain ID |
| Max message size | 32 MB | After framing |
| Ping interval | 30s | |
| Peer drop after | 90s no pong | |
| Max peers | 25 | Per-node, configurable |

## P2P endpoint on the testnet

External miners can bootstrap to the public seed:

```
seed.waveledger.net:18333
```

Add this to your node's `bootstrap_nodes` list:

```toml
[discovery]
bootstrap_nodes = ["seed.waveledger.net:18333"]
```

The seed node accepts inbound P2P from anywhere on the internet over
TCP. Behind the scenes that's the public chat dApp's P2P listener
exposed via fly's dedicated IPv4.
