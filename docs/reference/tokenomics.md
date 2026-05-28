# Genesis and tokenomics

The complete emission and supply specification.

## Headline numbers

| Property | Value |
|---|---|
| Symbol | WAVE |
| Max supply | 21,000,000 WAVE |
| Initial block reward | 5 WAVE |
| Halving interval | 2,100,000 blocks |
| Block time target | 60 s (mainnet) / 5 s (testnet) |
| Expected lifetime | ~4 years per halving epoch |
| Genesis premine | 1,000,000 WAVE to the foundation address |
| Genesis address | `34378b1ba5be9d0999acd60be3a8a1f1` |

All values are constants in `core/constants.py::BlockchainConstants`.

## Emission curve

Block subsidy at height `h`:

```text
epoch       = h // HALVING_INTERVAL
subsidy(h)  = INITIAL_BLOCK_REWARD / (2 ** epoch)
```

The subsidy halves every `HALVING_INTERVAL = 2,100,000` blocks. At ~60s
per block that is approximately 4 years per epoch — matched to
Bitcoin's halving cadence.

Why 5 WAVE × 2.1M blocks instead of 50 WAVE × 210k? Block time is 10×
faster than Bitcoin's, so per-block reward and halving interval are
both scaled by 1/10× to preserve a four-year halving cadence on the
same 21M cap. The geometric series still sums to 21M:

```text
sum(2 × INITIAL_BLOCK_REWARD × HALVING_INTERVAL)
  = 2 × 5 × 2,100,000
  = 21,000,000
```

## Genesis block (block 0) { #genesis }

Genesis is constructed deterministically by every node on first boot.

| Field | Value |
|---|---|
| `index` | 0 |
| `previous_hash` | `"0" * 64` |
| `difficulty` | 0 |
| `nonce` | 0 |
| `quantum_signature` | `null` |
| `transactions[0]` | The genesis distribution tx (see below) |
| `timestamp` | `GENESIS_TIMESTAMP = 1700000000.0` (mainnet) <br> `TESTNET_GENESIS_TIMESTAMP = 1800000000.0` (testnet) |

### Genesis distribution tx

```json
{
  "sender":    "genesis",
  "recipient": "34378b1ba5be9d0999acd60be3a8a1f1",
  "amount":    1000000.0,
  "fee":       0.0,
  "data":      null,
  "nonce":     -1
}
```

The sender `"genesis"` is a sentinel string accepted only in block 0.
The 1M WAVE premine lands at `GENESIS_FOUNDATION_ADDRESS` — a real
ML-DSA-87-derived foundation address whose private key is held offline
by the BDFL.

Every node computes the same genesis hash because every input
(timestamp, distribution amount, foundation address) is a compile-time
constant.

## Coinbase (per block)

Each non-genesis block's first transaction is a coinbase:

```json
{
  "sender":    "mining_reward",
  "recipient": "<miner address>",
  "amount":    "<subsidy(h) + sum(tx_fees)>",
  "fee":       0.0,
  "signature": "reward_signature"
}
```

- `signature` is the literal string `"reward_signature"` — validated
  structurally, not via ML-DSA.
- Adding the coinbase must not cause total supply to exceed
  `MAX_SUPPLY`. If it would, the block is rejected.

## Supply check

```python
total_supply = GENESIS_DISTRIBUTION + sum(subsidy(h) for h in 1..tip)
assert total_supply <= MAX_SUPPLY
```

Enforced on every block apply. Past max supply, blocks can still be
mined but the coinbase must be zero (fees only) — otherwise the block
is rejected.

## Fees

| Component | Default | Constant |
|---|---|---|
| Transaction fee floor | 0.0001 WAVE | `MEMPOOL_MIN_FEE` |
| Default tx fee | 0.01 WAVE | `DEFAULT_TRANSACTION_FEE` |
| Min tx amount | 0.01 WAVE | `MIN_TRANSACTION_AMOUNT` |
| Max tx amount | 1,000,000 WAVE | `MAX_TRANSACTION_AMOUNT` |

Fees are collected entirely by the block miner via the coinbase
`amount`. No protocol-level burn.

## Bohms (gas unit)

Smart contract gas is denominated in **bohms**, a sub-unit of WAVE:

```text
1 WAVE = 10^18 bohms
GAS_PRICE = 10^9 bohms per gas unit
```

A contract tx's `fee` (in WAVE) must cover `gas_limit × GAS_PRICE`
bohms or it is rejected before mempool admission with
`InsufficientFee`. Unused gas is **not refunded** in v1.

Example: a 100,000-gas call requires
`100,000 × 10^9 = 10^14` bohms = `0.0001` WAVE, exactly at the floor.

## Staking (Phase 2, disabled)

Hybrid PoS + Proof-of-Hardware design is reserved in code but disabled:

| Property | Value |
|---|---|
| `STAKING_ENABLED` | `False` |
| `STAKING_MIN_AMOUNT` | 100 WAVE |
| `STAKING_LOCK_PERIOD` | 1000 blocks |
| `STAKING_REWARD_SHARE` | 10% of block reward |
| `STAKING_DIFFICULTY_BONUS` | 0.5 leading-zeros reduction |

Stakers will earn the right to mine but **must still produce a valid
QRNG attestation**; staking is Sybil resistance, not a replacement for
hardware-backed PoW.

## Mainnet vs testnet

| Parameter | Mainnet | Testnet |
|---|---|---|
| Block time | 60 s | 5 s |
| Initial difficulty | 4 | 2 |
| Max difficulty | 8 | 4 |
| Network magic | `b"WAVE"` | `b"TWAV"` |
| Default port | 8333 | 18333 |
| Genesis timestamp | 1700000000 | 1800000000 |

The testnet difficulty cap exists because single-CPU performance-1x VMs
spiral past difficulty 4 into multi-minute blocks; capping at 4 hex
zeros keeps blocks at ~1s on shared CPUs.
