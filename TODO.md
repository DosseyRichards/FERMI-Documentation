# Documentation Followups

Things noticed while writing that should be filled in.

## API Reference
- [ ] Document the `quantum_signature` field on blocks — what fields,
  how to verify, where the attestation key comes from
  (`mining/attestation.py`)
- [ ] Document rate limits per endpoint (currently only chat send has one)
- [ ] Document CORS policy (we set `Access-Control-Allow-Origin: *` on
  JSON responses but no preflight handling — confirm + document)
- [ ] Document admin session model — is HTTP Basic stateless, or does
  setting `WAVELEDGER_ADMIN_PASSWORD` rotate at restart? (it's stateless;
  document explicitly)
- [ ] Document the `signing_key` vs `private_key` distinction on wallet
  objects (and which is needed for `sign_transaction`)

## Nodes
- [ ] Document P2P protocol wire format (`network/protocol.py`) for
  third-party miner implementers
- [ ] Document the headers-first IBD sync protocol + batch sizes
- [ ] Document fork resolution (cumulative-PoW comparison)
- [ ] Document mDNS / DNS-seed discovery in detail
- [ ] Document UPnP behavior (when it runs, what ports)

## Cross-cutting
- [ ] Add an "auditing a WaveLedger chain" page — how to verify the
  chain end-to-end (PoW + PQC attestation chain + tx signatures)
- [ ] Add a "post-quantum threat model" page — what we resist, what we
  don't (e.g., we still rely on SHA3 collision resistance)
- [ ] Add a tokenomics-deep-dive page showing the emission schedule
  with charts
- [ ] Add a comparison table vs Ethereum + Bitcoin for positioning

## Site infrastructure
- [ ] Wire a "View on GitHub" link per page that goes to the source
  Markdown (currently `edit_uri: ""`)
- [ ] Set up real Algolia DocSearch when traffic justifies it (lunr
  local search ships out of the box for now)
- [ ] Add changelog / release notes section
- [ ] Add OpenGraph images per page (auto-generated would be ideal)
- [ ] Add Mermaid + math support if any pages need it (extensions are
  loaded but no pages use them yet)
