# Join and mine on the Qumbra testnet

中文：[`join-and-mine-zh.md`](./join-and-mine-zh.md) · **English is authoritative on technical detail.**

> **Note for readers of this public mirror:** inline code links (`../crates/…`,
> `../deploy/…`) reference the development tree, which is private during T1 — they document
> exactly where each claim is tested, but will 404 for outside readers until that tree
> opens. Nothing you need to join depends on following them.


This is the T1 path for a participant the project does not control.

> **Values filled 2026-08-16 as the T1 announcement package** (design
> `t1-sg-posture-decision`, sequence: Gate A remeasure → this package → SG open). They are
> read from the deployed fleet, not guessed: image digest/revision from the compose pin and
> its OCI label readback, seeds from the four OPEN entry points of the SG posture decision
> (node0 is deliberately not an entry point). **Until the announcement is published (Larry's
> go), the port these seeds listen on is not yet publicly open** — a connection refused before
> that moment is the posture working, not a wrong address. If the fleet rolls between this
> fill and the announcement, the digest/revision rows are re-verified at publication.

## 1. Obtain and verify the release

The node image is public at `ghcr.io/qumbra-labs/qumbra-node`. Use the **digest from the T1
announcement**, never a mutable tag. The image records its source revision in the OCI label
`org.opencontainers.image.revision`; read it back and compare it with the revision in the
announcement rather than trusting the tag or a successful pull. The label and node binary are
part of the runtime image ([`deploy/docker/Dockerfile:123-160`](../deploy/docker/Dockerfile#L123-L160));
the readback requirement is the lesson of [lab #224](https://github.com/qumbra-labs/qumbra-lab/issues/224).

```sh
IMAGE='ghcr.io/qumbra-labs/qumbra-node@sha256:c7b8b3340d35d7461daaa83acea6a8eef045bdba74d5173f9f059322ec18adbb'
EXPECTED_REV='e0b624596e29dc8a95d1f6715ed061b4247a73e2'

docker pull "$IMAGE"
ACTUAL_REV="$(docker image inspect \
  --format '{{index .Config.Labels "org.opencontainers.image.revision"}}' "$IMAGE")"
test "$ACTUAL_REV" = "$EXPECTED_REV" || {
  echo "wrong image revision: got $ACTUAL_REV, want $EXPECTED_REV" >&2
  exit 1
}
```

The announcement also publishes these network-identity inputs together:

- `genesis.qmb` — **format v4**;
- `expected_genesis_hash` —
  `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb`;
- the initial P2P seed addresses for `dial_peers`.

The format and hash are pinned in the tree
([`genesis.rs:68-77`](../crates/qumbra-node/src/genesis.rs#L68-L77),
[`genesis.rs:775-779`](../crates/qumbra-node/src/genesis.rs#L775-L779)). Distribution:
`genesis.qmb` downloads from **`https://seed.qumbra.org/genesis.qmb`** (deploy PR #150) —
always byte-verify it against `expected_genesis_hash` above; `qumbra-node check` and startup
both refuse a wrong file, so a tampered download cannot pass silently. The seed list is the
four open entry points below (`t1-sg-posture-decision`).

## 2. Join as a non-mining node

Put the downloaded `genesis.qmb` beside this minimal `node.toml`:

```toml
data_dir = "/data"
listen_addr = "0.0.0.0:9400"
dial_peers = ["18.202.166.126:9444","18.141.177.109:9444","52.194.224.123:9444","52.5.0.21:9444"]
genesis_file = "/config/genesis.qmb"
expected_genesis_hash = "138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb"
mining = false
```

These are the fields a joiner needs. `mining` defaults to false, but it is explicit above. Do
**not** copy fleet configs and do **not** add `committee_key_paths`: a public joiner is a
verify-only node and holds no committee signing keys
([`config.rs:54-94`](../crates/qumbra-node/src/config.rs#L54-L94)).

Preflight the exact genesis and config before binding a socket, then run:

```sh
docker volume create qumbra-data

docker run --rm \
  -v "$PWD:/config:ro" -v qumbra-data:/data \
  --entrypoint /usr/local/bin/qumbra-node \
  "$IMAGE" check --config /config/node.toml

docker run --rm --name qumbra-node \
  -v "$PWD:/config:ro" -v qumbra-data:/data \
  --entrypoint /usr/local/bin/qumbra-node \
  "$IMAGE" run --config /config/node.toml
```

`check` exercises the same byte/format/hash gate as startup without opening a listener
([`run.rs:226-242`](../crates/qumbra-node/src/run.rs#L226-L242)). A different file hash produces
`WrongGenesisHash` and the node refuses to start
([`genesis.rs:528-555`](../crates/qumbra-node/src/genesis.rs#L528-L555)).

### NAT is stated, not discovered

**Outbound-only participation is accepted by design.** Behind NAT you sync, mine, and transact,
but your address is never gossiped and you serve no peers. Leave `advertise_addr` unset. Set it
**only** when the advertised host and port are genuinely publicly dialable all the way to this
node; for the container that also means publishing the P2P port, for example `-p 9400:9400/tcp`.
This behavior is the recorded 2026-07-26 NAT decision, not a workaround
([`config.rs:68-78`](../crates/qumbra-node/src/config.rs#L68-L78),
[`run.rs:588-595`](../crates/qumbra-node/src/run.rs#L588-L595)).

### What healthy joining looks like

Read successive `TELEMETRY` lines, not one isolated sample
([`run.rs:1342-1371`](../crates/qumbra-node/src/run.rs#L1342-L1371)):

- `peers=N` is the live peer count: `N > 0` means at least one connection is live; persistent
  `peers=0` means the node has not joined a peer.
- `slag=N` is fork-choice tip minus applied-state tip: it should fall toward `0` while joining;
  `slag=0` means the node has applied its own selected chain
  ([`run.rs:1017-1034`](../crates/qumbra-node/src/run.rs#L1017-L1034)).
- `mready=-` is correct for the non-mining config above. After mining is enabled, `unknown` or
  `behind` means the startup mining gate is correctly refusing; `synced` or `latched` permits
  mining. It does **not** prove `slag=0`
  ([`run.rs:376-421`](../crates/qumbra-node/src/run.rs#L376-L421)).

## 3. Turn the joined node into a miner

### Platform boundary — CORRECTED 2026-08-17 (was: Linux/glibc only)

> **Dated correction (2026-08-17, lab #437): native macOS mining is PERMITTED.** The original
> red boundary below was conditioned on the deterministic-emission boundary not yet being
> active — that boundary activated 2026-08-12 (#299/#303 closed). Above height 8,640,
> `body.coinbase == coinbase_exact(height)` is a hard consensus rule in pure integer
> arithmetic; no libm is reachable from consensus, so a native macOS miner can neither
> compute a divergent coinbase nor scar history. **Verified live, not just argued**: a native
> macOS arm64 build joined T1 through the public entry points and won 40 accepted, finalized
> blocks in its first ~100 minutes (lab #437). The container remains the paved, reproducible
> path; a native build is now a supported alternative.
>
> **If you build from source, one flag is load-bearing**: a bare `cargo build -p qumbra-node`
> produces the ARMED variant, which halts at 8,640 and cannot follow today's chain. Build with
> `--features qumbra-node/rule-boundary-resume`, and confirm with `qumbra-node check` — its
> `halt plan:` line must read `no halt scheduled`, not `ARMED`.

The original boundary text, preserved for the record: *Mine on Linux/glibc only until the
deterministic-emission boundary is publicly confirmed active… A macOS miner has been measured
to compute coinbase values that differ by ±1 bessel from glibc at specific heights
([lab #303](https://github.com/qumbra-labs/qumbra-lab/issues/303)); consensus at the time
accepted those values, and each such block became a permanent scar the activation rule in
[lab #299](https://github.com/qumbra-labs/qumbra-lab/issues/299) must grandfather.*

Generate the payout identity from the wallet whose address should receive coinbase:

```sh
qumbra-wallet miner-rkm --dir "$HOME/.qumbra-wallet"
```

The command derives the allocated address index (index 0 by default) and prints the exact
64-hex-character `miner_rkm = "…"` line expected by the node
([`qumbra-wallet/main.rs:147-175`](../crates/qumbra-wallet/src/main.rs#L147-L175)). Paste it into
`node.toml` and change only these mining fields:

```toml
mining = true
miner_rkm = "[64 hex characters printed by qumbra-wallet miner-rkm]"
```

On startup, prove the config took effect by finding this exact line:

```text
miner payout: coinbase notes paid to the configured miner_rkm
```

If you instead see the following warning, stop mining and fix the config: valid blocks are being
paid to an unspendable placeholder and the payout is unrecoverable
([`run.rs:596-612`](../crates/qumbra-node/src/run.rs#L596-L612)).

```text
⚠️  NO miner_rkm CONFIGURED: ... Every coin this node mines is BURNED.
```

### 🔴 The unresolved first-start silence

A node with `miner_rkm` configured has been observed once to spend **about 57 minutes at silent
100% CPU before its first log line** on first start. This is unresolved
([lab #300](https://github.com/qumbra-labs/qumbra-lab/issues/300)). It is not necessarily hung:
if the process is alive and a core is pegged while no log line appears, leave it running. A
restart does not fix the path and only discards the work already spent; wait for the first line.

### Reward, maturity, pacing, and odds

- A won block pays the wallet **65% of the block subsidy (including the integer rounding
  remainder) plus all transaction fees**
  ([`emission.rs:27-30`](../crates/qlab-node/src/emission.rs#L27-L30),
  [`coinbase.rs:133-139`](../crates/qlab-node/src/coinbase.rs#L133-L139)).
- The coinbase becomes spendable after **144 more blocks**. That is about three hours at target,
  not a wall-clock promise
  ([`emission.rs:47-55`](../crates/qlab-node/src/emission.rs#L47-L55)).
- The network target is **75 seconds per block**, retargeted every block with **LWMA-120**; the
  deployable binary uses real RandomX, not the Keccak simulation placeholder
  ([`params_devnet.rs:16-49`](../crates/qlab-devnet/src/params_devnet.rs#L16-L49),
  [`pow.rs:66-100`](../crates/qlab-devnet/src/pow.rs#L66-L100)).
- This is solo mining, not a pool and not one reward every 75 seconds. Your expected share is your
  effective RandomX work divided by all effective work currently competing. The public surface
  exposes current difficulty, not fleet hash rate, so this guide cannot honestly quote personal
  odds. Expect variance and potentially long dry spells.

## 4. Wallet quickstart — five commands

The CLI's own help is the authority for flags and units
([`qumbra-wallet/main.rs:43-66`](../crates/qumbra-wallet/src/main.rs#L43-L66)). These five commands
cover the shortest user journey; replace `RECIPIENT_QADDR` and keep in mind that `--amount` is in
**bessel** (`100,000,000` bessel = `1 QMB`;
[`emission.rs:34-35`](../crates/qlab-node/src/emission.rs#L34-L35)).
Command 3 uses `curl` and `jq`; `qumbra-wallet --help` gives the full command reference.

```sh
# 1 — create the wallet; this prints address [0]
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 — back up the Qumbra mnemonic in a private terminal
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# Paste the full address [0] into https://faucet.qumbra.org and wait for its grant.

# 3 — obtain the height against which the balance claim will be made
TIP="$(curl -fsS https://explorer.qumbra.org/v1/health.json | jq -r '.chain.tip_height')"

# 4 — discover outputs and subtract spent notes through the public node edge
qumbra-wallet scan --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --to "$TIP"

# 5 — rescan, build a real proof, and submit; --node defaults to --url if omitted
qumbra-wallet send --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --scan-to "$TIP" \
  --to RECIPIENT_QADDR --amount 100000000
```

HTTP **429 Too Many Requests is expected behavior for the current faucet limit**; do not hammer
the form. [Lab #308](https://github.com/qumbra-labs/qumbra-lab/issues/308) records that the
network-keyed bucket is currently seen through the reverse proxy as a shared bucket, so another
visitor may have consumed it. Retry after the stated window while that issue remains open.

## 5. Verification record for the image claim

This is evidence that the package path is public and that remote label readback works; it is
**not** the T1 image announcement. On 2026-08-10, without pulling or registry credentials:

```text
$ docker buildx imagetools inspect ghcr.io/qumbra-labs/qumbra-node:t0-wan-14
Name:   ghcr.io/qumbra-labs/qumbra-node:t0-wan-14
Digest: sha256:931afa0fe57844f52e5bbb390914ec3116a72c3ff04781b11d8a080dcc4c2f29

$ docker buildx imagetools inspect --format \
  '{{index .Image.Config.Labels "org.opencontainers.image.revision"}}' \
  ghcr.io/qumbra-labs/qumbra-node@sha256:931afa0fe57844f52e5bbb390914ec3116a72c3ff04781b11d8a080dcc4c2f29
e8d52d7b194d3560f70de5d1f26b99b6f37bdd2e
```

Do not substitute that historical T0 digest for the bracketed T1 digest in §1.

The §4 public reads were also checked on 2026-08-10: `GET
https://explorer.qumbra.org/v1/health.json` returned the pinned genesis hash and a numeric
`chain.tip_height`, while `GET https://seed.qumbra.org/v1/compact?from=6313&to=6313` returned
HTTP 200 with `application/octet-stream`. The JSON field is defined at
[`qumbra-explorer/json.rs:55-80`](../crates/qumbra-explorer/src/json.rs#L55-L80); the height
`6313` is only the one-sample probe height, not a network parameter.
