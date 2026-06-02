# Server-Sent Events

The messenger pushes every chain-side state change to connected
clients via a standard SSE stream: new blocks, new transactions of
any kind, chat messages, and contract receipts. Optional server-side
filtering by event type and by address keeps client code simple.

## GET /api/stream

Returns a `text/event-stream` response. The connection stays open
indefinitely; the server emits an event for every new block at the tip,
every confirmed transaction, every chat message landing in the mempool
or getting confirmed, and every contract receipt.

### Required headers

```
Accept: text/event-stream
```

The browser's built-in `EventSource` adds this automatically.

### Cookie

Requires a session cookie. SSE connections inherit cookies from the
page that opened them.

### Event format

Each event is a JSON object on a single `data:` line, terminated by
a blank line. The standard SSE format:

```
data: {"type":"message","msg":{...}}

data: {"type":"message","msg":{...}}
```

### Query params (optional filters)

| Param | Format | Effect |
|---|---|---|
| `types` | comma-separated subset of event types | Only emit events of these types. Unknown values are silently ignored. |
| `address` | 32-char chain-layer hex OR 40-char VM hex | Only emit events that touch this address (as sender / recipient / miner / contract / message author). Case-insensitive. |

Both filters are AND'd. Omit both to receive everything.

Examples:

```text
GET /api/stream?types=block
GET /api/stream?types=tx,receipt&address=34378b1ba5be9d0999acd60be3a8a1f1
```

### Event types

#### `message`

A chat-shaped transaction (data dict carries `memo`). The payload
mirrors what [`GET /api/messages`](chat.md#get-apimessages) returns:

```json
{
  "type": "message",
  "msg": {
    "sender": "alice",
    "sender_address": "34378b1b...",
    "address": "39848b50...",
    "text": "hello",
    "timestamp": 1780002999.123,
    "status": "confirmed",
    "block": 1042,
    "tx_id": "abc123..."
  }
}
```

#### `tx`

Any confirmed non-message transaction (transfer, contract deploy,
contract call):

```json
{
  "type": "tx",
  "tx": {
    "tx_id": "...",
    "sender": "34378b1b...",
    "recipient": "contract",
    "kind": "call",
    "to": "0x39c1a47bf68d0e2d9c43caaa10c1b2f3c4d5e6f7",
    "selector": 2,
    "arg_count": 1,
    "amount": 0,
    "fee": 0.01,
    "timestamp": 1780002999.0,
    "block_height": 1042
  }
}
```

#### `block`

A new block landed at the tip. One event per height; the SSE pumper
catches up across multiple new blocks as needed.

```json
{
  "type": "block",
  "block": {
    "height": 1042,
    "hash": "0000ab12...",
    "previous_hash": "0000fc99...",
    "timestamp": 1780002999.0,
    "tx_count": 3,
    "miner": "dc66f82c048f35144599737ed54ab702",
    "difficulty": 4,
    "quantum_verified": true
  }
}
```

#### `receipt`

A contract deploy or call produced a receipt. Same shape as
[Receipt format](../reference/receipt.md), with the originating `tx_id`
inlined:

```json
{
  "type": "receipt",
  "receipt": {
    "tx_id": "...",
    "type": "call",
    "success": true,
    "gas_used": 26305,
    "to": "39c1a47bf68d0e2d9c43caaa10c1b2f3c4d5e6f7",
    "return_data": "00000000...01",
    "logs": [],
    "error": null
  }
}
```

### Dedupe

Each event type has its own server-side dedupe window. Clients
should dedupe defensively on `tx_id` (for `message` / `tx` /
`receipt`) and `height` (for `block`) in case of restart.

### Example — browser

```javascript
const es = new EventSource('/api/stream');
es.onmessage = e => {
  const ev = JSON.parse(e.data);
  if (ev.type === 'message') {
    renderMessage(ev.msg);
  }
};
es.onerror = () => {
  // EventSource auto-reconnects with exponential backoff
};
```

### Example — Python

```python
import requests, json
r = requests.get('https://api.waveledger.net/api/stream',
                 cookies={'session': '...'}, stream=True)
for line in r.iter_lines():
    if line and line.startswith(b'data: '):
        event = json.loads(line[6:])
        print(event)
```

### Lifecycle

- Connection drops when the client disconnects, or after ~30 min of
  idle time (depending on reverse proxy timeouts; Fly's edge is
  configured for long-lived connections, nginx defaults to 60s and
  requires tuning).
- The server pushes a heartbeat comment (`: heartbeat\n\n`) every 30s
  to prevent intermediaries from timing out idle streams.
- On error or reconnect, clients should re-query
  [`GET /api/messages?limit=100`](chat.md#get-apimessages) to catch
  up any missed events.

### Backpressure

Each connected client has a 64-event in-memory queue. If the queue
fills (slow client), new events are dropped silently for that
connection only; the chain is unaffected.
