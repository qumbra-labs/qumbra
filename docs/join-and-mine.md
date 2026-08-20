# Join and mine on Qumbra T2

中文：[`join-and-mine-zh.md`](./join-and-mine-zh.md) · **English is authoritative on technical detail.**

> **Note for readers of this public mirror:** inline code links (`../crates/…`,
> `../deploy/…`) reference the development tree, which stays private on its own
> schedule — they document exactly where each claim is tested, but will 404 for
> outside readers until that tree opens. Nothing you need to join depends on
> following them.

This is the T2 path for a participant the project does not control. There are **two ways
to mine**, and you only need one of them:

| path | what you run | what you need |
|---|---|---|
| **Pool** (§1) | stock [XMRig](https://github.com/xmrig/xmrig) pointed at `pool.qumbra.org:3333` | a payout key from `qumbra-wallet` — **no node, no genesis file, no sync** |
| **Solo** (§2) | your own full node, `qumbra-node mine` | the node binary + the T2 genesis file |

Pool mining smooths variance and takes minutes to set up; solo mining keeps the whole
block reward when you win one and makes you a full validator of the chain. The wallet
(§4) is the same either way.

> ### ⚠️ T1 IS RETIRED — this guide is for T2
>
> **T1 has shut down.** If you were following the previous version of this guide: stop —
> its values are dead. T1 balances **do not carry over**; T2 is a fair relaunch from a
> fresh genesis — no premine, no carry, block 0 is the same starting line for everyone,
> including the operators. Anything you have that names the T1 genesis hash
> (`138e1524…addb`), a `t1-*` release tag, or the height-19,008 update deadline belongs
> to the retired net and does not apply here. T2 is a **different network by
> construction**: its genesis file is format **v5**, network `qumbra-t2`
> ([`genesis.rs:630-636`](../crates/qumbra-node/src/genesis.rs#L630-L636)), a node
> pinned to the wrong genesis refuses to start, and a wrong-net peer is refused by name
> at the P2P handshake ([`peer.rs:45-47`](../crates/qlab-p2p/src/peer.rs#L45-L47),
> lab #474) — you cannot join the wrong chain by accident, only fail loudly.

> **What is true at cutover, and what is not.** The genesis hash below is the real
> T2 value — pin it, verify it, trust nothing else. Two things this guide does
> **not** claim are live at 14:00 UTC+8:
>
> 1. **T2 binaries / image digest.** The T2 release is cut *after* the cutover (the
>    release guard refuses a T2 tag before the chain is live). Until a T2 tag
>    appears on <https://github.com/qumbra-labs/qumbra/releases>, there is no T2
>    binary to run. Every `t1-*` tag is a T1 artifact — do not run it against T2.
> 2. **The pool endpoint.** `pool.qumbra.org:3333` is the path (stock XMRig config
>    in §1). The pool is held until after cutover: a live-looking stratum that
>    issued jobs against a fake template would burn miners' hashpower, so the
>    hostname is **not** a paying endpoint until a later note on this repo says
>    it is. The config is here so it is one paste when that happens.
>
> Image digest, image revision, P2P seeds, and the T2 release tag remain written
> as `<… — filled at announcement>` until those values exist. A genesis hash from
> anywhere other than this page or the T2 announcement, including an older copy
> of this file, is wrong by definition.

## 1. Pool path — stock XMRig, no node

> **Held until after cutover.** The config in this section is the path, not a
> live claim. Do not point a miner at `pool.qumbra.org:3333` expecting payment
> until the T2 announcement (or a follow-up on this repo) says the pool is up.

New to mining pools? **Stratum** is the JSON-line protocol miners and pools use
over one long-lived TCP connection (login → jobs → share submits). Qumbra speaks
the Monero-family dialect that stock XMRig already knows — that is the whole
point of the T2 header layout. The short version: not one line changes on the
miner side.

The short version: Qumbra's pool speaks the Monero-family stratum dialect that stock
XMRig already knows, and the T2 block header was laid out so that **not one line changes
on the miner side**. The pool is custody-free: it never holds your coins — a won block's
coinbase pays miners directly on-chain
([`payee.rs`](../crates/qumbra-pool/src/payee.rs)).

### 1.1 Make a payout key

Your payout identity is a 64-hex-character key derived from a wallet you control. Get
`qumbra-wallet` from the release archive in §2.1 (any platform; you do **not** need to
run the node), then:

```sh
# 1 — create the wallet (prints address [0])
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 — back up the mnemonic in a private terminal, NOW — it is shown, not stored readable
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# 3 — derive the payout key the pool pays
qumbra-wallet miner-rkm --dir "$HOME/.qumbra-wallet"
```

Step 3 prints a `miner_rkm = "…"` line whose value is **exactly 64 hex characters**
([`qumbra-wallet/main.rs:476-495`](../crates/qumbra-wallet/src/main.rs#L476-L495)).
That value — the bare hex, without the `miner_rkm = ""` wrapper — is what goes in
XMRig's `user` field.

**Copy it exactly.** The pool credits precisely the key you log in with: a 64-hex login
*is* the payout key ([`payee.rs:45-62`](../crates/qumbra-pool/src/payee.rs#L45-L62)).
A mistyped or truncated login still mines and still scores shares, but it **can never be
paid** — the pool has no way to know who you meant. There is no account recovery,
because there are no accounts.

### 1.2 Point XMRig at the pool

Download stock XMRig from its official releases page
(<https://github.com/xmrig/xmrig/releases> — Windows, macOS, Linux). Expect your
antivirus or SmartScreen to flag it: it is a CPU miner, and flagging CPU miners is what
reputation filters do. Verify the download against XMRig's published hashes, then
decide.

Minimal `config.json`:

```json
{
    "pools": [
        {
            "url": "pool.qumbra.org:3333",
            "user": "<the 64 hex characters printed by qumbra-wallet miner-rkm>",
            "pass": "x",
            "algo": "rx/0",
            "keepalive": true,
            "tls": false
        }
    ]
}
```

Or the same thing as a command line:

```sh
xmrig -o pool.qumbra.org:3333 -a rx/0 -k \
      -u <the 64 hex characters printed by qumbra-wallet miner-rkm> -p x
```

`algo` must be `rx/0` — the pool refuses anything else by name
([`pool.rs:248`](../crates/qumbra-pool/src/pool.rs#L248)). `pass` is free-form and
unused. **When the pool is announced live**, healthy output is XMRig printing
`new job from pool.qumbra.org:3333` followed by a steady drip of `accepted` share
lines; a run with jobs but zero accepted shares after several minutes means
something is wrong — recheck `algo` first. Until that announcement, a connection
refused (or jobs that never become chain blocks) is the hold working, not a
wrong address.

### 1.3 How you get paid — read this before extrapolating earnings

- The pool tallies shares over a **PPLNS window: the last 1,024 accepted shares,
  weighted by difficulty**
  ([`pplns.rs:1-10`](../crates/qumbra-pool/src/pplns.rs#L1-L10); the window size is a
  devnet-placeholder constant and may be retuned).
- **At launch, each won block pays ONE miner** — the v5 coinbase payee list is capped at
  **N=1 at birth**
  ([`body.rs:94`](../crates/qlab-devnet/src/body.rs#L94)), so the whole miner reward of
  a block goes to the highest-weight payable miner in the window at that moment
  ([`payee.rs:107-131`](../crates/qumbra-pool/src/payee.rs#L107-L131)). Over many
  blocks this pays proportionally to shares; over a short window it behaves like a
  shares-weighted lottery. If your hashrate is a small fraction of the pool's, expect
  the same dry spells solo miners see, shortened in proportion to how often the pool
  wins blocks. Raising the cap (true per-block splitting) is a planned, declared rule
  change, not a knob the pool can turn.
- Payment is **on-chain coinbase**, so pool payouts obey coinbase maturity: spendable
  after **144 more blocks**, about three hours at target
  ([`emission.rs:83`](../crates/qlab-node/src/emission.rs#L83)).
- Your earnings are visible to your wallet, not on a pool dashboard: `miner-rkm` derives
  from your wallet's address index 0, so the §4 `scan` command shows what the pool has
  paid you, with no one to trust about it.

## 2. Solo path — run a node and mine

### 2.1 Obtain and verify the release

The node image is public at `ghcr.io/qumbra-labs/qumbra-node`. Use the **digest from the
T2 announcement**, never a mutable tag. The image records its source revision in the OCI
label `org.opencontainers.image.revision`; read it back and compare it with the revision
in the announcement rather than trusting the tag or a successful pull. The label and
node binary are part of the runtime image
([`deploy/docker/Dockerfile:144`](../deploy/docker/Dockerfile#L144)); the readback
requirement is the lesson of
[lab #224](https://github.com/qumbra-labs/qumbra-lab/issues/224).

```sh
IMAGE='ghcr.io/qumbra-labs/qumbra-node@<T2_IMAGE_DIGEST — filled at announcement>'
EXPECTED_REV='<T2_IMAGE_REVISION — filled at announcement>'

docker pull "$IMAGE"
ACTUAL_REV="$(docker image inspect \
  --format '{{index .Config.Labels "org.opencontainers.image.revision"}}' "$IMAGE")"
test "$ACTUAL_REV" = "$EXPECTED_REV" || {
  echo "wrong image revision: got $ACTUAL_REV, want $EXPECTED_REV" >&2
  exit 1
}
```

The announcement also publishes these network-identity inputs together:

- `genesis.qmb` — **format v5, network `qumbra-t2`**
  ([`genesis.rs:630-636`](../crates/qumbra-node/src/genesis.rs#L630-L636));
- `expected_genesis_hash` — `d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3`;
- the initial P2P seed addresses for `dial_peers` — `<T2_SEEDS — filled at announcement>`.

Distribution: `genesis.qmb` downloads from **`https://seed.qumbra.org/genesis.qmb`**
(the bare service names moved from T1 to T2 at cutover; `pool.qumbra.org` is new and
held — see the box above) — always byte-verify it against `expected_genesis_hash`
above. `qumbra-node check` and startup both refuse a wrong file
([`genesis.rs:525-530`](../crates/qumbra-node/src/genesis.rs#L525-L530)), so a tampered
download cannot pass silently. Until the cutover completes, that URL may still
serve T1's file or refuse — byte-verify against the T2 hash; a mismatch is the
cutover in progress, not a reason to ignore the hash.

#### Prebuilt binaries, no Docker

The container above stays the **reproducible baseline**. If Docker is the obstacle
rather than the answer, the same three binaries (`qumbra-node`, `qumbra-wallet`,
`qumbra-pool`, plus a `PROVENANCE.txt`) land as archives on the public repo's
releases page —
**<https://github.com/qumbra-labs/qumbra/releases>**, tag
`<T2_RELEASE_TAG — filled at announcement>` — **after the cutover**. That tag does
not exist yet; the release guard refuses to cut it before the chain is live.
Every currently published tag (`t1-91bdee4`, `t1-c5cfff8`, `t1-84f6f4c`) is a T1
artifact. Do not run a `t1-*` binary against T2.

| archive | for |
|---|---|
| `…-linux-x86_64-glibc.tar.gz` | Intel/AMD Linux, glibc 2.36+ (Debian 12, Ubuntu 22.04+) |
| `…-linux-aarch64-glibc.tar.gz` | arm64 Linux — what the fleet itself runs |
| `…-macos-arm64.tar.gz` | Apple Silicon, macOS 11+ |
| `…-windows-x86_64.zip` | Windows 10/11 x64 — native, no WSL2 |

```sh
# 1 — download the archive for your platform and SHA256SUMS from the release page, then:
sha256sum -c SHA256SUMS          # macOS: shasum -a 256 -c SHA256SUMS
tar -xzf qumbra-<T2_RELEASE_TAG>-<platform>.tar.gz
cd qumbra-<T2_RELEASE_TAG>-<platform>

# 2 — ask the binary what it is:
./qumbra-node halt-status
```

`build rev:` in that output is how a downloaded binary proves which source it came from —
a tarball carries no image label, so this line is the equivalent of the
`org.opencontainers.image.revision` readback in the container path above. It must match
the revision the release notes name. `halt plan:` must read `no halt scheduled`; if it
says **ARMED**, that binary is scheduled to stop following the chain at a fixed height —
do not run it, and treat its presence on the release page as evidence the artifact is
not what it claims to be (CI refuses to publish one).

**macOS only:** the binaries are unsigned and un-notarized. A browser download
quarantines them and Gatekeeper refuses to run them. Fetch with `curl`, or clear the
attribute: `xattr -d com.apple.quarantine qumbra-node qumbra-wallet qumbra-pool`.

**If you build from source, one flag is load-bearing**: a bare
`cargo build -p qumbra-node` produces the ARMED variant — a T1-upgrade artifact whose
scheduled halt still fires on any net, T2 included
([`release.rs:712-714`](../crates/qumbra-node/src/release.rs#L712-L714)). Build with
`--features qumbra-node/rule-boundary-resume` and confirm with `qumbra-node halt-status`:
`halt plan:` must read `no halt scheduled`, not `ARMED`. The published archives are
already built with that flag.

### 2.2 One command: `qumbra-node mine`

Everything in §2.3 — a wallet, its backup, the payout key, a hand-written `node.toml`,
the genesis download and verification — is what `qumbra-node mine` does for you. It is
the same node with the same config; the difference is that the five steps are one:

```sh
./qumbra-node mine --dir ~/.qumbra-miner
```

On a terminal, with no wallet in that directory, it generates one, prints its mnemonic
**once** behind a red banner, and **waits for you to press Enter** before it goes any
further. Write the phrase down at that moment: it is not stored anywhere you can read
back, and every coin this node mines is paid to it.

Then it downloads `genesis.qmb` (only if the directory does not already have one),
verifies it against the genesis hash this binary was built with **before anything binds
a socket**, writes an ordinary `node.toml` into the directory, and runs it. Nothing is
hidden: read `~/.qumbra-miner/node.toml` afterwards and it is the same file §2.3 tells
you to write by hand. **Use the T2 release binary from §2.1** — `mine`'s baked-in
defaults (seeds, genesis URL, expected genesis hash) belong to the release you run, and
only a T2 release carries T2's
([`mine.rs:61-83`](../crates/qumbra-node/src/mine.rs#L61-L83)).

| flag | for |
|---|---|
| `--yes-i-backed-up` | the confirmation, for a run with no terminal (a systemd unit, a container). **Without a terminal and without this flag, `mine` refuses to create a wallet** rather than creating one silently — capture the mnemonic from the command's output yourself. |
| `--rkm <64 hex>` | pay a key you already have. No wallet is read, created, or looked for; this is the manual path of §2.3, unchanged. |
| `--seeds`, `--genesis-url`, `--listen`, `--index` | override the baked defaults. A non-zero `--index` is **allocated** in the wallet as part of the run, so what this node mines stays inside what `scan` covers; it is capped at 1024, and a higher index is served by `qumbra-wallet address --new` plus `--rkm`. |

The wallet lands in `~/.qumbra-miner/wallet`, so every wallet command in §4 works
against it — `qumbra-wallet backup --dir ~/.qumbra-miner/wallet --reveal` shows the
phrase again, and `scan` reads what this node has mined. Re-running `mine` on a prepared
directory changes nothing and just starts the node; if you have edited `node.toml` by
hand it refuses rather than overwriting your edit, and tells you to use `run --config`
instead.

On Windows this is `.\qumbra-node.exe mine --dir $HOME\.qumbra-miner` in PowerShell, and
it is the shortest native path there is — it writes the `node.toml` itself, so the TOML
backslash trap in §3 cannot bite you.

### 2.3 The manual path — join as a node, then turn on mining

Put the downloaded `genesis.qmb` beside this minimal `node.toml`:

```toml
data_dir = "/data"
listen_addr = "0.0.0.0:9400"
dial_peers = ["<T2_SEEDS — filled at announcement>"]
genesis_file = "/config/genesis.qmb"
expected_genesis_hash = "d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3"
mining = false
```

These are the fields a joiner needs. `mining` defaults to false, but it is explicit
above. Do **not** copy fleet configs and do **not** add `committee_key_paths`: a public
joiner is a verify-only node and holds no committee signing keys
([`config.rs:86`](../crates/qumbra-node/src/config.rs#L86)).

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
([`main.rs:596`](../crates/qumbra-node/src/main.rs#L596)). A different file hash
produces `WrongGenesisHash` and the node refuses to start
([`genesis.rs:796`](../crates/qumbra-node/src/genesis.rs#L796)).

From the prebuilt binaries, the same joiner is: identical config with ordinary paths
(`data_dir = "./qumbra-data"`, `genesis_file = "./genesis.qmb"`), then

```sh
curl -fsSL https://seed.qumbra.org/genesis.qmb -o genesis.qmb
./qumbra-node check --config node.toml     # same preflight, exits 0 and prints the genesis hash
./qumbra-node run   --config node.toml
```

`run` writes into `data_dir`, so run it from a directory you own.

**NAT is stated, not discovered.** Outbound-only participation is accepted by design:
behind NAT you sync, mine, and transact, but your address is never gossiped and you
serve no peers. Leave `advertise_addr` unset. Set it **only** when the advertised host
and port are genuinely publicly dialable all the way to this node; for the container
that also means publishing the P2P port, for example `-p 9400:9400/tcp`
([`config.rs:61-78`](../crates/qumbra-node/src/config.rs#L61-L78),
[`run.rs:821-826`](../crates/qumbra-node/src/run.rs#L821-L826)).

**What healthy joining looks like** — read successive `TELEMETRY` lines, not one
isolated sample:

- `peers=N` is the live peer count: `N > 0` means at least one connection is live;
  persistent `peers=0` means the node has not joined a peer.
- `slag=N` is fork-choice tip minus applied-state tip: it should fall toward `0` while
  joining; `slag=0` means the node has applied its own selected chain
  ([`run.rs:1292-1308`](../crates/qumbra-node/src/run.rs#L1292-L1308)).
- `mready=-` is correct for the non-mining config above. After mining is enabled,
  `unknown` or `behind` means the startup mining gate is correctly refusing; `synced` or
  `latched` permits mining. It does **not** prove `slag=0`
  ([`run.rs:1327-1343`](../crates/qumbra-node/src/run.rs#L1327-L1343)).

**Then turn on mining.** Generate the payout identity from the wallet whose address
should receive coinbase (§1.1 steps 1–3, or reuse the same wallet), and change only
these fields in `node.toml`:

```toml
mining = true
miner_rkm = "[64 hex characters printed by qumbra-wallet miner-rkm]"
```

On startup, prove the config took effect by finding this exact line:

```text
miner payout: coinbase notes paid to the configured miner_rkm
```

If you instead see the following warning, stop mining and fix the config: valid blocks
are being paid to an unspendable placeholder and the payout is unrecoverable
([`run.rs:838-840`](../crates/qumbra-node/src/run.rs#L838-L840)).

```text
⚠️  NO miner_rkm CONFIGURED: ... Every coin this node mines is BURNED.
```

### 2.4 Reward, maturity, pacing, and odds

- A won block pays your key **65% of the block subsidy (including the integer rounding
  remainder) plus all transaction fees**, minus any name-registration fees that block
  burned ([`emission.rs:85-87`](../crates/qlab-node/src/emission.rs#L85-L87),
  [`coinbase.rs:172-185`](../crates/qlab-node/src/coinbase.rs#L172-L185)). On T2 the
  coinbase is a v5 payee list; a solo miner is simply its only entry.
- Emission is **integer-exact from block 0** — no floating point is reachable from
  consensus, so the schedule is reproducible by anyone, to the unit. (T1 earned this
  rule at a mid-chain boundary and carries one scarred block; T2 is born with it.)
- The coinbase becomes spendable after **144 more blocks**. That is about three hours at
  target, not a wall-clock promise
  ([`emission.rs:83`](../crates/qlab-node/src/emission.rs#L83)).
- The network target is **75 seconds per block**, retargeted every block with
  **LWMA-120**; the deployable binary uses real RandomX
  ([`params_devnet.rs:16-42`](../crates/qlab-devnet/src/params_devnet.rs#L16-L42),
  [`pow.rs`](../crates/qlab-devnet/src/pow.rs)).
- Solo mining is not one reward every 75 seconds. Your expected share is your effective
  RandomX work divided by all effective work currently competing. The public surface
  exposes current difficulty, not fleet hash rate, so this guide cannot honestly quote
  personal odds. Expect variance and potentially long dry spells — the pool path exists
  precisely to smooth this.

### 🔴 The unresolved first-start silence

A node with `miner_rkm` configured has been observed once to spend **about 57 minutes at
silent 100% CPU before its first log line** on first start (observed on T1; the code
path is unchanged and unresolved —
[lab #300](https://github.com/qumbra-labs/qumbra-lab/issues/300)). It is not necessarily
hung: if the process is alive and a core is pegged while no log line appears, leave it
running. A restart does not fix the path and only discards the work already spent; wait
for the first line.

## 3. Platform notes — Windows and macOS

Solo mining works natively on Linux, macOS, and Windows; the pool path (§1) is
platform-trivial since XMRig ships official builds everywhere. What follows is only what
is *different* about Windows and macOS for the solo path.

**Why native mining is safe on T2, in one line:** T2's emission rule is exact integer
arithmetic from block 0 — no platform's math library is reachable from consensus, so no
platform can compute a divergent coinbase. (This was a genuine T1-era hazard, retired
there at a mid-chain boundary; on T2 the class is structurally excluded from birth.)

### Windows: native

`windows-x86_64.zip` holds `qumbra-node.exe` and `qumbra-wallet.exe`, built for
`x86_64-pc-windows-msvc` with the same RandomX C++ implementation every other platform
uses. CI runs RandomX's four official reference vectors on the MSVC build, so a Windows
miner's hashes are the network's hashes, not a near-miss.

Everything in §2 applies unchanged — same `genesis.qmb`, same `node.toml` fields, same
seeds. The differences:

**1. Download and verify, in PowerShell.** Windows has no `sha256sum`:

```powershell
# from the release page: the zip for your platform, and SHA256SUMS
Get-FileHash .\qumbra-<T2_RELEASE_TAG>-windows-x86_64.zip -Algorithm SHA256
# compare the printed hash against the matching line in SHA256SUMS — by eye, all 64 chars
Expand-Archive .\qumbra-<T2_RELEASE_TAG>-windows-x86_64.zip -DestinationPath .
cd qumbra-<T2_RELEASE_TAG>-windows-x86_64
.\qumbra-node.exe halt-status
```

The `halt-status` reading is the same one §2.1 describes: `build rev:` must match the
release notes, and `halt plan:` must say `no halt scheduled` and not **ARMED**.

**2. 🔴 SmartScreen will stop you, and it is right to.** These executables are
**unsigned** — there is no code-signing certificate on this project, and buying one is a
separate decision nobody has taken. The first run of either binary shows *"Windows
protected your PC"*. The path through it is **More info → Run anyway**. Microsoft
Defender may additionally flag a CPU miner on reputation alone.

This is the honest position and not a reassurance: an unsigned binary from a small
project is exactly the shape of thing SmartScreen exists to warn about, and *"click
through the security warning"* is advice you should be suspicious of by default. The
only thing that makes it reasonable here is that you can check the download yourself —
**verify the SHA-256 against SHA256SUMS before you click Run anyway**, not after.

**3. Paths in `node.toml` need single quotes.** TOML's double-quoted strings treat `\`
as an escape character, so `data_dir = "C:\Users\you\qumbra-data"` is either a parse
error or a different directory than you meant. Use a TOML *literal* string, or forward
slashes:

```toml
data_dir = 'C:\Users\you\qumbra-data'          # literal string — backslashes are literal
genesis_file = 'C:\Users\you\genesis.qmb'
# or, equally valid on Windows:
# data_dir = "C:/Users/you/qumbra-data"
```

(`qumbra-node mine` writes its own `node.toml`, so this trap only exists on the manual
path.)

**4. Run it from a console you opened, and stop it with Ctrl-C.** Open PowerShell or
Windows Terminal and run `.\qumbra-node.exe run --config node.toml` there — do not
double-click it. **Ctrl-C is the stop that reliably flushes the snapshot.** Closing the
console window flushes too, but Windows gives any program about five seconds after a
window-close before killing it, and a node busy inside a RandomX round can miss that
budget.

Nothing is lost when it does: the block log is fsync'd per record and is the source of
truth, so a node that missed its snapshot flush replays the log on next start and
reaches exactly the same state. What a missed flush costs is **replay time**, not coins
or history.

**5. The wallet's seed file is not owner-only on Windows.** On Linux and macOS
`qumbra-wallet keygen` writes `wallet.seed` with mode `0600`. Windows has no such mode
and this build does not set an ACL, so the file inherits whatever the folder gives it —
under your own profile that is normally you *plus* SYSTEM and Administrators. `keygen`
prints this rather than claiming a protection it does not have. To make the wallet
folder owner-only, run once:

```powershell
icacls "$env:USERPROFILE\.qumbra-wallet" /inheritance:r /grant:r "${env:USERNAME}:(OI)(CI)F"
```

Anyone who can read that file owns every coin the wallet holds.

**6. Keep the machine awake.** Set Windows power settings not to sleep, and keep a
laptop plugged in — a sleeping host mines nothing. (This applies to the pool path too.)

**Not in this port** (named so nobody looks for it): no Windows service wrapper — to
survive sign-out, register the `run` command as a Task Scheduler task with *"Run whether
user is logged on or not"*, which is outside what this guide covers. No code signing. No
ARM Windows build.

### Windows: WSL2

Still supported: `wsl --install` from an admin PowerShell, then follow §2 inside Ubuntu
with the `linux-x86_64-glibc` tarball. It mines at effectively native speed. With a
native build available, WSL2 is the fallback rather than the path.

### macOS

Apple Silicon is supported natively (`macos-arm64.tar.gz`). Two things to know: the
quarantine/Gatekeeper note in §2.1 (fetch with `curl` or clear the attribute), and — as
everywhere — keep the machine awake and plugged in while mining.

## 4. Wallet quickstart — five commands

The CLI's own help is the authority for flags and units
([`qumbra-wallet/main.rs:270-290`](../crates/qumbra-wallet/src/main.rs#L270-L290)).
These five commands cover the shortest user journey; replace `RECIPIENT_QADDR` and keep
in mind that `--amount` is in **bessel** (`100,000,000` bessel = `1 QMB`;
[`emission.rs:69`](../crates/qlab-node/src/emission.rs#L69)). On T2 there is no
pre-funded shortcut: **coins come from mining** — either path in this guide — and become
spendable 144 blocks after the block that paid them.

**On Windows**, the same five commands run in PowerShell with `.\qumbra-wallet.exe` in
place of `qumbra-wallet`; `$HOME` and `$env:USERPROFILE` both work for `--dir`.
Command 3 needs a PowerShell equivalent, since `jq` is not present by default:
`$TIP = (Invoke-RestMethod https://explorer.qumbra.org/v1/health.json).chain.tip_height`.

```sh
# 1 — create the wallet; this prints address [0]
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 — back up the Qumbra mnemonic in a private terminal
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# (mine to this wallet — §1 pool or §2 solo — and wait out the 144-block maturity)

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

The explorer at <https://explorer.qumbra.org> shows the chain the wallet is claiming
against — its `/v1/health.json` carries the chain's served `network` field
([`qumbra-explorer/main.rs:113`](../crates/qumbra-explorer/src/main.rs#L113)), which on
T2 reads `qumbra-t2`; if it reads anything else, you are looking at the wrong net's
explorer.
