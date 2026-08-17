# Qumbra T1 — public mining is open

中文：[`t1-announcement-zh.md`](./t1-announcement-zh.md) · **English is authoritative on technical detail.**

> **PUBLISHED 2026-08-16.** If a seed connection is refused in the first minutes after
> publication, the door is mid-open — the firewall change lands right behind this page;
> retry shortly.

Qumbra is a post-quantum privacy chain: one shielded pool, one STARK per transaction,
RandomX-class CPU PoW for permissionless block production, and a BFT finality committee
that checkpoints the chain. T1 is its first public-facing testnet. **Anyone can now run a
node and mine — no registration, no permission, a CPU is enough.**

## The five values that define the network

| what | value |
|---|---|
| node image (by digest, never a tag) | `ghcr.io/qumbra-labs/qumbra-node@sha256:c7b8b3340d35d7461daaa83acea6a8eef045bdba74d5173f9f059322ec18adbb` |
| image revision (OCI label, read it back) | `e0b624596e29dc8a95d1f6715ed061b4247a73e2` |
| genesis file | `https://seed.qumbra.org/genesis.qmb` (format v4) |
| `expected_genesis_hash` | `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb` |
| P2P seeds (`dial_peers`) | `18.202.166.126:9444` · `18.141.177.109:9444` · `52.194.224.123:9444` · `52.5.0.21:9444` |

**How to join, step by step: [`join-and-mine.md`](./join-and-mine.md).** The short version:
verify the image by digest and revision label, byte-verify genesis against the hash, join as
a non-mining node, watch `slag` fall to 0, then set `mining = true` with a `miner_rkm` from
`qumbra-wallet miner-rkm` — **without `miner_rkm`, everything you mine is burned**, and the
node tells you so at startup. Mining is in-node and solo; there is no separate miner
program and no pool yet (both are deliberate — see the design repo's pool brief).

## Public services

- `https://seed.qumbra.org` — wallet discovery edge (`/v1/compact`, `/v1/coinbase`,
  `/v1/nullifiers`, `/v1/tree/leaves`, `POST /v1/tx`) and the genesis download
- `https://explorer.qumbra.org` — chain health (tip, finality, committee, exact supply audit)
- `https://faucet.qumbra.org` — grants for trying transactions before your coinbase matures

## Name service — activation pre-announced, by design

Qumbra will carry a native name layer: `NAME.qmb` resolving to a payment address, registered
on-chain. It is **built and merged but inert** — it **activates at height 19,008**
(stamped 2026-08-17; expected around the morning of 2026-08-21 +08 at the ~48 blk/h pace.
The height is exact — the chain activates at that block, wherever the clock stands; the date
is only an estimate).

**Why the height is pre-announced:** registration fees are **burned QMB**, and mining is how
QMB comes into existence — so the activation height is public information from day one, and
everyone (including you, from your first mined block) gets the same accumulation window
before anyone can register a single name. There is no insider registration period.

What you can prepare for (shape is final; numbers marked pending are not):

- **Commit–reveal registration**: commit first (window 8 blocks), reveal within 2,304
  blocks — pre-announcing a height does not enable sniping, the two-step blunts it.
- **Annual terms**: 365 epochs, plus a 90-epoch renewal grace window.
- **Fee table by name length** — 1 / 32 / 128 / 512 / 2048 QMB — **FINAL**
  (re-ratified with the height stamp, 2026-08-17). Annual term 365 epochs + 90 grace as
  above; all registration fees are burned.

## What T1 is, honestly

- **Coins have no value** and will not survive the next re-mint: a re-genesis is already
  queued (payee-list coinbase + stratum-compatible header layout ride the next genesis).
  Mine to exercise the system, not to accumulate.
- **The finality committee is centralized for now** — all 21 keys are the operator's. What
  is permissionless today is block production; committee decentralization is a later phase.
- Coinbase matures after **144 blocks** (~3 h at the ~75 s target); the wallet's `scan`
  shows spendable vs maturing explicitly.
- Your node syncs from genesis in minutes on current code (checkpoint-sync + incremental
  tree); a laptop-class CPU mines meaningfully — the fleet is four modest cloud hosts.
- Outbound-only participation (behind NAT) is fully supported: you sync, mine, and
  transact; you just serve no peers.
- The seed edge sees which block ranges your wallet requests (bandwidth-pattern side
  channel, Cloudflare-proxied). The gated future answer is an OMR overlay; until then this
  is stated rather than hidden.

Found something broken? That is what T1 is for — [open an issue here](https://github.com/qumbra-labs/qumbra/issues).
