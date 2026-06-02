# Network parameters

All protocol-level constants. Values are sourced from
`core/constants.py`.

## Consensus

| Parameter | Mainnet | Testnet | Constant |
|---|---|---|---|
| Block time target | 60 s | 5 s | `BLOCK_TIME_TARGET` / `TESTNET_BLOCK_TIME_TARGET` |
| Initial difficulty | 4 | 2 | `INITIAL_DIFFICULTY` / `TESTNET_INITIAL_DIFFICULTY` |
| Min difficulty | 2 | 2 | `MIN_DIFFICULTY` |
| Max difficulty | 8 | 4 | `MAX_DIFFICULTY` / `TESTNET_MAX_DIFFICULTY` |
| Difficulty adjustment interval | 10 blocks | 10 blocks | `DIFFICULTY_ADJUSTMENT_INTERVAL` |
| Max reorg depth | 100 blocks | 100 blocks | `MAX_REORG_DEPTH` |

The PoW target is `difficulty` leading hex zeros on the SHA3-512 block
hash. Adjustment looks at the time spanned by the last 10 blocks and
moves difficulty up or down by 1, clamped to `[MIN, MAX]`.

## Economics { #economics }

<a id="max_supply"></a>

| Parameter | Value | Constant |
|---|---|---|
| Max supply | 21,000,000 WAVE | `MAX_SUPPLY` |
| Initial block reward | 5 WAVE | `INITIAL_BLOCK_REWARD` |
| Halving interval | 2,100,000 blocks | `HALVING_INTERVAL` |
| Genesis distribution | 1,000,000 WAVE | `GENESIS_DISTRIBUTION` |
| Default tx fee | 0.01 WAVE | `DEFAULT_TRANSACTION_FEE` |
| Min tx amount | 0.01 WAVE | `MIN_TRANSACTION_AMOUNT` |
| Max tx amount | 1,000,000 WAVE | `MAX_TRANSACTION_AMOUNT` |
| Max txs per block | 100 | `MAX_TRANSACTIONS_PER_BLOCK` |

See [Tokenomics](tokenomics.md) for the emission curve.

## Gas

| Parameter | Value | Constant |
|---|---|---|
| Initial wallet bohms | 500.0 | `INITIAL_WALLET_BOHMS` |
| Small tx gas | 10.0 (amount ≤ 1000) | `SMALL_TX_GAS` |
| Medium tx gas | 15.0 (amount ≤ 10000) | `MEDIUM_TX_GAS` |
| Large tx gas | 25.0 (amount > 10000) | `LARGE_TX_GAS` |
| Default contract gas limit | 1,000,000 units | `DEFAULT_GAS_LIMIT` |
| Gas price | 10^9 bohms / unit | `GAS_PRICE` |
| Bohms per WAVE | 10^18 | (denomination) |

## Mempool

| Parameter | Value | Constant |
|---|---|---|
| Max size | 5,000 txs | `MEMPOOL_MAX_SIZE` |
| Min fee | 0.0001 WAVE | `MEMPOOL_MIN_FEE` |
| TTL | 72 hours (259,200 s) | `MEMPOOL_TX_TTL` |
| Max pending per address | 25 | `MEMPOOL_MAX_PER_ADDRESS` |

Eviction policy: when full, lowest-fee txs are dropped first.

RBF (Replace-By-Fee): a tx with the same `(sender, nonce)` as a pending
tx and a fee ≥ 1.1× the existing fee replaces it.

## P2P networking

| Parameter | Value | Constant |
|---|---|---|
| Default port | 8333 (mainnet) / 18333 (testnet) | `DEFAULT_PORT` / `TESTNET_DEFAULT_PORT` |
| Max peers | 25 | `MAX_PEERS` |
| Min peers | 3 | `MIN_PEERS` |
| Protocol version | 1 | `PROTOCOL_VERSION` |
| Network magic | `b"WAVE"` / `b"TWAV"` | `NETWORK_MAGIC` / `TESTNET_NETWORK_MAGIC` |
| Max frame size | 4 MB | `MAX_FRAME_SIZE` |
| Peer timeout | 60 s | `PEER_TIMEOUT` |
| Handshake timeout | 10 s | `HANDSHAKE_TIMEOUT` |
| Ping interval | 30 s | `PING_INTERVAL` |
| Pong timeout | 60 s | `PONG_TIMEOUT` |

Frame format: `MAGIC(4) + LENGTH(4) + JSON_PAYLOAD`. Every message is
ML-DSA-87 signed.

## Sync

| Parameter | Value | Constant |
|---|---|---|
| Max blocks per request | 500 | `MAX_BLOCKS_PER_REQUEST` |
| Max headers per request | 2,000 | `MAX_HEADERS_PER_REQUEST` |
| Sync timeout | 300 s | `SYNC_TIMEOUT` |

Headers-first IBD: pull headers in 2,000-batch increments to validate
the chain shape, then pull full blocks in 500-batch increments.

## Discovery

| Parameter | Value | Constant |
|---|---|---|
| mDNS service type | `_waveledger._tcp.local.` | `MDNS_SERVICE_TYPE` |
| Discovery interval | 60 s | `DISCOVERY_INTERVAL` |
| External IP check interval | 1 hour | `EXTERNAL_IP_CHECK_INTERVAL` |
| Max addresses per ADDR msg | 1,000 | `MAX_ADDR_PER_MSG` |
| INV cache TTL | 10 minutes | `INV_CACHE_TTL` |
| Max INV entries per msg | 50,000 | `MAX_INV_PER_MSG` |

### Seeds

| Mainnet seed nodes | Mainnet DNS seeds |
|---|---|
| `seed1.waveledger.net:8333` | `seed.waveledger.net` |
| `seed2.waveledger.net:8333` | `seed2.waveledger.net` |

| Testnet seed nodes | Testnet DNS seeds |
|---|---|
| `seed1-testnet.waveledger.net:18333` | `seed1-testnet.waveledger.net` |
| `seed2-testnet.waveledger.net:18333` | `seed2-testnet.waveledger.net` |

## Service bits

Advertised in the `VERSION` handshake to signal node capabilities.

| Bit | Value | Meaning |
|---|---|---|
| `NODE_NETWORK` | 1 | Full node — relays blocks and transactions |
| `NODE_MINER` | 2 | Has a working QRNG attestation source |
| `NODE_ARCHIVE` | 4 | Stores full chain history (no pruning) |
| `NODE_REACHABLE` | 8 | Accepts inbound connections (passed reachability probe) |

Bits are bitmasked; one node may advertise any combination.

## QRNG attestation bounds

Per-block health checks applied by every validator before accepting a
new block's `quantum_signature`. See `core/constants.py` for the
authoritative values.

| Test | Bound |
|---|---|
| Monobit ratio (proportion of 1-bits in seed) | 0.40 ≤ p ≤ 0.60 |
| Fano factor (variance/mean of ADC reads) | 0.5 ≤ F ≤ 1.5 |
| Chi-squared p-value | ≥ 0.001 |

A block whose attestation fails any of these is rejected.

## Crypto

| Parameter | Value | Constant |
|---|---|---|
| ML-KEM parameter set | ML-KEM-1024 | `MLKEM_PARAMETER_SET` |
| ML-KEM private key size | 3,168 bytes | `MLKEM_PRIVATE_KEY_SIZE` |
| ML-KEM public key size | 1,568 bytes | `MLKEM_PUBLIC_KEY_SIZE` |
| ML-DSA parameter set | ML-DSA-87 | `MLDSA_PARAMETER_SET` |
| ML-DSA signature size | ~4,627 bytes | `MLDSA_SIGNATURE_SIZE` |
| Hash algorithm | SHA-3-512 | `HASH_ALGORITHM` |
| Hash output | 512 bits / 128 hex chars | `HASH_OUTPUT_BITS` |
| NIST security level | 5 | `NIST_SECURITY_LEVEL` |
| Quantum entropy per attestation | 256 bits | `QUANTUM_ENTROPY_BITS` |

See [Crypto primitives](crypto.md) for algorithm details.

## Auth

| Parameter | Value | Constant |
|---|---|---|
| PBKDF2 iterations | 100,000 | `PBKDF2_ITERATIONS` |
| PBKDF2 key length | 32 bytes (AES-256) | `PBKDF2_KEY_LENGTH` |
| AES mode | GCM | `AES_MODE` |
| Salt length | 32 bytes | `SALT_LENGTH` |
| Nonce length | 12 bytes (GCM standard) | `NONCE_LENGTH` |

The keystore uses Argon2id (not PBKDF2) for password derivation; the
`PBKDF2_*` constants are legacy names retained for backwards
compatibility with older wallet files. See
`auth/wallet_encryption.py`.

## Genesis

| Parameter | Mainnet | Testnet |
|---|---|---|
| Genesis timestamp | 1700000000.0 | 1800000000.0 |
| Genesis foundation address | `34378b1ba5be9d0999acd60be3a8a1f1` | (same) |
