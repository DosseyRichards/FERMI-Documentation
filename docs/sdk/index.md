# SDK and examples

The official Python SDK ships in the WaveLedger source tree at
`clients/python/waveledger/`. One `Client` class covers every messenger
surface plus the SSE event stream. See
[Python client](python.md) for installation + the full API.

A JavaScript SDK is on the roadmap (the messenger is plain HTTP +
cookies + SSE, so `fetch` and `EventSource` work today — see the
[browser examples](javascript.md)).

!!! info "Base URL"

    Every code sample in this section talks to:

    ```text
    https://chat.waveledger.net
    ```

    Self-hosted nodes serve the same routes on `http://localhost:8081`
    by default (override with `--messenger-port`).

## Pick your path

<div class="grid cards" markdown>

-   :material-language-python:{ .lg .middle } **Python**

    ---

    Use the chain's own `crypto`, `core`, and `fourier` modules
    directly. The most ergonomic option, since the reference impl is
    Python.

    [:octicons-arrow-right-24: Python client](python.md)

-   :material-language-javascript:{ .lg .middle } **JavaScript / browser**

    ---

    REST + cookies. Wallet signing requires server-side help (ML-DSA-87
    in browser is on the roadmap via WASM).

    [:octicons-arrow-right-24: Browser](javascript.md)

-   :material-key:{ .lg .middle } **Signing transactions**

    ---

    What the canonical tx envelope looks like, what's signed, how to
    produce a valid signature outside the messenger.

    [:octicons-arrow-right-24: Signing](signing.md)

-   :material-broadcast:{ .lg .middle } **Subscribing to events**

    ---

    SSE for chat messages; polling patterns for everything else.

    [:octicons-arrow-right-24: Events](events.md)

</div>
