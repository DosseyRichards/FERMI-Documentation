# Crypto primitives

WaveLedger is post-quantum from the ground up. No classical
cryptography appears anywhere in the consensus path: every signature,
key encapsulation, and hash is NIST-standardized PQC or SHA-3.

## Algorithm summary

| Standard | Algorithm | Use |
|---|---|---|
| NIST FIPS 203 | ML-KEM-1024 | Wallet key encapsulation |
| NIST FIPS 204 | ML-DSA-87 | Transaction + P2P message signatures |
| NIST FIPS 205 | SLH-DSA-SHA2-128s | Optional contract-level signatures (precompile) |
| NIST FIPS 202 | SHA3-512 | Block hashing, Merkle trees, address derivation |
| NIST FIPS 202 | SHA3-256 | VM-side hashing, contract address derivation, calldata hashing |

All three signature primitives target NIST security level 5
(equivalent classical strength: AES-256). Implementations: `kyber-py`
for ML-KEM, `dilithium-py` for ML-DSA, `pqcrypto` (PQClean) for
SLH-DSA.

## ML-KEM-1024 (FIPS 203)

Key encapsulation mechanism used for wallet creation and (eventually)
session-key establishment between nodes.

| Property | Value |
|---|---|
| Parameter set | ML-KEM-1024 |
| Public key size | 1,568 bytes |
| Private key size | 3,168 bytes |
| Ciphertext size | 1,568 bytes |
| Shared secret | 32 bytes |
| Security level | NIST 5 |

Active use: the public-key material whose SHA3-512 derives a wallet
address. Reserved for future P2P channel encryption.

## ML-DSA-87 (FIPS 204)

Digital signature scheme used for every transaction and every P2P
message.

| Property | Value |
|---|---|
| Parameter set | ML-DSA-87 |
| Public key size | 2,592 bytes |
| Private key size | 4,896 bytes |
| Signature size | ~4,627 bytes (approximate; variable) |
| Security level | NIST 5 |

A transaction's `signature` field is the ML-DSA-87 signature over
SHA3-512 of the canonical signed-form dict (see
[Transaction format](tx.md)).

A P2P message's signature appears alongside the payload in the frame
metadata; receivers verify against the sender's advertised verifying
key before accepting.

## SLH-DSA-SHA2-128s (FIPS 205) { #slh-dsa }

Hash-based signature scheme, available as a VM precompile at
`0x...03`. Slower and larger than ML-DSA but with the most conservative
PQ security assumptions (only hash-function preimage resistance).

| Property | Value |
|---|---|
| Parameter set | SLH-DSA-SHA2-128s |
| Public key size | 32 bytes |
| Signature size | ~7,856 bytes |
| Security level | NIST 1 |
| Status | Optional — falls back to "always reject" if `pqcrypto` is not installed |

Contracts use it via the precompile call; it is **not** part of the
consensus path.

## SHA3-512 (FIPS 202)

Hash function for everything on the chain layer.

| Property | Value |
|---|---|
| Output | 512 bits / 64 bytes / 128 hex chars |
| Used for | Block hash, Merkle root, transaction id, wallet address derivation |

```python
import hashlib
digest = hashlib.sha3_512(payload).hexdigest()    # 128-char hex
```

Examples in code:

- Block hash: `sha3_512(json.dumps(header, sort_keys=True))`
- Merkle leaf: `sha3_512(tx.to_dict_canonical())`
- Merkle internal: `sha3_512(left || right)`
- Wallet address: `sha3_512(pub_key_bytes).hexdigest()[:32]`

## SHA3-256 (FIPS 202)

Hash function for VM-layer operations.

| Property | Value |
|---|---|
| Output | 256 bits / 32 bytes / 64 hex chars |
| Used for | VM `SHA3` opcode, contract address derivation, wallet→VM address bridge |

```python
import hashlib
digest = hashlib.sha3_256(payload).digest()   # 32 bytes
```

Examples in code:

- Contract address: `sha3_256(deployer || nonce.be_32)[:20]`
- Wallet→VM bridge: `sha3_256(wallet_address_utf8)[:20]`
- VM `SHA3` opcode: pops `(offset, length)`, hashes memory window,
  pushes 256-bit digest

## QRNG entropy

Block attestations carry 64 bytes (512 bits) of entropy drawn from the
configured entropy source (drand on testnet, or a QRNG device planned
for mainnet). The seed half (32 bytes) feeds the block header; the
proof half is committed by SHA3-512 hash.

| Test | Bound | Constant |
|---|---|---|
| Monobit ratio | 0.40 — 0.60 | `ATTESTATION_MONOBIT_MIN/MAX` |
| Fano factor | 0.5 — 1.5 | `ATTESTATION_FANO_MIN/MAX` |
| Chi-squared p-value | ≥ 0.001 | `ATTESTATION_CHI_SQUARED_P_MIN` |
| Attestation timestamp window | ±60 s from block timestamp | (validator) |

A block whose entropy fails any of these bounds is rejected.

## Precompiles

The VM short-circuits CALLs targeting reserved low addresses and runs
native code.

| Address | Name | Cost | Input | Output |
|---|---|---|---|---|
| `0x0..01` | SHA3-512 | 60 + 12 × ⌈len/32⌉ | arbitrary bytes | 64-byte digest |
| `0x0..02` | ML-DSA verify | 30,000 | `pk_len(u32) || msg_len(u32) || sig_len(u32) || pk || msg || sig` | 32-byte word (1 or 0) |
| `0x0..03` | SLH-DSA verify | 50,000 | same layout | 32-byte word (1 or 0) |

Signature verify precompiles share a common calldata layout: three
4-byte big-endian length prefixes followed by the concatenated
public key, message, and signature. The precompile parses lengths
defensively — malformed calldata returns 0 (failure), not a revert.

Implementation: `vm/precompiles.py`. Adding a new PQC scheme: pick the
next free low address, register it in `GAS`, and (for Fourier
support) add it to `fourier/codegen.py::CRYPTO_SCHEMES`.

## Wallet encryption (off-chain)

Wallets at rest are encrypted with AES-256-GCM. The encryption key is
derived from the user's password via Argon2id (the constants in
`AuthConstants` mention PBKDF2 — that is a legacy field name; the
implementation in `auth/wallet_encryption.py` is Argon2id-backed).

| Property | Value |
|---|---|
| Cipher | AES-256-GCM |
| Key derivation | Argon2id |
| Salt length | 32 bytes |
| GCM nonce | 12 bytes (NIST SP 800-38D standard) |
| Authenticated data | None |

The wallet keystore is per-node and never leaves the host.

## What WaveLedger does **not** use

- No ECDSA, EdDSA, RSA, or other classical signatures.
- No Curve25519, P-256, or other classical key exchange.
- No SHA-256 or SHA-512 (SHA-3 only).
- No BLS, no Schnorr, no Ring signatures.

A primitive not listed on this page is not used in the consensus path.
TLS from a browser to the dashboard is the one exception (TLS remains
classical for browser compatibility); on-chain artifacts are fully
post-quantum.
