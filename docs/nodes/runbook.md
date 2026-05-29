# Operational runbook

For when something goes wrong, or for the daily-driver operations of
keeping a node alive.

## Healthy-state checklist

A healthy mining node will show all of:

```bash
# Dashboard status
$ curl -s http://127.0.0.1:8080/api/status
{ "node_running": true, "chain_height": 12345, "peer_count": ≥1,
  "mining_active": true, "qrng_reachable": true, "sync_state": "synced" }

# Tip age (should be < 2× block_time_target)
$ curl -s http://127.0.0.1:8080/api/chain | jq '.tip_timestamp - now'
< 10s on testnet, < 120s on mainnet

# Mempool not pinned high
$ curl -s http://127.0.0.1:8080/api/mempool | jq '.pending_count'
< 100 normally; higher under heavy load
```

## Common failures + fixes

### Chain not advancing

**Symptom:** `tip_timestamp` is more than 60s old.

```bash
# Is the miner running at all?
curl -s http://127.0.0.1:8080/api/mining | jq '.running'

# Is entropy reachable?
curl -s http://127.0.0.1:8080/api/qrng | jq '.reachable'

# Are peers connected?
curl -s http://127.0.0.1:8080/api/peers | jq '.peer_count'

# Logs
journalctl -u waveledger-miner -n 100 --no-pager
```

Typical causes:

| Cause | Fix |
|---|---|
| Entropy source unreachable | Restart entropy service / check DNS / check firewall |
| All peers dropped | `systemctl restart waveledger-miner`; check bootstrap_nodes |
| Difficulty spiraled | See "difficulty too high" below |
| QRNG returned bad attestation | Check entropy upstream health |

### Difficulty too high (mining stuck on hard blocks)

**Symptom:** `mining_active: true` but block age keeps growing. Logs
show `Block mined` events stopping. The chain's difficulty got pushed
above what the available CPU can sustain.

Fix:

1. Stop the miner.
2. If you control the chain, lower `TESTNET_MAX_DIFFICULTY` in
   `core/constants.py` and redeploy. (Mainnet uses the higher
   `MAX_DIFFICULTY = 8` ceiling; this is a testnet problem.)
3. Wait for ~10 blocks to land at the new ceiling — the difficulty
   adjustment will start ramping down.
4. If the chain is truly stuck and no one is producing blocks, you
   may need to reset the chain entirely (see "reset the chain" below).

The difficulty adjustment in WaveLedger only re-evaluates at interval
boundaries (every 10 blocks), so once you're stuck, you stay stuck
until someone mines one more block.

### Faucet failing to credit users

**Symptom:** New signups get approved but balance stays at 0.

```bash
journalctl -u waveledger-chat | grep -iE 'faucet|approve' | tail
```

Possible diagnoses:

| Log line | Fix |
|---|---|
| `faucet skipped: node has no miner_address` | Set `[mining].miner_address` or enable `--mine` |
| `faucet skipped: miner wallet missing from blockchain.wallets` | Wallet store out of sync — restart node |
| `faucet underfunded (miner_balance < ...)` | Miner hasn't earned enough coinbase yet; wait, or seed the miner wallet via direct transfer |
| `faucet tx rejected by mempool: <reason>` | The reason explains; usually duplicate-tx-id or balance race |

### Forks + reorgs everywhere

**Symptom:** Logs show many `Fork detected` lines, `Cannot find fork
point`, `Failed to download competing branch`. Two miners are racing
and the chain is unstable.

Causes:

- Two miners running independent chains because tx propagation is
  failing (txs sit in one node's mempool, the other never sees them)
- Network partition (peers came back after losing connection)
- Block time too aggressive — mining is faster than gossip can converge

Mitigation:

1. Stop all but one miner. Let the chain stabilize.
2. Restart the others one at a time, with proper bootstrap nodes set.
3. If the chain has truly diverged, reset (below).

### Reset the chain

When all else fails:

```bash
# Stop every node
sudo systemctl stop 'waveledger-*'

# Wipe each node's data dir
sudo rm -rf /var/lib/waveledger-testnet

# Start them back up
sudo systemctl start waveledger-*
```

All on-chain state is gone (balances, contracts, message history,
approvals). In-memory messenger state (sessions, invites) was already
gone the moment you stopped the process.

## Monitoring + alerting

A minimal monitoring loop:

```bash
#!/usr/bin/env bash
# alarm if tip > 60s old on a testnet node
TIP_AGE=$(curl -s http://127.0.0.1:8080/api/chain | \
          jq '(.tip_timestamp // 0) | now - .')
if [ "${TIP_AGE%.*}" -gt 60 ]; then
  echo "ALERT: tip age $TIP_AGE > 60s" | mail -s "WaveLedger tip stale" you@example.com
fi
```

Put it in cron every minute. Add similar checks for `peer_count` and
`qrng_reachable`.

For Prometheus + Grafana, the dashboard exposes Prometheus
text-format metrics at `/metrics`:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: waveledger
    metrics_path: /metrics
    static_configs:
      - targets: ['127.0.0.1:8080']
```

Exported gauges (one line per metric):

| Metric | Meaning |
|---|---|
| `waveledger_chain_height` | Tip block index |
| `waveledger_tip_timestamp` | Unix seconds of the tip block |
| `waveledger_total_supply` | WAVE minted so far |
| `waveledger_difficulty` | Current PoW leading-zero count |
| `waveledger_mempool_size` | Pending tx count |
| `waveledger_peer_count` | Connected P2P peers |
| `waveledger_mining_active` | `1` if miner running, else `0` |
| `waveledger_synced` | `1` if IBD complete, else `0` |
| `waveledger_uptime_seconds` | Process uptime |

No auth on `/metrics` even when `require_auth=true`; scrape over a
private network or behind a reverse-proxy ACL.

## Upgrades

```bash
cd /opt/waveledger
sudo -u waveledger git pull
sudo systemctl restart waveledger-chat waveledger-miner waveledger-entropy
```

Most chain-state changes are non-breaking (new tx kinds, new opcodes,
new precompiles). Breaking changes (genesis change, new merkle scheme,
etc.) require a chain reset and are documented in the release notes.

## Backups

| What | How often | Where |
|---|---|---|
| `chain.db` | daily | Off-machine (R2, S3, etc.) |
| `wallets/` | once per wallet creation | Encrypted, off-machine |
| `api_key.json` | once per node | Off-machine (rotate if leaked) |
| Foundation keypair (`~/.waveledger/genesis_foundation.json`) | once at chain genesis | Cold storage; this is the only key that can spend the genesis premine |

For VPSes, `rsync` to a separate location nightly is fine. For fly,
volume snapshots are automatic (retained 5 days by default).

## Capacity planning

- Each block is ~5-50 KB depending on tx count
- 1 block per minute = ~7 GB chain growth per year (mainnet)
- 1 block per 5s = ~88 GB/yr (testnet — wipe periodically)
- Mempool max ~5,000 txs × ~6 KB each = ~30 MB RAM max
- Peer connections ~1 MB RAM each
- Plan ~2 GB RAM for a comfortable miner; 256 MB for entropy

For a 5-year horizon plan ~50 GB disk per mainnet node, conservatively.
