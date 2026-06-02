# Running the entropy aggregator

The entropy aggregator is a small HTTP service that:

1. Pulls bytes from one or more upstream entropy sources (drand,
   QRNG hardware, future federated beacons)
2. XOR-mixes them into a single byte pool
3. Serves the pool to miners via the chain's standard
   [REST contract](../concepts/entropy.md#source-rest-contract)

Run a local aggregator to avoid dependence on the public testnet
aggregator (`entropy.waveledger.net`) — for operational control, for
redundancy, or to integrate dedicated entropy hardware.

## Run the reference aggregator

```bash
# In the WaveLedger repo:
python3 qrng_aggregator_service.py --port 8420 --host 0.0.0.0
```

The reference setup wires in drand as the sole source. Miners on the
same machine can point at `127.0.0.1:8420`; miners elsewhere require
the host to be reachable (firewall + DNS).

On boot the aggregator warms up by fetching a drand round; this takes
approximately 2 seconds.

## Endpoints

See [QRNG and entropy → REST contract](../concepts/entropy.md#source-rest-contract)
for the full surface. The aggregator exposes three endpoints:

| Endpoint | Use |
|---|---|
| `GET /api/health` | Pool fill status, source health, last refill metadata |
| `GET /api/random/bytes?n=N` | Raw entropy (binary) |
| `GET /api/random/hex?n=N` | Same, hex-encoded |

## Adding sources

The aggregator picks up sources from `make_aggregator()` in
`qrng_aggregator_service.py`:

```python
from quantum.providers.drand_provider import DrandProvider
from quantum.providers.local_qrng_provider import LocalQRNGProvider
from quantum.providers.aggregator_provider import AggregatorProvider

def make_aggregator():
    return AggregatorProvider(
        sources=[
            DrandProvider(),
            LocalQRNGProvider(host='10.0.0.5', port=8421),  # your QRNG box
            # AnotherProvider(...),
        ],
        min_quorum=2,         # require at least 2 sources to return valid bytes
    )
```

Each source must implement `async def generate_qrng(num_bits: int)`
returning a `QRNGResult` (see `quantum/providers/local_qrng_provider.py`
for the reference impl).

`min_quorum=2` requires at least 2 configured sources to respond
successfully before the aggregator serves output. Failing closed is the
correct default for entropy.

## Pool sizing

The aggregator maintains a 64 KB pool by default. When the pool drops
below ~50% full, a background coroutine refills from all configured
sources (XOR-mixed).

| Knob | Default | When to change |
|---|---|---|
| `pool max size` | 65536 bytes | Raise for high-throughput chains; lower for memory-constrained boxes |
| `refill threshold` | 50% | Raise for jittery upstreams |
| `refill batch` | 32 KB per source | — |

A `performance-1x` fly VM serves approximately 10 MB/sec of entropy
from this pool, sufficient for any plausible miner count.

## Running on fly

```bash
# From the WaveLedger repo, with fly.entropy.toml in place
flyctl apps create my-entropy
flyctl ips allocate-v6 --config fly.entropy.toml
flyctl ips allocate-v4 --config fly.entropy.toml --shared
flyctl deploy --config fly.entropy.toml --ha=false
flyctl certs add entropy.example.com --config fly.entropy.toml
```

Then add DNS pointing `entropy.example.com` at Fly. See
[Self-hosting on Fly.io](fly.md) for the full walkthrough.

## Running on a VPS

```bash
# As root on Ubuntu 22.04+
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git /opt/waveledger
sudo bash /opt/waveledger/deploy/testnet/setup_entropy_vps.sh
# Then put TLS in front:
ENTROPY_HOST=entropy.example.com sudo -E bash /opt/waveledger/deploy/testnet/setup_entropy_proxy.sh
```

The setup scripts use systemd + Caddy. See the
[VPS guide](vps.md) for the breakdown.

## Trust list considerations

A new aggregator's source ID (the `source` field in its `/api/health`)
must be in the chain's trust list before its blocks are accepted by
miners. The testnet trust list includes `aggregator:drand-default` and
a handful of placeholder IDs.

Operating an aggregator for a private set of miners is unconstrained —
the trust list applies only when **external miners** must accept
attestations from the source.
