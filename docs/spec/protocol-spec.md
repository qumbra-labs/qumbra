# Qumbra Protocol Specification — v0.1-core (M8-core)

> **Provenance (2026-08-17, staged open-sourcing stage 1 — design tracker #203):** this file is
> published from Qumbra's private design repository, authoritative as published here and synced
> forward on spec changes. Cross-references to design docs or lab PRs/issues not yet published
> will 404 for outside readers — they document where each claim's full argument and test lives,
> and resolve as later stages open. Nothing was removed from the source text.

> [中文版](protocol-spec-zh.md) — EN is normative.

**Status: DRAFT v0.1-core (2026-07-23).** The testnet-sufficient core of the M8 yellowpaper-equivalent ([testnet-plan §3](testnet-plan.md)): wire formats, derivation chains, and consensus rules, precise enough for an independent implementation of a T0/T1 node and wallet. Sections marked **[full-M8]** are deliberately deferred to the complete spec (pre-mainnet). Constants are the **FROZEN v1.0** set ([consensus-parameters](consensus-parameters.md) — the canonical table); this spec never restates a frozen value without citing it. Normative language: **MUST / MUST NOT / SHOULD** per RFC 2119. Every byte-level statement below is extracted from the measured prototype (`qumbra-lab`, golden-locked tests cited as `crate/file.rs`) — where the prototype is placeholder-grade the section says **[devnet-placeholder]** and binds the *shape*, not the constant.

## 0. Versioning

The protocol is **NU-style versioned from v1**: every consensus rule and wire format carries an integer protocol version; upgrades activate at a pre-announced **halt-height** (committee-and-governance §4) and MUST be accompanied by a revision doc amending this spec. Frozen constants change only through that mechanism. Wire-format bytes that are version-tagged in this spec (address version, seed version, envelope version, clue-slot version) MUST be treated as reject-unknown by consumers unless a section states otherwise.

## 1. Notation and primitives

- **H(·)** = Keccak-256 (the only hash in the consensus-critical path — frozen §1). `H_r(·)` denotes the Keccak-f[1600] sponge with rate 1088/capacity 512 as instantiated by the circuit's sponge layouts.
- **Field** = KoalaBear p = 2³¹ − 2²⁴ + 1; challenge field = degree-4 extension (≈ 2^124). Field elements on the wire are **4-byte little-endian** in the fixed-width proof format.
- **KEM** = ML-KEM-768 (FIPS 203 final): ek 1,184 B, ct 1,088 B, shared secret 32 B.
- **AEAD** = ChaCha20-Poly1305 (RFC 8439): 16-B tag.
- **SIG** = ML-DSA-65 (FIPS 204) — infrastructure roles only (checkpoints, governance); **no per-spend signatures exist in this protocol** (spend authorization is inside the transaction proof, §4).
- **[u64;4]** denotes a 256-bit value as four 64-bit limbs; packing into sponge inputs follows the circuit convention (extraction-cited per site).
- Integers on the wire are little-endian unless stated. Amounts are **u64 counts of bessel** (1 QMB = 10⁸ bessel — frozen §8); the balance relation is checked in-circuit over 16-bit limbs.
- **Digests**: 256-bit values are `[u64;4]` — four little-endian 64-bit lanes; byte form = lane-major LE (`lane i` → bytes `8i..8i+8`). Keccak padding throughout is the **original pad10*1** (Ethereum-style, NOT SHA-3/NIST), rate 1088/capacity 512; digest = output lanes 0..3.
- **Three preimage disciplines**, one hash: (i) **raw-sponge fixed-state packings** for circuit-bound derivations (cm, nf, nk, rkm, Merkle nodes) — the message is written directly into a zeroed Keccak-f[1600] state, one permutation, domain separation by *marker bits*, not strings; (ii) **domain-string byte sponges** for all wallet/note KDFs (`b"qumbra:..."` ASCII prefixes, plain concatenation); (iii) **fixed-width/length-prefixed concatenation** for consensus object preimages (injective without delimiters). A conforming implementation MUST NOT mix them.

## 2. Keys, addresses, and seeds

*(Every derivation below is golden-locked in `qlab-wallet`/`qlab-note` tests and, where circuit-bound, byte-locked against `build_bucket`.)*

**2.1 Circuit-bound key chain** (raw-sponge discipline; one Keccak-f[1600] permutation each, zeroed state, pad10*1 = a `1` bit immediately after the message + `st[16] = 1≪63`):

| Derivation | State layout (u64 lanes) | Pad start |
|---|---|---|
| `nk = H(sk ‖ D_N)` | `st[0..4] = sk`, `st[4] = 1` (D_N marker bit) | `st[5] = 1` |
| `rkm = H(nk ‖ D_R ‖ d)` | `st[0..4] = nk`, `st[4] = 2` (D_R marker bit), `st[5..7] = d` (16 B as two LE words) | `st[7] = 1` |
| `nf = H(nk ‖ ρ)` | `st[0..4] = nk`, `st[4..8] = ρ` | `st[8] = 1` |

Domain separation is the **marker bit in lane 4** (values 1 = domain N, 2 = domain R) — NOT an ASCII string; this is the one-block-absorb optimization the circuit shares. Digest = output lanes 0..3.

**2.2 Viewing split** (type-level): `SpendingKey{sk}` alone can produce a spend witness. `Fvk{nk, div_seed}` derives rkm(d), generates addresses, computes nf, detects+decrypts — cannot spend. `Ivk{div_seed}` only detects+decrypts and enumerates d(index) — deliberately no nk, so no rkm/address/nf capability (post-#32, rkm binds nk). `Fvk → Ivk` is a one-way downgrade.
`div_seed = Keccak256(b"qumbra:wallet:div-seed:v1" ‖ sk_bytes)`; per-diversifier ML-KEM keypair seed = `Keccak256(b"qumbra:wallet:mlkem-div:v1" ‖ div_seed ‖ d)`.

**2.3 Diversifiers**: `d(index) = Keccak256(b"qumbra:wallet:div-index:v1" ‖ div_seed ‖ index_u64le)[..16]` — 16 B, pseudorandom-of-index (issued addresses reveal neither sequence nor count). DiversifierLedger (wallet-local, non-consensus): monotone cursor, reuse + collision guards; wire = `version(1, =1) ‖ next_index(8 LE) ‖ count(8 LE) ‖ [index(8 LE) ‖ d(16)]*` ascending, reject-unknown/trailing/duplicates.

**2.4 Address**: raw = `version(1 B, ADDRESS_VERSION = 1) ‖ d(16) ‖ rkm(32) ‖ ML-KEM ek(1,184)` = **1,233 B**; unknown version MUST be rejected. Encoding = bech32m (BIP-350 constant, hand-rolled, vector-locked; the BIP-173 90-char cap is deliberately not enforced): full form HRP **`qaddr`** → **1,985 chars**; short form HRP **`qs`** → **35 chars**, encoding `Keccak256(b"qumbra:wallet:short-addr:v1" ‖ raw_1233)[..16]` — a hash commitment requiring a resolution layer, not a decodable address.

**2.5 HD seed** (deliberately not BIP-32/HMAC — one hash family): `MasterSeed{version = SEED_VERSION = 1, entropy[32]}`; chain (domain-string discipline, all-hardened, path `m/account/role`):
`node_m = Keccak256(b"qumbra:hd:v1:master" ‖ version ‖ entropy)` → `node_a = Keccak256(b"qumbra:hd:v1:account" ‖ node_m ‖ account_u32le)` → `sk = Keccak256(b"qumbra:hd:v1:role" ‖ node_a ‖ role_u32le)`; only role 0 (Spend) is defined. Golden vector: entropy `e[i] = (7i+1) mod 256` → `sk(0, Spend)` = `1a6937e0…80cac4` (change = SEED_VERSION bump).

**2.6 Mnemonic**: 24 words, canonical BIP-39 English-2048 list; 256-bit entropy + 8-bit checksum = **first byte of `Keccak256(entropy)`** — deliberately NOT SHA-256, so a Qumbra phrase is not stock-BIP-39-recoverable (a self-identifying Qumbra artifact; ratified 2026-07-23). Bit stream is big-endian, 24 × 11-bit indices; no PBKDF2/passphrase (the phrase is a reversible encoding of the entropy). Pinned vector: zero entropy → 23 × "abandon" + **"ahead"** (BIP-39 would end "art").

## 3. Notes, commitments, nullifiers, and the tree

- A **note** = `(value: u64 bessel, rkm, ρ, rseed)`; plaintext wire = `value(8 LE) ‖ rkm(32) ‖ ρ(32) ‖ rseed(32)` = **104 B** exactly (no memo field, no d field; parse rejects ≠ 104).
- **cm = H(note)**, one Keccak block: `st[0] = value`, `st[1..5] = rkm`, `st[5..9] = ρ`, `st[9..13] = rseed`, pad `st[13] = 1`, `st[16] = 1≪63`; digest = lanes 0..3. The wallet/note-side recomputation is regression-locked byte-for-byte against the circuit's `build_bucket` output.
- **nf = H(nk ‖ ρ)** (§2.1); publishable exactly once; the nullifier set is consensus state.
- **Commitment tree** (depth **32**, frozen §1): node = `H(left ‖ right)` — single 512-bit block (`st[0..4] = left`, `st[4..8] = right`, pad `st[8] = 1`), **no level tags, no extra domain bytes**; **leaf = cm directly** (no leaf-hash wrapper). Empty leaf = `[0;4]`; `zeros[i] = H(zeros[i−1] ‖ zeros[i−1])` (incremental-tree convention). Witness = 32 siblings + 32 path bits, leaf-level first; path bit true ⇒ running digest is the RIGHT child. Root byte form = lane-major LE (`digest_bytes`).
- **Frontier serialization** (light-client sync; versioned, not yet golden-frozen — interop O5): `version(0x01) ‖ depth(u8 = 32) ‖ n_leaves(varint) [‖ rightmost_leaf(32) ‖ per level: present(u8) ‖ node(32 iff present)]`, `present` MUST equal bit i of `n_leaves−1`.

## 4. Transactions

- **Shape**: fixed 2×2 arity bucket at launch (2 inputs, 2 outputs, in-circuit dummies — transaction-model §5); 4×4/8×8 are versioned additions. Bucket geometry: 617 columns × 2^18 rows, 83 Keccak permutations.
- **Statement** (one monolithic STARK; no per-spend signatures): per input — membership of cm under an approved anchor (depth-32 path, §3), correct `nf = H(nk‖ρ)`, spending-key knowledge (`nk = H(sk‖D_N)` in-circuit); per output — cm well-formedness; balance `Σin = Σout + fee` as an exact 16-bit-limb borrow chain against the fee public values.
- **Public values** (84 field elements, this exact order):

| Offset | Field | Encoding |
|---|---|---|
| 0 | anchor | 16 × 16-bit chunks |
| 16 | nf₁ | 16 chunks |
| 32 | nf₂ | 16 chunks |
| 48 | cm′₁ | 16 chunks |
| 64 | cm′₂ | 16 chunks |
| 80 | fee | 4 × 16-bit LE chunks of the u64 |

Digest chunking: chunk index = `4·lane + j`, value = bits `[16j, 16j+16)` of lane `⌊i/4⌋` (chunk-major within lane, lanes LE). Each chunk is one KoalaBear element.
- **Proof wire**: bincode **fixint** encoding of the `p3_uni_stark::Proof` struct (pinned 0.6.1) — components: `commitments{trace, quotient_chunks}`, `opened_values{trace_local/next, quotient_chunks}`, `opening_proof` (FRI), `degree_bits`. At the frozen config b16/**q21**/g22/fp16/a16 (Merkle cap height 3): **145,609 B measured, byte-deterministic**. No hand-rolled layout exists; the struct encoding IS the wire **[full-M8: freeze a golden byte dump]**.
> **Measured update (2026-08-04 — THE MINT, [lab PR #252](https://github.com/qumbra-labs/qumbra-lab/pull/252), coordinator-accepted 1218/0 unfiltered, genesis reproduced byte-identical twice through the real CLI): the consensus wire is now 148,625 B (145.1 KB), still ≤ 150 KB (3.3 % margin against the 153,600 B ceiling). The figure above is the pre-mint record.** One pre-T1 re-mint (legitimate only because no chain existed under the old genesis) carried three changes: **(1)** #215 (i)/option-4 ρ′ derivation — `ρ′_0 = nf_0`, `ρ′_1 = H(nf_0 ‖ D_P|1)` via a new `ROLE_ARHO` permutation (83 → 84 perms) plus a third equality bank, closing the faerie-gold free witness structurally; **(2)** the #219 dummy-liveness latch (`dv`) made unconditional — single-note spends are in-circuit legal; **(3)** the #188 (a) discovery payload — 120 B/note full plaintext per the amended ruling (an `Ivk` deliberately cannot derive `rkm`; the served wire carries no nullifier) — relocated into the **committed** body region, `/v1/compact` serving it by projection. Trace width 617 → 643 (accounted column by column); max constraint degree and quotient chunks **unmoved at 4**; genesis `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb`, `GENESIS_FORMAT_VERSION` 4. b4-fallback interior caveat: see the aggregation-rung1 §6 breach block ([lab #257](https://github.com/qumbra-labs/qumbra-lab/issues/257)).
- **Fees**: posted price by bucket — 0.01 / 0.02 / 0.04 QMB (frozen §5), paid to miners, never burned. A transaction whose fee ≠ its bucket's posted price is **invalid**.
- **Anchors**: MUST reference a finalized root, aged ≤ 1,152 blocks, quantized to 10-minute buckets (= 8 blocks) (frozen §7).

## 5. Note-discovery wire (PQ compact blocks)

- **Per-note compact entry** = `cm(32) ‖ tag(8) ‖ clue(1)` = **41 B**; clue byte MUST be `0x00` at protocol v1 (a non-zero clue is reject — the byte is reserved from genesis as the OMR door).
- **Recipient bundle**: the ML-KEM ct (1,088 B) appears **once per (tx, recipient)** — `ct ‖ k entries`; amortized bytes/note = `41 + ⌈1088/k⌉` (measured: 1,129 @ k=1, **585 @ k=2**).
- **Detection tag** = `Keccak256(b"qumbra:note-detect:v1" ‖ K ‖ cm_bytes)[..8]` (85-B preimage; K = KEM shared secret). **AEAD**: key = `Keccak256(b"qumbra:note-aead-key:v1" ‖ K ‖ index_u32le)`, nonce = `Keccak256(b"qumbra:note-aead-nonce:v1" ‖ K ‖ index_u32le)[..12]`, `index` = output slot in the bundle (key/nonce bind (K, index), tag binds (K, cm)); ciphertext = 104 + 16 = **120 B**, no AAD.
- **Serving framing** (compact-block protocol; +3 B per group ≈ +1.5 B/note at k=2): group = `tx_index(LEB128) ‖ n_recipients(u8) ‖ per recipient [ct(1088) ‖ n_outputs(u8) ‖ entries]`; response = `version(0x01) ‖ n_blocks(varint) ‖ per block [height(varint) ‖ n_groups(varint) ‖ groups]`. Varints = unsigned LEB128 ≤ 10 B. Decoders MUST reject trailing bytes, unknown version, non-zero clue length. Golden vector on record (whole-buffer digest `3ee2a5e6…f54017`).
- **Scan rule**: decap once per (tx, recipient) → tag pre-filter per entry → on match, fetch + AEAD-decrypt + parse. **Every tag match MUST run full standardized FO decapsulation before any secret is used** (on-match-FO, adopted 2026-07-22); FO-skip-as-default remains gated (F1/F2). False-positive rate 2^-64 per tag.

## 6. Blocks, PoW, and emission

- **Header** (shape binding; byte encoding **[full-M8]** — the devnet pins the *hash preimages*, which ARE binding): fields = parent, height, timestamp, difficulty, nonce, tx-body commitment, **reserved aggregate-proof slot** (empty at rung 0), **reserved epoch-supply-attestation slot**. Devnet header-hash preimage (98 B, [devnet-placeholder] shape): `prev(32) ‖ height(8 LE) ‖ timestamp(8 LE) ‖ difficulty(8 LE) ‖ nonce(8 LE) ‖ tx_body_commitment(32) ‖ 0xA6 ‖ 0x59` — the two tag bytes bind the reserved slots.
  > **Correction (2026-07, [qumbra-lab PR #57](https://github.com/qumbra-labs/qumbra-lab/pull/57) acceptance):** v0.1-core labeled this preimage "106 B"; the enumerated fields sum to **98 B** (32 + 8×4 + 32 + 2), matching the golden-locked prototype (`BlockHeader::preimage()`, now also `qlab-p2p`'s `HEADER_WIRE_LEN = 98` with round-trip + hash-identity tests). Arithmetic label only — the field list was and is correct.
- **Body commitment** preimage ([devnet-placeholder] shape, injective by fixed widths/length prefixes): per tx `anchor(32) ‖ nfs ‖ cms ‖ bucket(u8) ‖ fee(8 LE) ‖ proof_len(8 LE) ‖ proof_bytes`, then `coinbase(8 LE)`; Keccak-256.
- **PoW**: CPU-friendly RandomX-class **[full-M8: instantiation + difficulty retarget]**; target block time **75 s** (frozen §2).
- **Emission (normative integer form — the supply-audit anchor)**: real-valued closed form `S(h) = r0·(1−(1−d)^h)/d` for `h ≤ h_t`, then `+ tail·(h−h_t)`, with the frozen B2 constants (r0 = 50 QMB, d = 8.237e-7, tail = 1.22441 QMB/block; h_t = first h where decay ≤ tail). Define `S_atomic(h) = round(S(h) · 10⁸)` bessel. **`coinbase(h) = S_atomic(h+1) − S_atomic(h)`** — non-negative and telescoping exactly (test-locked), so `Σ_{i<h} coinbase(i) = S_atomic(h)` **exactly**; the epoch supply attestation reconciles `Σcoinbase − Σfees` against it. Split 65/15/20 per block (frozen §3); coinbase maturity **144 blocks** (frozen §2).

  > **Amendment (2026-08, [lab #299](https://github.com/qumbra-labs/qumbra-lab/issues/299)): the schedule is a validity rule, not only a definition.** As written above, this section defined `coinbase(h)` and no rule required a block to commit it — the prototype validated only `coinbase > 0 ⇒ payee present`, and the miner's note is priced at 65 % of the **committed** field plus fees, so a miner could write any emission and be paid proportionally (found via a live −4114 bessel divergence on T0; root-caused and verified on the issue). **Rule, activation-gated:** for `height ≥ 1`, a body with `body.coinbase ≠ coinbase(height)` is INVALID. Genesis is exempt **explicitly** — it commits `coinbase = 0` while `coinbase(0) = 5×10⁹`; the first scheduled mint is height 1 (the attestation's `first_mint = max(start, 1)` encodes the same fact). The rule activates at a halt-height boundary (committee-and-governance §4 mechanism); history before the boundary is grandfathered as recorded — on the current T0 chain that is exactly one under-paying block in epoch 1, kept visible as a known scar in the supply attestation rather than patched over. **The rule must be active before public mining opens (T1 gate).**
- **Block weight**: two-median governor with the frozen §6 constants; the governor is a backstop (anti-gigantism + surge damping) — the fee floor is the spam bound (measured correction, params §6).

## 7. Finality: committee and checkpoints

- **Committee**: N = 21 named entities, quorum **15** (⌊2N/3⌋+1), epoch 1,152 blocks; self-bond 10⁴ QMB with the frozen genesis ramp; equivocation (two conflicting signed votes — two signature checks, no human in the path) → 10 % slash + permanent tombstone; downtime → jail, no slash, at < 33 % signed of the trailing 100 (all frozen §4). Tombstone is terminal and idempotent; tombstoned votes MUST NOT count toward quorum. Membership changes at epoch boundaries only, in-protocol (committee-and-governance §2) **[devnet-placeholder: the prototype runs a static genesis set]**.
- **Checkpoint object** = `{height: u64, block_hash: 32 B, root: 32 B}` (root = the finalized commitment-tree root anchors reference). **Signed message** ([devnet-placeholder] domain string): `b"qumbra:devnet:checkpoint:v1" ‖ height(8 LE) ‖ block_hash(32) ‖ root(32)`; signature = ML-DSA-65 deterministic (3,309 B); a vote = (signer index, signature). **[full-M8: production domain string + checkpoint cadence]**.
  > **Update (2026-07, [qumbra-lab PR #59](https://github.com/qumbra-labs/qumbra-lab/pull/59) — both items were inline delegated stamps at M9 GO, 2026-07-23):** the **production domain string is realized**: signed message = `b"qumbra:checkpoint:v1" ‖ height(8 LE) ‖ block_hash(32) ‖ root(32)` (golden-locked in `qlab-devnet::committee` incl. a no-`devnet:`-substring assertion; the placeholder form above is retired — a deliberate versioned signature break per §0). **Checkpoint cadence pinned at 8 blocks = one 10-minute anchor bucket** (§7 anchors row); cadence remains **testnet-tunable, NOT frozen** — its freeze stays with full-M8 v1.1. Epoch-boundary membership machinery (staged changes, tombstone forced-exit, per-epoch roster history) and the trailing-100/33 % downtime-jail window are prototyped per frozen §4 (integer-exact comparison), with two annotated prototype boundaries (cross-epoch penalty indexing; boundary window reset).
- **Finalization**: a checkpoint is final on ≥ quorum distinct valid votes and MUST strictly advance the finalized height. Committee stall ⇒ Ebb-and-Flow degradation to probabilistic PoW; anchors keep referencing the last finalized roots (liveness never hostage; the ≤ 24 h anchor window bounds how long degraded-mode spending stays live — by design).
  > **Update (2026-07-25, [qumbra-lab PR #72](https://github.com/qumbra-labs/qumbra-lab/pull/72), issue #70 — coordinator-accepted; delegated stamp, this section specifies a relay, not a new consensus rule):** the rule above was always right and is unchanged; what it never said is **how the distinct votes reach one counter**. On a real net the committee is split across machines (the T0 rehearsal runs 21 keys 6/5/5/5), so **no single gossip message ever carries a quorum** — a prototype that counted only the votes inside one message could never finalize anything, which is exactly what the Phase B-lite WAN-shaped rehearsal found. **Vote relay is therefore part of the protocol, not an implementation detail**, and is specified as follows:
  > - **Partial vote sets are first-class objects.** A node MUST accept, accumulate, and relay a well-formed vote set that is below quorum. A below-quorum set is honest progress toward finality and MUST NOT be treated as an invalid object or scored against its sender; only a malformed set, or one carrying an unverifiable, non-roster, or duplicate-signer vote, is misbehaviour.
  > - **Accumulation is per checkpoint variant, de-duplicated by signer index**, across any number of messages, and MUST be bounded (the prototype pins three named caps — heights tallied, variants per height, and how far above the local tip a height may still be tallied). Accumulated state needs no persistence: it rebuilds from re-gossip, and finalized checkpoints already survive in the block log.
  > - **Accumulation never lowers the bar.** The quorum gate is unchanged and stays the sole authority: it re-checks roster membership, signer distinctness, **every signature**, and the ⌊2N/3⌋+1 threshold over the *accumulated* set before anything is finalized. Tombstoned/jailed signers are excluded before the count (frozen §4) and re-excluded against their status at finalization time, so a signer tombstoned after their vote was accumulated cannot be counted.
  > - **Wire form** (prototype, golden-locked in `qlab-p2p`): message type `CheckpointVotes = 0x0024`, body **byte-identical** to the existing `Checkpoint` (0x0022) body — `height(8 LE) ‖ block_hash(32) ‖ root(32) ‖ varint(vote_count) ‖ votes…`, 73 bytes for the empty set — so the codec is not forked; only the envelope type differs. Relay is direct push, gated on "the local tally grew", which both terminates the relay and keeps inventory dedup on a finalized checkpoint's identity from suppressing the message that would complete a quorum. Codepoints and the push-vs-inventory choice are **testnet-tunable, NOT frozen**; the signed message of §7 above is untouched by all of this.
  >
  > Measured: a 4-node docker net with the 21-key 6/5/5/5 split — **no node holding a quorum** — finalizes on the cadence grid (`final` 32 → 232 monotone over a 135-min soak, ≤ 1-block tip spread); 2+2 partition correctly stalls both sides (11 and 10 keys, both < 15) and resumes on heal **without a restart**; a 3-node committee outage stalls and then re-forms, finality catching up across the stalled span. Still owed to Phase B-WAN: real RTT, LWMA at WAN pacing, ≥ 48 h.

## 8. Selective disclosure envelope

Versioned envelope (wallet-interop §3; zero consensus surface): `ver(u8 = 0x01) ‖ claim_type(u8 = 0x01 "sent-payment") ‖ tx_ref(32) ‖ value(8 LE) ‖ addr_commitment(32) ‖ output_index(u8) ‖ proof_len(LEB128) ‖ proof_bytes`. Unknown `ver` OR `claim_type` MUST be rejected at parse and re-checked at verify; trailing bytes rejected. `addr_commitment = Keccak256(raw 1,233-B address)` (plain, no domain prefix — distinct from the short address). Statement public values: `cm(16 chunks) ‖ addr_commitment(16) ‖ value(4)` = 36 elements, same chunking as §4; the statement proves cm opens to (value, …) ∧ addr_commitment matches ∧ the rkm inside cm equals the rkm inside the address preimage (cross-binding: "this payment went to THAT address"). Proof = 594-col narrow-Keccak circuit, measured 121.9 KB — comparable to a payment, stated honestly. `proof_bytes` uses postcard encoding (envelope-internal; unlike the consensus fixed wire).

## 9. Genesis **[shape only; execution = M15]**

Genesis block: no premine outputs (fair launch — zero insider allocation, frozen §3 grounds), the frozen constant set baked, committee₀ per candidate-net output, treasury accrual active from block 0. Format **[full-M8]**.

## 10. Deferred to full M8 (honest list)

Exact header/block byte encoding; RandomX instantiation + difficulty algorithm; P2P message formats; mempool policy details; multi-bucket (4×4/8×8) activation; aggregation rung-1 wire (block-proof + Σfee attestation) — activates post-launch with its own spec section; the complete test-vector appendix.

## Decided vs open

**Decided:** everything cited to the frozen table or a golden-locked test is normative as written.

**Open:** the [full-M8] list (§10). All byte-level content is extraction-filled from the golden-locked prototype (two read-only extraction passes, 2026-07-23).
