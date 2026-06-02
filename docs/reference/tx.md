# Transaction format

The canonical on-chain representation of a transaction. Used by
serializers, signers, and consensus validation.

## JSON shape

```json
{
  "sender": "34378b1ba5be9d0999acd60be3a8a1f1",
  "recipient": "39848b500426d8f10a3c2b4e6d7f8901",
  "amount": 1.0,
  "timestamp": 1780002999.123,
  "transaction_id": "<128-char hex>",
  "signature": "<hex, ~9KB>",
  "fee": 0.001,
  "data": null,
  "nonce": -1,
  "quantum_verified": true
}
```

## Field reference

| Field | Type | Notes |
|---|---|---|
| `sender` | string | 32-char hex chain-layer address, OR one of the sentinels: `"mining_reward"`, `"genesis"` |
| `recipient` | string | Same — 32-char hex chain-layer address, OR `"contract"` for deploy/call txs |
| `amount` | float | WAVE; must be ≥ `MIN_TRANSACTION_AMOUNT` (0.01) except for contract txs |
| `timestamp` | float | Unix epoch, seconds, sub-second precision |
| `transaction_id` | string | SHA3-512 hex of the canonical JSON of (sender, recipient, amount, timestamp, fee) |
| `signature` | string | ML-DSA-87 hex signature over the same canonical dict |
| `fee` | float | WAVE; must be ≥ `MEMPOOL_MIN_FEE` (0.0001) |
| `data` | object \| null | Optional payload. See "Data field" below |
| `nonce` | int | `-1` = legacy (no nonce tracking); else ≥ 0 |
| `quantum_verified` | bool | True after signature verify; ignored on submit |

## Signed-data canonical form

Only **five fields** are covered by the signature:

```json
{
  "sender":    "<hex>",
  "recipient": "<hex|sentinel>",
  "amount":    1.0,
  "timestamp": 1780002999.123,
  "fee":       0.001
}
```

Serialized as `json.dumps(d, sort_keys=True)` then hashed with SHA3-512
and signed with ML-DSA-87. See [Signing](../sdk/signing.md) for details.

## Data field

The `data` field is unsigned (its integrity comes from `transaction_id`).
Three documented shapes:

### Chat message

```json
{ "memo": "any string up to 200 chars" }
```

The explorer renders txs with this shape as `kind: "message"`.

### Contract deploy

```json
{
  "type": "deploy",
  "code": "<hex bytecode>",
  "gas_limit": 1000000,
  "value": 0,
  "init_calldata": ""
}
```

`recipient` must be `"contract"`. The deployed address is derived from
`(sender, sender_nonce)` and lands in the receipt.

### Contract call

```json
{
  "type": "call",
  "to": "<20-byte hex address of target contract>",
  "calldata": "<hex, selector + 32-byte args>",
  "gas_limit": 1000000,
  "value": 0
}
```

See [Fourier ABI](https://fourier.fermi.world/abi/) for calldata
encoding.

## Validation order

When a tx is presented (via mempool admission or block validation):

1. Amount ≥ dust limit (skipped for contract txs)
2. Fee ≥ `MEMPOOL_MIN_FEE`
3. RBF: if `(sender, nonce)` matches an existing pending tx, require
   fee ≥ 1.1 × old_fee, then evict old
4. Duplicate id check
5. Per-address pending-count limit (50)
6. Balance check: `sender_balance - sum(pending_outflows_from_sender)
   ≥ amount + fee`
7. Nonce check (for non-legacy txs)
8. Signature verify
9. Mempool size cap with fee-priority eviction
10. Accept

## Sentinel senders + recipients

| String | Used as | Meaning |
|---|---|---|
| `mining_reward` | sender | Coinbase tx |
| `genesis` | sender | Genesis distribution tx (block 0 only) |
| `contract` | recipient | Contract deploy or call |
| `"0" * 32` (`BURN_ADDRESS`) | recipient (convention) | Burn |

A real address never starts with these strings; sentinel detection is
a literal match.

## Examples

### Simple transfer

```json
{
  "sender":   "34378b1ba5be9d0999acd60be3a8a1f1",
  "recipient":"39848b500426d8f10a3c2b4e6d7f8901",
  "amount":   1.5,
  "timestamp":1780002999.123,
  "transaction_id":"<sha3-512 hex>",
  "signature":"<ml-dsa-87 hex>",
  "fee":      0.01,
  "data":     null,
  "nonce":    -1
}
```

### Chat message

```json
{
  "sender":   "34378b1ba5be9d0999acd60be3a8a1f1",
  "recipient":"39848b500426d8f10a3c2b4e6d7f8901",
  "amount":   0.0,
  "timestamp":1780002999.123,
  "transaction_id":"<hex>",
  "signature":"<hex>",
  "fee":      0.001,
  "data":     { "memo": "hello world" },
  "nonce":    -1
}
```

(`amount=0` because chat messages do not transfer value; the message
text resides in `data.memo`. The recipient is the sender's own address
in the dApp — effectively a self-tx with a payload.)

### Coinbase (cannot be submitted; appears at block index 0 of every block)

```json
{
  "sender":   "mining_reward",
  "recipient":"dc66f82c048f35144599737ed54ab702",
  "amount":   5.001,
  "timestamp":1780002999.123,
  "transaction_id":"reward_12345_a1b2c3d4",
  "signature":"reward_signature",
  "fee":      0.0,
  "data":     null,
  "nonce":    -1,
  "quantum_verified": true
}
```

(`amount = subsidy + collected_fees`. `signature` is the literal
string `"reward_signature"` — synthetic, validated by structural rule
rather than ML-DSA.)
