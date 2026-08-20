# Qumbra T2: what launched, what retired, and what to do about it

中文：[`t2-announcement-zh.md`](./t2-announcement-zh.md) · **English is authoritative on technical detail.**

> **T2 is live as of 2026-08-20 14:00 UTC+8 (06:00 UTC).** T1 is retired. This note
> is the long version of the announcement: what the new net is, what happened to
> the old one, and how to join. Step-by-step: [`join-and-mine.md`](./join-and-mine.md).
> The T1 announcement is kept as the record, not deleted:
> [`t1-announcement.md`](./t1-announcement.md).

**What T2 is.** T2 runs the v5 chain format — the one T1 existed to earn:

- **A payee-list coinbase.** A block's reward can be split across a list of recipients
  inside consensus. This is the primitive that makes a custody-free mining pool
  possible: the pool never holds your coins; the block itself pays you.
- **A stratum-compatible header.** The 97-byte block header was laid out so that stock
  XMRig grinds it without a single modified line — the nonce sits exactly where XMRig
  expects to write, and share judging reads exactly the bytes XMRig reads.
- **Exact integer emission from block 1.** T1 launched with a floating-point emission
  schedule and earned an integer-exact one at a mid-chain boundary (and one block paid
  4,114 bessel short before the rule existed — the scar is in T1's archive). T2 has the
  exact rule from its first block. Total supply arithmetic is reproducible by anyone,
  to the unit.
- **The name service, native from block 0.** Short human names (`alice.qmb`) with
  commit–reveal registration (front-running is structurally excluded), burned
  length-tiered fees (1/32/128/512/2048 QMB), write-once records — no transfers, no
  seizures, nothing to steal. On T1 this would have activated mid-chain at block
  19,008; T2 is born with it. That boundary never happens here.

**What happened to T1.** T1 was the rehearsal net, and it did its job: a WAN soak
across three continents, a live emission-boundary repair, the first real user journey,
seven defects found by real money and fixed in a day, and the entire v5 format proven
against it. T1 halted at **Thursday 20 August 2026, 14:00 UTC+8** (06:00 UTC); its
history is archived, not deleted. **T1 balances do not carry over** — T2 is a fair
relaunch: no premine, no carry, block 0 is the same starting line for everyone,
including us.

**How to join.** Two paths — details in [`join-and-mine.md`](./join-and-mine.md):

- **Pool (no node):** install stock XMRig, point it at `pool.qumbra.org:3333` with a
  Qumbra payout address (the join guide walks through making a wallet). Shares are
  tallied PPLNS; block rewards pay the miner list directly in the coinbase.
  **The pool is held until after cutover** — a live-looking stratum that issued jobs
  against a fake template would burn miners' hashpower, so that hostname is not a
  paying endpoint until a later note on this repo says it is. The config is in the
  join guide so it is one paste when that happens.
- **Solo (run a node):** download, verify the genesis hash below, `qumbra-node mine`.
  The join guide covers Linux, macOS, and Windows.
  **There is no T2 binary at cutover.** The T2 release is cut *after* the chain is
  live (the release guard refuses before). Until a T2 tag appears on
  <https://github.com/qumbra-labs/qumbra/releases>, there is nothing to download
  that will follow this chain. Every `t1-*` tag is a T1 artifact — do not run it
  against T2.

Pin this value and trust nothing else:

| what | value |
|---|---|
| network | `qumbra-t2` |
| genesis format | v5 |
| `expected_genesis_hash` | `d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3` |
| genesis file (after cutover) | `https://seed.qumbra.org/genesis.qmb` — byte-verify against the hash |
| T2 binaries | land on <https://github.com/qumbra-labs/qumbra/releases> **after** cutover |
| pool (stock XMRig) | `pool.qumbra.org:3333` — path documented, **held** until announced live |

Image digest, P2P seeds, and the T2 release tag are not on this page because they
do not exist at 14:00. They will be filled on this repo when they do — not guessed.

**What has not changed.** The design that was announced with T1 is the design T2 runs:
post-quantum by construction (hash-based STARK proofs, ML-KEM note encryption, no
elliptic curves anywhere), one shielded pool, 148,625-byte transactions with proving in
seconds on a laptop, key-level selective disclosure instead of protocol backdoors. The
spec and the verifier circuit remain public in the same repos
([whitepaper](spec/whitepaper.md) · [protocol specification](spec/protocol-spec.md) ·
[frozen consensus parameters](spec/consensus-parameters.md)).

**Honesty register, unchanged in spirit.** T2 is still a testnet. The finality
committee's 21 keys are operator-held (opening the committee to outside signers is its
own milestone, with a weak-subjectivity checkpoint design already ruled and waiting);
parameters marked testnet-tunable can still move at declared boundaries; and the word
"mainnet" stays unused until those two things are no longer true. Fair start — and the
record of every claim above is in the public spec on this repo, not in a post.

Found something broken? [Open an issue.](https://github.com/qumbra-labs/qumbra/issues)
