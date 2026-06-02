---
title: WaveLedger Documentation
hide:
  - navigation
  - toc
---

# WaveLedger

A post-quantum Layer-1 blockchain. Every transaction is signed with
**ML-DSA-87** (NIST FIPS 204). Every key exchange uses **ML-KEM-1024**
(FIPS 203). Every hash is **SHA3-512** (FIPS 202). No classical
cryptography sits in the consensus path.

Cryptographic schemes are dispatched through an on-chain registry,
entropy sources are catalogued in a typed allow-list, and additional
PQC primitives plug in as precompile addresses. Adding a NIST-standard
scheme is a node release coordinated through governance; it is not a
chain split.

Mining requires a verifiable attestation that the block's entropy came
from a registered source. The public testnet uses the
[drand](https://drand.love) federated beacon today through a
vendor-agnostic contract; hardware QRNG and additional sources land
through the same registry path. See
[Crypto agility](concepts/agility.md) for what is swappable today and
[Source registry](concepts/entropy.md#source-registry) for the
enforcement model.

[Public testnet :material-arrow-right:](https://chat.waveledger.net){ .md-button .md-button--primary }

## SDK

```bash
pip install waveledger-sdk        # PyPI
npm  install waveledger-sdk        # npm
```

A single `Client` covers every messenger surface — auth, chat, wallet,
explorer, playground, admin — plus the SSE event stream. Both libraries
default to `https://api.waveledger.net`. See [SDK](sdk/index.md) for
the full surface.

---

## Documentation map

<div class="grid cards" markdown>

-   :material-book-open-page-variant:{ .lg .middle } **Concepts**

    ---

    Post-quantum cryptography, block structure, mining and entropy,
    economics, addresses, networking.

    [:octicons-arrow-right-24: Concepts](concepts/index.md)

-   :material-api:{ .lg .middle } **API Reference**

    ---

    Every public REST endpoint: wallet, chat, contract playground,
    explorer, admin. Request and response shapes for each.

    [:octicons-arrow-right-24: API](api/index.md)

-   :material-server:{ .lg .middle } **Running a node**

    ---

    Miner, seed, and entropy aggregator deployments. Fly.io and
    bare-VPS guides. Operational runbook.

    [:octicons-arrow-right-24: Operator guide](nodes/index.md)

-   :material-code-braces:{ .lg .middle } **SDK**

    ---

    Send WAVE, deploy contracts, subscribe to events from Python and
    TypeScript. Tested snippets for each call.

    [:octicons-arrow-right-24: SDK](sdk/index.md)

-   :material-file-document:{ .lg .middle } **Reference**

    ---

    Wire formats for blocks, transactions, receipts, addresses;
    network parameters; the PQC primitive catalogue.

    [:octicons-arrow-right-24: Format specifications](reference/index.md)

-   :material-script-text-outline:{ .lg .middle } **Fourier (smart contracts)**

    ---

    The smart-contract language has its own site at
    [fourier.fermi.world](https://fourier.fermi.world).

    [:octicons-arrow-right-24: Fourier](https://fourier.fermi.world)

</div>

---

## Parameters at a glance

| Property | Value |
|---|---|
| Consensus | Proof-of-Work with mandatory entropy attestation |
| Block time | 60s mainnet target, 5s testnet |
| Block reward | 5 WAVE; halves every 2,100,000 blocks |
| Maximum supply | 21,000,000 WAVE |
| Genesis allocation | 1,000,000 WAVE to the foundation address |
| Signature scheme | ML-DSA-87 (NIST FIPS 204) |
| Key encapsulation | ML-KEM-1024 (NIST FIPS 203) |
| Hash function | SHA3-512 (NIST FIPS 202) |
| Address length | 20 bytes, lowercase hex without `0x` prefix |
| Smart-contract VM | Stack-based, 256-bit words, post-quantum precompiles |
| Smart-contract language | [Fourier](https://fourier.fermi.world) |

## Design boundaries

- **Original implementation.** The VM, the language, and the consensus
  client are written from scratch.
- **Post-quantum from the consensus root.** Every primitive in the
  chain was selected for PQ safety; classical cryptography is not used
  to sign blocks, transactions, or P2P messages.
- **Public testnet today.** A mainnet specification is published; the
  mainnet network is not yet live.
- **EVM-aligned VM ergonomics.** The smart-contract VM borrows
  liberally from EVM where the ergonomics are well-understood;
  deviations are documented in the Fourier reference.
