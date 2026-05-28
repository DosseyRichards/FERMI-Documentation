# Address format

WaveLedger has **two address spaces**: chain-layer wallet addresses
(16 bytes) and VM-layer addresses (20 bytes). They are connected by a
deterministic bridge function. Knowing which space you are in matters
whenever you cross the boundary between a transaction and a contract.

## Chain-layer (wallet) addresses

| Property | Value |
|---|---|
| Length | 16 bytes (32 hex characters) |
| Prefix | None — no `0x`, no checksum |
| Encoding | Lowercase hex string |
| Derived from | First 32 hex of `SHA3-512(public_key_material)` |
| Used in | `tx.sender`, `tx.recipient`, `block.miner`, balances, mempool keys |

Derivation (`crypto/kyber_crypto.py`):

```python
address_input = pub_key_bytes
# (or quantum_entropy_seed || pub_key_bytes for quantum-entropy wallets)
address = hashlib.sha3_512(address_input).hexdigest()[:32]
```

Example: `34378b1ba5be9d0999acd60be3a8a1f1` (the genesis foundation
address).

## VM-layer addresses

| Property | Value |
|---|---|
| Length | 20 bytes (40 hex characters) |
| Used in | All contract code, storage keys, log envelopes, precompile dispatch |
| Storage | `vm/state.py` indexes accounts by 20-byte `bytes` |

Why 20 bytes? It matches EVM ergonomics so existing tooling and
calldata conventions translate directly.

## Bridging: wallet → VM

When a wallet address enters the VM (as `caller()` / `origin()`, or as
the `sender` of a deploy/call), it is hashed to 20 bytes:

```python
# core/contract_engine.py::_to_addr_bytes
SHA3-256(wallet_address_utf8)[:20]
```

This is a pure function — the same wallet address always maps to the
same VM address. It is **not** invertible: given a VM address, you
cannot recover the wallet address.

If a `str` is already 40 hex chars (optionally `0x`-prefixed), it is
treated as a literal 20-byte address (identity). Anything else passes
through the SHA3-256 mapping.

## Contract addresses

Contract addresses are pure VM-layer. Derived at deploy time:

```python
# vm/deploy.py::contract_address
contract_address = SHA3-256(deployer_20b || nonce.to_bytes(32, "big"))[:20]
```

- `deployer_20b` is the deployer's VM-layer address (already bridged).
- `nonce` is the deployer's VM-level account nonce *before* the deploy.
- Deterministic: two nodes processing the same deploy tx produce the
  same address. No collision recovery — a duplicate address raises
  `DeploymentCollision` (defensive; near-impossible with monotonic
  nonces).

See [Receipt format](receipt.md) for how the deploy address surfaces
on-chain.

## Sentinel strings

Some address slots in the protocol carry strings that are not real
addresses. They are matched literally and never derived from any key.

| String | Where | Meaning |
|---|---|---|
| `"mining_reward"` | `tx.sender` | Coinbase transaction sender |
| `"genesis"` | `tx.sender` | Genesis distribution tx (block 0 only) |
| `"contract"` | `tx.recipient` | Contract deploy or call target |
| `"0" * 32` | `tx.recipient` | Burn address (convention) |

Real wallet addresses never collide with these because they are
deterministically derived from a 1568-byte ML-KEM public key and the
sentinels are short ASCII.

## Precompile addresses

The VM reserves the low end of the 20-byte address space for native
operations dispatched without bytecode lookup:

| Hex | Name | Purpose |
|---|---|---|
| `0x0..01` | SHA3-512 | 64-byte hash precompile |
| `0x0..02` | ML-DSA verify | FIPS 204 signature verification |
| `0x0..03` | SLH-DSA verify | FIPS 205 signature verification (if installed) |

See [Crypto primitives](crypto.md) for calling conventions.

## Display conventions

The dashboard and explorer truncate long hex for readability:

| Source | Display |
|---|---|
| Wallet address | `34378b1ba5be...8a1f1` (first 12 / last 8) |
| Block hash | First 32 chars |

Truncation is purely for the UI. APIs always return the full address.

## Validation rules

- Wallet addresses: 32 lowercase hex chars, no separators, no prefix.
- VM addresses: 20 bytes / 40 hex chars (case-insensitive on input;
  lowercase on output).
- A field expecting an address that receives a sentinel string is
  accepted in the slots listed above and rejected everywhere else.
- A `tx.sender` that is not a sentinel must hash, under SHA3-512, to
  produce a value whose first 32 hex chars equal the claimed address —
  enforced indirectly by signature verification against the wallet's
  public key.
