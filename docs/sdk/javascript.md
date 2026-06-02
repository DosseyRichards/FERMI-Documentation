# JavaScript / Node.js client

The `waveledger` package ships in the source tree at
`clients/typescript/`. A single `Client` class — TypeScript-typed,
ESM-native, dependency-free — runs in Node 18+ and modern browsers
(Chrome, Safari, Firefox).

!!! info "Base URL"

    Default: `https://api.waveledger.net`. Override via `baseUrl`
    for self-hosted nodes (`http://localhost:8081`) or to use the
    legacy `https://chat.waveledger.net` host.

## Install

```bash
npm install waveledger-sdk
```

Or, from source:

```bash
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git
cd Fermi-Mining-ASIC-Software/clients/typescript
npm install && npm run build
```

## Quickstart

```ts
import { Client } from "waveledger-sdk";

const c = new Client();   // defaults to https://api.waveledger.net

await c.signup("alice", { inviteCode: "WAVE-ABC123" });
await c.sendMessage("hello world");

for await (const ev of c.subscribe({ types: ["block"] })) {
  console.log(`block ${ev.block.height}`);
}
```

## API surface

```ts
// Auth
c.signup(name, { inviteCode? })
c.login(name, token)
c.me()
c.logout()
c.session                            // current cookie value

// Chat
c.sendMessage(text)                  // returns { tx_id, status }
c.messages(limit?)

// Wallet
c.wallet()
c.walletSend({ to, amount, memo? })
c.walletExport(passphrase)
c.walletImport({ name, encrypted, passphrase })

// Explorer (public)
c.explorer.stats()
c.explorer.blocks({ limit?, offset? })
c.explorer.block(height)
c.explorer.tx(txId)
c.explorer.address(addr, { limit?, offset? })

// Playground
c.playground.compile(source)
c.playground.deploy(source)
c.playground.call({ contract, method, args })
c.playground.receipt(txId)
c.playground.contracts()

// Admin (HTTP Basic at construction)
const ac = new Client({ admin: { user: "admin", password: "PASSWORD" } });
ac.admin.pending()
ac.admin.approve(name)
ac.admin.block(name, reason?)
ac.admin.unblock(name)
ac.admin.inviteCreate({ maxUses })
ac.admin.inviteRevoke(code)
ac.admin.tokenCreate({ label, name, scope? })
ac.admin.tokensList()
ac.admin.tokenRevoke({ token?, tokenHash? })

// SSE event stream
for await (const ev of c.subscribe({ types?, address?, signal? })) {
  // ev.type: "block" | "tx" | "message" | "receipt"
}
```

## Filtering events

Server-side filtering — the messenger only emits events that match the
`types=` and `address=` query parameters:

```ts
for await (const ev of c.subscribe({ types: ["block"] })) {
  console.log(ev.block.height);
}

for await (const ev of c.subscribe({
  address: "34378b1ba5be9d0999acd60be3a8a1f1",
})) {
  console.log(ev.type, ev);
}
```

Combine both, and pass `signal` to abort early:

```ts
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 60_000);

for await (const ev of c.subscribe({
  types: ["tx", "receipt"],
  address: "34378b1ba5be9d0999acd60be3a8a1f1",
  signal: ctrl.signal,
})) { /* ... */ }
```

## Errors

```ts
import {
  AuthError, NotFoundError, RateLimitedError,
  ValidationError, ServerError, WaveLedgerError,
} from "waveledger-sdk";

try {
  await c.walletSend({ to: "bad", amount: -1 });
} catch (e) {
  if (e instanceof ValidationError) {
    console.log(e.status, e.payload);
  } else if (e instanceof RateLimitedError) {
    await new Promise(r => setTimeout(r, 2000));
  }
}
```

| HTTP | Exception |
|---|---|
| 400 | `ValidationError` |
| 401 / 403 | `AuthError` |
| 404 | `NotFoundError` |
| 429 | `RateLimitedError` |
| 5xx | `ServerError` |

## API token auth { #api-token-auth }

For CI pipelines and unattended workloads, an administrator can mint
a Bearer token bound to an approved user. The token's user owns
contracts deployed or called through it; that user's wallet pays the
fees.

```ts
// Operator (one-off):
const admin = new Client({
  admin: { user: "admin", password: "PASSWORD" },
});
await admin.admin.approve("ci-bot");
const { token } = await admin.admin.tokenCreate({
  label: "release-pipeline",
  name:  "ci-bot",
});
console.log(token);                  // wlg_… — save this once
```

```ts
// Pipeline:
const c = new Client({ apiToken: "wlg_…" });
await c.playground.deploy(source);   // signed under ci-bot's wallet
```

The raw token is returned once and stored only as its SHA3-256 hash.
Revocation: `admin.admin.tokenRevoke({ token: "wlg_…" })`.

The TypeScript SDK does not ship a local Fourier compiler in v1.
Programmatic compilation goes through
`c.playground.compile(source)`, which is unauthenticated when no
Bearer token is configured.

## Browser usage

The client uses `globalThis.fetch` and `ReadableStream`. It runs in
modern browsers as-is. The `subscribe()` iterator works with the
browser's native fetch streaming.

CORS is handled server-side: the messenger answers preflight `OPTIONS`
requests and sets `Access-Control-Allow-Origin: *` on every response,
so cross-origin calls from a dApp work without additional configuration.

## Persisted sessions

```ts
// Save
const token = c.session;
localStorage.setItem("waveledger-session", token!);

// Restore
const saved = localStorage.getItem("waveledger-session");
const c = new Client({ session: saved ?? undefined });
```

Server-side, sessions persist across node restarts (SQLite-backed in
the [admin store](../api/admin.md#persistence)).

## Build + test

```bash
cd clients/typescript
npm install      # only to run the test runner
npm run build    # tsc → dist/
npm test         # node:test, 25 tests, no extra deps
```

## Versioning

Pre-1.0. Method names and response shapes track the REST API. Latest
release: [`waveledger-sdk@0.2.0`](https://www.npmjs.com/package/waveledger-sdk).
