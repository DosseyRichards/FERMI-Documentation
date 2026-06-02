# Glossary

Terms specific to WaveLedger, the Fourier language, and the entropy
attestation protocol. EVM-flavored terms with EVM-identical meaning
(stack, memory, calldata) are omitted.

---

**Address (chain)**
:   A 16-byte (32-hex-char) hash derived from a wallet's ML-KEM-1024
    public key via `SHA3-512(pub_key)[:32]`. Used in transactions,
    balances, and the mempool. See [Address format](address.md).

**Address (VM)**
:   A 20-byte (40-hex-char) identifier used inside contract execution.
    Derived from a chain address via the bridge function
    `SHA3-256(chain_addr_utf8)[:20]` at the moment a contract is
    called. Contract addresses live in this space too. See
    [Address format](address.md).

**Application-Specific Mining (ASM)**
:   The design property of WaveLedger that every block must contain a
    verifiable entropy attestation in addition to a PoW solution. The
    attestation source (drand on testnet; QRNG hardware in the mainnet
    design) is advertised on every block and validated by every full
    node.

**Attestation**
:   The `quantum_signature` envelope on a block. Contains the entropy
    seed, a SHA3-512 commitment, the source identifier, and a signature
    proving the source generated this entropy. See
    [Block format](block.md#qrng-attestation-envelope).

**Bohm**
:   The smallest unit of WAVE; `1 WAVE = 10^18 bohms`. Smart contract
    gas is denominated in bohms. Named after the physicist **David
    Bohm**, in keeping with the Fermi-themed naming.

**Coinbase**
:   The first transaction in every non-genesis block. Sender is the
    sentinel string `"mining_reward"`. Recipient is the miner's wallet
    address. Amount is `subsidy(height) + sum(fees)`.

**Commitment**
:   In the QRNG attestation: `SHA3-512(entropy_seed || canonical(health))`.
    The block-level entropy commitment signed by the attestation source.
    Allows validators to check that the seed matches the signed
    commitment without requiring the full pre-hash payload.

**Contract address**
:   A VM-layer address derived deterministically at deploy time from
    `SHA3-256(deployer_20b || nonce.be_32)[:20]`. Two nodes processing
    the same deploy tx land on the same address. See
    [Receipt format](receipt.md#address-derivation-deploy).

**Difficulty**
:   The number of leading hex zeros required on the SHA3-512 block hash
    to satisfy PoW. Adjusts every 10 blocks within `[MIN_DIFFICULTY,
    MAX_DIFFICULTY]` based on observed block times.

**drand**
:   The federated randomness beacon used by the testnet as the
    attestation source. Wraps a threshold BLS signature scheme run by
    several independent operators. Out of scope for the mainnet design;
    used on testnet because it is publicly verifiable without QRNG
    hardware.

**Entropy aggregator**
:   `qrng_aggregator_service.py`. An optional in-tree service that
    XOR-combines entropy from N upstream providers and exposes the
    result as a REST endpoint on port 8420. Allows a node to treat
    several independent sources as one logical entropy source.

**Fano factor**
:   `variance / mean` of a count process. For shot noise from a
    photodiode the expected value is ~1.0 (Poisson). One of three
    statistical checks applied to attestation entropy.

**Fourier**
:   WaveLedger's smart-contract language. Rust-flavored syntax,
    compiles directly to VM bytecode. Documented separately at
    [fourier.fermi.world](https://fourier.fermi.world).

**Genesis foundation address**
:   `34378b1ba5be9d0999acd60be3a8a1f1`. Receives the 1M WAVE genesis
    distribution. Real ML-DSA-87-derived address; private key held
    offline by the foundation operator. Predates any sentinel string.

**Headers-first IBD**
:   Initial Block Download strategy. Nodes pull block headers in batches
    of 2,000 to validate chain shape before fetching full bodies in
    batches of 500. Cheaper than naive download for catching up over
    long ranges.

**Mainnet**
:   The intended production network. Block time 60s, difficulty
    `[2, 8]`, network magic `b"WAVE"`. Not yet active.

**Merkle root**
:   `SHA3-512` Merkle root over the block's transaction list, with
    odd-leaf doubling (Bitcoin convention). Coinbase is always leaf 0.

**ML-DSA-87**
:   NIST FIPS 204 signature scheme used for every transaction signature
    and every P2P message. NIST security level 5.

**ML-KEM-1024**
:   NIST FIPS 203 key encapsulation mechanism. Public-key half is what
    wallet addresses are derived from. Reserved for future P2P channel
    encryption.

**Monobit ratio**
:   Proportion of 1-bits in a binary sequence. For a true random source
    the expectation is 0.5. Attestation acceptance window: `[0.40,
    0.60]`.

**Nonce (PoW)**
:   The 64-bit integer the miner increments in the block header search.
    The final value satisfies the difficulty target.

**Nonce (account)**
:   Per-(sender, account) monotonically increasing integer. Used both
    for replay protection on chain-layer txs and for deriving contract
    deploy addresses on the VM layer.

**Precompile**
:   A native VM operation dispatched by address. SHA3-512, ML-DSA
    verify, and SLH-DSA verify are precompiles in the low end of the
    20-byte address space. See [Crypto primitives](crypto.md#precompiles).

**Proof type / source**
:   Versioned fields on the attestation envelope identifying which
    entropy source produced this block's entropy and which validation
    rules apply. Allows the chain to swap sources (drand → QRNG →
    multi-source) without forking.

**Quantum verified**
:   Boolean flag on a block or tx, set to `true` only after the relevant
    cryptographic check has passed. On a tx it means the ML-DSA-87
    signature was verified. On a block it means the QRNG attestation
    was validated.

**RBF (Replace-by-Fee)**
:   A mempool entry can be replaced by a tx with the same `(sender,
    nonce)` and at least 1.1× the original fee. The original is
    evicted.

**Receipt**
:   Structured result of a contract transaction (deploy or call).
    Includes success flag, gas used, return data, logs, optional error.
    See [Receipt format](receipt.md).

**Relay-only node**
:   A node that validates and forwards blocks and transactions but does
    not mine. Does not need a QRNG attestation source. Run on VPSes as
    public bootstrap peers.

**Sentinel**
:   A reserved string used in an address slot where a real address
    would be expected. The four sentinels are `"mining_reward"`,
    `"genesis"`, `"contract"`, and `"0" * 32`. See
    [Address format](address.md#sentinel-strings).

**SLH-DSA-SHA2-128s**
:   NIST FIPS 205 stateless hash-based signature. Optional VM
    precompile. Falls back to "always reject" if the `pqcrypto` package
    is not installed.

**Testnet**
:   The active network. Block time 5s, difficulty capped at 4, network
    magic `b"TWAV"`. Uses drand as the attestation source.

**WAVE**
:   The native token. 21M maximum supply, 18-decimal subdivision into
    bohms.
