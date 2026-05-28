# Post-quantum cryptography

WaveLedger uses NIST-standardized post-quantum primitives for every
piece of in-band cryptography. There is no classical-crypto fallback.

## Why

Shor's algorithm, run on a sufficiently large quantum computer, breaks
all widely-deployed asymmetric primitives in polynomial time. That
includes:

- **RSA** — used in TLS, code signing, and most certificate authorities
- **ECDSA** — used in Bitcoin, Ethereum, and most other chains
- **Ed25519** — used in many newer chains and SSH
- **Diffie-Hellman** (classical + elliptic-curve variants)

Symmetric primitives (AES, SHA-3) are weakened by Grover's algorithm,
but only quadratically — a 256-bit key becomes "128-bit-equivalent",
which is still infeasible to brute-force. NIST classifies AES-256 and
SHA3-512 as PQ Category 5.

A chain that needs to keep proving past transactions valid 40 years
from now cannot afford to use cryptography that's broken by a quantum
computer 20 years from now.

## The primitive set

| Use | Primitive | NIST standard |
|---|---|---|
| Transaction signatures | ML-DSA-87 | FIPS 204 |
| Key encapsulation (planned: encrypted handoff, p2p session keys) | ML-KEM-1024 | FIPS 203 |
| Hashes (block hash, merkle root, tx id, address derivation) | SHA3-512 | FIPS 202 |
| Wallet backup encryption | AES-256-GCM + Argon2id | SP 800-208 (Cat 5) |

ML-DSA, ML-KEM, and SHA3 are all NIST-standardized as of 2024-25.
There are no proprietary primitives in the chain.

## Sizes

PQ keys + signatures are bigger than classical equivalents. Plan for it.

| Quantity | Bytes | vs Ed25519 |
|---|---|---|
| ML-DSA-87 public key | 2,592 | 81× larger |
| ML-DSA-87 signature | 4,627 | 72× larger |
| ML-KEM-1024 public key | 1,568 | — |
| ML-KEM-1024 ciphertext | 1,568 | — |

A transaction with one signature is ~5 KB on the wire. Block size is
not artificially capped; the only effective limit is
`MAX_TRANSACTIONS_PER_BLOCK = 100` (network parameter, raisable on
mainnet).

## What we don't claim

- **Symmetric crypto** is not post-quantum *replaced* — it's
  post-quantum *safe* by virtue of being symmetric. AES-256 has 128 bits
  of quantum security; SHA3-512 has 256 bits of quantum collision
  resistance.

- **Side-channel resistance** is not in scope. ML-DSA implementations
  have published timing-attack mitigations; we use the reference
  Python implementation (`dilithium-py`), which is not hardened.
  Hardware-backed signing is on the roadmap.

- **Long-term forward secrecy of historical data** depends on the
  ciphers being unbroken. If ML-DSA is broken in 2055, every tx ever
  signed becomes forgeable in retrospect. Same caveat applies to every
  signature scheme ever invented.

## Crypto agility

The chain has a runtime **signature scheme registry** — see
[`verify_sig(scheme, ...)`](https://fourier.fermi.world/language/cross-contract/)
in Fourier and [`SLH-DSA-SHA2-128s`](../reference/crypto.md#slh-dsa)
as the second registered scheme. Contracts can choose which scheme to
require, and new schemes can be added without a hard fork.

The intent is that if any one scheme is broken or deprecated, the chain
keeps running on the others while we migrate.

## Further reading

- [NIST IR 8413](https://csrc.nist.gov/pubs/ir/8413/upd1/final) — PQC
  standardization process report
- [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final) — ML-DSA spec
- [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final) — ML-KEM spec
- [FIPS 202](https://csrc.nist.gov/pubs/fips/202/final) — SHA-3 spec
