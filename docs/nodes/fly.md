# Self-hosting on Fly.io

The reference deployment runs three fly apps:

| App | Role | Public? |
|---|---|---|
| `waveledger-entropy` | Entropy aggregator | Yes (HTTPS) |
| `waveledger-chat` | dApp + seed + miner #1 | Yes (HTTPS chat + raw TCP P2P) |
| `waveledger-miner2` | Miner #2 | No (private) |

Total cost: ~$27/mo (one $25 perf-1x for the chat node, +$2 for the
dedicated IPv4 the raw-TCP P2P needs).

## Prereqs

- `flyctl` installed (`brew install flyctl` or `curl -L https://fly.io/install.sh | sh`)
- A fly account (`fly auth signup`)
- Docker daemon running locally (or accept `--remote-only` for slower builds)
- A domain you can edit DNS for

## Deploy

From the repo root:

```bash
# Entropy
fly apps create waveledger-entropy
fly ips allocate-v6 --config fly.entropy.toml
fly ips allocate-v4 --config fly.entropy.toml --shared
fly deploy --config fly.entropy.toml --ha=false

# Chat / seed / miner-1
fly apps create waveledger-chat
fly volumes create waveledger_chat_data \
    --config fly.chat.toml --region iad --size 1 --yes
fly secrets set \
    WAVELEDGER_ADMIN_USER=admin \
    WAVELEDGER_ADMIN_PASSWORD='your-strong-password' \
    --config fly.chat.toml --stage
fly ips allocate-v4 --config fly.chat.toml --yes      # for P2P raw TCP
fly deploy --config fly.chat.toml --ha=false

# Miner-2
fly apps create waveledger-miner2
fly volumes create waveledger_miner2_data \
    --config fly.miner2.toml --region iad --size 1 --yes
fly deploy --config fly.miner2.toml --ha=false
```

## Certs (custom domains)

```bash
fly certs add chat.example.com    --config fly.chat.toml
fly certs add entropy.example.com --config fly.entropy.toml
fly certs add seed.example.com    --config fly.chat.toml
```

Each command prints the DNS records you need to set.

## DNS

For three subdomains served at fly:

| Type | Name | Value |
|---|---|---|
| CNAME | `chat` | `waveledger-chat.fly.dev` |
| CNAME | `entropy` | `waveledger-entropy.fly.dev` |
| A | `seed` | `<dedicated-v4-from-fly-ips-list>` |
| AAAA | `seed` | `<dedicated-v6-from-fly-ips-list>` |

`seed` needs explicit A/AAAA (not CNAME) because raw TCP can't ride
on fly's shared SNI-routed IPv4. `chat` and `entropy` are HTTPS, so
shared IPv4 + CNAME work.

Check propagation + cert status:

```bash
dig +short chat.example.com
fly certs check chat.example.com --config fly.chat.toml
```

The cert is issued automatically by Let's Encrypt once DNS resolves.
First-time validation takes 30 sec to a few minutes.

## Smoke test

```bash
curl -sI https://chat.example.com | head -3
curl -s  https://entropy.example.com/api/health | jq .pool
nc -zv   seed.example.com 18333
```

## Logs + operations

```bash
# Tail
fly logs --config fly.chat.toml

# SSH into a machine
fly ssh console --config fly.chat.toml

# Restart
fly machine restart --config fly.chat.toml --select

# Roll back to previous release
fly releases  --config fly.chat.toml
fly deploy --image registry.fly.io/waveledger-chat:deployment-<prev>

# Scale
fly scale memory 4096 --config fly.chat.toml
fly scale count 0     --config fly.chat.toml     # pause (chain halts)
fly scale count 1     --config fly.chat.toml     # resume
```

## Wipe state (reset the chain)

```bash
# Destroy the volume (machines must be stopped first)
fly machine stop <id> --config fly.chat.toml
fly machine destroy <id> --config fly.chat.toml --force
fly volumes destroy <vol_id> --config fly.chat.toml --yes
fly volumes create waveledger_chat_data \
    --config fly.chat.toml --region iad --size 1 --yes
fly deploy --config fly.chat.toml --ha=false
```

Repeat for miner2 if you want a totally fresh chain. The entropy app
has no volume — no reset needed.

## Why fly

- One-command deploys
- Auto-issued TLS
- Private network (`.internal`) between sibling apps with no extra config
- Anycast — your chat URL feels close everywhere
- Free shared IPv4 for HTTPS apps; cheap dedicated IPv4 for raw TCP

The same setup works on Render, Railway, or any container host with
custom domain + volume support. For a bare-VPS path see
[VPS guide](vps.md).
