# HF22 consensus changes — build spec (for Dom)

Branch `feat/hf22-consensus-fixes` (from `feat/hf22-sn-policy`). Decisions below are final — this is the
implementation list, not a design discussion. All items are consensus-critical: implement here, then
validate with the 20-operator harness (`testnet/hf22-multiop/`, run single-host per
`testnet/hf22-multiop/localhost/README-localhost.md`) plus a mixed old/new-binary pass before merging to
`release/hf22`.

## Already done on this branch (do not redo)
- **Q5b** — fallback signer skips Pulse blocks: `&& !b.has_pulse()` added at `cryptonote_core.cpp:2415`. ✅
- **Q11** — 14-day lock moved to `network_config.DEREGISTRATION_LOCK_DURATION_V2` (mainnet 14d, testnet 30s);
  replaces the hardcoded `hours(14*24)` at `service_node_list.cpp:1337`. ✅

## Already set on feat (do not change)
- Fallback wait: `PULSE_MINER_FALLBACK_ROUNDS = 2` (~60s) on mainnet (b7f3213). Consensus-critical
  (`pulse.cpp:985 miner_fallback_timestamp` → `service_node_list.cpp:3224`); set as part of HF22.
- Operator-fee cap stays 10% (`cryptonote_config.h MAX_OPERATOR_FEE_BASIS = 1000`) — no change.

## To implement

### Q5a — fork-gate the fallback sign+verify on `hf22_sn_policy`
Gate on `hf_version >= hf::hf22_sn_policy` (pattern: `service_node_quorum_cop.cpp:154`):
- `cryptonote_core.cpp:2415` signing condition (currently `>= hf::hf16_pulse`).
- The verify side that accepts the fallback signature (`sig.voter_index==0xFFFF`, `service_node_list.cpp:2788`).
- The `miner_fallback_timestamp` block-type gate.
Both sides move together (old/new nodes must agree during the upgrade window).

### Q5c — multiple authorized fallback miners (set of dedicated keys)
- Add `FALLBACK_MINER_PUBKEYS` (a **set**) to `network_config`; verify accepts a fallback block signed by
  **any** key in the set. Keep the `voter_index=0xFFFF` sentinel.
- Fallback key is loaded from a **file** (arg = path, not hex). Remove the unsynchronized static cache in
  `get_fallback_miner_pubkey`. Do **not** use the governance spend key.
- Production: two authorized miners on independent providers — **maple (OVH)** and **OCI (Oracle)**, each
  with its own dedicated key; both pubkeys go in the mainnet `FALLBACK_MINER_PUBKEYS`. (Ops generates the
  keys and fills the pubkeys — not Dom.)
- Per-miner **local start-delay** flag (primary = 0, secondary = +N rounds) to avoid simultaneous
  production; runtime-only (validity stays "from round 2"), no fork needed.

### Q6 — refill deduped obligations/checkpoint quorums to 10/20/10
After operator-dedup, refill from the remaining shuffled candidate list so obligations/checkpoint/blink
quorums keep full size (`service_node_list.cpp`). Pulse keeps 1-seat-per-operator.

### Q7 — unify the Pulse candidate threshold on 12
Make `update_from_block`'s round-0 bail (currently 11) match `generate_pulse_quorum`
(`PULSE_QUORUM_NUM_VALIDATORS`, +1 for round>0) and the dedup fallback (`PULSE_MIN_SERVICE_NODES = 12`).
Add a shared constant/assert so they can't drift. (12 is required: at 11 a single-SN leader can drop
round-0 to 10 sigs, below the 7-sig floor.)

### QL — Lokinet ships in HF22 (fork-gated + grace period)
Lokinet is a separate daemon run alongside xeqm-d (xeqm-core only checks its ping); it has never run on
XEQM mainnet. First produce + verify a working XEQM Lokinet build (network id/ports/reachability).
- Fork-gate the enforcement on `hf_version >= hf::hf22_sn_policy` (pattern at
  `service_node_quorum_cop.cpp:154`), then set mainnet `HAVE_LOKINET=true`:
  - `cryptonote_core.cpp:2684` — the "no uptime proof without a recent Lokinet ping" check.
  - `service_node_list.cpp:5966` — the min-Lokinet-version proof reject.
- **Grace period**: no decommission for missing/unreachable Lokinet for N days after the fork
  (new `network_config` constant), then enforce.
- Confirm/set the min Lokinet version in `service_node_rules.h`.
- Operator delivery (every SN must run Lokinet before the fork):
  - Installer (`~/xeqm-node-installer-script/`, no Lokinet today): add install+config to `install.sh`,
    ports to `firewall.sh`, a reachability check to `doctor.sh`.
  - **Docker image** (Dom builds): extend the repo `Dockerfile` to bundle xeqm-d + Lokinet + config so
    non-installer operators run a ready SN via `docker run`.
- Testnet: add a Lokinet-enabled profile + run Lokinet on the test SNs (harness currently HAVE_LOKINET=false).
- Docs: update the whitepaper + defense.xeqmlabs.com (both currently say "HF22 has no Lokinet dependency").

## Validation gate (before merge to release)
1. All platforms build on CI (macOS Intel on the `macmini-intel` self-hosted runner).
2. 20-operator stall/recovery run → confirm dedup + refill + multi-miner fallback (kill primary miner
   mid-stall, secondary keeps producing) + Lokinet gating + grace period.
3. Mixed old/new-binary pass → confirm the fork-gates (Q5a, QL) don't split the chain.
4. Merge from Fernando so commits/tag are signed (Verified).
