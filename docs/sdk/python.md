# Python client

The `waveledger` package ships in the source tree at
`clients/python/waveledger/`. A single `Client` class wraps every
messenger surface (auth, chat, wallet, explorer, playground, admin)
plus the SSE event stream.

!!! info "Base URL"

    Every example below talks to `https://api.waveledger.net`. Point
    the `Client` at any other host (e.g. `http://localhost:8081` for a
    self-hosted node) and the same code works.

## Install

```bash
pip install waveledger-sdk
```

The PyPI distribution is `waveledger-sdk`; the importable module is
`waveledger`. Imports remain `from waveledger import Client`.

From the WaveLedger source tree:

```bash
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git
cd Fermi-Mining-ASIC-Software
pip install -e clients/python
```

Or vendor `clients/python/waveledger/` directly — pure Python with one
dependency (`requests`).

```bash
pip install requests
cp -r clients/python/waveledger ./vendor/
```

## Quickstart

```python
from waveledger import Client

c = Client("https://api.waveledger.net")

# Sign up with an invite (instant approval + 100 testnet WAVE)
c.signup("alice", invite_code="WAVE-ABC123")

# Post a chat message — real on-chain ML-DSA-87 tx
c.send_message("hello world")

# Send WAVE
c.wallet_send(to="34378b1ba5be9d0999acd60be3a8a1f1", amount=1.0)

# Live event stream — block, tx, message, receipt
for ev in c.subscribe(types=["block"]):
    b = ev["block"]
    print(f"block {b['height']} from {b['miner'][:16]}")
```

## API surface

```python
# Auth / session
c.signup(name, invite_code=None)        # request access
c.login(name, token)                    # activate via one-shot token
c.me()                                  # who am i?
c.logout()                              # client-side cookie clear

# Chat
c.send_message(text)                    # post on-chain message
c.messages(limit=50)                    # recent messages

# Wallet
c.wallet()                              # balance + recent txs
c.wallet_send(to=..., amount=..., memo=None)
c.wallet_export(passphrase=...)         # AES-256-GCM + Argon2id bundle
c.wallet_import(name=..., encrypted=..., passphrase=...)

# Explorer (public, no auth)
c.explorer.stats()
c.explorer.blocks(limit=25, offset=0)
c.explorer.block(height)
c.explorer.tx(tx_id)
c.explorer.address(address)

# Playground (Fourier)
c.playground.compile(source)            # server compile → {bytecode_hex, abi}
c.playground.compile_local(source)      # local compile — no network call
c.playground.deploy(source)             # → {tx_id, abi}
c.playground.call(contract=..., method=..., args=[...])
c.playground.receipt(tx_id)
c.playground.contracts()

# Admin (HTTP Basic — pass admin=(user, pw) at construction)
ac = Client("https://api.waveledger.net", admin=("admin", "PASSWORD"))
ac.admin.pending()
ac.admin.approve(name)
ac.admin.block(name, reason=None)
ac.admin.unblock(name)
ac.admin.invite_create(max_uses=25)
ac.admin.invite_revoke(code)
ac.admin.invites_list()
ac.admin.token_create(label="ci", name="alice", scope="playground")
ac.admin.tokens_list()
ac.admin.token_revoke(token="wlg_...")

# SSE event stream (block / tx / message / receipt)
for ev in c.subscribe(types=None, address=None):
    handle(ev)
```

## Filtering events

Filtering is **server-side** — the messenger only emits events that
match the `?types=` and `?address=` filters:

```python
# Every block as it lands
for ev in c.subscribe(types=["block"]):
    print(ev["block"]["height"])

# Everything that touches one address
for ev in c.subscribe(address="34378b1ba5be9d0999acd60be3a8a1f1"):
    print(ev["type"], ev)

# Both filters AND'd
for ev in c.subscribe(types=["tx", "receipt"],
                       address="34378b1ba5be9d0999acd60be3a8a1f1"):
    print(ev)
```

For a production indexer, wrap the loop in
`while True: try: ... except ConnectionError: ...`. The underlying HTTP
connection drops occasionally on long-running streams; reconnecting at
the SSE level is safe because the server's event window covers the
last ~1000 IDs per type.

## Errors

```python
from waveledger import (
    Client,
    AuthError, NotFoundError, RateLimitedError,
    ValidationError, ServerError, WaveLedgerError,
)

try:
    c.wallet_send(to="bad", amount=-1)
except ValidationError as e:
    print(e.status, e.payload)        # 400, {'error': '...'}
except RateLimitedError:
    time.sleep(2)                     # back off, retry
```

| HTTP | Exception |
|---|---|
| 400 | `ValidationError` |
| 401 / 403 | `AuthError` |
| 404 | `NotFoundError` |
| 429 | `RateLimitedError` |
| 5xx | `ServerError` |

Every exception carries `.status` (int) and `.payload` (decoded JSON if
any).

## API token auth { #api-token-auth }

For CI pipelines and other unattended use, an administrator can mint a
Bearer token bound to an approved user. The token's user owns any
contracts deployed or called through it; that user's wallet pays the
fees.

```python
# Operator side (one-off):
admin_c = Client("https://api.waveledger.net",
                 admin=("admin", "PASSWORD"))
admin_c.admin.approve("ci-bot")   # if not already approved
out = admin_c.admin.token_create(label="release-pipeline",
                                  name="ci-bot",
                                  scope="playground")
print(out["token"])               # wlg_... — save this once
```

```python
# Pipeline side:
c = Client("https://api.waveledger.net", api_token="wlg_…")
c.playground.deploy(source)       # signed under ci-bot's wallet
```

Tokens are stored as their SHA3-256 hash; the raw value is returned
once and never recoverable. Revocation:
`admin_c.admin.token_revoke(token="wlg_…")`.

## Local Fourier compile { #local-compile }

The Python package vendors the Fourier compiler at
`waveledger._fourier`. `compile_local()` runs in-process and returns
the same shape as the server compile:

```python
out = c.playground.compile_local("""
    contract Counter {
        storage value: uint @ 0;
        pub fn inc() -> uint { value = value + 1; return value; }
    }
""")
print(out["bytecode_size"], out["abi"]["contract"])
```

Useful for:

- **CI builds** that produce bytecode artifacts without network access
- **Local iteration** with no round-trip to the server
- **Reproducibility** — the vendored compiler is pinned with the SDK
  release, so a given `waveledger-sdk` version produces byte-identical
  bytecode regardless of when or where it runs

`compile_local` raises `ValidationError` (status 400, payload
`{"error": ..., "phase": "compile"}`) for lex, parse, codegen, and ABI
errors, mirroring the response shape the server returns from
`compile()`.

## Persisted sessions

The session cookie is stored on the client's `requests.Session`. To
resume a session across restarts, save the cookie value and pass it
back at construction:

```python
# First run
c = Client("https://api.waveledger.net")
c.signup("alice", invite_code="WAVE-ABC123")
token = c._http.cookies.get("session")   # save somewhere safe
```

```python
# Later
c = Client("https://api.waveledger.net", session=token)
c.me()                                    # still logged in
```

Server-side, sessions persist across node restarts (SQLite-backed in
the [admin store](../api/admin.md#persistence)).

## Building from chain primitives

The Python `Client` covers the messenger API. To address the chain at
a lower level — offline tx signing, direct SQLite DB reads, scripted
mempool interaction — import the relevant modules from the WaveLedger
source tree:

```python
from crypto.kyber_crypto import WaveLedgerCrypto
from core.blockchain     import Transaction
from core.contract_engine import build_deploy_tx_data, build_call_tx_data
from fourier             import compile_source
from vm                  import VM, WorldState, Env
```

These are the same modules used by the messenger and miner; the public
surface is stable per the [reference docs](../reference/index.md).

## Tests

```bash
cd clients/python
python3 -m pytest tests/ -q
```

The suite uses a mock `requests.Session` — no network, no fixtures, no
additional test dependencies.

## Versioning

Pre-1.0. Method names and response shapes track the REST API. Latest
release: [`waveledger-sdk 0.2.0`](https://pypi.org/project/waveledger-sdk/).
