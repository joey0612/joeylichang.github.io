# Fast HotStuff

## Ovewview

Fast HotStuff 主要特点有两个：
1. 三轮投票变两轮，沿用[HotSuff-2](./hotstuff2.md) 的分析方式后续分析 Unhappy Path 使用什么方式解决。
2. 解决分叉攻击，[HotStuff](./hotstuff.md) 的 safeNode-predicate 规则要求在 LockQC 之后追加新的 proposal，并不是 HighQC 这就给恶意节点留下了分叉攻击的机会（恶意节点在 LockQC 而非 HighQC 追加proposal），这不会造成安全问题，但是确实影响了整个系统的吞吐。

基于 Pos 共识的区块链中，除非 bug 导致，恶意的分叉攻击会有严重的惩罚，我们接下来重点分析第一点。

## Model Analysis

Fast HotStuff 在 Unhappy Path 的情况下，使用了类似 PBFT 的解决方案，收集 2f + 1 个节点的 HighQC 并且新的提案会带有这 2f + 1 个节点的 HighQC 的证明（AggQC），让全网统一 view，如前对于 PBFT 的介绍，view change 阶段可能会替换"隐藏锁"，这也是 AggQC 的作用（详细举例可以见引用链接，不在此赘述）。

Fast HotStuff 引入了聚合签名减少了 AggQC 这步的通信复杂度，但是他的通信签名，验签等依然存在不小的复杂度，在大规模节点上依然存在瓶颈。

## Reference
[Fast HotStuff 理解与思考](https://www.xufeisofly.xyz/blog/fast-hotstuff)