# BSC Validator Output Block Flow

## Workflow

1. 导入父块：收到并验证父块,设为链头,触发出新块。
2. Prepare 组头：造 header(coinbase / 难度 / gasLimit / T)、开父状态、初始化 gas 池。
3. 打包执行：从 mempool 取交易执行,累积 receipt/gas,prefetch 预热，硬限制 T-15ms。
4. Finalize：停止打包,跑系统交易 + 算 state root + 组装块（15ms 之内）。
5. Seal：validator 私钥签 header,写进 extraData。
6. 广播：把封好的块发给 peers,约 T 到达（走 EVN，有概率会占用“导入父块”的时间）。

`0 → 435ms`：导入父块 + Prepare + **打包执行**(一整段连续,导入/Prepare 只占很小头部)。
`435ms → 450ms`：Finalize + Seal + 广播。

## Builder

在继续介绍后续的打包出块流程之前，需要先了解 Builder 的 PBS 架构：

| 角色 | 职责 | 链上收入 |
|---|---|---|
| **searcher** | 发现 MEV 机会(套利/清算/夹子…),为抢先付费 | 捕获的 MEV − 付出的费用 |
| **builder** | 聚合 searcher 的 bundle、排序、(bidblock 还要执行)交出块 | **BuilderFee**(validator 付的服务费) |
| **validator** | 选最优块、签名、广播 | 块全部 gas 费 + MEV 付款 − BuilderFee − 系统税 |
| **sentry**(bsc-mev-sentry) | 夹在 builder 和 validator 之间,转发 bid、构造 payBidTx | — |

Builder 注册是链下注册，非共识。目前的机制是：
1. **注册**:链下、**每个 validator 各自配置**接受的 builder 列表(`Mev.Builders []{Address, URL}`);另有官方公示库 [`bnb-chain/bsc-mev-info`](https://github.com/bnb-chain/bsc-mev-info)(`mainnet/builder-list.toml`)做发现。身份靠**签名**认证(`EcrecoverSender` → `ExistBuilder`)。
2. **质押**:**无**。是白名单许可模型,不是经济抵押。
3. **惩罚**:**吊销权限,非链上罚没**。bidblock 无效 → 封 24h(`bidBlockRevokeDuration`);gas 价过低 → 封 1 epoch(`bidBlockGasPriceLowRevokeDuration=450s`);可手动吊销。另有限流 `MaxBidsPerBuilder`。

### Builder 经济模型

#### Basic Model
```
searcher/用户交易的 gas 费 + 为抢先给 coinbase 的 MEV 付款
        ↓ 全进 coinbase = validator = GasFee(整块收益)
validator 的 payAccount ──BuilderFee(≤ commission%×GasFee,gasPrice=0)──→ builder
        ↑ 这笔就是 payBidTx,由 sentry 追加在块尾
```

- **GasFee** = 块交给 validator coinbase 的总收益(`bidder.go` 里 = `state.GetBalance(SystemAddress)`)。
- **payBidTx** = **sentry 的 `payAccount` → builder**,`Value = BuilderFee`,`GasPrice = 0`,追加为块最后一笔([bsc-mev-sentry `node/validator.go` GeneratePayBidTx](https://github.com/bnb-chain/bsc-mev-sentry/blob/main/node/validator.go))。**方向是付给 builder**。

#### Builder Fee

| 角色 | 净收益 |
|---|---|
| **validator** | `GasFee − BuilderFee`(≥ 99% 的块收益) |
| **builder** | `BuilderFee`(≤ 1% 的块收益) |

当然，Builder 具备打包交易的排序权，有能力进行套利（Builder 的充分竞争可以抑制恶意套利）。

- **builder 自己声明** `RawBid.BuilderFee`(想收多少填多少;参考实现 bsc-builder 里还是 TODO=0)。**没有 validator 侧计算公式**。
- 两道上限:
  - **绝对**:`BuilderFeeCeil`(sentry 校验,超了拒,[bsc-mev-sentry `service/sentry.go`](https://github.com/bnb-chain/bsc-mev-sentry/blob/main/service/sentry.go))。
  - **相对**:`≤ commission% × 块收益`(validator 选标时 `validatorReward<0` 则拒,见下)。

- **builder 自己声明** `RawBid.BuilderFee`(想收多少填多少;参考实现 bsc-builder 里还是 TODO=0)。**没有 validator 侧计算公式**。
- 两道上限:
  - **绝对**:`BuilderFeeCeil`(sentry 校验,超了拒,[bsc-mev-sentry `service/sentry.go`](https://github.com/bnb-chain/bsc-mev-sentry/blob/main/service/sentry.go))。
  - **相对**:`≤ commission% × 块收益`(validator 选标时 `validatorReward<0` 则拒,见下)。

**为什么是 1%(ValidatorCommission)**
`ValidatorCommission` 默认 `100/10000 = 1%`([config.go:55](miner/minerconfig/config.go))。它是 **validator 愿意分给 builder 的最高比例 = BuilderFee 的相对上限**:
```go
validatorReward = blockReward × commission/10000 − BuilderFee   // bid_simulator.go:1293/1329
if validatorReward < 0 { reject }                               // ⇔ BuilderFee ≤ 1% × 块收益
```
所以 **builder 最多收走块收益的 1%,validator 至少留 99%**。validator 可调高(愿多分抢好块)或调低(自己多留)。

> 注:`validatorReward = blockReward×1% − BuilderFee` 是**选标/卡上限的度量**,不是 validator 实际净收益(实际是 `GasFee − BuilderFee`,≥99%)。它的用途:① 卡 BuilderFee ≤ 1%;② 给候选 bid 排序(同收益下 BuilderFee 越低越优)。

**validator 如何选最优 bid(为什么不直接比 gasFee)**
```
① bid 总块收益 > 本地块收益          ← 直接比 gasFee
② bid 的 validatorReward > 本地的      ← 扣掉 builder 抽成后的"净收益"闸
```

为什么光比 ① 不够?

gasFee 里有一块要付给 builder(BuilderFee),只看头条会被"高总额 + 高抽成"忽悠——反例(commission=1%)进行排序。

## Bid

Builder 向 Validator 发送的是 Bid，所以这里针对 Bid 进行详细的介绍。

1. 提交：builder 提交`mev_sendBid`,`BidArgs`(交易列表 + GasFee 承诺 + BuilderFee),签名。
2. 准入校验：validator  `EcrecoverSender` → `ExistBuilder` → `ReservePending`(限流)
3. 处理： validator 丢弃过期;与"当前最优待跑 bid"比 `expectedBlockReward`,更优才考虑打断。
4. 打断判定：validator 判断是否 `canBeInterrupted`（T-135ms = 315ms 是否到），剩余 `≥ NoInterruptLeftOver(~135ms)` 才打断当前模拟。
5. 模拟：**本地重放** bid 交易(不信 builder 报的收益),prefetch,算真实收益 
6. 贪婪合并（看时间是否够用，可选）：bid 交易跑完后用 mempool 交易填满剩余 gas,截止 `T−25ms`，如果时间超过了（T−25ms=425ms 则不会贪婪）。
7. 收尾：追加 payBidTx → `packReward` → 与 `bestBid` 比,更高则更新 `SetBestBid`
8. 封块决策： T−15ms时刻，取 `GetBestBid`(**已模拟完**的最优)vs 本地 PK,采纳则 seal

| 相对时刻 | 事件 |
|---:|---|
| `0 → 315ms` | 主窗口:收 bid、逐个模拟;更优 bid **可打断**当前模拟 |
| `315ms`(T−135ms) | **打断闸关**:此后新 bid 只记录、不打断在跑的模拟 |
| `415ms`(T−35ms) | BidBetterBefore:bid 最好在此前到 |
| `425ms`(T−25ms) | GreedyMerge 收尾截止(`delayLeftOver+10ms` buffer) |
| `435ms`(T−15ms) | BidMustBefore:停止收 bid;取最优已完成 bid vs 本地,选定,Finalize |
| `435→450ms` | 15ms 预留:Finalize + Seal + 广播 |


说明：
1. 315ms：315ms 后若模拟器空闲,新 bid 仍会开新模拟;真正"不模拟/不上链"的硬线是 435ms。
2. 事件驱动、无自触发：mainLoop 跑完一个模拟不会主动去拾起 bestBidToRun;待跑的 bid 要等下一个 bid 到达才被踢起来。所以可能出现"有时间、有待跑 bid,却因无新事件而空闲"的缝隙(繁忙 validator 上 bid 密集,影响小)。
3. GreedyMerge 不是独立阶段,是每个 bid 模拟内部的收尾,以 T−25ms 为界;那个 bid 执行本身超了 425ms 就跳过 GreedyMerge。

* T-135ms 时刻正在模拟 bid 不会被打断，但是后面进来的 bid 会执行吗？

如果正在模拟的 Bid A 在 T-15ms 之前执行完，只有一个更优的 Bid B 在 [T-135ms, T-15ms] 之间进来，不会执行因为上述第二点。如果又有一个 Bid C 进来，那么在 Bid B 和 Bid C 中选择一个最优进行模拟。 如果在 T-15ms 之前执行完并且正确，那么 [T-135ms, T-15ms] 之间来的 Bid B 或者 C 会上链，如果没有执行完也不会管它了，直接用 Bid A 上链接。在流量高峰期间 T-135ms 是一个非常关键的点位，在这个时间点之前越接近的位置打进来一个收益更好的 Bid 来打断正在执行的 Bid 开始模拟大概率会获胜，因为剩余的 120ms（135ms - 15ms）没有足够时间模拟两个以上的 bid。 在流量低峰期间（120ms 可以2个 bid 以上） T-135ms 到 T-15ms 是一个关键的时间段，需要在这期间打尽来至少 2 个 bid 并能在 T-15ms 之前执行完。不过流量低峰可能没有那么多 bid，所以整体上 T-135ms 是最重要的博弈时间点。

## BidBlock

BidBlock 是 builder 交"已建好的完整块",validator 不执行、直接采纳、签名后先广播、之后再异步验证。最快,靠信任 + 封禁兜底。

因为不模拟,没有 NoInterrupt(315)/ BidBetterBefore(415)/ GreedyMerge(425)那些窗口,**只有一个硬截止 `bidMustBefore = T−15ms`**。

| 相对时刻 | 事件 |
|---:|---|
| `0 → 435ms` | 持续收 bidblock;**每个到达即同步 preSeal 校验**(头+tx root+GasFee,不执行),存最优(按 GasFee) |
| `435ms`(T−15ms) | 硬截止,过后 "too late" 拒;worker 取最优 bidblock,和 simBid + 本地三方 PK |
| `435→450ms` | bidblock 胜:lane quota + 签系统交易 + 签 header → **先广播** |
| 广播之后(异步) | `InsertChain` 验证;失败 → 封 builder 24h |

**含义**:bidblock 不需给 validator 留模拟时间 → **builder 可拖到接近 T−15ms 才提交**(信息更全、延迟更低),代价是自己扛执行 + 出错被封。