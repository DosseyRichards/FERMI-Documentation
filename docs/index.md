---
title: WaveLedger Documentation
hide:
  - navigation
  - toc
---

# WaveLedger

A post-quantum Layer-1 blockchain. Every transaction is signed with
**ML-DSA-87** (NIST FIPS 204), every key exchange uses **ML-KEM-1024**
(FIPS 203), every hash is **SHA3-512**. No classical cryptography
anywhere in the chain.

Mining requires a verifiable entropy attestation — the testnet uses the
[drand](https://drand.love) federated beacon, with a vendor-agnostic
contract that lets us swap in QRNG hardware (or any other source) without
forking the chain.

[Live testnet :material-arrow-right:](https://chat.waveledger.net){ .md-button .md-button--primary }
[Source on GitHub :material-arrow-right:](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software){ .md-button }

---

## What's in these docs

<div class="grid cards" markdown>

-   :material-book-open-page-variant:{ .lg .middle } **Concepts**

    ---

    The chain from first principles — post-quantum crypto, block
    structure, mining + entropy, economics, addresses, networking.

    [:octicons-arrow-right-24: Read the concepts](concepts/)

-   :material-api:{ .lg .middle } **API Reference**

    ---

    Every public REST endpoint: wallet, chat, contract playground,
    explorer, admin. With request/response examples for each.

    [:octicons-arrow-right-24: Browse the API](api/)

-   :material-server:{ .lg .middle } **Running a node**

    ---

    Run your own miner, seed, or entropy aggregator. Fly.io and
    bare-VPS guides. Operational runbook for production.

    [:octicons-arrow-right-24: Node operator guide](nodes/)

-   :material-code-braces:{ .lg .middle } **SDK and examples**

    ---

    Send WAVE, deploy contracts, subscribe to events — from Python
    and from the browser. Working snippets, not just signatures.

    [:octicons-arrow-right-24: Get coding](sdk/)

-   :material-file-document:{ .lg .middle } **Reference**

    ---

    Wire formats for blocks, transactions, receipts, addresses;
    network parameters; the full crypto primitive list.

    [:octicons-arrow-right-24: Format specs](reference/)

-   :material-script-text-outline:{ .lg .middle } **Fourier (smart contracts)**

    ---

    The smart-contract language gets its own site —
    [fourier.fermi.world](https://fourier.fermi.world).

    [:octicons-arrow-right-24: Fourier docs](https://fourier.fermi.world)

</div>

---

## Five-minute orientation

| Property | Value |
|---|---|
| Consensus | Proof-of-Work with mandatory QRNG attestation |
| Block time | 60s (mainnet), 5s (testnet) |
| Block reward | 5 WAVE, halves every 2,100,000 blocks (~4 yrs) |
| Max supply | 21,000,000 WAVE |
| Genesis premine | 1,000,000 WAVE to foundation address |
| Signature scheme | ML-DSA-87 (NIST FIPS 204) |
| Key exchange | ML-KEM-1024 (NIST FIPS 203) |
| Hash function | SHA3-512 |
| Address length | 20 bytes (hex without `0x` prefix) |
| Smart contract VM | Stack-based, EVM-flavored, post-quantum precompiles |
| Smart contract language | [Fourier](https://fourier.fermi.world) |

## What WaveLedger is not

- **Not a fork of anything.** The VM is original; the language is
  original; the consensus client is original Python.
- **Not "Bitcoin with PQC bolted on."** Every primitive in the chain
  was selected for PQ safety from the start.
- **Not production-ready.** This is a testnet. The mainnet design is
  documented, but not running.
- **Not anti-Ethereum.** The smart-contract VM borrows liberally from
  EVM ergonomics, because EVM ergonomics work. Where we differ from
  EVM, it's documented.
