# 加入 Qumbra T2 并挖矿

English: [`join-and-mine.md`](./join-and-mine.md) · **技术细节以英文版为准。**

> **公开镜像读者注意：** 文中的代码链接（`../crates/…`、`../deploy/…`）指向开发树，
> 该树按自己的时间表保持私有——它们标明每条主张在哪里被测试，但对外部读者在该树
> 公开前会 404。加入所需的一切都不依赖点开它们。

这是项目无法控制的参与者进入 T2 的操作路径。现在有**两种挖矿方式**，你只需要其中一种：

| 路径 | 你运行什么 | 你需要什么 |
|---|---|---|
| **矿池**（§1） | 原版 [XMRig](https://github.com/xmrig/xmrig) 指向 `pool.qumbra.org:3333` | 一个 `qumbra-wallet` 生成的收款密钥——**不需要节点、不需要创世文件、不需要同步** |
| **独立挖矿**（§2） | 自己的全节点，`qumbra-node mine` | 节点二进制 + T2 创世文件 |

矿池挖矿能抹平方差，几分钟就能跑起来；独立挖矿在你赢下区块时独得全部奖励，并让你成为
这条链的完整验证者。钱包（§4）两条路径通用。

> ### ⚠️ T1 已退役——本指南面向 T2
>
> **T1 已经关停。** 如果你在按本指南的旧版本操作：停下——它的数值已经作废。T1 余额
> **不会带入**；T2 是从全新创世开始的公平重启——没有预挖、没有继承，区块 0 对所有人
> （包括运营者）都是同一条起跑线。任何写着 T1 创世哈希（`138e1524…addb`）、`t1-*`
> release tag、或高度 19,008 更新期限的东西，都属于已退役的网络，在这里不适用。T2 在
> 构造上就是**另一个网络**：其创世文件为格式 **v5**、网络名 `qumbra-t2`
> （[`genesis.rs:630-636`](../crates/qumbra-node/src/genesis.rs#L630-L636)），钉错创世的
> 节点拒绝启动，接错网的对等节点在 P2P 握手时被点名拒绝
> （[`peer.rs:45-47`](../crates/qlab-p2p/src/peer.rs#L45-L47)，lab #474）——你不可能
> 误入错误的链，只会响亮地失败。

> **切换时刻什么是真的、什么不是。** 下方创世哈希是 T2 的真值——钉住它、验证它，
> 别信任何别的。本指南在 14:00 UTC+8 **不主张**下面两件事已经上线：
>
> 1. **T2 二进制 / 镜像 digest。** T2 release 在切换**之后**才切割（release 闸门在
>    链未上线前拒绝打 T2 tag）。在
>    <https://github.com/qumbra-labs/qumbra/releases> 出现 T2 tag 之前，没有可运行的
>    T2 二进制。每一个 `t1-*` tag 都是 T1 产物——不要拿它连 T2。
> 2. **矿池端点。** `pool.qumbra.org:3333` 是路径（§1 的 stock XMRig 配置）。矿池在
>    切换之后才启用：一个看起来活着、却对着假模板发任务的 stratum 会烧掉矿工算力，
>    所以在本仓库后续说明它已上线之前，该主机名**不是**付款端点。配置写在这里，是
>    为了那一刻只需粘贴一次。
>
> 镜像 digest、镜像 revision、P2P 种子、T2 release tag 在这些值存在之前，仍写作
> `<… — filled at announcement>`。来自本页或 T2 公告以外任何地方（包括本文件旧副本）
> 的创世哈希，定义为错。

## 1. 矿池路径——原版 XMRig，不需要节点

> **切换之后才启用。** 本节配置是路径，不是活端点主张。在 T2 公告（或本仓库后续
> 说明）声明矿池已上线之前，不要把矿机指向 `pool.qumbra.org:3333` 并指望付款。

第一次接触矿池？**Stratum** 是矿工与矿池之间用一条长 TCP 连接说话的 JSON 行协议
（登录 → 任务 → 提交份额）。Qumbra 说的是原版 XMRig 天生就懂的 Monero 系方言——
T2 区块头布局的全部目的就在于此。一句话：矿工侧一行代码都不用改。

一句话版本：Qumbra 的矿池说的是原版 XMRig 天生就懂的 Monero 系 stratum 方言，T2 区块头
的布局就是为了**矿工侧一行代码都不用改**而设计的。矿池是免托管的：它从不经手你的
币——赢下的区块由 coinbase 直接在链上付给矿工
（[`payee.rs`](../crates/qumbra-pool/src/payee.rs)）。

### 1.1 生成收款密钥

你的收款身份是一个从你自己控制的钱包派生出的 64 位十六进制密钥。从 §2.1 的 release
归档取得 `qumbra-wallet`（任意平台；你**不需要**运行节点），然后：

```sh
# 1 —— 创建钱包（打印地址 [0]）
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 —— 立即在私密终端备份助记词——它只显示、不可回读
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# 3 —— 派生矿池付款的收款密钥
qumbra-wallet miner-rkm --dir "$HOME/.qumbra-wallet"
```

第 3 步打印一行 `miner_rkm = "…"`，其值**恰好是 64 个十六进制字符**
（[`qumbra-wallet/main.rs:476-495`](../crates/qumbra-wallet/src/main.rs#L476-L495)）。
这个值——裸的十六进制，不含 `miner_rkm = ""` 外壳——就是填进 XMRig `user` 字段的东西。

**一个字符都不能错。** 矿池付款给的正是你登录用的那个密钥：64 位十六进制的登录名
**本身就是**收款密钥
（[`payee.rs:45-62`](../crates/qumbra-pool/src/payee.rs#L45-L62)）。打错或截断的登录名
照样挖矿、照样计份额，但**永远收不到钱**——矿池无从知道你本来想写什么。没有账户找回，
因为根本没有账户。

### 1.2 把 XMRig 指向矿池

从 XMRig 官方 releases 页下载原版
（<https://github.com/xmrig/xmrig/releases>——Windows、macOS、Linux 都有）。预期你的
杀毒软件或 SmartScreen 会标记它：它是 CPU 矿工，而标记 CPU 矿工正是信誉过滤器的本职。
先对照 XMRig 公布的哈希校验下载，再做决定。

最小 `config.json`：

```json
{
    "pools": [
        {
            "url": "pool.qumbra.org:3333",
            "user": "<qumbra-wallet miner-rkm 打印的那 64 个十六进制字符>",
            "pass": "x",
            "algo": "rx/0",
            "keepalive": true,
            "tls": false
        }
    ]
}
```

同样的东西写成命令行：

```sh
xmrig -o pool.qumbra.org:3333 -a rx/0 -k \
      -u <qumbra-wallet miner-rkm 打印的那 64 个十六进制字符> -p x
```

`algo` 必须是 `rx/0`——其余的矿池会点名拒绝
（[`pool.rs:248`](../crates/qumbra-pool/src/pool.rs#L248)）。`pass` 任意填写、未被使用。
**矿池宣布上线之后**，健康的输出是 XMRig 打印 `new job from pool.qumbra.org:3333`，
随后稳定滴答出现 `accepted` 份额行；有任务但几分钟后仍零 accepted 说明有问题——
先复查 `algo`。在那之前，连接被拒（或任务永远变不成链上块）是暂缓在起作用，不是
地址写错。

### 1.3 你如何拿到钱——外推收益前先读这一节

- 矿池在 **PPLNS 窗口内计份额：最近 1,024 个被接受的份额，按难度加权**
  （[`pplns.rs:1-10`](../crates/qumbra-pool/src/pplns.rs#L1-L10)；窗口大小是
  devnet-placeholder 常量，可能被重调）。
- **启动时，每个赢下的区块只付给一名矿工**——v5 coinbase 收款人列表在诞生期上限为
  **N=1**（[`body.rs:94`](../crates/qlab-devnet/src/body.rs#L94)），所以一个区块的全部
  矿工奖励付给那一刻窗口中权重最高的可付款矿工
  （[`payee.rs:107-131`](../crates/qumbra-pool/src/payee.rs#L107-L131)）。拉长看这是
  按份额比例支付；短窗口内它表现为按份额加权的抽签。若你的算力只占矿池一小部分，预期
  会有独立挖矿者式的枯水期，只是按矿池赢块频率成比例缩短。提高上限（真正的每块分账）
  是一个计划中的、需宣告的规则变更，不是矿池能拧的旋钮。
- 支付走**链上 coinbase**，因此矿池付款同样服从 coinbase 成熟期：**再过 144 个区块**
  才可花费，按目标出块约三小时
  （[`emission.rs:83`](../crates/qlab-node/src/emission.rs#L83)）。
- 你的收益由你的钱包可见，而不是矿池的仪表盘：`miner-rkm` 派生自钱包地址索引 0，所以
  §4 的 `scan` 命令能展示矿池付给你的一切，不需要信任任何人。

## 2. 独立路径——运行节点并挖矿

### 2.1 获取并验证发布版本

节点镜像公开位于 `ghcr.io/qumbra-labs/qumbra-node`。只用 **T2 公告给出的 digest**，绝不
依赖可变 tag。镜像把源码 revision 记录在 OCI 标签
`org.opencontainers.image.revision` 中；必须读回并与公告中的 revision 比较，不能相信
tag 或一次成功的 pull。标签与节点二进制均在 runtime 镜像中
（[`deploy/docker/Dockerfile:144`](../deploy/docker/Dockerfile#L144)）；必须读回是
[lab #224](https://github.com/qumbra-labs/qumbra-lab/issues/224) 的教训。

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

公告同时一并发布这些网络身份输入：

- `genesis.qmb`——**格式 v5，网络 `qumbra-t2`**
  （[`genesis.rs:630-636`](../crates/qumbra-node/src/genesis.rs#L630-L636)）；
- `expected_genesis_hash`——`d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3`；
- `dial_peers` 的初始 P2P 种子地址——`"18.202.166.126:9444", "18.141.177.109:9444", "52.194.224.123:9444", "52.5.0.21:9444"`。

分发：`genesis.qmb` 从 **`https://seed.qumbra.org/genesis.qmb`** 下载（裸服务名在
切换时从 T1 移交给 T2；`pool.qumbra.org` 是新增且暂缓的——见上方框）——务必对照
上面的 `expected_genesis_hash` 用 **`qumbra-node check`** 验证——那是这项检查的可执行形式:
把哈希填进配置,`check` 会从文件重算创世身份并拒绝不匹配。(**不要**指望文件的 `sha256`
等于上面那个哈希:钉住的值是对规范编码取的 keccak256,与文件摘要是两个不同的数,
拿 `shasum` 直接比会看起来像不匹配,而其实一切正常。)`check` 与启动都会拒绝错误
文件（[`genesis.rs:525-530`](../crates/qumbra-node/src/genesis.rs#L525-L530)），被
篡改的下载无法静默通过。切换完成前，该 URL 可能仍在提供 T1 文件或拒绝连接——
对照 T2 哈希校验；对不上说明切换正在进行，不是忽略哈希的理由。

#### 预编译二进制，不用 Docker

上面的容器仍是**可复现基线**。若 Docker 是障碍而非答案，同样的三个二进制
（`qumbra-node`、`qumbra-wallet`、`qumbra-pool`，外加 `PROVENANCE.txt`）以归档形式
落在公开仓库的 releases 页——**<https://github.com/qumbra-labs/qumbra/releases>**，
tag 为 `<T2_RELEASE_TAG — filled at announcement>`——**在切换之后**。该 tag 此刻不
存在；release 闸门在链未上线前拒绝切割。当前已发布的每一个 tag（`t1-91bdee4`、
`t1-c5cfff8`、`t1-84f6f4c`）都是 T1 产物。不要拿 `t1-*` 二进制连 T2。

| 归档 | 适用 |
|---|---|
| `…-linux-x86_64-glibc.tar.gz` | Intel/AMD Linux，glibc 2.36+（Debian 12、Ubuntu 22.04+） |
| `…-linux-aarch64-glibc.tar.gz` | arm64 Linux——fleet 自身运行的平台 |
| `…-macos-arm64.tar.gz` | Apple Silicon，macOS 11+ |
| `…-windows-x86_64.zip` | Windows 10/11 x64——原生，无需 WSL2 |

```sh
# 1 —— 从 release 页下载对应平台的归档与 SHA256SUMS，然后：
sha256sum -c SHA256SUMS          # macOS 用：shasum -a 256 -c SHA256SUMS
tar -xzf qumbra-<T2_RELEASE_TAG>-<platform>.tar.gz
cd qumbra-<T2_RELEASE_TAG>-<platform>

# 2 —— 问二进制它是什么：
./qumbra-node halt-status
```

输出中的 `build rev:` 是下载来的二进制证明自己出自哪份源码的方式——tarball 没有镜像
标签，这一行就等价于容器路径里的 `org.opencontainers.image.revision` 读回。它必须与
release notes 点名的 revision 一致。`halt plan:` 必须读作 `no halt scheduled`；若显示
**ARMED**，那个二进制被排定在某个固定高度停止跟链——不要运行它，并把它出现在
release 页这件事本身当作制品名不符实的证据（CI 拒绝发布这样的东西）。

**仅 macOS：** 二进制未签名、未公证。浏览器下载会打上隔离标记，Gatekeeper 拒绝运行。
用 `curl` 获取，或清除属性：
`xattr -d com.apple.quarantine qumbra-node qumbra-wallet qumbra-pool`。

**若你从源码构建，有一个 flag 是承重的**：裸的 `cargo build -p qumbra-node` 产出 ARMED
变体——一个 T1 升级期的产物，但其排定的停机在任何网上都会触发，T2 也不例外
（[`release.rs:712-714`](../crates/qumbra-node/src/release.rs#L712-L714)）。带
`--features qumbra-node/rule-boundary-resume` 构建，并用 `qumbra-node halt-status`
确认：`halt plan:` 必须是 `no halt scheduled`，不是 `ARMED`。已发布的归档都已带此
flag 构建。

### 2.2 一条命令：`qumbra-node mine`

§2.3 里的一切——钱包、备份、收款密钥、手写的 `node.toml`、创世下载与校验——都是
`qumbra-node mine` 替你做的事。它是同一个节点、同一份配置；区别只是五步变一步：

```sh
./qumbra-node mine --dir ~/.qumbra-miner
```

在终端上，若目录里没有钱包，它会生成一个，在红色横幅后**只显示一次**助记词，并
**等你按下回车**才继续。就在那一刻把短语抄下来：它不存放在任何你能回读的地方，而这个
节点挖到的每一枚币都付给它。

随后它下载 `genesis.qmb`（仅当目录里还没有时），在**绑定任何 socket 之前**用这份二进制
构建时内置的创世哈希校验它，往目录里写一份普通的 `node.toml`，然后运行。没有任何隐藏：
事后读 `~/.qumbra-miner/node.toml`，它就是 §2.3 教你手写的那份文件。**请使用 §2.1 的
T2 release 二进制**——`mine` 内置的默认值（种子、创世 URL、期望创世哈希）属于你运行的
那个 release，只有 T2 release 携带 T2 的
（[`mine.rs:61-83`](../crates/qumbra-node/src/mine.rs#L61-L83)）。

| flag | 用途 |
|---|---|
| `--yes-i-backed-up` | 无终端运行（systemd unit、容器）时的确认。**无终端且无此 flag 时，`mine` 拒绝创建钱包**而不是静默创建——请自行从命令输出中截获助记词。 |
| `--rkm <64 hex>` | 付给你已有的密钥。不读取、不创建、不寻找钱包；就是 §2.3 的手动路径，原样不变。 |
| `--seeds`、`--genesis-url`、`--listen`、`--index` | 覆盖内置默认值。非零 `--index` 会作为运行的一部分在钱包中**登记分配**，使这个节点挖到的东西始终在 `scan` 的覆盖范围内；上限 1024，更高的索引由 `qumbra-wallet address --new` 加 `--rkm` 提供。 |

钱包落在 `~/.qumbra-miner/wallet`，因此 §4 的每条钱包命令都能对着它用——
`qumbra-wallet backup --dir ~/.qumbra-miner/wallet --reveal` 再次显示短语，`scan` 读出
这个节点挖到了什么。对已就绪的目录重跑 `mine` 不改变任何东西、只是启动节点；若你手动
编辑过 `node.toml`，它会拒绝而不是覆盖你的修改，并提示改用 `run --config`。

Windows 上是 PowerShell 里的 `.\qumbra-node.exe mine --dir $HOME\.qumbra-miner`，这是
最短的原生路径——`node.toml` 由它自己写，§3 的 TOML 反斜杠陷阱咬不到你。

### 2.3 手动路径——先作为节点加入，再打开挖矿

把下载的 `genesis.qmb` 放在这份最小 `node.toml` 旁边：

```toml
data_dir = "/data"
listen_addr = "0.0.0.0:9400"
dial_peers = ["18.202.166.126:9444", "18.141.177.109:9444", "52.194.224.123:9444", "52.5.0.21:9444"]
genesis_file = "/config/genesis.qmb"
expected_genesis_hash = "d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3"
mining = false
```

这就是加入者需要的全部字段。`mining` 默认即为 false，上面写明只为显式。**不要**照抄
fleet 配置，**不要**添加 `committee_key_paths`：公众加入者是只验证节点，不持有委员会
签名密钥（[`config.rs:86`](../crates/qumbra-node/src/config.rs#L86)）。

先在绑定 socket 之前预检创世与配置，再运行：

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

`check` 走的是与启动完全相同的字节/格式/哈希闸门，只是不开监听
（[`main.rs:596`](../crates/qumbra-node/src/main.rs#L596)）。文件哈希不同会产生
`WrongGenesisHash`，节点拒绝启动
（[`genesis.rs:796`](../crates/qumbra-node/src/genesis.rs#L796)）。

用预编译二进制跑同样的加入者：配置相同，只是路径写普通形式
（`data_dir = "./qumbra-data"`、`genesis_file = "./genesis.qmb"`），然后

```sh
curl -fsSL https://seed.qumbra.org/genesis.qmb -o genesis.qmb
./qumbra-node check --config node.toml     # 同一预检，退出码 0 并打印创世哈希
./qumbra-node run   --config node.toml
```

`run` 会写入 `data_dir`，请在你拥有的目录里运行。

**NAT 靠声明，不靠探测。** 仅出站参与是设计上接受的：在 NAT 后你照常同步、挖矿、交易，
但你的地址永不被 gossip，也不服务任何对等节点。让 `advertise_addr` 保持不设。**只有**
当所宣告的主机和端口真正可从公网一路拨到这个节点时才设置它；对容器而言这还意味着发布
P2P 端口，例如 `-p 9400:9400/tcp`
（[`config.rs:61-78`](../crates/qumbra-node/src/config.rs#L61-L78)、
[`run.rs:821-826`](../crates/qumbra-node/src/run.rs#L821-L826)）。

**健康的加入长什么样**——读连续的 `TELEMETRY` 行，不要只看孤立一条：

- `peers=N` 是存活对等数：`N > 0` 说明至少一条连接活着；持续 `peers=0` 说明还没接上
  任何对等节点。
- `slag=N` 是分叉选择 tip 与已应用状态 tip 之差：加入过程中它应向 `0` 回落；`slag=0`
  表示节点已应用了自己选定的链
  （[`run.rs:1292-1308`](../crates/qumbra-node/src/run.rs#L1292-L1308)）。
- 对上面的非挖矿配置，`mready=-` 是正确读数。打开挖矿后，`unknown` 或 `behind` 表示
  启动挖矿闸门在正确地拒绝；`synced` 或 `latched` 允许挖矿。它**不能**证明 `slag=0`
  （[`run.rs:1327-1343`](../crates/qumbra-node/src/run.rs#L1327-L1343)）。

**然后打开挖矿。** 从应当收取 coinbase 的钱包生成收款身份（§1.1 第 1–3 步，或复用同一
钱包），在 `node.toml` 里只改这两个字段：

```toml
mining = true
miner_rkm = "[qumbra-wallet miner-rkm 打印的 64 个十六进制字符]"
```

启动时，找到这一行原文以证明配置生效：

```text
miner payout: coinbase notes paid to the configured miner_rkm
```

若你看到的是下面这条警告，停止挖矿并修配置：有效区块正被付给一个不可花费的占位符，
付款不可恢复（[`run.rs:838-840`](../crates/qumbra-node/src/run.rs#L838-L840)）。

```text
⚠️  NO miner_rkm CONFIGURED: ... Every coin this node mines is BURNED.
```

### 2.4 奖励、成熟期、节奏与胜率

- 赢下的区块付给你的密钥**区块补贴的 65%（含整数舍入余数）加全部交易手续费**，再减去
  该区块烧毁的名字注册费
  （[`emission.rs:85-87`](../crates/qlab-node/src/emission.rs#L85-L87)、
  [`coinbase.rs:172-185`](../crates/qlab-node/src/coinbase.rs#L172-L185)）。T2 的
  coinbase 是 v5 收款人列表；独立矿工只是它唯一的条目。
- 排放从**区块 0 起就是整数精确的**——共识路径上没有任何浮点可达，任何人都能把排放
  表复现到个位。（T1 是在链中途的边界上才挣到这条规则的，还留着一个疤痕区块；T2 生来
  就有。）
- coinbase 要**再过 144 个区块**才可花费。按目标出块约三小时，不是墙钟承诺
  （[`emission.rs:83`](../crates/qlab-node/src/emission.rs#L83)）。
- 网络目标是**每 75 秒一个区块**，每块用 **LWMA-120** 重定难度；可部署的二进制用真实
  RandomX（[`params_devnet.rs:16-42`](../crates/qlab-devnet/src/params_devnet.rs#L16-L42)、
  [`pow.rs`](../crates/qlab-devnet/src/pow.rs)）。
- 独立挖矿不是每 75 秒一份奖励。你的期望份额是你的有效 RandomX 算力除以当前全部竞争
  算力。公开面暴露的是当前难度而非全网算力，所以本指南无法诚实地报出个人胜率。预期
  方差，可能有很长的枯水期——矿池路径的存在正是为了抹平它。

### 🔴 未解决的首次启动静默

配置了 `miner_rkm` 的节点曾被观察到一次：首启时**在打印第一行日志前，以 100% CPU 静默
运行约 57 分钟**（在 T1 上观察到；代码路径未变，问题未解决——
[lab #300](https://github.com/qumbra-labs/qumbra-lab/issues/300)）。它未必是卡死：
若进程存活、一个核被打满而没有日志行，让它继续跑。重启并不修复该路径，只会丢弃已经
花掉的工作量；等第一行出现。

## 3. 平台说明——Windows 与 macOS

独立挖矿在 Linux、macOS、Windows 上均可原生运行；矿池路径（§1）对平台无感，XMRig 在
各平台都有官方构建。下面只写 Windows 与 macOS 在独立路径上*不同*的部分。

**为什么原生挖矿在 T2 上是安全的，一句话：** T2 的排放规则从区块 0 起就是精确整数
算术——共识路径上没有任何平台数学库可达，所以没有平台能算出发散的 coinbase。（这在
T1 时代是真实存在的隐患，在链中途的边界上才退役；T2 上这一类问题从出生起就被结构性
排除。）

### Windows：原生

`windows-x86_64.zip` 里是 `qumbra-node.exe` 与 `qumbra-wallet.exe`，为
`x86_64-pc-windows-msvc` 构建，用的是与其他平台完全相同的 RandomX C++ 实现。CI 在
MSVC 构建上跑 RandomX 的四个官方参考向量，所以 Windows 矿工的哈希就是全网的哈希，
不是近似。

§2 的一切原样适用——同样的 `genesis.qmb`、同样的 `node.toml` 字段、同样的种子。
不同之处：

**1. 在 PowerShell 里下载并校验。** Windows 没有 `sha256sum`：

```powershell
# 从 release 页取：对应平台的 zip 和 SHA256SUMS
Get-FileHash .\qumbra-<T2_RELEASE_TAG>-windows-x86_64.zip -Algorithm SHA256
# 与 SHA256SUMS 中对应行逐字比对打印出的哈希——用眼睛，64 个字符全部对上
Expand-Archive .\qumbra-<T2_RELEASE_TAG>-windows-x86_64.zip -DestinationPath .
cd qumbra-<T2_RELEASE_TAG>-windows-x86_64
.\qumbra-node.exe halt-status
```

`halt-status` 的读法与 §2.1 相同：`build rev:` 必须与 release notes 一致，`halt plan:`
必须是 `no halt scheduled` 而非 **ARMED**。

**2. 🔴 SmartScreen 会拦你，而且它拦得对。** 这些可执行文件**未签名**——这个项目没有
代码签名证书，买一张是一个还没人拍板的独立决定。首次运行任一二进制会弹出
*"Windows 已保护你的电脑"*。通过路径是 **更多信息 → 仍要运行**。Microsoft Defender
还可能仅凭信誉就标记 CPU 矿工。

这是诚实的立场，不是安抚：一个小项目的未签名二进制正是 SmartScreen 存在意义上要警告
的东西，而"点掉安全警告"这类建议你默认就该怀疑。让它在这里变得合理的唯一一件事是你
可以自己校验下载——**在点"仍要运行"之前对照 SHA256SUMS 校验 SHA-256**，不是之后。

**3. `node.toml` 里的路径要用单引号。** TOML 的双引号字符串把 `\` 当转义符，所以
`data_dir = "C:\Users\you\qumbra-data"` 要么解析错误、要么是另一个目录。用 TOML
*字面*字符串，或正斜杠：

```toml
data_dir = 'C:\Users\you\qumbra-data'          # 字面字符串——反斜杠就是反斜杠
genesis_file = 'C:\Users\you\genesis.qmb'
# 或在 Windows 上同样有效：
# data_dir = "C:/Users/you/qumbra-data"
```

（`qumbra-node mine` 自己写 `node.toml`，所以这个陷阱只存在于手动路径。）

**4. 从你自己打开的控制台运行，用 Ctrl-C 停止。** 打开 PowerShell 或 Windows
Terminal，在那里运行 `.\qumbra-node.exe run --config node.toml`——不要双击。
**Ctrl-C 是能可靠刷写快照的停止方式。** 关闭控制台窗口也会刷写，但 Windows 在窗口
关闭后只给程序约五秒就杀掉，正忙在一轮 RandomX 里的节点可能赶不上这个预算。

即便没赶上也不丢东西：区块日志逐条 fsync，是事实来源，错过快照刷写的节点下次启动时
重放日志、到达完全相同的状态。错过刷写付出的是**重放时间**，不是币也不是历史。

**5. Windows 上钱包种子文件不是仅属主可读。** Linux 和 macOS 上
`qumbra-wallet keygen` 以 `0600` 写 `wallet.seed`。Windows 没有这种模式，本构建也不
设 ACL，文件继承文件夹给什么就是什么——在你自己的配置文件夹下通常是你*加上* SYSTEM
和 Administrators。`keygen` 会把这一点打印出来，而不是声称一种它没有的保护。要把钱包
文件夹改成仅属主，运行一次：

```powershell
icacls "$env:USERPROFILE\.qumbra-wallet" /inheritance:r /grant:r "${env:USERNAME}:(OI)(CI)F"
```

能读那个文件的人拥有这个钱包的每一枚币。

**6. 让机器保持清醒。** 把 Windows 电源设置改为不睡眠，笔记本插上电——睡着的主机
什么也挖不到。（矿池路径同样适用。）

**本移植不包含**（写明以免有人去找）：没有 Windows 服务包装——要在注销后存活，把
`run` 命令注册为任务计划程序任务并选 *"不管用户是否登录都要运行"*，这超出本指南范围。
没有代码签名。没有 ARM Windows 构建。

### Windows：WSL2

仍然支持：管理员 PowerShell 里 `wsl --install`，然后在 Ubuntu 内用
`linux-x86_64-glibc` tarball 按 §2 走。挖矿速度实际等同原生。有了原生构建，WSL2 现在
是后备而非主路径。

### macOS

Apple Silicon 原生支持（`macos-arm64.tar.gz`）。要知道两件事：§2.1 的隔离标记/
Gatekeeper 说明（用 `curl` 获取或清除属性），以及——与所有平台一样——挖矿时让机器
保持清醒并插电。

## 4. 钱包速通——五条命令

flag 与单位以 CLI 自己的帮助为权威
（[`qumbra-wallet/main.rs:270-290`](../crates/qumbra-wallet/src/main.rs#L270-L290)）。
这五条命令覆盖最短用户旅程；替换 `RECIPIENT_QADDR`，并记住 `--amount` 的单位是
**bessel**（`100,000,000` bessel = `1 QMB`；
[`emission.rs:69`](../crates/qlab-node/src/emission.rs#L69)）。T2 上没有预注资的
捷径：**币来自挖矿**——本指南的任一路径——并在付款区块之后 144 个区块才可花费。

**Windows 上**，同样五条命令在 PowerShell 里运行，把 `qumbra-wallet` 换成
`.\qumbra-wallet.exe`；`--dir` 用 `$HOME` 或 `$env:USERPROFILE` 都行。命令 3 需要
PowerShell 等价形式，因为默认没有 `jq`：
`$TIP = (Invoke-RestMethod https://explorer.qumbra.org/v1/health.json).chain.tip_height`。

```sh
# 1 —— 创建钱包；打印地址 [0]
qumbra-wallet keygen --dir "$HOME/.qumbra-wallet"

# 2 —— 在私密终端备份 Qumbra 助记词
qumbra-wallet backup --dir "$HOME/.qumbra-wallet" --reveal

# （向这个钱包挖矿——§1 矿池或 §2 独立——并等过 144 区块成熟期）

# 3 —— 取得余额声明所依据的高度
TIP="$(curl -fsS https://explorer.qumbra.org/v1/health.json | jq -r '.chain.tip_height')"

# 4 —— 经公共节点边缘发现产出并扣除已花费的 note
qumbra-wallet scan --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --to "$TIP"

# 5 —— 重扫、构建真实证明并提交；省略 --node 时默认取 --url
qumbra-wallet send --dir "$HOME/.qumbra-wallet" \
  --url https://seed.qumbra.org --scan-to "$TIP" \
  --to RECIPIENT_QADDR --amount 100000000
```

<https://explorer.qumbra.org> 的浏览器展示的就是钱包所声明的那条链——其
`/v1/health.json` 携带链上提供的 `network` 字段
（[`qumbra-explorer/main.rs:113`](../crates/qumbra-explorer/src/main.rs#L113)），在
T2 上读作 `qumbra-t2`；若读到别的，你看的是错误网络的浏览器。
