# Subscribing to events

WaveLedger exposes a single SSE stream that pushes **every** chain-side
state change — new blocks, confirmed transactions, chat messages, and
contract receipts. Server-side filtering by event type and address
keeps client code small.

See [Server-Sent Events](../api/sse.md) for the full endpoint spec.

## All events (firehose)

```python
import requests, json

with requests.get(
    'https://api.waveledger.net/api/stream',
    cookies={'session': '...'}, stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b'data: '):
            print(json.loads(line[6:]))
```

The stream emits four event types: `block`, `tx`, `message`, `receipt`.
Dedupe defensively on `tx_id` for tx-shaped events and `height` for
block events.

## New blocks only

```python
with requests.get(
    'https://api.waveledger.net/api/stream?types=block',
    cookies={'session': '...'}, stream=True,
) as r:
    for line in r.iter_lines():
        if not line.startswith(b'data: '):
            continue
        ev = json.loads(line[6:])
        b = ev['block']
        print(f"block {b['height']}: {b['tx_count']} txs, miner {b['miner'][:16]}")
```

## Watch one address

```python
addr = '34378b1ba5be9d0999acd60be3a8a1f1'
with requests.get(
    f'https://api.waveledger.net/api/stream?address={addr}',
    cookies={'session': '...'}, stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b'data: '):
            ev = json.loads(line[6:])
            # ev['type'] in {'tx', 'message', 'receipt', 'block'}; filtered
            # server-side to events that name `addr` in any role.
            print(ev['type'], ev)
```

The `address` filter matches sender, recipient, `to`, contract address,
miner, and message author. Combine with `types=` to narrow:

```text
/api/stream?types=tx,receipt&address=<addr>
```

## Contract receipts only

```python
with requests.get(
    'https://api.waveledger.net/api/stream?types=receipt',
    cookies={'session': '...'}, stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b'data: '):
            print(json.loads(line[6:])['receipt'])
```

## Fallback: poll the explorer

The SSE stream is the recommended path. The polling examples below
apply to clients that cannot open long-lived connections (e.g. some
serverless platforms).

### Poll for new blocks

```python
import requests, time

last_height = 0
while True:
    s = requests.get('https://api.waveledger.net/api/explorer/stats').json()
    if s['height'] > last_height:
        for h in range(last_height + 1, s['height'] + 1):
            b = requests.get(f'https://api.waveledger.net/api/explorer/block/{h}').json()
            print(f"block {h}: {b['tx_count']} txs")
        last_height = s['height']
    time.sleep(2)
```

For mainnet (60s block times), `sleep(5)` is sufficient. For testnet
(5s blocks), use `sleep(1-2)` to catch every block.

### Poll a specific receipt

After submitting a deploy or call, poll the receipt endpoint:

```python
import requests, time

def wait_for_receipt(tx_id, base='https://api.waveledger.net', timeout=30):
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = requests.get(f'{base}/api/explorer/tx/{tx_id}').json()
        if r.get('status') == 'confirmed' and r.get('receipt') is not None:
            return r['receipt']
        time.sleep(1)
    raise TimeoutError(f"receipt for {tx_id} never landed")
```

## Mempool changes: poll

Same pattern — poll `/api/mempool` (dashboard, local only) or
`/api/explorer/stats` for the `mempool_size` field.

## Address activity: poll

To watch when a specific address receives or sends:

```python
last_tx_count = 0
while True:
    r = requests.get(f'.../api/explorer/address/{addr}').json()
    if r['tx_count'] > last_tx_count:
        new = r['transactions'][:r['tx_count'] - last_tx_count]
        for tx in new:
            print(f"{tx['direction']}: {tx['amount']} WAVE")
        last_tx_count = r['tx_count']
    time.sleep(3)
```

## Why no WebSockets?

The testnet uses SSE for the one stream that requires it (chat). SSE
has half the complexity of WebSockets, works through every CDN without
special configuration, has auto-reconnect built into browsers, and
fits a one-way push pattern.

Bidirectional channels would warrant WebSockets. The chain has no
requirement for them in v1.

## Future extensions

| Feature | Status |
|---|---|
| `block` SSE event | Shipped |
| `tx` SSE event | Shipped |
| `receipt` SSE event | Shipped |
| Per-address SSE filter | Shipped |
| Per-type SSE filter | Shipped |
| `eth_subscribe`-style WS | Not planned |
| Webhook (HTTP POST on event) | Under consideration |
