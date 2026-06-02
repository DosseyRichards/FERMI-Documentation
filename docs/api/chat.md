# Chat / Messenger API

The chat dApp is a thin layer over on-chain transactions: every
message is an ML-DSA-87-signed transfer of 0 WAVE with the message
body in `tx.data.memo`. The "chat room" is a view of the chain.

## Onboarding

### POST /api/signup

Request access. Two paths:

**Path A — admin gate (no invite code):** the signup lands in a
pending queue. An admin manually approves via the [admin API](admin.md).

**Path B — invite code (instant):** if the request includes a valid
`invite_code`, the signup is auto-approved on the spot. A wallet is
generated, the [faucet](../concepts/economics.md) drips 100 WAVE, a
session cookie is set, and the response carries the login token for
logging in from another device.

#### Request

```json
{
  "name": "alice",
  "invite_code": "WAVE-ABC123"
}
```

| Field | Required | Notes |
|---|---|---|
| `name` | yes | 2–32 chars, `[A-Za-z0-9_-]` only |
| `invite_code` | no | If provided and valid, path B; otherwise path A |

#### Response (path A — pending)

```json
{
  "status": "pending",
  "message": "an admin will review your request shortly"
}
```

#### Response (path B — auto-approved)

```json
{
  "status": "approved",
  "auto": true,
  "name": "alice",
  "address": "34378b1ba5be9d0999acd60be3a8a1f1",
  "login_token": "AWeRfZo5skP6HHGIzV6whU1q0HJFw6rN",
  "invite_code": "WAVE-ABC123"
}
```

`Set-Cookie: session=...` is sent with this response. The browser is
already logged in and can redirect to `/chat`.

#### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"name must be 2-32 chars, alphanumeric / - / _"}` |
| 403 | `{"error":"this name is blocked"}` |
| 403 | `{"error":"invalid or revoked invite code"}` |
| 403 | `{"error":"invite code has reached its usage limit"}` |
| 409 | `{"error":"name already taken"}` |

---

### POST /api/login

Log in as an approved user using their login token.

#### Request

```json
{
  "name": "alice",
  "token": "AWeRfZo5skP6HHGIzV6whU1q0HJFw6rN"
}
```

#### Response

```json
{ "status": "ok", "name": "alice" }
```

`Set-Cookie: session=...` is set.

#### Errors

| Status | Body |
|---|---|
| 400 | `{"error":"name and token required"}` |
| 401 | `{"error":"unknown name or wrong token"}` |
| 403 | `{"error":"this name has been blocked"}` |

---

### GET /api/me

Returns the current logged-in user's name, address, and balance.

#### Response

```json
{
  "name": "alice",
  "address": "34378b1ba5be9d0999acd60be3a8a1f1",
  "balance": 99.998
}
```

Used by the UI to populate the header. Returns 401 if no session is
present.

---

## Sending + reading messages

### POST /api/send

Submit a chat message. The server builds a transaction, signs it with
the user's wallet, and submits it to the mempool.

#### Request

```json
{ "text": "hello world" }
```

| Field | Required | Notes |
|---|---|---|
| `text` | yes | 1–512 chars. Stored in `tx.data.memo`. |

#### Response

```json
{
  "status": "submitted",
  "tx_id": "abc123...",
  "pending": true
}
```

#### Rate limit

**1 message per 5 seconds per address.** Exceeding this returns:

```http
HTTP/1.1 429 Too Many Requests
```
```json
{ "error": "rate limit: wait N seconds" }
```

The window is per-address; sessions are not used as the key, so
opening additional sessions does not bypass the limit.

---

### GET /api/messages

Returns the most recent messages (confirmed + pending), newest first.

#### Query params

| Param | Default | Max | Notes |
|---|---|---|---|
| `limit` | 50 | 200 | Number of messages |

#### Response

```json
{
  "count": 42,
  "messages": [
    {
      "sender": "alice",
      "address": "34378b1b...",
      "text": "hello world",
      "timestamp": 1780002999.123,
      "status": "confirmed",
      "block": 42,
      "tx_id": "abc123..."
    },
    {
      "sender": "39848b500426d8f1...",
      "address": "39848b50...",
      "text": "pending one",
      "timestamp": 1780003001.5,
      "status": "pending",
      "block": null,
      "tx_id": "def456..."
    }
  ]
}
```

| Field | Notes |
|---|---|
| `sender` | Display name if known, else a shortened address |
| `address` | Full sender address |
| `status` | `"confirmed"` (in a block) or `"pending"` (mempool only) |
| `block` | Block height or `null` if pending |
| `tx_id` | First 16 chars (full id available via `/api/explorer/tx/{id}`) |

Messages are identified by walking the chain backwards for
`tx.data.memo` entries, then prepending the mempool. They are not
stored in a separate table.

---

## Real-time updates

See [Server-Sent Events](sse.md) for the `/api/stream` endpoint that
pushes new messages to connected clients as they appear.
