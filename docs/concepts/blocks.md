# Blocks and consensus

## Block structure

Every block in WaveLedger has the following fields:

| Field | Type | Notes |
|---|---|---|
| `index` | uint | Block height. Genesis = 0. |
| `timestamp` | float | Unix epoch (seconds, float). |
| `transactions` | list | Ordered; first tx is always the coinbase. |
| `previous_hash` | hex string | SHA3-512 of the previous block. |
| `merkle_root` | hex string | SHA3-512 Merkle root of the tx list. |
| `nonce` | uint | PoW nonce. |
| `hash` | hex string | SHA3-512 of the block header (computed last). |
| `difficulty` | uint | Number of required leading hex zeros in `hash`. |
| `miner` | string (20-byte hex) | The miner's address; receives the coinbase. |
| `quantum_signature` | object \| null | QRNG attestation envelope (see [Entropy](entropy.md)). |
| `quantum_verified` | bool | True when `quantum_signature` is present + valid. |

Genesis is the only block with `difficulty = 0` and no PoW.

## Consensus rules

A block is valid iff:

1. `previous_hash` matches the tip's hash.
2. `merkle_root` matches a recomputed Merkle root of `transactions`.
3. `hash` matches a recomputed SHA3-512 of the header.
4. `hash` has at least `difficulty` leading hex zeros.
5. `difficulty` matches what the chain expects at this height.
6. `quantum_signature` is present and verifies against the block's
   `quantum_seed` (see [Entropy](entropy.md)).
7. The coinbase tx (`transactions[0]`) has `sender = "mining_reward"`,
   `recipient = block.miner`, and `amount == subsidy + collected_fees`
   where `subsidy` is the schedule-derived block reward at this height.
8. Every non-coinbase tx passes the standard tx validation rules (see
   [Transaction format](../reference/tx.md)).
9. Adding the coinbase would not push `total_supply` above
   [`MAX_SUPPLY`](../reference/parameters.md#max_supply).

## Fork resolution

The canonical chain is the one with **highest cumulative work**, defined
as $\sum 2^{\text{difficulty}_i}$ summed over every block.

If two blocks arrive at the same height with the same parent, the node
keeps both until one chain extends. The longer-work chain wins; the
other branch is reorged out and its non-coinbase txs are put back into
the mempool.

Reorgs deeper than `MAX_REORG_DEPTH` (100 blocks) are rejected — a
node that has been offline long enough to see a 100-block reorg must
re-sync from peers, not from disk.

## Difficulty adjustment

Difficulty is recalculated every `DIFFICULTY_ADJUSTMENT_INTERVAL` (10)
blocks, at block boundaries only (not on every block). At each boundary:

- If the last interval took less than half the expected time → `+1`
- If the last interval took more than 2× the expected time → `-1`
- Otherwise hold

Mainnet bounds: `[MIN_DIFFICULTY, MAX_DIFFICULTY]` = `[2, 8]`.
Testnet bounds: `[2, 4]` — `performance-1x` Fly VMs cap out at difficulty 4.

Expected time = `BLOCK_TIME_TARGET * (DIFFICULTY_ADJUSTMENT_INTERVAL - 1)` =
60s × 9 = 540s mainnet, 45s testnet.

## Coinbase

The first transaction in every block is a "coinbase" — synthetic, no
signature, sender = `"mining_reward"`. Its amount is the schedule
subsidy plus the sum of fees from all other txs in the block.

The subsidy is:
$$ \text{subsidy}(h) = \frac{\text{INITIAL\_BLOCK\_REWARD}}{2^{\lfloor h / \text{HALVING\_INTERVAL} \rfloor}} $$

with current values `INITIAL_BLOCK_REWARD = 5 WAVE`,
`HALVING_INTERVAL = 2,100,000` blocks. See
[Tokenomics](../reference/tokenomics.md) for the full curve.

## Mempool eligibility

To enter the mempool a tx must:

- Have a valid ML-DSA-87 signature
- Have fee `≥ MEMPOOL_MIN_FEE` (0.0001 WAVE)
- Have amount `≥ MIN_TRANSACTION_AMOUNT` (0.01 WAVE), **unless** it is
  a contract deploy/call (`recipient == "contract"` or `data.type in
  ("deploy", "call")`), which are exempt from the dust limit
- Pass the balance check: `sender_balance - sum(pending_outflows) ≥
  amount + fee`
- Not be a duplicate of a tx already pending
- Not exceed `MEMPOOL_MAX_PER_ADDRESS` pending entries (50) from one sender

Mempool capacity is `MEMPOOL_MAX_SIZE` (5,000) with fee-priority eviction.
Replace-by-fee is supported with a 10% minimum bump on the same
`(sender, nonce)` pair.
