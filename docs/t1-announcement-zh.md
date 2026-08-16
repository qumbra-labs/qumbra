# Qumbra T1——公开挖矿开放

English: [`t1-announcement.md`](./t1-announcement.md) · **技术细节以英文版为准。**

> **已发布 2026-08-16。** 若发布后最初几分钟内种子连接被拒,说明门正在开——防火墙变更
> 紧随本页落地;稍候重试即可。

Qumbra 是一条后量子隐私链:单一屏蔽池、每笔交易一个 STARK、RandomX 系 CPU PoW 无许可出块、
BFT finality 委员会为链盖检查点。T1 是它第一张面向公众的测试网。**现在任何人都能跑节点、
挖矿——不注册、不审批,有 CPU 就够。**

## 定义这张网的五个值

| 项 | 值 |
|---|---|
| 节点镜像(按 digest,永不用 tag) | `ghcr.io/qumbra-labs/qumbra-node@sha256:c7b8b3340d35d7461daaa83acea6a8eef045bdba74d5173f9f059322ec18adbb` |
| 镜像 revision(OCI 标签,务必回读) | `e0b624596e29dc8a95d1f6715ed061b4247a73e2` |
| genesis 文件 | `https://seed.qumbra.org/genesis.qmb`(格式 v4) |
| `expected_genesis_hash` | `138e1524ba889bd49644f0eeafafa53533584caa2c0c851330cd27965223addb` |
| P2P 种子(`dial_peers`) | `18.202.166.126:9444` · `18.141.177.109:9444` · `52.194.224.123:9444` · `52.5.0.21:9444` |

**逐步操作:[`join-and-mine-zh.md`](./join-and-mine-zh.md)。** 短版:按 digest 和 revision
标签验证镜像,按哈希逐字节校验 genesis,先以非挖矿节点加入,看 `slag` 落到 0,再配上
`qumbra-wallet miner-rkm` 打印的 `miner_rkm` 并置 `mining = true`——**不配 `miner_rkm`,
挖到的一切都会被烧掉**,节点启动时会明说。挖矿在节点内、solo;没有独立矿工程序也暂无矿池
(两者都是刻意的——见设计仓的矿池简报)。

## 公开服务

- `https://seed.qumbra.org`——钱包发现边缘(`/v1/compact`、`/v1/coinbase`、
  `/v1/nullifiers`、`/v1/tree/leaves`、`POST /v1/tx`)及 genesis 下载
- `https://explorer.qumbra.org`——链健康(tip、finality、委员会、精确供应审计)
- `https://faucet.qumbra.org`——你的 coinbase 成熟前可先领币试交易

## T1 是什么,说实话

- **币没有价值**,且不会活过下一次 re-mint:再铸已在排期(收款人列表 coinbase + stratum
  兼容 header 布局搭下次创世)。挖矿是为了操练系统,不是为了积累。
- **finality 委员会目前是中心化的**——21 把钥匙都在运营者手里。今天无许可的是出块;
  委员会去中心化是后面的阶段。
- coinbase 成熟期 **144 块**(按 ~75 秒目标约 3 小时);钱包 `scan` 明确区分可花与成熟中。
- 现行代码下从 genesis 同步只要几分钟(checkpoint-sync + 增量树);笔记本级 CPU 就能有效
  参与——fleet 本身只是四台普通云主机。
- 完全支持只出不进的参与(NAT 之后):照常同步、挖矿、交易,只是不为别人服务。
- seed 边缘能看到你的钱包请求了哪些区块范围(带宽模式侧信道,经 Cloudflare 代理)。门控中
  的未来答案是 OMR 覆盖层;在那之前如实说明,不加掩饰。

发现坏东西?T1 就是为此存在的——[在这里提 issue](https://github.com/qumbra-labs/qumbra/issues)。
