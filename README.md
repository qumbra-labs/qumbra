# Qumbra

**A post-quantum privacy chain** — one shielded pool, one STARK per transaction (no
per-spend signatures), conservative Keccak/SHA-class hashing throughout consensus, ML-KEM
note encryption, and hybrid PoW + BFT finality: RandomX-class CPU proof-of-work makes block
production permissionless while a finality committee checkpoints the chain.

## T2 public testnet — the chain that counts

> **T1 is retired** (2026-08-20 14:00 UTC+8). T2 is a fair relaunch from a fresh genesis:
> no premine, no carry. T1 balances do not survive. The T1 announcement is archived as
> the record, not deleted: [`docs/t1-announcement.md`](docs/t1-announcement.md)
> ([中文](docs/t1-announcement-zh.md)).

**[Read the T2 announcement →](docs/t2-announcement.md)** ([中文](docs/t2-announcement-zh.md))

Anyone can run a node and mine, or point stock XMRig at the pool. No registration, no
permission — a CPU is enough.

**[How to join and mine →](docs/join-and-mine.md)** ([中文](docs/join-and-mine-zh.md))

Two join paths:

| path | what you run | status at cutover |
|---|---|---|
| **Pool** | stock [XMRig](https://github.com/xmrig/xmrig) at `pool.qumbra.org:3333` | path documented; **held** until a later note says it is live |
| **Solo** | `qumbra-node mine` with the T2 genesis | **T2 binaries are published: [`t2-644a129`](https://github.com/qumbra-labs/qumbra/releases/tag/t2-644a129)** (linux x86_64/aarch64, macOS arm64, Windows) — verify against `SHA256SUMS`; do not run a `t1-*` tag against T2 |

**Pin this genesis hash and trust nothing else:**

`d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3`

Network name `qumbra-t2`. Names are native from block 0 (the T1 height-19,008 boundary
never happens here).

**What am I mining? The spec is public:**
[whitepaper](docs/spec/whitepaper.md) · [protocol specification](docs/spec/protocol-spec.md) ·
[frozen consensus parameters](docs/spec/consensus-parameters.md)

| service | URL |
|---|---|
| chain health | https://explorer.qumbra.org |
| wallet / discovery edge | https://seed.qumbra.org |
| genesis file | https://seed.qumbra.org/genesis.qmb — byte-verify against the hash above |
| pool (stock XMRig) | `pool.qumbra.org:3333` — held until announced live |

The T1 faucet is retired with T1. Coins on T2 come from mining.

## Status, honestly

T2 is a testnet. Coins have no value. Block production is permissionless today; the
finality committee is still operator-run — decentralizing it is a later phase. The
development tree (design documents, node source, deployment) stays private on its own
schedule; these documents mirror it at publication and it remains authoritative.

**Found something broken? [Open an issue.](https://github.com/qumbra-labs/qumbra/issues)**

## License

Documents and released artifacts in this repository are dual-licensed under
[MIT](LICENSE-MIT) OR [Apache-2.0](LICENSE-APACHE), at your option.
