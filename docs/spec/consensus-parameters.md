# Qumbra Consensus Parameters — FROZEN v1.0

> **Provenance (2026-08-17, staged open-sourcing stage 1 — design tracker #203):** this file is
> published from Qumbra's private design repository, authoritative as published here and synced
> forward on spec changes. Cross-references to design docs or lab PRs/issues not yet published
> will 404 for outside readers — they document where each claim's full argument and test lives,
> and resolve as later stages open. Nothing was removed from the source text.

> [中文版](consensus-parameters-zh.md)

> **🧊 GENESIS FREEZE v1.0 (2026-07-23, Larry — the explicit "freeze" stamp; every technical gate verified cleared: 2025/2197 close-read §fri-soundness-6, B″ measured lab PR #47, all rows decided).** The table below is the canonical frozen constant set. *Frozen* means: these are the **genesis values**; every one is a versioned consensus parameter (per [degree5-field-migration](degree5-field-migration.md)'s door) changeable henceforth **only** via a halt-height upgrade carrying its own revision doc — never silently. The section-by-section record below the table (grounds, sources, measured updates) is preserved unchanged as the audit trail.
>
> | § | Constant | FROZEN v1.0 |
> |---|---|---|
> | 1 | Base field / extension | KoalaBear (2³¹−2²⁴+1), degree-4 (challenge ≈ 2^124) |
> | 1 | Consensus FRI | b16 / **q21** / g22 / fp16 / a16 — measured 145,609 B (142.2 KB) |
> **Measured update (2026-08-04 — THE MINT, [lab PR #252](https://github.com/qumbra-labs/qumbra-lab/pull/252), coordinator-accepted 1218/0 unfiltered, genesis reproduced byte-identical twice through the real CLI): the consensus wire is now 148,625 B (145.1 KB), still ≤ 150 KB (3.3 % margin against the 153,600 B ceiling). The figure above is the pre-mint record.** One pre-T1 re-mint (legitimate only because no chain existed under the old genesis) carried three changes: **(1)** #215 (i)/option-4 ρ′ derivation — `ρ′_0 = nf_0`, `ρ′_1 = H(nf_0 ‖ D_P|1)` via a new `ROLE_ARHO` permutation (83 → 84 perms) plus a third equality bank, closing the faerie-gold free witness structurally; **(2)** the #219 dummy-liveness latch (`dv`) made unconditional — single-note spends are in-circuit legal; **(3)** the #188 (a) discovery payload — 120 B/note full plaintext per the amended ruling (an `Ivk` deliberately cannot derive `rkm`; the served wire carries no nullifier) — relocated into the **committed** body region, `/v1/compact` serving it by projection. Trace width 617 → 643 (accounted column by column); max constraint degree and quotient chunks **unmoved at 4**; genesis `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb`, `GENESIS_FORMAT_VERSION` 4. b4-fallback interior caveat: see the aggregation-rung1 §6 breach block ([lab #257](https://github.com/qumbra-labs/qumbra-lab/issues/257)).
> | 1 | Aggregation lanes | leaf b4/**q43**/g22 · interior b2/**q86**/g22 (≤ 32 GB measured) |
> | 1 | Security label | ~100-bit conjectured, 2197-corrected (measured 100.6/101.6/100.2); proven-Johnson field-capped ~80 |
> | 1 | Hash / tree depth | Keccak-256 everywhere in consensus / depth 32 |
> | 2 | Emission | 75-s blocks; r0 = 50 QMB; half-life 2 y; tail 1.22441 QMB/block (d = 8.237e-7); no hard cap |
> | 2 | Coinbase maturity | 144 blocks (3.0 h) |
> | 3 | Split | 65 % miners / 15 % committee / 20 % treasury |
> | 4 | Committee | N = 21; quorum 15; epoch 1,152 blocks (24 h) |
> | 4 | Self-bond | 10⁴ QMB steady-state; **ramp pinned at freeze: 0 (epochs 0–89) → 10² (from epoch 90) → 10³ (from 180) → 10⁴ (from 360)** — supply at those boundaries (≈ 5M / 9.8M / 18.6M QMB) makes each step trivially acquirable then binding |
> | 4 | Slashing / jail | equivocation 10 % + permanent tombstone; downtime jail (no slash) at < 33 % signed of trailing 100 |
> | 5 | Fees | 0.01 / 0.02 / 0.04 QMB (2×2 / 4×4 / 8×8), emission-anchored floor |
> | 6 | Block weight | free zone 10 MB; hard cap 2×; long window 100,000; lt 1.4× / st 50 |
> | 7 | Anchors | finalized-only; ≤ 1,152 blocks (24 h); 10-min buckets (= 8 blocks); ~144 retained roots |
> | 8 | Denomination | **1 QMB = 10⁸ bessel**; ticker QMB |

> **DECIDED (2026-07-22; Larry delegated the stamps to the coordinator with the instruction "no shortcuts — do it right"; grounds recorded per item):** §2 emission = **candidate B2 ADOPTED as proposed**. §4 = **N=21 and epoch 1,152 blocks ADOPTED; equivocation slash AMENDED 100 % → 10 % + permanent tombstone** — the coordinator's own 100 % proposal failed review against committee-and-governance §3's decided reasoning ("first offenses should sting, not destroy — equivocation must end tenure"): tenure ends via the tombstone; the slash need only sting (10 % = an order above Cosmos's 5 %, ~3× Ethereum's initial 1/32), and total-bond destruction for what the field's record shows is almost always an HA misconfiguration would deter competent candidate-net operators. §5 fee ratios 1/2/4 **ADOPTED** (triple-checked: action count, output count, and circuit work all independently give 1:2:4). The [open] rows remain open; genesis freeze remains separate and 2025/2197-gated.

> **DECIDED (2026-07-22 evening — the denomination pass, same delegated chain; grounds + inline sources in §8, calibration in §5):** §8 denomination = **1 QMB = 10^8 atomic units; the atomic unit is named the *bessel*; ticker QMB confirmed clean (QUM fallback retained, QTUM-confusability flagged)**. This pins every "denomination-relative" value in one pass: §2 r0 = **50 QMB/block absolute** (D1's 6.25 "21M-class" rescale REJECTED — pure headline marketing the tokenomics doc's honest-framing stance already refuses; the curve is identical); §5 fee absolutes = **0.01 / 0.02 / 0.04 QMB** for 2×2 / 4×4 / 8×8 (anti-sandblasting calibration, native-anchored, §5); §4 self-bond minimum = **10^4 QMB steady-state, genesis ramp 0 → 10² → 10³ → 10⁴** (fair launch means zero supply at genesis — day-1 bonding is arithmetically impossible). One carve-out from the delegation: the unit *name* is a brand surface, so **"bessel" carries Larry's veto at PR review**; the numeric decisions ride the delegated chain.

**Status: PROPOSAL v0.1 (2026-07-22) — the single place every "parked to the consensus-parameters appendix" constant is gathered and given a *proposed* value with its basis. Larry decides each; nothing here is frozen.** Its stated gate — "gated on prototype benchmarks" (tokenomics §7, committee-and-governance §7, consensus §10) — is now met: M1–M6 supply the measured basis, and `qumbra-lab`'s emission simulator ([qlab-econ, PR #36](https://github.com/qumbra-labs/qumbra-lab/pull/36), `docs/econ-sweep.md`) supplies the emission candidate trade-space. Every value is **PROPOSED**; genesis freeze is a separate, later act (and gated additionally on the 2025/2197 close-read for the security row — [fri-soundness §5](fri-soundness-accounting-2026-07.md)).

Convention: **[decided]** = already fixed in a prior doc, restated here for one-place reference; **[proposed]** = this doc's recommendation for Larry; **[open]** = surfaced but not yet recommended.

## 1. Security / proof-system parameters **[decided — restated]**

| Parameter | Value | Source |
|---|---|---|
| Base field | KoalaBear (2^31 − 2^24 + 1) | prototype-bench, all milestones |
| Extension degree | 4 (challenge field ≈ 2^124) | [degree5-field-migration](degree5-field-migration.md): stay degree-4, versioned |
| Consensus FRI config | blowup 16 / **21 queries** / grind 22 / final-poly 16 / fold-arity 16 | [self-proving §4](self-proving-vs-proof-size.md) + B′ + **B″ (queries 20→21, measured 2026-07-23 — [lab PR #47](https://github.com/qumbra-labs/qumbra-lab/pull/47))** |
| Aggregation leaf lane | b4 / **q43** / g22 | aggregation-rung1 §4 + B″ |
| Aggregation interior lane | b2 / **q86** / g22 | aggregation-rung1 §4 (decided 2026-07-21) + B″ |
| Security label | **~100-bit conjectured (2197-corrected: list-decoding capacity in base-field entropy — 100.6/101.6/100.2 measured)**; proven-Johnson field-capped ~80 | fri-soundness §3 + §6 |
| Commitment/PRF hash | Keccak-256 everywhere in consensus | performance-budget §2 (conservative-hash decision) |
| Tree depth | 32 | transaction-model §4 |

**Versioned, not baked (per degree5-field-migration):** extension degree and FRI config are recorded here as consensus parameters upgradable at halt-height — no protocol surface hard-codes them.

> **2025/2197 close-read + B″ (2026-07-22, [fri-soundness §6](fri-soundness-accounting-2026-07.md)):** the conjectured label above was repriced — for extension codes the list-size entropy base is the **base field**, so the measured q20-era configs honestly carry **96.9 / 96.1 / 94.8 conjectured** (consensus / leaf / interior), not ~100; no efficient attack exists and the proven-Johnson floor is untouched. **B″ DECIDED (delegated): queries 20→21 / 40→43 / 80→86, restoring 100.6 / 101.6 / 100.2 under the corrected accounting** — the ~100-bit floor is a standing invariant and pre-freeze is the cheap side of the asymmetry. The FRI-config row above becomes **target b16/q21/g22**.
>
> **B″ IMPLEMENTED AND MEASURED (2026-07-23, [lab PR #47](https://github.com/qumbra-labs/qumbra-lab/pull/47), coordinator-accepted — 268/268 unfiltered, consensus wire byte-identical ×2 + coordinator rerun):** consensus b16/**q21**/g22 = **145,609 B (142.2 KB) ≤ 150 KB ✓**; leaf b4/**q43**, interior b2/**q86** = 20.2–20.9 GB (coordinator-independent 22.5 GB on a busy machine) **≤ 32 GB ✓**; both pre-registered rectangle fit-checks PASS as permanent tests (leaf 2^16 @ 91.0 %, interior 2^19 @ 82.3 %). The corrected conjectured labels are now **measured at 100.6 / 101.6 / 100.2**. **The genesis-freeze gate on this section is CLEARED** — freezing is now a pure act of record.

## 2. Emission constants **[proposed]**

Basis: the [qlab-econ sweep](https://github.com/qumbra-labs/qumbra-lab/pull/36)'s 15-candidate table. All Monero-class candidates satisfy tokenomics §1's four jobs by construction (smooth closed-form decay, exact `S(h)`, perpetual tail, no cliffs); the choice is the trade-space, not correctness.

**Recommended: candidate B2 — Monero-class, 75 s blocks, 2-year half-life, 0.87 % tail target.**

| Constant | Proposed value | Rationale |
|---|---|---|
| Emission family | Monero-class `reward(h) = max(r0·(1−d)^h, tail)` | tokenomics §3: value-continuous, exact closed-form `S(h)` = the supply-audit anchor (§1 job 4); Todd's asymptotic-non-inflation applies to the fixed tail |
| Block time | **75 s** | consensus §7: Zcash ZIP-208 precedent; the propagation budget was priced at this cadence |
| Half-life | **2 years** | mid of the swept {1, 2, 4}: front-loads distribution moderately without a 4-year founder-dilution lag |
| Initial reward r0 | **50 QMB/block [decided 2026-07-22]** (= 5×10⁹ bessel — §8) | pure denomination knob; sets S_inf scale, not the curve shape; D1 rescale rejected (§8) |
| Tail target | **0.87 %** inflation at activation | Monero's measured landing (0.87 %) — the field's one real datapoint for "perpetual tail that didn't break the chain" |
| Derived per-block decay d | 8.237e-7 | closed-form from {r0, half-life, block time} — not an independent knob |
| Derived tail | 1.22441 coin/block | closed-form from the above |
| Coinbase maturity delay | **[decided 2026-07-23]** **144 blocks** (= 3.0 h at 75 s; ⅛ epoch) | reorg behavior now characterized ([lab PR #46](https://github.com/qumbra-labs/qumbra-lab/pull/46) Monte-Carlo, degraded mode): worst measured depth for q ≤ 0.40 incl. a pathological 30-day stall = **118**; 144 covers it with 22 % margin; the q→½ tail is finality's job, not maturity's (unbounded as q→½ — no depth covers it) |

B2 lands (from the sweep): tail activates **year ~10.7** (block 4,503,708) at 0.87 % inflation, 0.65 % at year 50, trending to ~0.09 % at year 1000. Decay-part asymptote r0/d ≈ 60.7M coin (r0-relative); supply then grows linearly via the tail — 79.5M at year 50, **no hard cap** (tokenomics: honest framing over "21M" marketing). **Alternatives on the table:** M5 (same but 60 s blocks, if throughput is later prioritized over the propagation budget); D1 (r0 = 6.25, if a "~21M-class" headline supply is wanted — pure denomination rescale, curve identical). **(D1 REJECTED 2026-07-22, denomination pass — §8: a rescale buys a headline number and nothing else, and the honest-no-hard-cap framing already refuses cap marketing.)**

## 3. Reward split **[decided — restated]**

**65 % miners / 15 % finality committee / 20 % consensus-native treasury** ([tokenomics §4](tokenomics-and-issuance.md)). Applied to `reward(h)` per block; treasury accrues consensus-natively (DCP-0006 shape), committee accrues per-checkpoint claimed per-epoch (committee-and-governance §7).

## 4. Committee / staking constants **[proposed + open]**

| Constant | Proposed | Source / rationale |
|---|---|---|
| Genesis committee size N | **21** (odd, in the decided N≈20 band) | committee-and-governance §1; odd N eases ⅔ quorum arithmetic |
| Quorum | ⅔ + 1 (15 of 21) | BFT standard |
| Epoch length | **[proposed]** 1,152 blocks (= 24 h at 75 s) | the Sui end of the field's 2–24 h band — at N=21 *named* entities membership churn is rare, so a daily boundary (round, human-schedulable) beats a fast one; committee-payout cadence keys off it |
| Self-bond minimum | **[decided 2026-07-22; ramp steps FROZEN 2026-07-23]** 10⁴ QMB steady-state; ramp 0 (epochs 0–89) → 10² (from 90) → 10³ (from 180) → 10⁴ (from epoch 360) | committee §2: power-of-10 quantized (fingerprint resistance). Sizing: 21 × 10⁴ = 210k QMB ≈ 1.1 % of year-1 supply (~18.6M, qlab-econ S(h)) — acquirable by named entities without a market squeeze; 10 % slash = 10³ QMB. The ramp exists because fair launch ⇒ **zero supply at genesis**: day-1 bonding is arithmetically impossible, so launch-era entry is track-record + committee-vote gated (committee §2) while the bond phases in. Steady-state 10⁵ was considered and set aside: at year-1 committee income (~411 QMB/day/validator) even 10⁴ is recouped in ~24 days — early-era skin-in-game is the tombstone + candidate-net reputation, not the bond; 10⁵ (11 % of year-1 supply locked) would squeeze entry for no additional deterrence |
| Equivocation slash | **[decided 2026-07-22]** 10 % of bond + permanent tombstone | committee §3's own register: tenure ends via tombstone; the slash stings (order above Cosmos 5 %, ~3× Ethereum 1/32) without destroying an operator over the HA-misconfig accidents the field record shows dominate |
| Downtime penalty | jail, **no slash**; re-admit after timeout | committee §3 (decided posture) |
| Downtime jail threshold | **[decided 2026-07-23]** signing < **33 %** of the trailing **100-block** window | [lab PR #46](https://github.com/qumbra-labs/qumbra-lab/pull/46) grid: false jails are a NON-constraint at named-entity uptimes (zero to 80 % uptime across the whole swept grid) → pick purely for detection speed; (33 %, 100) detects a dark validator in 67 blocks ≈ 1.4 h with huge margin |

## 5. Fee schedule **[decided 2026-07-22]**

Posted-price, `fee = f(arity bucket)`, ZIP-317-shape ([consensus §8](consensus-and-network.md)). A small public table keyed on the transaction's padded bucket:

| Bucket | Fee **[decided 2026-07-22]** | Basis |
|---|---|---|
| 2×2 | **0.01 QMB** (10⁶ bessel) — the base | the launch bucket; calibration below |
| 4×4 | **0.02 QMB** (2×10⁶ bessel) | ~2× circuit work (ratio decided 2026-07-22) |
| 8×8 | **0.04 QMB** (4×10⁶ bessel) | ~4× circuit work |

**Anti-sandblasting calibration (the absolute, decided 2026-07-22 — native-anchored on emission, never fiat).** The field's one measured fiat-anchored floor is the failure case: ZIP-313 priced Zcash fees at "1 U.S. cent per transaction" at an assumed ZEC price ([ZIP-313](https://zips.z.cash/zip-0313)) and bought 13+ months of sandblasting — shielded outputs 42,600/month → 21,622,590/month (≈507×) from June 2022, at ~$10/day to the attacker ([ECC retrospective](https://electriccoin.co/blog/a-look-back-nu5-and-network-sandblasting/), [Lopp's estimate](https://cryptopotato.com/how-someone-is-attacking-the-zcash-network-for-10-a-day/)), tripling the chain to >100 GB ([protos](https://protos.com/zcash-chain-triples-in-size-thanks-to-10-a-day-spam-attack/)). The fix, ZIP-317 (5,000 zatoshis per logical action, no fiat reference — [ZIP-317](https://zips.z.cash/zip-0317)), is credited with ending the episode; at that level, sustaining the June-2022 rate (~720k outputs/day) costs ≈36 ZEC/day ≈ **1.0 % of Zcash's then-daily emission** [computed: 3,600 ZEC/day at 3.125 ZEC / 75 s]. The countercase is Monero: a floor at ~0.58 % of daily tail emission measurably failed — the March 2024 black-marble flood bought ~75 % of all new outputs for 2.5 XMR/day ([Rucknium](https://rucknium.me/html/monero-black-marble-flood.pdf)); Rucknium's response analysis is also the field's one explicit emission-relative budget framework (adversary budgets capped at the daily tail-emission security budget — [Defeating a Black Marble Flood](https://github.com/Rucknium/misc-research/blob/main/Monero-Black-Marble-Flood/pdf/monero-black-marble-optimal-fee-ring-size.pdf)).

Qumbra's exposure differs in kind from Monero's: no rings, so attacker-owned notes shrink nobody's anonymity set (transaction-model §3) — the defended resource is *linear*: the **585 B/note forever-stream** every light wallet scans (note-discovery §2, measured [PR #28](https://github.com/qumbra-labs/qumbra-lab/pull/28)). But each Qumbra output costs ~5× Zcash's 116-B compact output in stream bytes, and the recovery asymmetry is severe: lowering a too-high fee at an upgrade is painless, while Zcash needed 13 months to recover from a too-low one. So the floor lands **~5× above ZIP-317's level after adjusting for supply scale and per-output bytes** (0.005 QMB/output vs 5×10⁻⁵ ZEC/action: 100× nominal, ~26× supply-adjusted, ~5× byte-adjusted): at 0.01 QMB per 2×2 bucket, sustaining a Zcash-scale injection (~720k outputs/day) costs **3,600 QMB/day ≈ 6.3 % of launch daily emission (57,600 QMB/day), 9.6 % of the miner share — rising monotonically as emission decays, past 2.5× total daily emission in the tail era**. Per forever-stream byte: ≈8.5 QMB/MB (Monero's measured-too-low floor: ≈0.02 XMR/MB). Deliberately **not** adopted: Monero's reward-linked dynamic floor (fee declines with emission — exactly the property that let the 2024 flood get cheap, [A note on fees](https://www.getmonero.org/2017/12/11/A-note-on-fees.html)); a fixed absolute strengthens with time instead. The table is a versioned consensus parameter (§1 note): lowering is the cheap direction if organic demand warrants; the dynamic-median direction ([Shielded Labs' fee laboratory](https://fees.shieldedinfra.net/), with power-of-ten bucketing for fee-privacy) is a standing watch, not a launch dependency. Fees paid to miners, not burned (tokenomics §6: one transparent flow). Note the fee reveals only the bucket — which the padded transaction shape already makes public — so posted-price is also the fee-privacy optimum (no fee-entropy side channel).

## 6. Block-weight anti-spam **[decided 2026-07-23]**

Monero-style long-term-median + quadratic penalty ([consensus §8](consensus-and-network.md), two-median form). *(Original framing, preserved: constants deferred to M6 devnet load-testing.)*

> **Measured correction (2026-07, [lab PR #46](https://github.com/qumbra-labs/qumbra-lab/pull/46) — the load harness):** any reading of this section as the *primary* spam bound is wrong. The two-median governor is **demand-adaptive and does not bound a budgeted flood** — at launch, governed chain growth equals the governor-OFF baseline at every attacker budget (the quadratic penalty has zero slope just above the median, so a myopic-rational miner ratchets the median up to the attacker's sustainable byte-rate). **The §5 fee floor is the binding anti-spam control** (reinforcing its deliberately-high setting); the governor is an anti-gigantism (hard 2M cap — the one unconditional bound) + surge-damping backstop, whose one high-leverage knob is the long window.

**Decided constants (2026-07-23, delegated; data: PR #46 sweep tables):** free zone `min_weight` = **10 MB** (sized to organic launch capacity ~1 TPS × 75 s × 136 KiB — larger free zones are cheaper to bloat); hard cap `max_multiple` = **2** (Monero); long-term window = **100,000 blocks** (≈ 87 days at 75 s, Monero-parity — the measured creep damper: a 30-day flood never turns the window over; the swept 5k devnet default let the median ratchet 348×, 50k cut it to 209×); `lt_cap` = **1.4×**, `st_cap` = **50** (Monero values — measured low-sensitivity for sustained abuse, retained absent a surge-accommodation reason).

## 7. Anchor policy **[decided — restated]**

| Constant | Value | Source |
|---|---|---|
| Anchor validity window | ≤ 24 h | performance-budget §8 |
| Anchor quantization | 10-minute buckets | performance-budget §8 |
| Retained historical roots | ~144 (24 h / 10 min) | derived |
| Anchor source | finalized roots only; finality lag = minimum anchor age | consensus §6 |

## 8. Denomination **[decided 2026-07-22]**

*(Original framing, preserved: ticker + smallest-unit naming from the [naming doc](naming-and-branding.md)'s grab list — decides r0's absolute scale and every "denomination-relative" value above; r0 and the fee table scale together with denomination; the curve shape and the split are denomination-invariant, so this was pinned last.)*

**Decision: 1 QMB = 10⁸ atomic units; the atomic unit is the *bessel*; ticker QMB (fallback QUM).** Grounds:

**(a) The exponent is u64-bounded, and the measured circuit freezes u64.** Value is `u64` end-to-end in the prototype (qlab-air witness, qlab-note wire, qlab-wallet), and the in-circuit balance is an exact 4×16-bit-limb borrow chain (prototype-bench §10) — u128 would mean re-opening a measured, gate-passing circuit. With B2's no-hard-cap emission (tail ≈ 515,190 QMB/yr), the u64 overflow horizon by exponent:

| Exponent | u64 capacity (QMB) | Overflow horizon | Verdict |
|---|---|---|---|
| 10⁸ | 1.84×10¹¹ | ~358,000 y | **adopted** |
| 10⁹ | 1.84×10¹⁰ | ~35,700 y | viable — Grin's shape (10⁹ / u64 / no cap, ~585-y horizon at its far faster tail — [Grin emission](https://docs.grin.mw/about-grin/emission/), [nanogrin](https://github.com/mimblewimble/grin-explorer/issues/10)) |
| 10¹⁰ | 1.84×10⁹ | ~3,500 y | marginal — Polkadot at 10¹⁰ already occupies ~92 % of u64 and survives only on Substrate's u128 ([planck](https://support.polkadot.network/support/solutions/articles/65000168663-how-many-planck-are-in-a-dot-), [u128 Balance](https://github.com/paritytech/substrate-open-working-groups/blob/main/SOWG/1-polkadot-token-standard.md)) |
| 10¹² | 1.84×10⁷ | **< the 60.7M decay asymptote — overflows mid-decay** | dead — Monero's 10¹² works only by saturating its cumulative-emission counter at u64::MAX in the tail era, an explicit special case in consensus code ([blockchain.cpp](https://github.com/monero-project/monero/blob/master/src/cryptonote_core/blockchain.cpp), [atomic units](https://www.getmonero.org/resources/moneropedia/atomic-units.html)) |

10⁸ is the Bitcoin/Zcash/Beam/Decred point ([amount.h](https://github.com/bitcoin/bitcoin/blob/master/src/consensus/amount.h) et al.): the supply-attestation sum (performance-budget §9 — u64 arithmetic over Σcoinbase) stays ~2,300× below overflow even at year-50 supply, no Monero-style saturation clause is ever needed, and Bitcoin's own 2010 value-overflow incident ([CVE-2010-5139](https://en.bitcoin.it/wiki/Value_overflow_incident)) is the standing argument for generous headroom plus explicit sum checks. The cost — 10⁻⁸ is not an SI power, so no "nanoqumbra" — is a naming constraint, not an engineering one, and the honorific style it forces has the field's best aging record (below).

**(b) The unit name: bessel.** Friedrich Bessel's *Besselian elements* are the standard mathematics of eclipse prediction — literally the computation of where the umbra falls ([Besselian elements](https://en.wikipedia.org/wiki/Besselian_elements)) — squarely in the brand register (naming §5: eclipse geometry, astronomical not gothic) and in the honorific tradition (satoshi, lovelace, groth, planck — [satoshi](https://en.bitcoin.it/wiki/Satoshi_(unit)), [lovelace](https://cardano.org/glossary/lovelace/), [groth](https://beam.mw/beampedia-item/groth), [planck](https://support.polkadot.network/support/solutions/articles/65000168663-how-many-planck-are-in-a-dot-)), the naming style with the best survival record (Ethereum's honorific ladder decayed to the SI-hybrid gwei for *display*, but atomic "wei" itself held — [denominations](https://platform.chainbase.com/blog/article/understanding-wei-and-gwei-ethereum-s-smallest-denominations-explained)). The 2026-07-22 collision scan (CoinGecko search API + web, point-in-time) found **bessel clean — the only surviving shortlist candidate**: shade ([Shade Protocol](https://www.coingecko.com/en/coins/shade-protocol), privacy niche), saros ([Solana DeFi](https://www.coingecko.com/en/coins/saros-finance)), midnight ([IOG's privacy chain NIGHT, top-150](https://www.coindesk.com/markets/2025/12/11/cardano-ecosystem-gets-a-privacy-boost-as-midnight-s-night-goes-live)), eclipse ([major SVM L2](https://coinmarketcap.com/currencies/eclipse-xyz/)), umbral ([NuCypher's PRE primitive](https://blog.nucypher.com/unveiling-umbral/)), obscura (zk-branded DEX squatters), antumbra ([reads Penumbra-affiliated](https://antumbra.net/)), syzygy (dust squatters + unpronounceable), occult\* (register violation) — all compromised. The wider \*-umbra space is crowded on three fronts (Penumbra the chain, [Umbra the stealth-address protocol](https://github.com/ScopeLift/umbra-protocol) + [a second Umbra on Solana](https://thedefiant.io/news/tokens/umbra-trades-5x-above-ico-price-after-oversubscribed-raise), [Umbrel the node OS](https://umbrel.com/)) — reinforcing a unit name *outside* that family.

**(c) Ticker: QMB confirmed.** Zero hits on the CoinGecko search API for QMB (2026-07-22; point-in-time — DEX-only squatters not excludable by API). QUM also clean but **demoted on QTUM confusability** ([QTUM](https://www.coingecko.com/en/coins/qtum), rank ~306); fallback retained.

**(d) Protocol vs display.** The protocol only ever sees bessels — every consensus amount (values, fees, bonds, Σfee riders, S(h)) is a u64 count of atomic units; "1 QMB = 10⁸ bessel" is a display convention. Polkadot's Denomination Day is the precedent on record: the display unit rebased 100× by community vote while the atomic planck stayed fixed ([redenomination](https://ihodl.com/analytics/2020-08-17/polkadots-dot-redenomination-goes-ahead-following-member-vote/)) — recorded so any future display rebase is understood as a wallet-level act, never a consensus change.

**Pinned by this decision:** §2 r0 = 50 QMB absolute (5×10⁹ bessel/block at genesis; decay asymptote 6.07×10¹⁵ bessel); §5 fee absolutes; §4 bond minimum; the [naming doc](naming-and-branding.md) §4/§5 carry the brand-side record.

## Decided vs open

**Restated-decided (one-place reference):** §1 security params, §3 split, §7 anchor policy.

**Decided (2026-07-22, delegated stamps):** §2 emission = B2 with r0 = 50 QMB absolute (D1 rejected); §4 N=21, epoch 1,152 blocks (24 h), equivocation slash 10 % + tombstone, self-bond 10⁴ QMB + genesis ramp; §5 fee table 0.01/0.02/0.04 QMB (ratios 1/2/4 + native-anchored floor); §8 denomination 10⁸, unit = bessel (**Larry's brand veto pending at PR review**), ticker QMB.

**Decided (2026-07-23, delegated, on PR #46's measured basis + PR #47's B″ measurements):** §1 FRI configs = q21/q43/q86 MEASURED (142.2 KB / fit-checks / ≤32 GB — the freeze gate is cleared); §2 coinbase maturity = 144 blocks; §4 jail = <33 % of 100; §6 weight constants = 10 MB / 2× / 100k / 1.4× / 50 (with the governor-is-a-backstop correction on record).

**Open (surfaced, not recommended):** exact genesis-ramp step heights for the bond (genesis freeze). **Every technical gate in front of genesis freeze is now cleared** — the freeze itself is a deliberate act of record awaiting Larry's word.
