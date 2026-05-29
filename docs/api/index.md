# API Reference

!!! info "Base URL"

    On the public testnet, the canonical base URL is:

    ```text
    https://api.waveledger.net
    ```

    Every path in this section is relative to that base — so
    `/api/messages` means `https://api.waveledger.net/api/messages`.

    `chat.waveledger.net` and `seed.waveledger.net` resolve to the same
    Fly machine and serve the same routes; `api.` is the durable name
    for programmatic clients, while `chat.` is for the dApp UI and
    `seed.` is what other P2P peers dial. Any of the three works for
    HTTP clients.

    Self-hosted nodes serve the same routes on `http://localhost:8081`
    by default (override with `--messenger-port`).

Every WaveLedger node exposes three HTTP APIs:

| API | Default port | Auth | Audience |
|---|---|---|---|
| **Messenger / dApp** | 8081 | Session cookie | End users (chat, wallet, playground) |
| **Explorer** | 8081 (same as messenger) | None | Anyone (public chain data) |
| **Dashboard** | 8080 (loopback) | API key | Node operator / monitoring |

On the public testnet, the messenger + explorer are reached via
[`https://api.waveledger.net`](https://api.waveledger.net) (Fly's
edge does TLS termination + proxies to port 8081). The dashboard
stays on loopback by default and is reached via `fly ssh console`.

## Conventions

All endpoints use JSON request/response bodies unless noted (e.g.
`/api/random/bytes` returns raw bytes; `/api/stream` returns SSE
text events).

Status codes:

| Code | Meaning |
|---|---|
| `200` | Success |
| `400` | Bad request (invalid JSON, missing field, validation fail) |
| `401` | Not authenticated (session missing, login token wrong) |
| `403` | Not authorized (admin endpoint with non-admin creds; blocked name) |
| `404` | Resource not found |
| `409` | Conflict (name already taken) |
| `429` | Rate limited |
| `500` | Server error (file a bug + include the response body) |

Errors come back as:

```json
{ "error": "amount must be > 0" }
```

Some errors include extra fields — e.g. `phase: "compile"` on
playground errors, `hint: "..."` on auth errors.

## Authentication models

Three different auth schemes, one per audience:

| Endpoint family | Auth |
|---|---|
| `/api/signup`, `/api/login`, `/api/explorer/*` | None (public) |
| `/api/me`, `/api/wallet/*`, `/api/send`, `/api/stream`, `/api/playground/*` | Session cookie (`session=...`) |
| `/api/admin/*` | HTTP Basic (`WAVELEDGER_ADMIN_PASSWORD` env on node) |
| Dashboard `/api/*` | Bearer API key (loopback-only by default) |

See [Authentication](auth.md) for details.

## API groups

<div class="grid cards" markdown>

-   :material-account-circle:{ .lg .middle } **Wallet**

    ---

    Address, balance, sign and send txs, encrypted backup download
    + restore.

    [:octicons-arrow-right-24: Wallet endpoints](wallet.md)

-   :material-message-text:{ .lg .middle } **Chat / Messenger**

    ---

    Submit chat messages (on-chain), list messages, signup + login
    flow.

    [:octicons-arrow-right-24: Chat endpoints](chat.md)

-   :material-code-braces:{ .lg .middle } **Playground**

    ---

    Compile Fourier source, deploy contracts, call methods, poll
    receipts, list your deployed contracts.

    [:octicons-arrow-right-24: Playground endpoints](playground.md)

-   :material-magnify:{ .lg .middle } **Explorer**

    ---

    Public chain data: blocks (paginated), transactions, addresses,
    chain stats. No auth.

    [:octicons-arrow-right-24: Explorer endpoints](explorer.md)

-   :material-shield-account:{ .lg .middle } **Admin**

    ---

    Pending signup queue, approval, blocking, invite-code generation.
    HTTP Basic auth.

    [:octicons-arrow-right-24: Admin endpoints](admin.md)

-   :material-chart-line:{ .lg .middle } **Dashboard**

    ---

    Node-local monitoring (chain info, peer list, mempool, mining
    state). Bearer-key auth.

    [:octicons-arrow-right-24: Dashboard endpoints](dashboard.md)

-   :material-broadcast:{ .lg .middle } **Server-Sent Events**

    ---

    Live message stream for the chat dApp. Standard SSE.

    [:octicons-arrow-right-24: SSE](sse.md)

</div>
