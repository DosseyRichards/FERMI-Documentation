# Subscribing to events

WaveLedger exposes one push stream — the chat dApp's SSE feed for new
messages — and everything else is polling.

## Chat messages: SSE

See [Server-Sent Events](../api/sse.md) for the full endpoint spec.

```python
import requests, json

with requests.get(
    'https://chat.waveledger.net/api/stream',
    cookies={'session': '...'}, stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b'data: '):
            print(json.loads(line[6:]))
```

The stream emits a `message` event whenever a new message lands in the
mempool, and again with `status: "confirmed"` when it's mined. Dedupe
on `(tx_id, status)`.

## New blocks: poll

There is no native block-notification stream yet. The two patterns:

### Poll the explorer

```python
import requests, time

last_height = 0
while True:
    s = requests.get('https://chat.waveledger.net/api/explorer/stats').json()
    if s['height'] > last_height:
        # New block(s) — fetch them
        for h in range(last_height + 1, s['height'] + 1):
            b = requests.get(f'https://chat.waveledger.net/api/explorer/block/{h}').json()
            print(f"block {h}: {b['tx_count']} txs")
        last_height = s['height']
    time.sleep(2)
```

For mainnet (60s block times) `sleep(5)` is plenty. For testnet (5s
blocks) `sleep(1-2)` to catch every block.

### Use the dashboard locally

If you run a local node, the dashboard's `/api/blocks` endpoint serves
the same data instantly from in-memory cache, no network hop. Useful
for indexers that need block-by-block coverage.

## New contract receipts: poll

After submitting a deploy or call, poll the receipt endpoint:

```python
import requests, time

def wait_for_receipt(tx_id, base='https://chat.waveledger.net', timeout=30):
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

For watching when a specific address receives or sends:

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

The testnet uses SSE for the one stream that needs it (chat) because
SSE is half the complexity of WebSockets, works through every CDN
without special config, has auto-reconnect built into browsers, and
fits a one-way push perfectly.

Bidirectional channels (which the chain currently doesn't need) would
warrant WebSockets. There's no roadmap item to add them yet.

## Roadmap

| Feature | Status |
|---|---|
| `block-mined` SSE event | Planned — straightforward extension of the existing pumper |
| `tx-confirmed` SSE event | Planned — same |
| Per-address SSE filter | Planned — server-side filter |
| `eth_subscribe`-style WS | Not planned |
| Webhook (HTTP POST on event) | Considering |
