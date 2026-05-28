# Receipt format

A receipt is the structured result of a contract transaction (deploy or
call). One receipt per contract tx, persisted in the chain database
keyed by `transaction_id`, and surfaced via the explorer API.

Non-contract transactions (transfers, chat messages) do not produce
receipts.

## JSON shape

### Deploy receipt

```json
{
  "type": "deploy",
  "contract": "39c1a47bf68d0e2d9c43caaa10c1b2f3c4d5e6f7",
  "success": true,
  "gas_used": 42118,
  "logs": [ ... ],
  "error": null
}
```

### Call receipt

```json
{
  "type": "call",
  "to": "39c1a47bf68d0e2d9c43caaa10c1b2f3c4d5e6f7",
  "success": true,
  "gas_used": 26305,
  "return_data": "0000000000000000000000000000000000000000000000000000000000000001",
  "logs": [ ... ],
  "error": null
}
```

## Field reference

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"deploy"` or `"call"` |
| `contract` | string | (deploy only) Deployed contract address, 40-char hex (20 bytes) |
| `to` | string | (call only) Target contract address, 40-char hex (20 bytes) |
| `success` | bool | `true` if the VM halted via STOP/RETURN; `false` if REVERT or runtime error |
| `gas_used` | int | Gas consumed by the call frame and any sub-calls |
| `return_data` | string | (call only) Hex-encoded RETURN payload; empty string if nothing was returned |
| `logs` | list[Log] | Events emitted via `LOG` opcode in declaration order |
| `error` | string \| null | Short error description if `success=false`; `null` otherwise |

## Log shape

Each entry in `logs`:

```json
{
  "address": "<20-byte hex>",
  "topics":  ["<32-byte hex>", "<32-byte hex>", ...],
  "data":    "<hex>"
}
```

| Field | Notes |
|---|---|
| `address` | Contract that emitted the log (the currently-executing contract, not `tx.sender`) |
| `topics` | Up to 4 indexed values; for Fourier events, `topics[0]` is the SHA3-256 event signature hash |
| `data` | Concatenated non-indexed event arguments, 32 bytes each, no length prefix |

## Lifecycle

1. Miner builds a block including a contract tx (`type` of `deploy` or `call`).
2. During block application, `contract_engine.process_tx()` runs the VM
   and returns the receipt dict.
3. Receipt is persisted via `ChainDB.put_receipt(tx_id, block_height, receipt)`.
4. Explorer API surfaces it via `GET /api/receipt/{tx_id}`.

## Address derivation (deploy)

The `contract` address is deterministic:

```
contract_address = SHA3-256(deployer_addr || nonce.to_bytes(32, "big"))[:20]
```

`deployer_addr` is the deployer's 20-byte VM address (see
[Addresses](address.md) for the wallet→VM mapping). `nonce` is the
deployer's VM-level nonce *before* the deploy (incremented atomically
during the deploy).

## Gas accounting

- Gas is denominated in *units*; the tx pays `gas_used × GAS_PRICE`
  bohms, deducted from `tx.fee`.
- `GAS_PRICE = 10^9` bohms/unit (see `core/contract_engine.py`).
- `DEFAULT_GAS_LIMIT = 1_000_000`.
- A tx whose `fee` does not cover `gas_limit × GAS_PRICE` is rejected at
  mempool admission (`InsufficientFee` exception).
- Unused gas is **not** refunded in v1.

## Failure semantics

When `success = false`:

- All state changes from the failed call frame are reverted (atomic).
- Outer balance/nonce changes (the fee deduction, the deployer nonce
  bump) still apply.
- `gas_used` reflects what the VM consumed before failing.
- Common `error` values: `"REVERT"`, `"out of gas"`, `"stack underflow"`,
  `"invalid jump"`, `"call depth exceeded"`, `"bad code: ..."` (deploy
  with malformed bytecode), `"bad fields: ..."` (malformed call data).

## Querying

```
GET /api/receipt/{tx_id}        → receipt JSON, or 404
GET /api/receipts/block/{idx}   → list of receipts for that block
```

See [Explorer API](../api/explorer.md).
