# Playground (contracts) API

Endpoints for compiling Fourier source, deploying it as a contract,
and calling methods on a deployed contract.

## Authentication

Each endpoint accepts either:

- a [session cookie](auth.md#session-cookies-end-users), or
- an `Authorization: Bearer wlg_…` [API token](admin.md#api-tokens)
  scoped to `playground`.

The session cookie (or the token's bound user) determines whose wallet
pays the fee. If both are presented, the session cookie wins. The
SDKs construct the right header automatically — see
[SDK / Python](../sdk/python.md#api-token-auth) and
[SDK / TypeScript](../sdk/javascript.md#api-token-auth).

The paying wallet must hold sufficient WAVE; the
[signup faucet](../concepts/economics.md) drips 100 WAVE, sufficient
for approximately 100,000 contract operations.

## Compile

### POST /api/playground/compile

Compile Fourier source to VM bytecode and extract its ABI. No
on-chain state changes; no gas cost.

#### Request

```json
{ "source": "contract Counter { storage value: uint @ 0; ... }" }
```

#### Response

```json
{
  "bytecode_hex": "0120ffff...0100",
  "bytecode_size": 198,
  "abi": {
    "contract": "Counter",
    "methods": [
      { "name": "get", "selector": 1, "params": [], "returns": "uint" },
      { "name": "inc", "selector": 2, "params": [], "returns": null }
    ],
    "storage": [
      { "name": "value", "type": "uint", "slot": 0 }
    ],
    "events": []
  }
}
```

The ABI is extracted by parsing the source with
[`fourier.fourier_abi.extract_abi`](https://fourier.fermi.world/abi/),
walking the AST to find every `pub fn`, and assigning sequential
1-byte selectors starting at `0x01`. See
[Fourier ABI](https://fourier.fermi.world/abi/) for the encoding.

#### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"<compile error>", "phase": "compile"}` |
| 401 | `{"error":"not logged in"}` |

---

## Deploy

### POST /api/playground/deploy

Compile and submit a deploy transaction. The contract address is
derived from the sender and sender's nonce; it is not known until the
deploy transaction is mined.

#### Request

```json
{ "source": "contract Counter { ... }" }
```

Optional fields (use defaults if absent):

| Field | Default | Notes |
|---|---|---|
| `gas_limit` | 1,000,000 | Max gas the deploy may consume |
| `value` | 0 | WAVE to credit to the contract at deploy time |
| `init_calldata` | `""` | Bytes passed to the contract's `init`. Fourier v1 requires this to be empty. |

#### Response

```json
{
  "status": "submitted",
  "tx_id": "a2a7dfa5e8b26a1b...",
  "abi": { /* same as /compile */ }
}
```

The transaction enters the mempool. Poll
[`GET /api/playground/receipt/{tx_id}`](#receipt) to learn the
contract address once the block lands (~5 sec testnet).

#### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"<compile error>", "phase": "compile"}` |
| 400 | `{"error":"insufficient WAVE for deploy fee"}` |
| 400 | `{"error":"deploy tx rejected by mempool"}` |
| 401 | `{"error":"not logged in"}` |

---

## Call

### POST /api/playground/call

Submit a call transaction against a deployed contract. The server
encodes calldata using the contract's ABI (selector plus 32-byte
arguments).

#### Request

```json
{
  "contract_addr": "0x082dd2359955d8d191d29a354d13add25c63336c",
  "selector": 2,
  "args": [],
  "params": []
}
```

| Field | Required | Notes |
|---|---|---|
| `contract_addr` | yes | Hex address (with or without `0x`) |
| `selector` | yes | 1-byte function selector from the ABI |
| `args` | yes | Argument values in declaration order |
| `params` | yes | Type info per arg (`[{"name":"to","type":"address"}, ...]`); required for encoding |
| `gas_limit` | no | Default 1,000,000 |
| `value` | no | WAVE attached to the call |

#### Response

```json
{ "status": "submitted", "tx_id": "f5c43575f6b520c1..." }
```

#### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"contract_addr + selector required"}` |
| 400 | `{"error":"<encoding error>", "phase": "encode"}` |
| 400 | `{"error":"insufficient WAVE for call fee"}` |
| 400 | `{"error":"call tx rejected by mempool"}` |
| 401 | `{"error":"not logged in"}` |

---

## Receipt

### GET /api/playground/receipt/{tx_id} {#receipt}

Returns the receipt for a deploy or call transaction once mined.

#### Response — pending

```json
{ "status": "pending" }
```

#### Response — confirmed (deploy)

```json
{
  "status": "confirmed",
  "success": true,
  "type": "deploy",
  "gas_used": 25248,
  "contract_addr": "0x082dd2359955d8d191d29a354d13add25c63336c"
}
```

#### Response — confirmed (call, with uint return)

```json
{
  "status": "confirmed",
  "success": true,
  "type": "call",
  "gas_used": 501,
  "return_hex": "0000...0001",
  "return_uint": 1
}
```

#### Response — reverted

```json
{
  "status": "confirmed",
  "success": false,
  "type": "call",
  "gas_used": 1023,
  "revert_reason": "insufficient balance"
}
```

| Field | Notes |
|---|---|
| `success` | `true` if the contract returned normally, `false` if it reverted |
| `type` | `"deploy"`, `"call"`, or `"contract"` (error before classification) |
| `gas_used` | Integer gas consumed |
| `contract_addr` | Present on successful deploy |
| `return_hex` | Raw return data, hex-encoded |
| `return_uint` | Decoded to integer if return is exactly one 32-byte word |
| `revert_reason` | Present if `success == false` and the contract supplied a reason |

---

## List your contracts

### GET /api/playground/contracts

Returns contracts deployed by the current user along with their
ABIs.

#### Response

```json
{
  "contracts": [
    {
      "tx_id": "a2a7dfa5e8b26a1b...",
      "contract_addr": "0x082dd2359955d8d191d29a354d13add25c63336c",
      "abi": { /* ABI from compile */ },
      "submitted_at": 1780002999.1,
      "status": "deployed"
    }
  ]
}
```

Pending deploys (block not yet mined) have `contract_addr: null` and
`status: "pending"`.

---

## Examples

### GET /api/playground/examples

Returns the bundled example contracts that the UI loads from the
"Load example" dropdown.

#### Response

```json
{
  "examples": [
    {
      "id": "counter",
      "title": "Counter",
      "description": "Minimal contract — stores an integer, exposes get/inc.",
      "source": "contract Counter { ... }"
    },
    {
      "id": "guestbook",
      "title": "Guestbook",
      "description": "Each address stores one message; anyone can read any.",
      "source": "contract Guestbook { ... }"
    },
    {
      "id": "multisig",
      "title": "M-of-N Threshold",
      "description": "Tracks a numeric tally toward a threshold.",
      "source": "contract Threshold { ... }"
    }
  ]
}
```

Examples ship with the testnet build and are extended by editing
the `_PG_EXAMPLES` dict in `api/messenger.py`.
