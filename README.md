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
| **Solo** | `qumbra-node mine` with the T2 genesis | **T2 binaries: [the latest release](https://github.com/qumbra-labs/qumbra/releases/latest)** (linux x86_64/aarch64, macOS arm64, Windows) — verify against `SHA256SUMS`. 🔴 **Use `t2-a89dce6` or newer**: earlier T2 builds cannot reopen their own data directory after a restart ([#521](https://github.com/qumbra-labs/qumbra-lab/issues/521)) — they start fine and fail the *second* time. Never run a `t1-*` tag against T2. 🔴 **No public release pins the re-launched genesis yet** — see below |

**Pin this genesis hash and trust nothing else:**

`59d9f054bb15116dac40c42ddb67c7d377407eec3010e9f98a5cc76e9e0544b1`

🔴 **T2 was re-launched on 2026-10-03 from genesis `59d9f054…44b1`; `d1dad4ea…e2f3` is
retired. No public release pins the new genesis yet.** Every release up to and including
`t2-2026.08.25-1` is built from a revision that pins `d1dad4ea…e2f3`, so it cannot join
the re-launched T2. A release for `59d9f054` is pending.

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
