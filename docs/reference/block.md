# Block format

## JSON shape

```json
{
  "index": 12345,
  "timestamp": 1780002999.123,
  "transactions": [ ... ],
  "previous_hash": "0000fc99...",
  "merkle_root": "abcd...",
  "nonce": 42031,
  "hash": "0000ab12...",
  "difficulty": 4,
  "miner": "dc66f82c048f35144599737ed54ab702",
  "quantum_signature": { ... },
  "quantum_verified": true
}
```

## Field reference

| Field | Type | Notes |
|---|---|---|
| `index` | int | Block height; genesis is 0 |
| `timestamp` | float | Unix epoch, set by the miner |
| `transactions` | list[Transaction] | First entry is always the coinbase |
| `previous_hash` | string (hex) | SHA3-512 of the previous block |
| `merkle_root` | string (hex) | SHA3-512 Merkle root of the tx list |
| `nonce` | int | The PoW solution |
| `hash` | string (hex) | SHA3-512 of the block header (computed last) |
| `difficulty` | int | Required leading hex zeros in `hash` |
| `miner` | string | Coinbase recipient. Real address (not a sentinel). |
| `quantum_signature` | object | QRNG attestation envelope (see below) |
| `quantum_verified` | bool | True if `quantum_signature` is present + valid |

## Header

The "block header" — what gets hashed for PoW + chain integrity — is:

```python
header = {
    'index': block.index,
    'timestamp': block.timestamp,
    'previous_hash': block.previous_hash,
    'merkle_root': block.merkle_root,
    'difficulty': block.difficulty,
    'miner': block.miner,
    'nonce': block.nonce,
    'quantum_seed': block.quantum_signature['qrng_attestation']['entropy_seed'],
}
hash = sha3_512(json.dumps(header, sort_keys=True))
```

`hash` must have `difficulty` leading hex zeros.

## Merkle root

The Merkle root is computed over the tx list:

```python
def merkle_root(txs):
    leaves = [sha3_512(tx.to_dict_canonical()) for tx in txs]
    if not leaves: return sha3_512(b'')
    while len(leaves) > 1:
        if len(leaves) % 2: leaves.append(leaves[-1])
        leaves = [sha3_512(a + b) for a, b in zip(leaves[::2], leaves[1::2])]
    return leaves[0].hex()
```

Standard pairwise SHA3-512 reduction, with odd-leaf doubling (Bitcoin
convention). Coinbase is always leaf 0.

## QRNG attestation envelope

The `quantum_signature` block has this shape (from
`mining/attestation.py::generate_attestation`):

```json
{
  "qrng_attestation": {
    "version":       1,
    "entropy_seed":  "<32-byte hex>",
    "entropy_proof": "<32-byte hex>",
    "commitment":    "<sha3-512 of entropy_proof, hex>",
    "source":        "aggregator:drand-default",
    "device_id":     "qrng_<node-id-prefix>",
    "proof_type":    "qrng_hardware_attestation_v1",
    "health": {
      "monobit_ratio": 0.503,
      "fano_factor":   1.02,
      "pool_size":     65536,
      "timestamp":     1780002999.0
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `version` | yes | Currently `1` |
| `entropy_seed` | yes | 32-byte hex, mixed into the block header for PoW |
| `entropy_proof` | yes | 32-byte hex, the revealed half whose hash matches `commitment` |
| `commitment` | yes | `SHA3-512(entropy_proof)`, hex |
| `source` | optional | One of the IDs registered in [`SourceRegistry`](../concepts/entropy.md#source-registry). Absent on v0 blocks. |
| `device_id` | yes | Free-form node identifier |
| `proof_type` | yes | Versioned tag; future schemes increment this |
| `health` | yes | Source-specific stats; the four shown are enforced bounds |

`signature` and `public_key` are **not** produced by the current
implementation — there is no source-side signing path yet. They are
reserved for a future revision (`proof_type` will increment when it
ships) where the entropy aggregator signs its responses and the
validator verifies them against a per-source registered public key.

Validation rules — enforced by `mining/attestation.py::verify_attestation`:

1. `entropy_seed` and `entropy_proof` are both exactly 32 bytes
2. `commitment` matches `sha3_512(entropy_proof)`
3. `health.monobit_ratio` is within `[0.40, 0.60]`
4. `health.fano_factor` is within `[0.5, 1.5]`
5. Chi-squared p-value on `entropy_proof` ≥ `0.001`
6. Monobit ratio of `entropy_proof` is within `[0.35, 0.65]`
7. `source` (when present) is in the
   [SourceRegistry trust list](../concepts/entropy.md#source-registry).
   Aggregator-composed IDs (`aggregator:a+b+c`) are allowed iff every
   component is itself registered.

Back-compat: v0 blocks lacking a `source` field still verify. Per-source
signature verification under a registered public key is on the roadmap;
when it ships, `proof_type` will increment from
`qrng_hardware_attestation_v1` to `_v2` and the envelope will gain
outer `signature` and `public_key` fields.

## Genesis block

Genesis (`index=0`) is special:

- `previous_hash = "0" * 64`
- `difficulty = 0`
- `nonce = 0`
- `quantum_signature = null` (no attestation needed)
- `transactions = [genesis_tx]` where `genesis_tx` sends
  `GENESIS_DISTRIBUTION` (1M WAVE) from sentinel `"genesis"` to
  `GENESIS_FOUNDATION_ADDRESS`
- `hash` is deterministic from the above

Every node computes its own genesis block on first boot; they all
agree because the inputs are deterministic constants.

## Validation rules

A block is accepted iff:

1. `previous_hash` matches the current tip hash
2. Recomputed merkle root matches `merkle_root`
3. Recomputed header hash matches `hash`
4. Header hash has at least `difficulty` leading hex zeros
5. `difficulty` matches the schedule for this height
6. `quantum_signature` validates (rules above)
7. Coinbase tx is well-formed: `sender = "mining_reward"`,
   `recipient = miner`, `amount = subsidy(height) + sum(fees)`
8. Every other tx passes mempool admission rules
9. Adding the coinbase would not exceed `MAX_SUPPLY`

Failed validation → block rejected, peer is not punished (could be a
race), block is not added to the chain.
