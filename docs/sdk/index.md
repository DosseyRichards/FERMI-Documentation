# SDK and examples

WaveLedger has no published SDK package yet. Until then, every API
operation is reachable via standard HTTP + JSON, and the chain's
internals are importable directly from the Python source.

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
