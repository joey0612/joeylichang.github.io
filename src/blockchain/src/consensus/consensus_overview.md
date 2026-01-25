# Consensus

# Overview

区块链的 permissionless 特性使得业界将目光抛向了开放类共识，[Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport) 是最早研究分布式共识的奠基人，他的三篇经典论文被再次被讨论，其中[Paxos](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/paxos/paxos.md) 作为非开放类共识协议在 Web2 领域被充分研究，经过研究和实践因输出经典的三阶段，两阶段，[Raft](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/raft/overview.md)，甚至[Gossip](https://github.com/joey0612/joeylichang.github.io/blob/master/src/distributed_protocol/gossip/overview.md) 等共识协议（被业界公认所有的共识协议均出自 Paxos）。

Leslie Lamport 在开放类共识提出了拜占庭将军问题（再次基础上进行了一系列研究，[FLP](./flp.md) 就是其中之一），再次基础上提出了 PBFT, 区块链的共识协议正式从 PBFT 开始，为了解决 View Change 复杂度提出了 Tendermint BFT，HotStuff 是第一个针对区块链的开放类共识协议，通过引入三阶段（之前都是两阶段）+ 阈值签名（后面引入聚合签名）实现了区块链场景下安全性与活性解耦，线性复杂度，响应性等区块链特性。

以太坊为了保持去中心化（哪怕只剩下一个节点也能恢复过来）打破了 BFT 假设（恶意节点少于 1/3）将共识协议分层，并相互配合。其中 Finality Rule 的研究中 Vitalik 提出的“弱主观性”和“最小削减条件”成为经典，将 BFT 类共识从共识区块（交易排序）转向“即时敲定”，推动业界其他区块链 finalized block 缩减到 2-3 个左右，让用户可以更快的确认交易上链（而不是之前的最长链原则）。

# BlockChain Consensus

* [PBFT](./pbft.md)：开放共识的基础协议，深刻理解两阶段，对等广播模型，异常情况下 view change 等时学习后续其他共识协议的基础。

* [Tendermint BFT](./tendermint_bft.md)：解决了 PBFT View Change 复杂度过高的问题，实现了线性复杂度，深刻理解 NewHight，NewRound 引入的必要性和含义，对 Tendermint 的特点有更深刻的理解。

* [HotStuff](./hotstuff.md)：推进了区块链共识的进程，通过三阶段的引入（之前都是两阶段），完成了安全性与活性解耦，正常/异常情况下线性 view change，比 Tendermint 有更好的响应性。

* [Partially Synchronization](./partially_synchronization.md)：部分同步网络是最符合现实情况的网络模型，目前流行的共识协议多数都是基于部分同步网络进行设计的，总结前三篇基础协议如何体现“部分同步“网络，并且结合以上基础，针对 View Change，复杂度，响应性等进行了讨论。

* [HotStuff2](./hotstuff2.md)：基于经典的三阶段 HotStuff 变两阶段的优化，在 Unhappy Path 采用了类似 Tendermint BFT 的做法，基于现实情况的观察，做出的简单改动获取了大的效果。

* [Fast HotStuff](./fast_hotstuff.md)：同样基于经典的三阶段 HotStuff 变两阶段的优化，Fast HotStuff 在 Unhappy Path 的情况下，使用了类似 PBFT 的解决方案，并且解决了分叉攻击。

* [Gasper](./eth_gasper.md)：以太坊的共识协议，应该是集大成者，打破了 BFT 假设的共识协议——哪怕只剩下一个节点也能恢复过来——，将共识协议划分为：Propose Rule（POS），Fork Choice Rule（LMD GHOST），Finality Rule（Casper FFG），其中 Vitalik 在 Finality Rule 的研究中提出了两个非常经典的理论：弱主观性和最少削减条件，使其与 Fork Choice 的配合打破了 BFT 假设（即恶意节点可以多于 1/3）。


