# Admin API

Admin endpoints require HTTP Basic auth using the `WAVELEDGER_ADMIN_USER`
+ `WAVELEDGER_ADMIN_PASSWORD` env vars set on the node. See
[Authentication](auth.md#http-basic-admin).

The UI for these endpoints lives at `/admin`.

## Pending signups

### GET /api/admin/pending

List signups awaiting approval.

```json
{
  "pending": [
    {
      "name": "alice",
      "signed_up_at": 1780002999.0,
      "ip": "1.2.3.4"
    }
  ]
}
```

Sorted oldest-first (FIFO queue).

---

## POST /api/admin/approve

Approve a pending signup. Creates a wallet, drips the [faucet](../concepts/economics.md)
(100 WAVE from the node's miner wallet), generates a login token.

```json
{ "name": "alice" }
```

Response:

```json
{
  "status": "approved",
  "name": "alice",
  "address": "34378b1b...",
  "login_token": "AWeRfZo5skP6HHGIzV6whU1q0HJFw6rN",
  "faucet_tx": "f5c43575f6b520c1"
}
```

Hand the `login_token` to the user (DM, email, in person). They use it
at `POST /api/login` or by visiting `/?name=alice&token=AWeRfZo5...`
(the landing page auto-redeems).

If the miner wallet is underfunded the faucet log emits `faucet underfunded`
and `faucet_tx` comes back `null` — the user is still approved, they just
have 0 WAVE until they receive some.

---

## POST /api/admin/block

Block a name. Revokes all sessions for that name, removes from
approved + pending sets.

```json
{ "name": "spammer" }
```

Response: `{"status":"blocked","name":"spammer"}`.

Blocked names are also removed from `approved`; if you want to "unblock"
later, currently requires editing in-memory state via a restart (and
re-approving). A first-class unblock endpoint is a [TODO](https://github.com/DosseyRichards/FERMI-Documentation/blob/main/TODO.md).

---

## Approved users

### GET /api/admin/users

List approved users with balance + login token (for re-sharing if a
user lost their original link).

```json
{
  "users": [
    {
      "name": "alice",
      "address": "34378b1b...",
      "balance": 99.998,
      "joined_at": 1780002999.0,
      "login_token": "AWeRfZo5skP6HHGIzV6whU1q0HJFw6rN"
    }
  ],
  "blocked": ["spammer"]
}
```

---

## Invite codes

Invite codes let approved users self-onboard without admin gating. Any
unauthenticated client that sends a valid invite code to `POST /api/signup`
gets auto-approved + faucet-credited + session-cookied — no admin
intervention required.

### POST /api/admin/invites/create

```json
{ "max_uses": 25 }
```

| Field | Default | Range |
|---|---|---|
| `max_uses` | 25 | 1..10000 |

Response:

```json
{
  "code": "WAVE-ABC123",
  "max_uses": 25,
  "signup_url": "/?invite=WAVE-ABC123"
}
```

The `signup_url` is what you share — anyone who opens it gets the
landing page with the invite code pre-filled.

Codes are 6-char with the prefix `WAVE-`, drawn from an alphabet that
excludes look-alikes (no `0/O`, no `1/I/L`).

### GET /api/admin/invites

List all codes ever issued.

```json
{
  "invites": [
    {
      "code": "WAVE-ABC123",
      "max_uses": 25,
      "used": 3,
      "remaining": 22,
      "created_at": 1780002999.0,
      "revoked": false
    }
  ]
}
```

### POST /api/admin/invites/revoke

Mark a code revoked. Future attempts to redeem return 403.

```json
{ "code": "WAVE-ABC123" }
```

Response: `{"status":"revoked","code":"WAVE-ABC123"}`.

Revoke does **not** undo signups that already used the code — they
remain approved with their faucet'd balance.

---

## Notes on the in-memory model

All admin state (pending queue, approved users, sessions, invite codes)
lives in **process memory** on the testnet node. A node restart
wipes all of it.

In a real deployment this gets promoted to SQLite. The TODO list at
[`TODO.md`](https://github.com/DosseyRichards/FERMI-Documentation/blob/main/TODO.md) tracks this. For the demo, the trade-off
is intentional: fewer moving parts, and the chain's data (which is
what actually matters) lives on disk via SQLite already.
