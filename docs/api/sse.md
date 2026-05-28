# Server-Sent Events

The chat dApp pushes new messages to connected clients via a standard
SSE (Server-Sent Events) stream. Use this when you want a live feed
without polling.

## GET /api/stream

Returns a `text/event-stream` response. The connection stays open
indefinitely; the server emits events as new messages land in the
mempool or get confirmed.

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

### Event types

#### `message`

A new message appeared in the mempool, OR an existing pending message
was confirmed. The `msg` payload matches what
[`GET /api/messages`](chat.md#get-apimessages) returns:

```json
{
  "type": "message",
  "msg": {
    "sender": "alice",
    "address": "34378b1b...",
    "text": "hello",
    "timestamp": 1780002999.123,
    "status": "pending",
    "block": null,
    "tx_id": "abc123..."
  }
}
```

The same `tx_id` will emit again with `status: "confirmed"` once mined.
Clients should de-duplicate on `(tx_id, status)`.

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
r = requests.get('https://chat.waveledger.net/api/stream',
                 cookies={'session': '...'}, stream=True)
for line in r.iter_lines():
    if line and line.startswith(b'data: '):
        event = json.loads(line[6:])
        print(event)
```

### Lifecycle

- Connection drops when the client disconnects, or after ~30 min of idle
  (depends on reverse proxy timeouts — fly's edge is configured for
  long-lived connections; nginx defaults to 60s and needs tuning).
- The server pushes a heartbeat comment (`: heartbeat\n\n`) every 30s
  so intermediaries don't time out idle streams.
- On error or reconnect, clients should re-query
  [`GET /api/messages?limit=100`](chat.md#get-apimessages) to catch
  up any events they missed.

### Backpressure

Each connected client has a 64-event in-memory queue. If the queue
fills (slow client), new events are dropped silently for that
connection only — the chain isn't affected.
