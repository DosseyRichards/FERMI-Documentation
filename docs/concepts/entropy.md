# QRNG and entropy

WaveLedger requires every block to carry a verifiable attestation of
unpredictable entropy. This is the **"quantum-attested mining"** rule
that distinguishes WaveLedger from a generic PoW chain.

## The rule

For a block to be valid:

1. The miner must fetch ≥ 64 bytes of entropy from a registered entropy
   source.
2. The miner must include an **attestation envelope** in the block
   (`quantum_signature`):
   - The raw entropy commitment: `SHA3-512(entropy)`
   - Health metadata from the source (timestamp, source ID, version)
   - The source's signature over the commitment + metadata
3. The entropy must be used as the **`quantum_seed`** in the block
   template — meaning the proof-of-work hash incorporates it.
4. Validators must accept the attestation: either the source's
   public key is registered on-chain, or the source ID is in the
   `ENTROPY_SOURCES_TRUST_LIST` network parameter.

## Why this matters

Generic PoW (Bitcoin, Litecoin, etc.) relies on the miner's secret
nonce search being computationally hard. WaveLedger adds the constraint
that the *input* to that search must be demonstrably unpredictable —
a miner can't pre-mine blocks for the future because they don't know
what entropy they'll have.

This makes the chain attractive to:

- **Smart contracts that need on-chain randomness** — instead of
  defrauding RANDAO-style schemes, contracts can read the attested
  entropy from the block header directly.
- **Auditors** — every block's randomness can be traced to a source
  and verified.
- **Hardware-QRNG vendors** — selling certified entropy as a service
  has a working consumer.

## Source registry

The block's `source` field records which entropy provider produced
this block's seed. Today the chain stores the field on every block as
an audit trail but does **not** yet reject blocks whose source ID is
unknown — `verify_attestation` only enforces a length bound on the
string. Allow-listing the registered IDs and verifying a per-source
signature are the next two enforcement steps (tracked in
`mining/attestation.py`); see
[Crypto agility](agility.md#what-is-agile-today) for the broader
design.

Sources currently seen on testnet:

| Source ID | Description | Enforced |
|---|---|---|
| `aggregator:drand-default` | drand "default" beacon via the in-tree aggregator | No allow-list yet |
| `self-hosted` | The miner's local entropy stack (default if a provider doesn't set `source_id`) | No allow-list yet |

Planned (not deployed):

- `fermi-qrng-v1` — Fermi shot-noise photodiode hardware
- `iqr-server-v0` — ID Quantique reference hardware

Adding a new source will be governance-controlled once the registry
ships; the design intent is BDFL-signed registration with a Timelock
delay, but the on-chain hooks for either are not yet built.

## The REST contract

A source must expose, on whatever port it likes:

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

## The aggregator pattern

The reference `qrng_aggregator_service.py` combines N upstream sources
by XOR-mixing their outputs:

$$ \text{output}_t = \bigoplus_{i=1}^{N} \text{source}_i(t) $$

As long as **at least one** source produces uniformly random bytes,
the output is uniformly random. The min-quorum parameter says how many
sources must respond successfully before the aggregator will serve any
output — set high to fail closed, low to favor liveness.

The testnet aggregator currently runs with `min_quorum = 1` and one
source (`drand`). Production setups should run min-quorum 2-of-3 with
a mix of [drand](https://drand.love), hardware QRNG, and a backup
public service.

## Why drand for the testnet

[drand](https://drand.love) is the League of Entropy's federated
beacon — a threshold BLS signature over a 30-second round number,
contributed to by Cloudflare, EPFL, Kudelski Security, and others.

- Free
- 100% historical uptime (since 2019)
- Cryptographically verifiable (BLS signature in every round)
- Publicly auditable

It is not *quantum* random — drand uses classical entropy mixed across
participants. The "quantum attested" framing on mainnet relies on
swapping drand for QRNG hardware once it's deployed. The aggregator
contract makes that a config change, not a fork.

## No classical fallback

There is deliberately **no** "fall back to /dev/urandom if all sources
fail" path. If every registered source is down, mining halts. This is
the only way the entropy attestation has any meaning — if mining
silently continues with software RNG when the verifiable source is
gone, the whole property is decorative.

Operators have six tiers of degradation available:

1. Local QRNG hardware (preferred)
2. Local aggregator (multiple sources, XOR'd)
3. Remote aggregator on a separate VPS (testnet default)
4. Lowered difficulty target (continues mining at a reduced rate
   while waiting for entropy)
5. Emergency federated beacon (humans rotate keys + republish)
6. Halt

If none of those apply, the chain stops accepting new blocks.
