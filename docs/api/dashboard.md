# Dashboard API (node-local)

The dashboard runs on **port 8080** of every node, bound to loopback
by default (`127.0.0.1`). It exposes monitoring and
chain-introspection endpoints intended for the node operator.

For public-facing chain data, use the [Explorer API](explorer.md)
instead. It offers similar coverage with no auth requirement.

## Reaching the dashboard

```bash
# On the node
curl http://127.0.0.1:8080/api/status

# From outside, via fly ssh:
fly ssh console --config fly.chat.toml -C 'curl -s http://127.0.0.1:8080/api/status'
```

To expose the dashboard publicly, set `[dashboard].host = "0.0.0.0"`
in `config.toml` and place a reverse proxy with auth in front. The
[`require_auth`](auth.md#bearer-api-key-dashboard-node-local) flag
controls whether GET endpoints also require the API key.

## Endpoints

### GET /api/status

Health check. No auth required even when `require_auth=true`.

```json
{
  "node_running": true,
  "chain_height": 12345,
  "peer_count": 2,
  "mining_active": true,
  "qrng_reachable": true,
  "simulate": false,
  "sync_state": "synced"
}
```

### GET /api/node

Node identity and uptime.

```json
{
  "node_id": "489d0c3879454833...",
  "version": "0.1.0-testnet",
  "started_at": 1780002000.0,
  "uptime_seconds": 9999.0,
  "data_dir": "/data"
}
```

### GET /api/chain

Chain summary.

```json
{
  "height": 12345,
  "tip_hash": "0000ab12...",
  "total_supply": 1062340.0,
  "difficulty": 4,
  "block_time_target": 5,
  "testnet": true
}
```

### GET /api/blocks?offset=&limit=

Same shape as the [explorer endpoint](explorer.md#get-apiexplorerblocks).
Predates the explorer; kept for backward compatibility.

### GET /api/block/{identifier}

Look up a block by height (`"42"`) or by hash (`"0000ab12..."`).

### GET /api/tx/{tx_id}

Look up a transaction by id, with full data including the raw `data`
field (deploy bytecode, call calldata, etc.).

### GET /api/address/{address}

Address detail with balance and transaction history.

### GET /api/mempool

Current mempool contents.

```json
{
  "pending_count": 3,
  "transactions": [
    {
      "tx_id": "abc123",
      "sender": "dc66f82c048f3514",
      "recipient": "39848b50",
      "amount": 100.0,
      "fee": 0.001,
      "data": { "memo": "faucet-drip" }
    }
  ]
}
```

### GET /api/mining

Miner state.

```json
{
  "enabled": true,
  "running": true,
  "blocks_mined": 42,
  "miner_address": "dc66f82c048f35144599737ed54ab702",
  "simulation_mode": false,
  "qrng_source": "drand-default"
}
```

### GET /api/qrng

Status of the configured entropy provider.

```json
{
  "reachable": true,
  "host": "waveledger-entropy.internal",
  "port": 8420,
  "last_fetch_at": 1780002999.0,
  "pool_available_bytes": 65536
}
```

### GET /api/qrng/random?n=N

Direct passthrough to the entropy provider's `/api/random/bytes?n=N`.
Returns hex.

### GET /api/peers

List connected peers.

```json
{
  "peer_count": 2,
  "peers": [
    {
      "peer_id": "5a5088d879aa4655...",
      "address": "1.2.3.4:18333",
      "version": "0.1.0-testnet",
      "best_height": 12345,
      "connected_at": 1780002000.0
    }
  ]
}
```

### GET /api/fee {#fee-estimation}

Fee estimation based on mempool depth and recent block contents.

```json
{
  "mempool_min": 0.0001,
  "mempool_median": 0.001,
  "min_fee": 0.0001,
  "low_fee": 0.0005,
  "medium_fee": 0.001,
  "high_fee": 0.005,
  "mempool_fullness": 0.001
}
```

### GET /api/search?q=

Heuristic search. An integer `q` triggers a block lookup; a long hex
string triggers a transaction lookup; otherwise the value is treated
as an address lookup. Returns the same structure as the corresponding
type-specific endpoint.

### POST /api/tx/submit

Submit a pre-signed transaction. Same body shape as a `Transaction`
to_dict (`sender`, `recipient`, `amount`, `timestamp`, `transaction_id`,
`signature`, `fee`, `data`, `nonce`).

Requires `Authorization: Bearer <api-key>` (write method).

```bash
curl -X POST http://127.0.0.1:8080/api/tx/submit \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{...tx...}'
```

Response:

```json
{ "status": "submitted", "tx_id": "..." }
```

Or on rejection:

```json
{ "status": "rejected", "reason": "insufficient balance (0.0001 < 1.001)" }
```
