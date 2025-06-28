# Casper FFG

[Casper FFG](https://arxiv.org/abs/1710.09437) 是 V 神为了在 ETH 上推进 PoS，参考 PBFT 和 Tendermint BFT
而设计的，V 神看好了 PBFT 的即时敲定，所以 Casper FFG 和其它的共识在区块链上的应用最大的区别是，Casper FFG 只负责
敲定区块，作为底层共识协议（PoW 或者 PoS）的覆盖协议，其他的共识协议主要是配合 PoS ，即时敲定是 PBFT 天然性质（之前不会特殊关注）。

## 原理

1. 独立于出块协议，Casper FFG 对区块进行投票 [S,T]。
2. 收集到 2/3 以上的签名赞成票。
   1. S 和 T 称作检查点。
   2. [S,T] 称作链接。
   3. S 进入 Finalized 状态，T 进入 Justified。 
   4. 下一次投票，T 进入 Finalized。

节点 Justified 和 Finalized 的递归变化，其实就是 PBFT 中三阶段的 Prepare 和 Commit 阶段的 pipeline 实现。

### 最小削减

V 神最早提出了[四条削减规则](https://medium.com/@VitalikButerin/minimal-slashing-conditions-20f0b500fc6c)
防止节点作恶，但是最后精简为两条，结果是很简洁的但是这个过程并不好理解（此处不会过多介绍）。

验证者不可以进行以下任何一种情况中 [S1, T1] 和 [S2, T2] 的投票：
1. 区块高度(T1) = 区块高度(T2),或者
2. 区块高度(S1) < 区块高度(S2) < 区块高度(T2) < 区块高度(T1)

S1 < S2 < T1 < T2，这种情况是不可能的，假设存在两条检查点路径都是正确的，
随着时间的推移，全局一定只有一个最检查点是 finalized，那么之后的路径必须从
这个 finalized 的检查点开始，那么另一条路径的检查点就不再获得 2/3 以上的
投票，无法继续了。

第一条，不可以在同一个高度投出两个不同的票（视域，消息等）。

第二条，保证了检查点的推进在全局路径是唯一的（结合 S1 < S2 < T1 < T2 是不成立的描述）。

### 安全性和活性

安全性证明：
两个相斥的区块 A 和 B 最终确定（且互不为对方的子区块）：
1. A = B，违背削减条件 1。
2. A < B，A能被确认，一定存在 [A,C] 收到 2/3 投票。
   B 能被确认一定是有一个序列都是收到 2/3 投票。
   这个序列任何一个点都不会和 A，C 相等否则违背削减条件 1。
   这个序列中一定存在一个 [Bn, Bn+1]，使得 Bn < A < C < Bn+1, 违反条件 2。

投票确认节点的 finliazed，出块可以根据地层的共识协议（PoW，PoS的最长链原则）
继续出块，不影响活性。

*注意：这里质押提取延后，最长链使用投票数最多的分支（LMD-GHOST，是投票数，不是检查点）等都没有介绍，因为重点是介绍 Casper FFG 的原理和正确性。*

## Reference
[Casper FFG：以實現權益證明為目標的共識協定](https://medium.com/taipei-ethereum-meetup/intro-to-casper-ffg-and-eth-2-0-95705e9304d6)
[什么是CASPER FFG](https://blog.csdn.net/qq_40713201/article/details/124691252)
