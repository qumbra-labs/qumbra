# Qumbra

**A post-quantum privacy chain** — one shielded pool, one STARK per transaction (no
per-spend signatures), conservative Keccak/SHA-class hashing throughout consensus, ML-KEM
note encryption, and hybrid PoW + BFT finality: RandomX-class CPU proof-of-work makes block
production permissionless while a finality committee checkpoints the chain.

## 🚀 T1 public testnet — mining is open

**[Read the announcement →](docs/t1-announcement.md)** ([中文](docs/t1-announcement-zh.md))

Anyone can run a node and mine. No registration, no permission — a CPU is enough.

**[How to join and mine →](docs/join-and-mine.md)** ([中文](docs/join-and-mine-zh.md))

| service | URL |
|---|---|
| chain health | https://explorer.qumbra.org |
| wallet / discovery edge | https://seed.qumbra.org |
| faucet | https://faucet.qumbra.org |

## Status, honestly

T1 is a testnet. Coins have no value and will not survive the next re-genesis. Block
production is permissionless today; the finality committee is still operator-run —
decentralizing it is a later phase. The development tree (design documents, node source,
deployment) is private during T1 and opens on its own schedule; these documents mirror it
at publication and it remains authoritative.

**Found something broken? [Open an issue.](https://github.com/qumbra-labs/qumbra/issues)**
