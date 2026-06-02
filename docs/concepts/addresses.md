# Addresses and wallets

## Address format

A WaveLedger address is **20 bytes**, displayed as a 40-character
lowercase hex string with no `0x` prefix in the canonical form:

```
34378b1ba5be9d0999acd60be3a8a1f1
```

Most APIs accept either form (`0x...` or plain hex). The explorer and
wallet UI display the canonical form (no prefix); RPC accepts both.

## Derivation

An address is derived from an ML-DSA-87 public key by:

1. Generating an ML-DSA-87 keypair (`dilithium-py` reference impl).
2. Serializing the public key.
3. Hashing it: `SHA3-512(serialized_pubkey)`.
4. Taking the **first 20 bytes** of the hash.

In code:

```python
from crypto.kyber_crypto import WaveLedgerCrypto

crypto = WaveLedgerCrypto()
wallet = crypto.create_wallet()
# wallet['address']     → 40-char hex string
# wallet['public_key']  → ML-DSA-87 public key (object or hex)
# wallet['private_key'] → ML-DSA-87 secret key (do not share)
```

## Wallet object shape

A wallet (in-memory) is a dict with these keys:

| Key | Type | Notes |
|---|---|---|
| `address` | str (40 hex) | Canonical address |
| `public_key` | hex string \| object | ML-DSA-87 verifying key |
| `private_key` | hex string | ML-DSA-87 signing key — **keep secret** |
| `signing_key` | object (optional) | Cached signer instance; speeds repeat-sign |
| `verifying_key` | object (optional) | Cached verifier instance |
| `created_at` | str | ISO-8601 timestamp |

`signing_key` / `verifying_key` are performance caches. If absent the
chain re-derives them from `private_key` / `public_key` on each call.

## Reserved sentinel addresses

Several strings appear as sender/recipient that are not real addresses.
They are sentinels recognized by the chain:

| Sentinel | Meaning |
|---|---|
| `mining_reward` | Coinbase tx sender. Never a real wallet. |
| `genesis` | Sender of the genesis distribution tx. |
| `contract` | Recipient of contract deploy + call txs; the real target is in `tx.data`. |
| `BURN_ADDRESS` (`"0" * 32`) | Convention for burned funds. |

The recipient of the genesis premine is a real ML-DSA-87 address —
see [`GENESIS_FOUNDATION_ADDRESS`](../reference/tokenomics.md#genesis).
Earlier testnet builds used the string `"wave_foundation"` as a sentinel,
which was unclaimable; that has been replaced with a real keypair.

## Signing transactions

To sign a tx, the chain computes:

```python
import hashlib, json
tx_data = {
  'sender': addr_hex,
  'recipient': recipient_hex,
  'amount': float,
  'timestamp': float,
  'fee': float,
}
tx_id = hashlib.sha3_512(
    json.dumps(tx_data, sort_keys=True).encode()
).hexdigest()
signature = crypto.sign_transaction(tx_data, private_key, signing_key=signing_key)
```

The signature covers only the five fields above (sender, recipient,
amount, timestamp, fee) — **not** the `data` field. This is deliberate:
contract-deploy/call data is large (full bytecode), and re-signing on
every change to a call's calldata would make hand-signing impractical.

The integrity of `data` is enforced by `tx_id`, which hashes the entire
canonical envelope including `data`.

!!! warning "Don't lose your private key"
    There is no recovery mechanism. Use the [encrypted backup](../api/wallet.md#export)
    feature in the wallet UI, or back up the keypair JSON offline.

## Wallet UX in the testnet dApp

The testnet chat dApp generates a wallet for every user at signup
(with or without an invite code). The keypair lives in server memory
until the user downloads an encrypted backup via
`POST /api/wallet/export`. See [Wallet API](../api/wallet.md).

The server-side custody model is the v1 specification.

### Future extensions

Browser-side ML-DSA via a WASM port of `dilithium-py` would make the
dApp non-custodial. See the
[`project-wallet-backup-crypto`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software)
design note for the AES-256-GCM choice on backup encryption.
