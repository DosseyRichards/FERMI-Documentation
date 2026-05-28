# Crypto agility

WaveLedger ships with NIST-standardized post-quantum primitives, but
**no one believes the first generation of PQC will be the last**.
Lattice schemes are young. Side-channel attacks against ML-DSA exist
already. SLH-DSA's parameters are conservative because nobody fully
trusts the lattice ones. New PQC standards are coming.

A blockchain that hard-codes one signature scheme bets the chain on
that scheme staying unbroken forever. WaveLedger does not take that
bet.

## What is agile today

| Surface | Scheme(s) shipped | How to add a new one |
|---|---|---|
| Smart-contract signature verify | ML-DSA-87, SLH-DSA-SHA2-128s | Add to `fourier/codegen.py::CRYPTO_SCHEMES` and register a precompile in `vm/precompiles.py` |
| QRNG attestation source | drand (testnet), aggregator (mainnet) | Implement the entropy REST contract; the block's `attestation.source` records exactly which provider was used |
| Attestation proof format | Version-tagged via `proof_type` | Add a new `proof_type` value; validators dispatch by the tag |
| Hash inside contract VM | SHA3-256 (the `SHA3` opcode) | Add a precompile (e.g., BLAKE3, SHA3-512) at a reserved address |

Contracts written today can already opt into any of the registered
signature schemes by passing the scheme ID to `verify_sig`:

```fourier
let ok: uint = verify_sig(scheme_id, pk, msg, sig);
```

where `scheme_id` is `1` (ML-DSA-87), `2` (SLH-DSA-SHA2-128s), or any
future value. Switching a contract from one scheme to another is a
constant change — no protocol upgrade, no hard fork.

## What is not agile yet (and why)

Some primitives are still hardcoded in v1 because they sit deeper in
the consensus path. Changing them requires either a hard fork or a
versioned-block format. They are listed here so you know what's
load-bearing:

| Surface | What's hardcoded | Why | Path to agility |
|---|---|---|---|
| Transaction signatures | ML-DSA-87 | Every node validates every tx; a registry would mean every node carries every scheme | A versioned `signature_scheme` field on `Transaction`, mapped through a node-wide registry |
| Block hash | SHA3-512 | Block hash defines chain identity; if two nodes hash differently they fork | A versioned `header_format` that names the hash, gated by activation height |
| Wallet address derivation | `SHA3-512(pub_key)[:32]` | Addresses persist on chain forever; cannot rotate without breaking historical balances | New addresses get a version prefix; old addresses keep working |
| QRNG attestation envelope shape | Single fixed schema | Same as block hash — every validator must agree | Already partially solved via `proof_type` versioning |

The honest pattern: anything that ends up on disk, in a balance, or
in a signature over a balance, is hard to change. Anything that's a
*verification step at the moment of use* is easy.

## The registry pattern

The two surfaces that are already agile use the same shape: a
**runtime-dispatched registry** keyed by an integer ID.

### Signature schemes

```python
# fourier/codegen.py
CRYPTO_SCHEMES = {
    1: 0x02,    # ML-DSA-87  → precompile at 0x...02
    2: 0x03,    # SLH-DSA    → precompile at 0x...03
    # 3, 4, ...    future schemes — add here + register precompile
}
```

A scheme is identified by an integer. The contract VM dispatches the
right verify routine by ID at runtime. Adding a scheme is a code
change in two files (the registry + the precompile implementation) and
deploying a new node release — no chain state changes, no fork.

### Entropy sources

```json
{
  "attestation": {
    "source": "aggregator:drand-default",
    "source_round": 4567890,
    "proof_type": "drand-bls",
    ...
  }
}
```

The `source` and `proof_type` fields on the attestation envelope are
opaque strings. Validators look up the verification rules by
`proof_type`. Adding a new entropy provider (QRNG vendor, multi-source
aggregator, threshold scheme) requires:

1. Implementing the entropy REST contract (`/api/entropy`).
2. Picking a new `proof_type` tag.
3. Registering verification rules for the new tag in the validator.

The new source can run alongside existing ones. Different miners can
use different sources at the same time; every block records which
source it used. There is a permanent on-chain audit trail.

## Why agility is a design feature, not a hedge

It is tempting to say "ML-DSA-87 is standardized, what could go
wrong" — and three things will, eventually:

1. **A specific implementation breaks.** A side-channel paper exposes
   ML-DSA-87 in particular conditions. Affected nodes can switch to
   SLH-DSA in their *next* block; no chain freeze, no migration
   window.
2. **A standard gets superseded.** NIST adopts a faster or
   better-analyzed scheme; older deployments want to migrate. Contracts
   can move scheme-by-scheme. The chain's signature registry is a
   migration ramp, not a rip-and-replace.
3. **A specific source goes dark.** The drand network has an outage,
   or the QRNG provider's hardware fails. Other registered sources
   keep producing blocks. There is no single source of truth for
   entropy — the chain only requires *a* trusted source per block.

Each of these failure modes is a "when," not an "if." Crypto agility
turns a coordinated chain-wide upgrade into a quiet rotation.

## How to read the chain's choices

Every block records:

- Which entropy source produced its attestation (`attestation.source`).
- Which proof format was used (`attestation.proof_type`).

Every transaction records:

- Which signature scheme its sender used (currently always ML-DSA-87
  on the chain layer; varies on the contract layer per call to
  `verify_sig`).

A simple aggregation over the chain tells you, at any point in time,
which schemes the network is leaning on. As schemes get added or
deprecated, the distribution shifts — visibly, and on-chain.

## What's next

Roadmap items for fuller agility:

- **Versioned tx signature field.** Lift the chain-layer signature
  into the registry. (Hard part: every node must still be able to
  verify *every* old signature scheme. Bytecode size grows with each
  new scheme.)
- **Block header format versioning.** A `header_format` tag activated
  at a specific height. Lets us swap the hash function without a fork.
- **Wallet-address rotation.** Versioned address derivation so future
  wallets can carry a scheme ID — sender address tells the network
  which signature scheme to expect.
- **Threshold attestation.** Require *N of M* entropy sources to
  agree before a block is accepted. Trades latency for resilience.

These are not promises; they are the natural extensions of the
registry pattern.

## Further reading

- [Crypto primitives](../reference/crypto.md) — what ships today.
- [Post-quantum cryptography](pqc.md) — why PQC at all.
- [QRNG and entropy](entropy.md) — the entropy source contract.
- [Reference: parameters](../reference/parameters.md#crypto) — the
  exact scheme IDs and parameter sets.
