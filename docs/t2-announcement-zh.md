# Qumbra T2：上线了什么，退役了什么，你该做什么

English: [`t2-announcement.md`](./t2-announcement.md) · **技术细节以英文版为准。**

> **T2 已于 2026-08-20 14:00 UTC+8（06:00 UTC）上线。** T1 已退役。这是公告的长版本：
> 新网是什么、旧网怎么了、如何加入。逐步操作见
> [`join-and-mine-zh.md`](./join-and-mine-zh.md)。T1 公告作为记录保留，不删除：
> [`t1-announcement-zh.md`](./t1-announcement-zh.md)。

**T2 是什么。** T2 跑的是 v5 链格式——T1 存在的意义就是挣来它：

- **收款人列表 coinbase。** 区块奖励可以在共识内拆给一串收款人。这是免托管矿池的
  基石：矿池从不经手你的币，区块本身直接付给你。
- **stratum 兼容区块头。** 97 字节的头是照着 stock XMRig 的习惯排布的——nonce 恰好
  落在 XMRig 要写的位置，share 判定读的恰好是 XMRig 读的那几个字节，一行矿机代码
  都不用改。
- **第 1 块起的整数精确排放。** T1 用浮点排放表启航，在链中途的边界才挣来整数精确
  规则（在规则存在之前还有一个块少付了 4,114 bessel——那道疤在 T1 的档案里）。T2
  从第一个块起就是精确规则，总量算术任何人可复算到个位。
- **第 0 块原生的名字服务。** 短人名（`alice.qmb`），commit–reveal 注册（抢注在结构
  上不可能）、按长度分档燃烧费（1/32/128/512/2048 QMB）、写一次不可改不可转——
  没有可偷的东西。在 T1 上它要等到 19,008 块；T2 生来就有。那条边界在这里不会发生。

**T1 怎么了。** T1 是排练网，而它完成了使命：横跨三大洲的 WAN 浸泡、一次活体排放
边界修复、第一个真实用户旅程、七个被真钱找出并当天修完的缺陷，以及整个 v5 格式在
它身上的验证。T1 已于 **2026 年 8 月 20 日（周四）14:00 UTC+8**（06:00 UTC）停机；
历史已归档，不是删除。**T1 余额不迁移**——T2 是公平重启：无预挖、无继承，第 0 块
对所有人是同一条起跑线，包括我们自己。

**如何加入。** 两条路——细节在 [`join-and-mine-zh.md`](./join-and-mine-zh.md)：

- **矿池（不用跑节点）：** 装 stock XMRig，指向 `pool.qumbra.org:3333`，填一个
  Qumbra 收款地址（加入指南教你建钱包）。share 按 PPLNS 记账，出块奖励由 coinbase
  直接付给矿工名单。
  **矿池在切换之后才启用**——一个看起来活着、却对着假模板发任务的 stratum 会烧掉
  矿工算力，所以在本仓库后续说明它已上线之前，该主机名不是付款端点。配置写在加入
  指南里，为的是那一刻只需粘贴一次。
- **solo（自己跑节点）：** 下载、核对下方创世哈希、`qumbra-node mine`。指南覆盖
  Linux、macOS、Windows。
  **切换时刻没有 T2 二进制。** T2 release 在链上线**之后**才切割（此前闸门拒绝）。
  在 <https://github.com/qumbra-labs/qumbra/releases> 出现 T2 tag 之前，没有任何
  下载能跟随这条链。每一个 `t1-*` tag 都是 T1 产物——不要拿它连 T2。

钉住这个值，别信其它：

| 项 | 值 |
|---|---|
| 网络名 | `qumbra-t2` |
| 创世格式 | v5 |
| `expected_genesis_hash` | `d1dad4ea2bc5bfc4880ecf25206d182cddeacc12b0f65eca1a1ce2f27a93e2f3` |
| 创世文件（切换后） | `https://seed.qumbra.org/genesis.qmb`——对照哈希逐字节校验 |
| T2 二进制 | 切换**之后**落在 <https://github.com/qumbra-labs/qumbra/releases> |
| 矿池（stock XMRig） | `pool.qumbra.org:3333`——路径已写明，**暂缓**至宣布上线 |

镜像 digest、P2P 种子、T2 release tag 不在本页，因为它们在 14:00 还不存在。存在时
会填进本仓库——不会猜。

**没变的部分。** 随 T1 公告的设计就是 T2 跑的设计：构造上后量子（哈希基 STARK 证明、
ML-KEM 笔记加密、全栈无椭圆曲线）、单一屏蔽池、148,625 字节交易且笔记本秒级出证、
密钥级选择性披露而非协议后门。spec 与验证电路照旧公开在本仓库
（[白皮书](spec/whitepaper.md) · [协议规范](spec/protocol-spec.md) ·
[冻结共识参数](spec/consensus-parameters.md)）。

**诚实登记，精神不变。** T2 仍是测试网：终局委员会 21 把钥匙由运营者持有（向外部
签名者开放是独立里程碑，弱主观性检查点设计已裁定待建）；标注 testnet-tunable 的参数
仍可在声明的边界处调整；在这两件事不再为真之前，“主网”一词不启用。公平出发——而
以上每一句的凭据都在本仓库的公开 spec 里，不在一篇帖子里。

发现坏东西？[在这里提 issue。](https://github.com/qumbra-labs/qumbra/issues)
