# Crypto agility

WaveLedger ships with NIST-standardized post-quantum primitives. The
first generation of PQC is unlikely to be the last: lattice schemes
are young, side-channel attacks against ML-DSA have been published,
SLH-DSA's parameters are conservative because the lattice schemes are
not yet fully trusted, and additional PQC standards are in progress.

A blockchain that hard-codes a single signature scheme stakes the
chain on that scheme remaining unbroken forever. WaveLedger does not
take that bet.

## Agile surfaces

| Surface | Scheme(s) shipped | How to add a new one |
|---|---|---|
| Smart-contract signature verify | ML-DSA-87, SLH-DSA-SHA2-128s | Add to `fourier/codegen.py::CRYPTO_SCHEMES` and register a precompile in `vm/precompiles.py` |
| QRNG attestation source | drand (testnet), aggregator (mainnet) | Implement the entropy REST contract; the block's `attestation.source` records exactly which provider was used |
| Attestation proof format | Version-tagged via `proof_type` | Add a new `proof_type` value; validators dispatch by the tag |
| Hash inside contract VM | SHA3-256 (the `SHA3` opcode) | Add a precompile (e.g., BLAKE3, SHA3-512) at a reserved address |

Contracts can opt into any registered signature scheme by passing the
scheme ID to `verify_sig`:

```fourier
let ok: uint = verify_sig(scheme_id, pk, msg, sig);
```

where `scheme_id` is `1` (ML-DSA-87), `2` (SLH-DSA-SHA2-128s), or any
future value. Switching a contract from one scheme to another is a
constant change — no protocol upgrade, no hard fork.

## Non-agile surfaces

Some primitives remain hardcoded in v1 because they sit deeper in the
consensus path. Changing them requires either a hard fork or a
versioned-block format. The load-bearing surfaces are:

| Surface | What's hardcoded | Why | Path to agility |
|---|---|---|---|
| Transaction signatures | ML-DSA-87 | Every node validates every tx; a registry would mean every node carries every scheme | A versioned `signature_scheme` field on `Transaction`, mapped through a node-wide registry |
| Block hash | SHA3-512 | Block hash defines chain identity; if two nodes hash differently they fork | A versioned `header_format` that names the hash, gated by activation height |
| Wallet address derivation | `SHA3-512(pub_key)[:32]` | Addresses persist on chain forever; cannot rotate without breaking historical balances | New addresses get a version prefix; old addresses keep working |
| QRNG attestation envelope shape | Single fixed schema | Same as block hash — every validator must agree | Already partially solved via `proof_type` versioning |

The pattern: anything that ends up on disk, in a balance, or in a
signature over a balance is hard to change. Anything that is a
*verification step at the moment of use* is straightforward to change.

## The registry pattern

The two agile surfaces share the same shape: a
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
correct verify routine by ID at runtime. Adding a scheme requires a
code change in two files (the registry and the precompile
implementation) plus a new node release — no chain state changes, no
fork.

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

A new source runs alongside existing ones. Different miners may use
different sources simultaneously; every block records which source it
used. The result is a permanent on-chain audit trail.

## Why agility is a design feature

Three failure modes for any fixed signature scheme are eventual, not
hypothetical:

1. **A specific implementation breaks.** A side-channel paper exposes
   ML-DSA-87 under particular conditions. Affected nodes switch to
   SLH-DSA in their *next* block; no chain freeze, no migration
   window.
2. **A standard is superseded.** NIST adopts a faster or
   better-analyzed scheme; older deployments migrate. Contracts move
   scheme-by-scheme. The chain's signature registry is a migration
   ramp, not a rip-and-replace.
3. **A specific source goes dark.** The drand network has an outage,
   or the QRNG provider's hardware fails. Other registered sources
   keep producing blocks. There is no single source of truth for
   entropy — the chain requires only *a* trusted source per block.

Each failure mode is a "when," not an "if." Crypto agility turns a
coordinated chain-wide upgrade into a quiet rotation.

## Reading the chain's choices

Every block records:

- Which entropy source produced its attestation (`attestation.source`).
- Which proof format was used (`attestation.proof_type`).

Every transaction records:

- Which signature scheme its sender used (always ML-DSA-87 on the
  chain layer in v1; varies on the contract layer per call to
  `verify_sig`).

Aggregating across the chain yields a point-in-time view of which
schemes the network relies on. As schemes are added or deprecated, the
distribution shifts visibly and on-chain.

## Future extensions

Natural extensions of the registry pattern:

- **Versioned tx signature field.** Lift the chain-layer signature
  into the registry. Every node must still verify *every* old
  signature scheme, so bytecode size grows with each new scheme.
- **Block header format versioning.** A `header_format` tag activated
  at a specific height. Enables swapping the hash function without a
  fork.
- **Wallet-address rotation.** Versioned address derivation so future
  wallets carry a scheme ID — sender address signals to the network
  which signature scheme to expect.
- **Threshold attestation.** Require *N of M* entropy sources to
  agree before a block is accepted. Trades latency for resilience.

## Further reading

- [Crypto primitives](../reference/crypto.md) — what ships today.
- [Post-quantum cryptography](pqc.md) — why PQC at all.
- [QRNG and entropy](entropy.md) — the entropy source contract.
- [Reference: parameters](../reference/parameters.md#crypto) — the
  exact scheme IDs and parameter sets.
