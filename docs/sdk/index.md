# SDK and examples

Two official client libraries ship the same `Client` surface — auth,
chat, wallet, explorer, playground, admin, and the SSE event stream.

| Language | Install | Package |
|---|---|---|
| Python | `pip install waveledger-sdk` | [PyPI](https://pypi.org/project/waveledger-sdk/) |
| Node / browser (TS) | `npm install waveledger-sdk` | [npm](https://www.npmjs.com/package/waveledger-sdk) |

Both are MIT-licensed, dependency-light (Python: just `requests`; Node:
zero runtime deps), and track the messenger REST API one-to-one. See
[Python client](python.md) and [JavaScript client](javascript.md) for
the full surface.

!!! info "Base URL"

    Every code sample in this section talks to:

    ```text
    https://api.waveledger.net
    ```

    Self-hosted nodes serve the same routes on `http://localhost:8081`
    by default (override with `--messenger-port`).

## Pick your path

<div class="grid cards" markdown>

-   :material-language-python:{ .lg .middle } **Python**

    ---

    `pip install waveledger-sdk`. Uses `requests` under the hood;
    importable as `from waveledger import Client`.

    [:octicons-arrow-right-24: Python client](python.md)

-   :material-language-javascript:{ .lg .middle } **JavaScript / TypeScript**

    ---

    `npm install waveledger-sdk`. Pure ESM, zero runtime deps, native
    `fetch` + `ReadableStream`. Works in Node 18+ and modern browsers.

    [:octicons-arrow-right-24: JavaScript client](javascript.md)

-   :material-key:{ .lg .middle } **Signing transactions**

    ---

    What the canonical tx envelope looks like, what's signed, how to
    produce a valid signature outside the messenger.

    [:octicons-arrow-right-24: Signing](signing.md)

-   :material-broadcast:{ .lg .middle } **Subscribing to events**

    ---

    SSE for block, tx, message, and receipt events; server-side
    filtering by type and address.

    [:octicons-arrow-right-24: Events](events.md)

</div>

## Release notes

| Version | Date | Notes |
|---|---|---|
| `0.1.1` | 2026-05 | Metadata bump: package URLs now point at `docs.fermi.world` (the canonical chain-docs host). No behavior changes. |
| `0.1.0` | 2026-05 | Initial release on both registries. Covers messenger, wallet, explorer, playground, admin, and SSE. Pre-1.0; the API tracks the REST surface, which is itself moving. |
