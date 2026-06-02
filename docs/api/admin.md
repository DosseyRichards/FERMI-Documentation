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

Deliver the `login_token` to the user (DM, email, in person). The
user redeems it at `POST /api/login` or by visiting
`/?name=alice&token=AWeRfZo5...` (the landing page auto-redeems).

If the miner wallet is underfunded the faucet log emits `faucet
underfunded` and `faucet_tx` returns `null`. The user is still
approved but holds 0 WAVE until receiving funds.

---

## POST /api/admin/block

Block a name. Revokes all sessions for that name, removes from
approved + pending sets.

```json
{ "name": "spammer" }
```

Response: `{"status":"blocked","name":"spammer"}`. Optionally include
`reason` in the body; it is stored on the blocked record for the
audit trail.

Blocked names are removed from `approved` and `pending`, and their
active sessions are revoked. To allow the name back, use
[`POST /api/admin/unblock`](#post-apiadminunblock); the user must then
re-sign-up to obtain a fresh wallet and login token.

## POST /api/admin/unblock

```json
{ "name": "spammer" }
```

Response: `{"status":"unblocked","name":"spammer"}` (200), or
`{"error":"not blocked"}` (404) if the name wasn't on the blocked list.

---

## Approved users

### GET /api/admin/users

List approved users with balance and login token (for re-sharing if
a user lost the original link).

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

Invite codes let users self-onboard without admin gating. Any
unauthenticated client that sends a valid invite code to
`POST /api/signup` is auto-approved, faucet-credited, and
session-cookied without admin intervention.

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

The `signup_url` is the shareable link; opening it loads the landing
page with the invite code pre-filled.

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

Revoke does **not** undo signups that already used the code; those
users remain approved with their faucet-credited balance.

---

## API tokens

Bearer tokens for programmatic access to the playground endpoints.
The token's bound user owns any contracts deployed or called through
the token; the user's wallet pays the deploy and call fees.

### POST /api/admin/tokens/create

```json
{
  "label": "ci-bot",
  "name":  "alice",
  "scope": "playground"
}
```

| Field | Required | Notes |
|---|---|---|
| `label` | yes | Human-readable, 1..64 chars |
| `name` | yes | Approved user the token acts as; must already exist in `approved` |
| `scope` | no | Comma-separated. Default `playground`. Only `playground` is recognized today. |

Response:

```json
{
  "token": "wlg_<64 hex>",
  "label": "ci-bot",
  "name":  "alice",
  "scope": "playground",
  "note":  "Save this token. The server stores only its hash and cannot recover the raw value."
}
```

The raw `token` is returned **once**. The server stores only its
SHA3-256 hash, so a lost token can only be replaced — never recovered.

### GET /api/admin/tokens

```json
{
  "tokens": [
    {
      "label": "ci-bot",
      "name":  "alice",
      "scope": "playground",
      "created_at":   1780100000.0,
      "last_used_at": 1780100500.0,
      "revoked_at":   null
    }
  ]
}
```

`token_hash` is internal and never returned.

### POST /api/admin/tokens/revoke

Revoke by raw token (server hashes) or by `token_hash` (useful when
revoking from the listing above):

```json
{ "token":      "wlg_..." }
```

```json
{ "token_hash": "<64 hex>" }
```

Response: `{ "status": "revoked", "token_hash": "<64 hex>" }`.

### Authenticating with a token

Pass `Authorization: Bearer wlg_...` on any
[playground endpoint](playground.md). The SDKs handle this for you:

```python
from waveledger import Client
c = Client("https://api.waveledger.net", api_token="wlg_...")
c.playground.deploy(source)
```

```typescript
import { Client } from "waveledger-sdk";
const c = new Client({ apiToken: "wlg_..." });
await c.playground.deploy(source);
```

If both a session cookie and a Bearer token are presented, the session
cookie wins — interactive sessions take precedence over service
accounts.

---

## Persistence

Admin state — pending queue, approved users, blocked names, active
sessions, and invite codes — is persisted to SQLite at
`{data_dir}/admin.db` alongside the chain DB. Every mutation writes
through synchronously; reads come from the in-memory mirror for
speed. Effects:

- Node restarts do **not** invalidate session cookies, drop the
  pending queue, void invite codes, or undo blocks.
- A clean re-sync of the chain data does not disturb admin state
  (`chain.db` and `admin.db` are separate files).
- `admin.db` is a standard SQLite file, inspectable with
  `sqlite3 /data/admin.db` for debugging.

Tables (see `api/admin_store.py`):

```text
approved  (name PK, address, login_token, joined_at, extra_json)
pending   (name PK, signed_up_at, ip)
blocked   (name PK, blocked_at, reason)
sessions  (token PK, name, created_at, last_seen)
invites   (code PK, max_uses, used, created_at, revoked, note)
```
