# 加入 Qumbra 测试网并挖矿

English: [`join-and-mine.md`](./join-and-mine.md) · **技术细节以英文版为准。**

> **公开镜像读者注意:** 文中的代码链接(`../crates/…`、`../deploy/…`)指向 T1 期间仍私有
> 的开发树——它们记录每个论断在哪里被测试,外部读者点开会 404,待该树公开后恢复。加入
> 网络所需的一切都不依赖这些链接。

这是项目无法控制的参与者进入 T1 的操作路径。

> **数值已于 2026-08-16 作为 T1 公告包填入**(设计文档 `t1-sg-posture-decision`,顺序:
> Gate A 复测 → 本包 → SG 开放)。它们读自已部署的 fleet,不是猜测:镜像 digest/revision
> 来自 compose pin 及其 OCI 标签回读,种子是 SG 姿态决定的四个**开放**入口(node0 刻意
> 不是入口)。**公告发布(Larry 放行)之前,这些种子监听的端口尚未对公网开放**——在那一刻
> 之前连接被拒是姿态在起作用,不是地址写错。若 fleet 在填写与公告之间又滚了镜像,
> digest/revision 两行在发布时重新核验。

## 1. 获取并验证发布版本

节点镜像公开位于 `ghcr.io/qumbra-labs/qumbra-node`。只用 **T1 公告给出的 digest**，绝不
依赖可变 tag。镜像把源码 revision 记录在 OCI 标签
`org.opencontainers.image.revision` 中；必须读回并与公告中的 revision 比较，不能相信 tag
或一次成功的 pull。标签与节点二进制均在 runtime 镜像中
([`deploy/docker/Dockerfile:123-160`](../deploy/docker/Dockerfile#L123-L160))；必须读回是
[lab #224](https://github.com/qumbra-labs/qumbra-lab/issues/224) 的教训。

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

公告还会把以下网络身份输入一同发布：

- `genesis.qmb` —— **格式 v4**；
- `expected_genesis_hash` ——
  `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb`；
- 写入 `dial_peers` 的初始 P2P 种子地址。

格式和哈希已在代码树中钉死
([`genesis.rs:68-77`](../crates/qumbra-node/src/genesis.rs#L68-L77)、
[`genesis.rs:775-779`](../crates/qumbra-node/src/genesis.rs#L775-L779))。分发:`genesis.qmb` 从
**`https://seed.qumbra.org/genesis.qmb`** 下载(deploy PR #150)——务必按上方
`expected_genesis_hash` 逐字节校验;`qumbra-node check` 与启动都会拒绝错误文件,被篡改的
下载无法静默通过。种子列表即下方四个开放入口(`t1-sg-posture-decision`)。

### 备选路径：预编译二进制，不用 Docker（2026-08-17 新增，lab #437）

上面的容器仍然是**可复现的基准路径**，本文编号步骤用的也是它。如果 Docker 本身就是障碍
而不是答案，同样的两个二进制以 tarball 形式发布在公开仓库的 releases 页：

**<https://github.com/qumbra-labs/qumbra/releases>**

> **在第一个版本被切出来之前，那个页面是空的，容器路径是唯一路径。** 发布流水线已经存在
> ([`release-binaries.yml`](../.github/workflows/release-binaries.yml))，由人手动触发；
> 一旦发布，T1 公告会给出对应的 tag。

| tarball | 适用于 |
|---|---|
| `…-linux-x86_64-glibc.tar.gz` | Intel/AMD Linux，glibc 2.36+（Debian 12、Ubuntu 22.04+） |
| `…-linux-aarch64-glibc.tar.gz` | arm64 Linux —— 测试网机队自己跑的就是它 |
| `…-macos-arm64.tar.gz` | Apple Silicon，macOS 11+ |

每个包内含 `qumbra-node`、`qumbra-wallet` 和一份 `PROVENANCE.txt`。**没有原生 Windows
构建**——节点有仅限 unix 的依赖；在 WSL2 下，按 Linux 用户的方式使用 Linux x86_64 包即可。

```sh
# 1 —— 从 release 页下载对应平台的 tarball 与 SHA256SUMS，然后：
sha256sum -c SHA256SUMS          # macOS：shasum -a 256 -c SHA256SUMS
tar -xzf qumbra-t1-<shortrev>-<platform>.tar.gz
cd qumbra-t1-<shortrev>-<platform>

# 2 —— 让二进制自己说明它是什么。下面两行都是承重的：
./qumbra-node halt-status
```

```text
  build rev:    <release notes 中给出的源码 revision>
  halt plan:    no halt scheduled
  resumes past: height 8640 (post-halt rules apply above it)
```

`build rev:` 是一个已下载的二进制证明自己来自哪份源码的方式——tarball 没有镜像标签，
所以这一行等价于上面容器路径里对 `org.opencontainers.image.revision` 的回读。如果
`halt plan:` 显示 **ARMED**，那个二进制会在 8,640 停住、无法跟随当前链，不要运行它。CI
拒绝发布 ARMED 产物，所以 release 页上出现 ARMED 二进制意味着该产物并非它所声称的东西。

**仅 macOS：** 这些二进制未签名、未公证。用浏览器下载会打上隔离属性，Gatekeeper 会拒绝
运行。请用 `curl` 下载，或显式清除该属性：
`xattr -d com.apple.quarantine qumbra-node qumbra-wallet`。

## 2. 作为不挖矿的节点加入

把下载的 `genesis.qmb` 与以下最小 `node.toml` 放在同一目录：

```toml
data_dir = "/data"
listen_addr = "0.0.0.0:9400"
dial_peers = ["18.202.166.126:9444","18.141.177.109:9444","52.194.224.123:9444","52.5.0.21:9444"]
genesis_file = "/config/genesis.qmb"
expected_genesis_hash = "138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb"
mining = false
```

这些就是加入者需要的字段。`mining` 默认是 false，但上面仍明确写出。**不要**复制机队配置，
也**不要**添加 `committee_key_paths`：公网加入者是只验证节点，不持有委员会签名密钥
([`config.rs:54-94`](../crates/qumbra-node/src/config.rs#L54-L94))。

先在不绑定 socket 的情况下预检确切的 genesis 与配置，再运行节点：

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

`check` 在不开 listener 的情况下执行与启动相同的字节、格式和哈希门
([`run.rs:226-242`](../crates/qumbra-node/src/run.rs#L226-L242))。文件哈希不同会产生
`WrongGenesisHash`，节点拒绝启动
([`genesis.rs:528-555`](../crates/qumbra-node/src/genesis.rs#L528-L555))。

### 用预编译二进制做同一件事

配置、genesis、种子完全相同，只有调用方式不同。把 `genesis.qmb` 和 `node.toml` 放在解压出
的二进制旁边，`data_dir` 与 `genesis_file` 写成普通路径，而不是容器里的 `/data` 和
`/config`：

```toml
data_dir = "./qumbra-data"
listen_addr = "0.0.0.0:9400"
dial_peers = ["18.202.166.126:9444","18.141.177.109:9444","52.194.224.123:9444","52.5.0.21:9444"]
genesis_file = "./genesis.qmb"
expected_genesis_hash = "138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb"
mining = false
```

```sh
curl -fsSL https://seed.qumbra.org/genesis.qmb -o genesis.qmb
./qumbra-node check --config node.toml     # 同一个预检，退出码 0 并打印 genesis 哈希
./qumbra-node run   --config node.toml
```

发布流水线在每个产物被允许上传到 release 页之前，都会针对同一个已发布的 genesis 跑一遍
上面这个 `check`——所以到你手上的 tarball 已经在它自己的平台上预检通过了。`run` 会写入
`data_dir`，请在你有权限的目录下运行；本文其余部分——下面的 telemetry 字段、§3 的挖矿、
§4 的钱包——两条路径读法完全一致。

### NAT 是声明，不是发现

**设计上接受仅出站参与。** 在 NAT 后你仍可同步、挖矿和交易，但你的地址永远不会被 gossip，
也不会为任何 peer 提供服务。不要设置 `advertise_addr`。只有当所填 host 与 port 确实能从公网
一路拨通到此节点时才设置它；对容器而言还须发布 P2P 端口，例如 `-p 9400:9400/tcp`。这是
记录在案的 2026-07-26 NAT 决定，不是变通办法
([`config.rs:68-78`](../crates/qumbra-node/src/config.rs#L68-L78)、
[`run.rs:588-595`](../crates/qumbra-node/src/run.rs#L588-L595))。

### 健康加入是什么样

要连续阅读多条 `TELEMETRY`，不要只看一个样本
([`run.rs:1342-1371`](../crates/qumbra-node/src/run.rs#L1342-L1371))：

- `peers=N` 是实时 peer 数：`N > 0` 表示至少一个连接在线；持续 `peers=0` 表示尚未加入任何
  peer。
- `slag=N` 是 fork-choice tip 减去 applied-state tip：加入过程中应向 `0` 下降；`slag=0` 表示
  节点已应用自己选中的链
  ([`run.rs:1017-1034`](../crates/qumbra-node/src/run.rs#L1017-L1034))。
- 对上面的非挖矿配置，`mready=-` 完全正确。启用挖矿后，`unknown` 或 `behind` 表示启动挖矿门
  正在正确拒绝；`synced` 或 `latched` 才允许挖矿。它**不能**证明 `slag=0`
  ([`run.rs:376-421`](../crates/qumbra-node/src/run.rs#L376-L421))。

## 3. 把已加入的节点改成矿工

### 平台边界——2026-08-17 已更正(原:只能用 Linux/glibc)

> **带日期更正(2026-08-17,lab #437):原生 macOS 挖矿已获准许。** 下方原红字边界的条件
> 是"确定性发行边界尚未激活"——该边界已于 2026-08-12 激活(#299/#303 已关)。高度 8,640
> 之上,`body.coinbase == coinbase_exact(height)` 是纯整数运算的硬共识规则,共识路径碰不到
> libm,原生 macOS 矿工既算不出偏差 coinbase,也造不成历史伤痕。**已实测验证,不止是论证**:
> 一个原生 macOS arm64 构建经公开入口加入 T1,同步后最初 ~100 分钟赢下 40 个被接受并
> finalize 的块(lab #437)。容器仍是铺好的可复现路径;原生构建现在是受支持的替代路径。
>
> **从源码构建时有一个 flag 性命攸关**:裸 `cargo build -p qumbra-node` 产出 ARMED 变体,
> 会在 8,640 停机、无法跟上今天的链。必须带
> `--features qumbra-node/rule-boundary-resume` 构建,并用 `qumbra-node check` 确认——
> `halt plan:` 行必须是 `no halt scheduled`,不能是 `ARMED`。**§1 中发布的 tarball 已经
> 带着这个 flag 构建**,且 CI 拒绝发布没带的产物,所以只有自己编译时才需要操心这个 flag。

原边界文字存档:*在确定性发行边界被公开确认已激活之前,只能在 Linux/glibc 上挖矿……实测
macOS 矿工在特定高度计算出的 coinbase 会与 glibc 相差 ±1 bessel
([lab #303](https://github.com/qumbra-labs/qumbra-lab/issues/303));当时的共识接受这些值,
每个这样的块都成为 [lab #299](https://github.com/qumbra-labs/qumbra-lab/issues/299) 激活
规则必须 grandfather 的永久伤痕。*

从应接收 coinbase 的钱包派生付款身份：

```sh
qumbra-wallet miner-rkm --dir "$HOME/.qumbra-wallet"
```

该命令从已分配地址 index 派生（默认 index 0），并打印节点所需的准确 64 位十六进制
`miner_rkm = "…"` 行
([`qumbra-wallet/main.rs:147-175`](../crates/qumbra-wallet/src/main.rs#L147-L175))。把它粘进
`node.toml`，只改变以下挖矿字段：

```toml
mining = true
miner_rkm = "[qumbra-wallet miner-rkm 打印的 64 个十六进制字符]"
```

启动时必须找到下面这条完全一致的行，以证明配置生效：

```text
miner payout: coinbase notes paid to the configured miner_rkm
```

如果看到下面的警告，立即停止挖矿并修正配置：有效块正在付给无法花费的占位身份，收益无法找回
([`run.rs:596-612`](../crates/qumbra-node/src/run.rs#L596-L612))。

```text
⚠️  NO miner_rkm CONFIGURED: ... Every coin this node mines is BURNED.
```

### 🔴 尚未解决的首次启动静默

曾有一次观察到：配置了 `miner_rkm` 的节点在首次启动、第一条日志出现之前，**约 57 分钟保持
100% CPU 且完全静默**。问题尚未解决
([lab #300](https://github.com/qumbra-labs/qumbra-lab/issues/300))。它不一定卡死：若进程仍活着、
一个核心被打满但没有日志，就让它继续运行。重启不能修复这条路径，只会丢掉已经花掉的工作；
等待第一条日志。

### 收益、成熟、节奏与胜率

- 赢下一个块会向钱包支付**区块补贴的 65%（包括整数舍入余数）加全部交易费**
  ([`emission.rs:27-30`](../crates/qlab-node/src/emission.rs#L27-L30)、
  [`coinbase.rs:133-139`](../crates/qlab-node/src/coinbase.rs#L133-L139))。
- Coinbase 再经过 **144 个块**才可花费。按目标约为三小时，但这不是墙上时钟承诺
  ([`emission.rs:47-55`](../crates/qlab-node/src/emission.rs#L47-L55))。
- 全网目标为**每块 75 秒**，用 **LWMA-120** 每块重定难度；可部署二进制使用真实 RandomX，
  不是 Keccak 模拟占位
  ([`params_devnet.rs:16-49`](../crates/qlab-devnet/src/params_devnet.rs#L16-L49)、
  [`pow.rs:66-100`](../crates/qlab-devnet/src/pow.rs#L66-L100))。
- 这是 solo mining，不是矿池，更不是每 75 秒给你一笔奖励。你的期望份额等于你的有效 RandomX
  工作量除以当前所有竞争工作量。公开面只暴露当前难度，不暴露机队总算力，因此本文无法诚实地
  报出个人胜率。要预期随机波动，也可能长时间一块不中。

## 4. 钱包快速开始 —— 五条命令

参数和单位以 CLI 自带帮助为准
([`qumbra-wallet/main.rs:43-66`](../crates/qumbra-wallet/src/main.rs#L43-L66))。以下五条命令
覆盖最短用户旅程；替换 `RECIPIENT_QADDR`，并记住 `--amount` 的单位是 **bessel**
（`100,000,000` bessel = `1 QMB`；
[`emission.rs:34-35`](../crates/qlab-node/src/emission.rs#L34-L35)）。
第 3 条命令使用 `curl` 和 `jq`；完整命令参考请运行 `qumbra-wallet --help`。

```sh
# 1 —— 创建钱包；命令会打印 address [0]
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 —— 在私密终端备份 Qumbra 助记词
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# 把完整 address [0] 粘贴到 https://faucet.qumbra.org，等待 grant。

# 3 —— 获取余额声明所针对的高度
TIP="$(curl -fsS https://explorer.qumbra.org/v1/health.json | jq -r '.chain.tip_height')"

# 4 —— 经公开节点边缘发现输出并扣除已花 note
qumbra-wallet scan --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --to "$TIP"

# 5 —— 重新扫描、构造真实证明并提交；省略 --node 时默认等于 --url
qumbra-wallet send --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --scan-to "$TIP" \
  --to RECIPIENT_QADDR --amount 100000000
```

HTTP **429 Too Many Requests 是当前 faucet 限制的预期行为**；不要反复轰击表单。
[Lab #308](https://github.com/qumbra-labs/qumbra-lab/issues/308) 记录了：通过反向代理后，按网络
取 key 的 bucket 目前会被看成共享 bucket，因此可能已被另一位访客用掉。只要该 issue 仍开放，
就按页面给出的窗口之后再试。

## 5. 镜像声明的验证记录

下面的记录只证明 package 路径公开且远程标签读回有效；它**不是** T1 镜像公告。2026-08-10，
未 pull、未使用 registry 凭证：

```text
$ docker buildx imagetools inspect ghcr.io/qumbra-labs/qumbra-node:t0-wan-14
Name:   ghcr.io/qumbra-labs/qumbra-node:t0-wan-14
Digest: sha256:931afa0fe57844f52e5bbb390914ec3116a72c3ff04781b11d8a080dcc4c2f29

$ docker buildx imagetools inspect --format \
  '{{index .Image.Config.Labels "org.opencontainers.image.revision"}}' \
  ghcr.io/qumbra-labs/qumbra-node@sha256:931afa0fe57844f52e5bbb390914ec3116a72c3ff04781b11d8a080dcc4c2f29
e8d52d7b194d3560f70de5d1f26b99b6f37bdd2e
```

不要拿这个历史 T0 digest 替换第 1 节方括号中的 T1 digest。

第 4 节的公开读取也于 2026-08-10 验证：`GET
https://explorer.qumbra.org/v1/health.json` 返回了钉死的 genesis 哈希和数字
`chain.tip_height`，`GET https://seed.qumbra.org/v1/compact?from=6313&to=6313` 则返回 HTTP 200
及 `application/octet-stream`。JSON 字段定义在
[`qumbra-explorer/json.rs:55-80`](../crates/qumbra-explorer/src/json.rs#L55-L80)；高度 `6313`
只是一次样本探测高度，不是网络参数。
