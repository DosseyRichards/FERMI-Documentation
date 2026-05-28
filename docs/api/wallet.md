# Wallet API

Wallet endpoints expose the user's address, balance, transaction
history, and let them sign + submit transfer transactions or download
an encrypted backup of their keypair.

All endpoints below require a [session cookie](auth.md#session-cookies-end-users).

---

## GET /api/wallet

Returns the current user's wallet info.

### Response

```json
{
  "name": "alice",
  "address": "34378b1ba5be9d0999acd60be3a8a1f1",
  "balance": 99.998,
  "public_key": "0a1b...4f78…",
  "public_key_full": "0a1b...4f78... (full 2592-byte hex)",
  "signature_scheme": "ML-DSA-87 (NIST FIPS 204)",
  "transactions": [
    {
      "tx_id": "f5c4357...",
      "direction": "out",
      "counterparty": "39848b500426d8f1...",
      "amount": 0.001,
      "fee": 0.001,
      "timestamp": 1780002999.123,
      "block": 42,
      "memo": "lunch"
    }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `name` | string | The user's display name |
| `address` | string (40 hex) | Their wallet address |
| `balance` | float | WAVE, confirmed (does not subtract pending outflows) |
| `public_key` | string | Truncated ML-DSA-87 public key (UI display) |
| `public_key_full` | string | Full hex of the public key |
| `signature_scheme` | string | Always `"ML-DSA-87 (NIST FIPS 204)"` in v1 |
| `transactions` | list | Last 25 in/out txs involving this address, newest first |

### Errors

| Status | Body | When |
|---|---|---|
| 401 | `{"error":"no session"}` | No `session` cookie present |
| 401 | `{"error":"session invalid"}` | Cookie present but name no longer in `approved` set |

---

## POST /api/wallet/send

Sign + submit a WAVE transfer from the user's wallet.

### Request

```json
{
  "to": "39848b500426d8f1...",
  "amount": 1.5,
  "memo": "optional note"
}
```

| Field | Required | Notes |
|---|---|---|
| `to` | yes | Recipient address (hex, with or without `0x`) |
| `amount` | yes | WAVE, must be > 0 |
| `memo` | no | Up to 200 chars; stored in `tx.data.memo` and visible to anyone |

### Response

```json
{
  "status": "submitted",
  "tx_id": "a2a7dfa5e8b26a1b..."
}
```

The tx is in the mempool. Confirmation lands in the next block (~5
sec testnet, ~60 sec mainnet target).

### Errors

| Status | Body | When |
|---|---|---|
| 400 | `{"error":"invalid amount"}` | Non-numeric or ≤ 0 |
| 400 | `{"error":"recipient and positive amount required"}` | Missing fields |
| 400 | `{"error":"insufficient balance (X.XXX WAVE)"}` | Balance too low for amount + 0.001 fee |
| 400 | `{"error":"tx rejected by mempool"}` | See mempool rules under [Blocks](../concepts/blocks.md#mempool-eligibility) |
| 401 | Standard session errors | |

The fee is fixed at **0.001 WAVE** for now (matches the contract gas
fee). Custom-fee transfers will land with the next mempool revision.

---

## POST /api/wallet/export

Download an encrypted backup of the user's wallet. The backup is an
AES-256-GCM ciphertext with the user's passphrase deriving the key
via Argon2id.

### Request

```json
{ "passphrase": "at least eight characters" }
```

### Response

```json
{
  "_format": "waveledger-wallet-backup/v1",
  "_warning": "Keep this file safe. Anyone with this file + your passphrase can spend from your wallet.",
  "address": "34378b1ba5be9d0999acd60be3a8a1f1",
  "encrypted_data": "<hex ciphertext>",
  "nonce": "<hex>",
  "salt": "<hex>",
  "created_at": "2026-05-28T20:00:00",
  "last_accessed": "2026-05-28T21:00:00",
  "version": "1.0",
  "encryption_algorithm": "AES-256-GCM",
  "kdf_algorithm": "Argon2id"
}
```

The wallet UI saves this as `waveledger-wallet-<address-prefix>.json`.

### Why AES-256-GCM on a post-quantum chain?

AES-256-GCM is **post-quantum safe** per NIST SP 800-208 (Cat 5).
Grover's algorithm halves symmetric security, so AES-256 → 128-bit
quantum security, which is still infeasible. ML-DSA / ML-KEM are
wrong tools for password-based file encryption (they target asymmetric
signatures / key exchange). For a future "encrypted wallet handoff"
between two on-chain ML-KEM keys, see the roadmap.

### Errors

| Status | Body | When |
|---|---|---|
| 400 | `{"error":"passphrase must be at least 8 chars"}` | Self-explanatory |
| 400 | `{"error":"Weak password: ..."}` | Argon2id rejected as too weak |
| 401 | Standard session errors | |

---

## POST /api/wallet/import

Restore a wallet from an encrypted backup. The user must have already
signed up (or been approved) under the target name.

### Request

```json
{
  "name": "alice",
  "passphrase": "...",
  "encrypted": { /* the full export object above */ }
}
```

### Response

```json
{
  "status": "imported",
  "address": "34378b1ba5be9d0999acd60be3a8a1f1"
}
```

This **replaces** whatever wallet the admin's approval created for that
name — useful for users who want to bring an existing wallet to a new
chat account.

### Errors

| Status | Body | When |
|---|---|---|
| 400 | `{"error":"name, encrypted, passphrase all required"}` | Missing field |
| 400 | `{"error":"decrypt failed: ..."}` | Wrong passphrase or corrupted file |
| 403 | `{"error":"name is blocked"}` | Admin blocked this name |
| 404 | `{"error":"name not approved — submit a signup first"}` | Sign up first, then import |
