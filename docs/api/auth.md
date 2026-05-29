# Authentication

WaveLedger uses **three distinct auth schemes** depending on the
audience.

## Session cookies (end users)

Set when the user logs in via `POST /api/login` (or self-onboards via
`POST /api/signup` with a valid invite code). The cookie:

| | Value |
|---|---|
| Name | `session` |
| Format | URL-safe random 24-byte token (`secrets.token_urlsafe`) |
| HttpOnly | yes |
| SameSite | Lax |
| Lifetime | Until restart (in-memory store on the testnet) |

Sessions are stored in process memory in the testnet build — a node
restart wipes them. End users have to re-login. (This is testnet
ergonomics; mainnet will persist sessions.)

To attach the cookie from `curl`:

```bash
curl -c cookies.jar -X POST https://api.waveledger.net/api/login \
  -H 'Content-Type: application/json' \
  -d '{"name":"alice","token":"login-token-from-admin"}'

# Subsequent requests
curl -b cookies.jar https://api.waveledger.net/api/me
```

## HTTP Basic (admin)

The admin endpoints (`/api/admin/*` and the `/admin` page) require
HTTP Basic auth using credentials set via env vars on the node:

```bash
WAVELEDGER_ADMIN_USER=admin
WAVELEDGER_ADMIN_PASSWORD=<your-strong-password>
```

If `WAVELEDGER_ADMIN_PASSWORD` is unset, the password defaults to
the literal string `"change-me-via-env"` — fine for local dev,
**never** acceptable in production. The node logs a warning on
startup if the default is in use.

Example:

```bash
curl -u admin:your-password https://api.waveledger.net/api/admin/pending
```

The browser will prompt for credentials when you visit `/admin`. Most
browsers cache them for the rest of the tab session.

## Bearer API key (dashboard, node-local)

The dashboard runs on port 8080 of every node, bound to loopback
(`127.0.0.1`) by default. It is **not** reachable from the public
internet unless you change the bind host or front it with a reverse
proxy + key check.

API key:

- Generated on first node boot (`auth.api_key_manager.APIKeyManager`)
- Stored at `{data_dir}/api_key.json` with mode `0600`
- Read by middleware on every dashboard write request

Read endpoints (`GET /api/*`) are public-by-default on the dashboard
unless the node was started with `--require-auth`. Write endpoints
(`POST/PUT/DELETE`) always require:

```http
Authorization: Bearer <api-key>
```

```bash
# Get your API key (run inside the container)
fly ssh console --config fly.chat.toml -C 'cat /data/api_key.json'

# Use it
KEY=$(fly ssh console --config fly.chat.toml -C 'cat /data/api_key.json' | jq -r .key)
curl -H "Authorization: Bearer $KEY" \
  http://127.0.0.1:8080/api/mining
```

The dashboard's API key is local to that node — it's never broadcast
to peers and changing it doesn't affect the chain.

## No auth (public chain data)

The explorer endpoints (`/api/explorer/*`) are **unauthenticated** by
design. Blockchain data is public; pretending otherwise is theater.

Likewise the signup + login endpoints take no auth (login itself is
the auth-grant step).

## CORS

All JSON responses include:

```
Access-Control-Allow-Origin: *
```

This is intentional — third-party explorers, monitoring dashboards,
and SDKs need to be able to read chain data from any origin. No
preflight handling is included; if you need OPTIONS-preflight support,
front the node with a reverse proxy that adds it.

## Future: signed-message login

The roadmap calls for **wallet-signed-nonce** login, where the user
proves ownership of a private key by signing a server-issued nonce
with their ML-DSA-87 keypair. That removes the need for server-side
session storage entirely. Currently blocked on shipping browser-side
ML-DSA via WASM.
