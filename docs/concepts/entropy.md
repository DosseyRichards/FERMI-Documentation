# Entropy and attestation

WaveLedger requires every block to carry a verifiable attestation that
its proof-of-work input came from a registered, unpredictable entropy
source. This requirement — *attested mining* — is what distinguishes
WaveLedger from generic proof-of-work.

## Block-level requirement

For a block to be accepted by validators:

1. The miner must obtain at least 64 bytes from a registered entropy
   source. The first 32 bytes are the **seed** that is mixed into the
   block header before PoW; the last 32 bytes are the **proof** that
   is revealed for verification.
2. The block must carry an **attestation envelope** in its
   `quantum_signature` field:
    - `entropy_seed` and `entropy_proof`, 32-byte hex each
    - `commitment = SHA3-512(entropy_proof)`, recomputed by every
      validator
    - `health`, a snapshot of monobit ratio, Fano factor, and pool
      size
    - `source`, the registered source identifier
    - `device_id`, `proof_type`, `version`
3. The seed is incorporated into the block header that PoW hashes. A
   miner cannot grind on different entropy after the fact.
4. Validators reject any block whose `source` is not in the
   [registry](#source-registry), whose `commitment` does not match
   `SHA3-512(entropy_proof)`, or whose health statistics fall outside
   the configured bounds. When a source has a registered public key,
   the envelope must also carry a valid ML-DSA-87 signature over the
   commitment; see [Source signing](#source-signing).

## Rationale

Generic proof-of-work depends only on the difficulty of the nonce
search. Attested mining adds the constraint that the *input* to that
search must come from a verifiable, unpredictable source: a miner
cannot pre-compute blocks for future heights, because they do not
control what entropy they will be permitted to use.

This makes the chain attractive to:

- **Smart contracts requiring on-chain randomness.** Instead of
  defrauding RANDAO-style schemes, contracts read the attested entropy
  from the block header directly.
- **Auditors.** Every block's randomness traces to a named source and
  is independently verifiable.
- **Entropy vendors.** Certified entropy services have a paying
  consumer in mining attestations.

## Source registry

The block's `source` field records which entropy provider produced the
block's seed. The registry of acceptable identifiers lives in
`core/constants.py::SourceRegistry`. `verify_attestation` rejects any
block whose `source` is not registered, either as an atomic identifier
or as an `aggregator:a+b+c` composition of registered atomic
identifiers.

Registered identifiers:

| Source ID | Description | Status |
|---|---|---|
| `self-hosted` | Default when a provider does not report a `source_id` | Testnet |
| `drand-default` | drand "default" beacon — threshold BLS, League of Entropy | Active |
| `drand-fastnet` | drand "fastnet" beacon | Active |
| `fermi-qrng-v1` | Fermi shot-noise photodiode hardware | Planned |
| `iqr-server-v0` | ID Quantique reference hardware | Planned |

Aggregator compositions, e.g. `aggregator:drand-default+self-hosted`,
are accepted when every component is itself registered.

Adding a new identifier is a governance-controlled code change. The
on-chain hooks for governance-driven registry edits are catalogued in
[Crypto agility](agility.md#agile-surfaces).

## Source signing

An attestation envelope may carry a `signature` and a `public_key`
alongside the commitment. The signature is **ML-DSA-87** (FIPS 204)
computed over the SHA3-512 commitment string.

Verification rules:

- If the registry has no public key on file for the source, the
  signature is optional. When present, it is verified against the
  envelope's `public_key`.
- If the registry has a public key on file for the source, the
  signature is mandatory and the envelope's `public_key` must equal
  the registered key. Mismatched or missing signatures cause the
  block to be rejected.

This binds each entropy provider to its on-chain identity once it
registers a key, while permitting older v1 providers to migrate at
their own pace.

## Source REST contract

A source exposes:

```
GET /api/health
GET /api/random/bytes?n=<int>
GET /api/random/hex?n=<int>
```

Response from `/api/health`:

```json
{
  "status": "running",
  "uptime_seconds": 12345.67,
  "pool": {
    "available_bytes": 65536,
    "capacity": 65536,
    "fill_percent": 100.0
  },
  "last_fill": {
    "source": "drand-default",
    "round": 4567890,
    "filled_at": "2026-05-28T20:00:00Z"
  },
  "source": "testnet-aggregator",
  "device": "WaveLedger Testnet Entropy Service"
}
```

Response from `/api/random/bytes?n=64`:

```
<raw bytes, Content-Type: application/octet-stream>
```

## Aggregator construction

The reference implementation `qrng_aggregator_service.py` combines
multiple upstream sources by XOR-mixing their outputs:

$$ \text{output}_t = \bigoplus_{i=1}^{N} \text{source}_i(t) $$

Provided at least one component is uniformly random, the output is
uniformly random. The `min_quorum` parameter specifies the minimum
number of sources that must respond before the aggregator returns
output. Higher values fail closed; lower values favor liveness.

Production deployments are expected to run with at least
`min_quorum = 2` across a mix of drand, hardware QRNG, and a
geographically separate public service.

## drand on the public testnet

[drand](https://drand.love) is the League of Entropy's federated
beacon: a threshold BLS signature over a 30-second round number, with
contributors including Cloudflare, EPFL, and Kudelski Security.

- No cost
- 100% historical uptime since 2019
- Cryptographically verifiable per round
- Publicly auditable

drand is classical randomness mixed across a federation. Mainnet
extends attested mining to hardware QRNG through the same registry,
without a chain fork.

## No silent fallback

There is no fallback to `/dev/urandom` when registered sources are
unavailable. If every registered source is down, mining halts. This is
the only operating mode under which the attestation has meaning: if
mining were permitted to continue against software randomness, the
attestation would be decorative.

Operators have a tiered degradation path:

1. Local QRNG hardware
2. Local aggregator across multiple sources
3. Remote aggregator on a separate host
4. Reduced difficulty target while the source is degraded
5. Emergency federated beacon, rotated by the operator network
6. Halt
