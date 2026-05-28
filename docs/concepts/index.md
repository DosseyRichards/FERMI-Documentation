# Concepts

The chain from first principles. Each page is short enough to read in
one sitting; they build on each other in this order:

1. [**Post-quantum cryptography**](pqc.md) — why ML-DSA / ML-KEM / SHA3,
   what they replace, what they don't.
2. [**Blocks and consensus**](blocks.md) — block structure, PoW + QRNG
   attestation, fork resolution.
3. [**QRNG and entropy**](entropy.md) — why mining requires verifiable
   entropy, what counts as a valid source, how the aggregator works.
4. [**Addresses and wallets**](addresses.md) — what an address is,
   how it's derived from an ML-DSA public key.
5. [**Gas, fees, and economics**](economics.md) — Bohms, the gas table,
   the emission schedule.
6. [**Mining**](mining.md) — the full mining loop, sync gates, block
   templating, difficulty adjustment.
7. [**Networking**](networking.md) — peer discovery, the P2P wire
   protocol, IBD, transaction propagation.
