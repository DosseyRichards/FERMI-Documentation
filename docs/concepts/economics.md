# Gas, fees, and economics

## Units

WAVE is the chain's native asset. The chain internally tracks balances
in **Bohms** — the smallest unit, where:

$$ 1 \text{ WAVE} = 10^{18} \text{ Bohms} $$

(Named after Niels Bohr. Bohms are to WAVE what wei is to ETH or
satoshis are to BTC.)

All on-the-wire amounts in JSON APIs use floats in WAVE units (e.g.
`0.001` is 1,000,000,000,000,000 Bohms). The contract VM works in
Bohms internally; conversion happens at the engine boundary.

## Block reward

| | Value |
|---|---|
| Initial subsidy | 5 WAVE / block |
| Halving interval | 2,100,000 blocks (~4 yrs at 60s mainnet block time) |
| Asymptotic max supply | 21,000,000 WAVE |
| Genesis premine | 1,000,000 WAVE (foundation) |

Halving math:

$$ \text{subsidy}(h) = \frac{5}{2^{\lfloor h / 2{,}100{,}000 \rfloor}} \text{ WAVE} $$

The geometric series sums to:

$$ \text{premine} + 2 \cdot \text{INITIAL\_REWARD} \cdot \text{HALVING\_INTERVAL} = 10^6 + 2 \cdot 5 \cdot 2{,}100{,}000 = 21{,}000{,}000 $$

The chain enforces this cap in `validate_received_block` — any block
whose coinbase would push `total_supply` past 21M is rejected, even if
the rest of it is valid. See [Tokenomics](../reference/tokenomics.md)
for the full emission curve.

## Transaction fees

Fees are paid in WAVE, set by the sender, collected by the miner of
the including block (added to the coinbase).

| | Value | Notes |
|---|---|---|
| `MEMPOOL_MIN_FEE` | 0.0001 WAVE | Floor — txs below are rejected from mempool |
| `DEFAULT_TRANSACTION_FEE` | 0.01 WAVE | What the wallet UI uses by default |
| Contract deploy / call fee | 0.001 WAVE | Charged from sender; `gas_limit * gas_price` |
| RBF bump | +10% over the existing fee | To replace a same-nonce tx |

Recommended fees are returned by the dashboard's `GET /api/fee`
endpoint (per [fee estimation](../api/dashboard.md#fee-estimation)),
based on current mempool depth and recent block contents.

## Gas

Contract execution is metered in gas. Conversion:

| Constant | Value |
|---|---|
| `DEFAULT_GAS_LIMIT` | 1,000,000 |
| `GAS_PRICE` | 1,000,000,000 (1 Gwei-equivalent, in Bohms/gas) |
| Default per-tx fee | `1,000,000 * 1,000,000,000 / 10^18` = **0.001 WAVE** |

The default gas limit is what the [Playground API](../api/playground.md)
sets when you don't specify one. Specific operations have published
costs (see [Reference: Crypto + gas](../reference/crypto.md)):

| Op | Gas |
|---|---|
| Base block tx | 21,000 |
| Contract deploy (per byte of bytecode) | 200 |
| `SLOAD` | 800 |
| `SSTORE` (new slot) | 20,000 |
| `SSTORE` (existing slot, non-zero) | 5,000 |
| `SHA3` (per word) | 6 + 6 per word over input |
| `CALL` / `DELEGATECALL` / `STATICCALL` | 700 + cost-of-callee |
| ML-DSA verify precompile | 50,000 |
| ML-KEM decapsulate precompile | 25,000 |
| Memory expansion | Quadratic (anti-DoS) |

Gas left over after execution is **not refunded** in v1.

## Transaction sizing

For wallet UX, a "typical" tx fits these envelopes:

| Tx kind | Wire size (est.) | Default fee | Typical gas |
|---|---|---|---|
| Plain transfer | ~5.5 KB | 0.01 WAVE | 21,000 |
| Chat message (memo ≤ 200B) | ~6 KB | 0.001 WAVE | 21,000 |
| Contract deploy (small, ~500B bytecode) | ~7 KB | 0.001 WAVE | 100,000–300,000 |
| Contract call (simple) | ~5.5 KB | 0.001 WAVE | 21,000–50,000 |

Most of the size budget goes to the ML-DSA-87 signature (4,627 bytes)
and public key (2,592 bytes if not yet known to the verifier).

## Mining economics

A block at mainnet's 60s target with 5 WAVE subsidy + occasional fees
produces ~7,200 WAVE/day to the global miner set. With one miner, that
miner gets all of it; with N miners, each gets `~5 * (their_hashrate /
total_hashrate)` WAVE per block in expectation.

Fee revenue is currently negligible vs subsidy — the chain has plenty
of empty blocks. As usage grows, fees should grow with it.

After the first halving (~4 years), subsidy drops to 2.5 WAVE/block;
after the second, 1.25; etc. By halving ~30, subsidy is below the
dust limit and fees become the only meaningful miner revenue.
