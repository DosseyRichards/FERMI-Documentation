# Signing transactions

This page documents exactly what a valid WaveLedger transaction
envelope looks like, what gets signed, and how to produce a valid
signature outside the dApp.

## The envelope

A `Transaction` object on the wire has these fields:

```json
{
  "sender": "34378b1ba5be9d0999acd60be3a8a1f1",
  "recipient": "39848b500426d8f10a3c2b4e6d7f8901",
  "amount": 1.0,
  "timestamp": 1780002999.123,
  "transaction_id": "f5c43575f6b520c1...",
  "signature": "<hex>",
  "fee": 0.001,
  "data": { "memo": "hello" },
  "nonce": -1,
  "quantum_verified": true
}
```

| Field | Type | Notes |
|---|---|---|
| `sender` | string (40 hex) | Sender address |
| `recipient` | string | Recipient address, OR `"contract"`, OR `"mining_reward"`, OR `"genesis"` |
| `amount` | float | WAVE |
| `timestamp` | float | Unix epoch (seconds, sub-second precision) |
| `transaction_id` | string (hex) | Deterministic id, computed from the canonical envelope |
| `signature` | string (hex) | ML-DSA-87 signature over the canonical signed-data dict |
| `fee` | float | WAVE; must be ≥ `MEMPOOL_MIN_FEE` (0.0001) |
| `data` | object \| null | Optional payload — chat memos, contract deploy/call envelopes |
| `nonce` | int | `-1` for legacy/auto, else explicit replay-protection nonce |
| `quantum_verified` | bool | Set true after the signature verifies; ignored on submit |

## What gets signed

The signature is over a **canonical 5-field dict** — not the whole envelope:

```python
signed_data = {
    'sender': tx.sender,
    'recipient': tx.recipient,
    'amount': tx.amount,
    'timestamp': tx.timestamp,
    'fee': tx.fee,
}
```

Specifically excluded from the signature: `data`, `nonce`,
`transaction_id`, `quantum_verified`.

The reason: `data` is large (full contract bytecode for deploys) and
re-signing it on every change makes hand-signing impractical. The
integrity of `data` is enforced by `transaction_id`, which hashes the
*entire* envelope including `data`'s effect — modifying `data` changes
the id, which means the tx is treated as a different tx and rejected
as a duplicate.

The mempool also rejects any tx where `recipient`, `amount`, or `fee`
differ from the signed values, so the 5-field signature is sufficient
to bind a sender's intent to those four values for that timestamp.

## Computing the signature

```python
import hashlib, json
from crypto.kyber_crypto import WaveLedgerCrypto

crypto = WaveLedgerCrypto()
wallet = crypto.create_wallet()

signed_data = {
    'sender': wallet['address'],
    'recipient': '39848b50...',
    'amount': 1.0,
    'timestamp': 1780002999.123,
    'fee': 0.001,
}

signature = crypto.sign_transaction(
    signed_data,
    wallet['private_key'],
    signing_key=wallet.get('signing_key'),    # optional perf cache
)
```

Internally, `sign_transaction`:

1. Serializes `signed_data` as canonical JSON (`sort_keys=True`)
2. Hashes with SHA3-512
3. Signs the hash with ML-DSA-87
4. Returns the signature as hex

## Computing the transaction id

```python
import hashlib, json

tx_id = hashlib.sha3_512(
    json.dumps(signed_data, sort_keys=True).encode()
).hexdigest()
```

The id is canonical SHA3-512 of the same `signed_data` dict used for
signing — but **without** the signature. It's a content-address.

## Verifying a signature

```python
from crypto.kyber_crypto import WaveLedgerCrypto

crypto = WaveLedgerCrypto()
ok = crypto.verify_signature(
    signed_data,
    signature_hex,
    sender_public_key_hex,
    verifying_key=cached_verifier,    # optional
)
```

The chain's mempool calls this on every inbound tx before accepting it.

## Nonces

The `nonce` field is set to `-1` for legacy transactions (the chat dApp
uses this — the address-count check in the mempool handles duplicate
prevention). For nonce-tracked replay protection (recommended for any
machine-driven signer), set:

```python
nonce = blockchain.get_next_nonce(sender_address)
```

The chain enforces `transaction.nonce == expected_nonce + addr_pending_count`
for non-legacy txs. RBF works on `(sender, nonce)` pairs: same nonce
with a 10% higher fee replaces the older tx.

## Contract txs

The signature math is the same for contract deploys + calls. Only the
envelope differs:

| Field | Deploy | Call |
|---|---|---|
| `recipient` | `"contract"` | `"contract"` |
| `amount` | 0 (or value to credit) | 0 (or value to pass) |
| `data.type` | `"deploy"` | `"call"` |
| `data.code` | hex of bytecode | — |
| `data.to` | — | hex of target contract addr |
| `data.calldata` | hex (optional init data) | hex (selector + args) |
| `data.gas_limit` | `1_000_000` (default) | same |

See [`core.contract_engine.build_deploy_tx_data`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/core/contract_engine.py)
and `build_call_tx_data` for the helpers that produce the `data` dict.
