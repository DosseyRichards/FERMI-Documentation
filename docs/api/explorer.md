# Explorer API

Public, unauthenticated endpoints exposing chain data. All endpoints
support pagination where it matters.

## GET /api/explorer/stats

Top-level chain stats. Cheap to call repeatedly — the UI polls this
every few seconds.

### Response

```json
{
  "height": 12345,
  "tip_hash": "0000ab12...",
  "tip_timestamp": 1780002999.123,
  "total_supply": 1062340.0,
  "max_supply": 21000000,
  "mempool_size": 3,
  "difficulty": 4,
  "block_time_target": 5,
  "testnet": true
}
```

---

## GET /api/explorer/blocks

Latest blocks, newest first. Paginated.

### Query params

| Param | Default | Max | Notes |
|---|---|---|---|
| `offset` | 0 | — | 0 = newest. `offset=25` skips the 25 most recent. |
| `limit` | 25 | 100 | |

### Response

```json
{
  "blocks": [
    {
      "height": 12345,
      "hash": "0000ab12...",
      "previous_hash": "0000fc99...",
      "timestamp": 1780002999.123,
      "tx_count": 3,
      "miner": "dc66f82c048f35144599737ed54ab702",
      "difficulty": 4,
      "quantum_verified": true
    }
  ],
  "pagination": {
    "offset": 0,
    "limit": 25,
    "total": 12346,
    "has_more": true
  }
}
```

---

## GET /api/explorer/block/{height}

Full block detail, including every transaction (decoded with kind +
counterparties).

### Response

```json
{
  "height": 12345,
  "hash": "0000ab12...",
  "previous_hash": "0000fc99...",
  "timestamp": 1780002999.123,
  "merkle_root": "abcd...",
  "nonce": 42031,
  "difficulty": 4,
  "miner": "dc66f82c048f35144599737ed54ab702",
  "tx_count": 3,
  "quantum_verified": true,
  "qrng_attestation_present": true,
  "transactions": [
    {
      "tx_id": "reward_12345_xyz",
      "sender": "mining_reward",
      "sender_name": null,
      "recipient": "dc66f82c048f35144599737ed54ab702",
      "recipient_name": null,
      "amount": 5.0,
      "fee": 0.0,
      "timestamp": 1780002999.123,
      "kind": "coinbase"
    },
    {
      "tx_id": "a2a7dfa...",
      "sender": "34378b1b...",
      "sender_name": "alice",
      "recipient": "contract",
      "amount": 0.0,
      "fee": 0.001,
      "timestamp": 1780002999.0,
      "kind": "deploy",
      "code_size": 198
    },
    {
      "tx_id": "f5c4357...",
      "sender": "34378b1b...",
      "sender_name": "alice",
      "recipient": "contract",
      "amount": 0.0,
      "fee": 0.001,
      "timestamp": 1780002999.1,
      "kind": "call",
      "to": "0x082dd2359955d8d191d29a354d13add25c63336c",
      "selector": 2,
      "arg_count": 0
    }
  ]
}
```

`kind` field values:

| Value | When |
|---|---|
| `coinbase` | Sender is `"mining_reward"` |
| `genesis` | Sender is `"genesis"` |
| `deploy` | Recipient is `"contract"` and `data.type == "deploy"` |
| `call` | Recipient is `"contract"` and `data.type == "call"` |
| `message` | `data` contains a `memo` field |
| `transfer` | Plain transfer (fallback) |

### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"height must be integer"}` |
| 404 | `{"error":"no such block"}` |

---

## GET /api/explorer/tx/{tx_id}

Look up a transaction by id. Searches the chain (newest blocks first)
and falls back to the mempool.

### Response — confirmed call with receipt

```json
{
  "tx_id": "a2a7dfa5e8b26a1b...",
  "sender": "34378b1b...",
  "sender_name": "alice",
  "recipient": "contract",
  "amount": 0.0,
  "fee": 0.001,
  "timestamp": 1780002999.0,
  "kind": "deploy",
  "code_size": 198,
  "block_height": 12345,
  "block_hash": "0000ab12...",
  "status": "confirmed",
  "confirmations": 3,
  "receipt": {
    "type": "deploy",
    "success": true,
    "gas_used": 25248,
    "contract_address": "0x082dd2359955d8d191d29a354d13add25c63336c",
    "return_data": null,
    "revert_reason": null
  }
}
```

### Response — pending

```json
{
  "tx_id": "def456...",
  "sender": "...",
  ...
  "status": "pending",
  "confirmations": 0
}
```

### Errors

| Status | Body |
|---|---|
| 404 | `{"error":"tx not found"}` |

---

## GET /api/explorer/address/{address}

Lookup an address. Returns the current balance + a paginated list of
all txs that involve this address (sender or recipient).

### Query params

| Param | Default | Max |
|---|---|---|
| `offset` | 0 | — |
| `limit` | 25 | 100 |

### Response

```json
{
  "address": "34378b1ba5be9d0999acd60be3a8a1f1",
  "name": "alice",
  "balance": 99.998,
  "tx_count": 42,
  "is_foundation": false,
  "transactions": [
    {
      "tx_id": "a2a7dfa...",
      "block_height": 12345,
      "direction": "out",
      "sender": "34378b1b...",
      "recipient": "39848b50...",
      "amount": 1.0,
      "fee": 0.001,
      "timestamp": 1780002999.0,
      "kind": "transfer"
    }
  ],
  "pagination": {
    "offset": 0, "limit": 25, "total": 42, "has_more": true
  }
}
```

`name` is set if the address is a known approved user or the foundation
address. `is_foundation: true` flags the address that holds the
[genesis premine](../reference/tokenomics.md#genesis).

The address argument accepts hex with or without `0x`. The 20-byte
canonical form (40 hex chars) is what gets returned in responses.
