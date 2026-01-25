# HotSuff-2

## Overview

HotSuff-2 基于 [HotSuff](./hotstuff.md) 做了轻量的改动（实现复杂度）将三阶段变成了两阶段。在 Happy Path 上仍然保持了 HotSuff 的响应性，在 Unhappy Path 上则由于引入的了等待超时丧失了原本 HotStuff 的响应性，HotSuff-2 将这点统称为“乐观响应性”。在多数情况下（网络无异常）两阶段的 finalized 只需要 2 轮投票集合敲定。总结：**HotSuff-2 牺牲了 Unhappy Path 的响应性，换来了 finalized 敲定速度**。下面的分析就是理解这个的过程。

## Model Analysis

要想理解 HotSuff 和 HotSuff-2 本质的区别，就需要明白三阶段和两阶段的本质区别，在此基础之上去理解 HotSuff 和 HotSuff-2 的深层逻辑。

[HotSuff](./hotstuff.md) 之前介绍过新引入一个 PrepareQC 用于线性 view change。[Partial Synchronization](./partially_synchronization.md) 介绍了响应性。对于两阶段和三阶段的对比又了一些了解，我们继续从另外一个侧面了解两阶段和三阶段。

如果领导者发生故障或网络停滞，我们可能会陷入下图所示的三种情况之一：
1. 没有节点持有锁，更不用说有值被提交了。
2. 一个或少数几个方锁定在某个值上，但没有诚实方已提交。
3. 2t+1 个节点被锁定，一些诚实方已提交，但并非所有诚实方都已提交。

1 并不理想，但所有方在随后的视图中都可以自由地对其他值进行投票，所以不会影响活性，在协议设计中是简单场景常常不需要讨论。2 和 3 才是难点，因为 LockQC 之后，无法确定是否 CommitQC，如果是 CommitQC 那么这个提案必须全网共识，否则可以不用共识而被新的提案替换。但是 2 和 3 对于新的 leader 来说确认它就是难点，这也是二阶段和三阶段的本质，三阶段通过 PrepareQC 就可以保证全网看到了，然后新 leader 基于 PrepareQC 就可以继续（HotStuff 新 proposal 就是基于最高的 PrepareQC），而两阶段就会存在隐藏锁（新 leader 未必能在 2f+1 中获取最高 PrepareQC）的情况，导致安全性和活性收到破坏，[HotStuff View Change 3-Phase](./hotstuff.md#view-change)有介绍。

所以针对针对 2 两阶段中的隐藏锁的问题，每种共识算法有自己的解决方案：
* PBFT

1. 收集 2f+ 1 个 LockQC，针对情况 3 一定能拿到最新的 LockQC（第一轮投票已经被全网共识了）。
2. 针对情况 2 可能拿不到最新的 LockQC（只有节点 Lock 但是网络不佳没有给新 leader）。
3. PBFT 会发送 2t+1 里面最高的 LockQC 让诚实的锁定节点替换新的提案。
4. 需要 2f +1 节点的证明在给全网，复杂度就会 O(n^2)（n 个证明），再加上每一个超时的节点都会发起 view change 所以最终的复杂度回事 O(n^3)——**复杂度高**。

* Tendermint BFT
1. 要求领导者在视图开始时等待 O(Delta) 时间。在 GST 之后，O(Delta) 的等待确保领导者收到所有诚实方的锁——**线性 view change**。
    1. 为什么需要等待所有节点的 LockQC? 防止前述的“隐藏锁”，存在情况 3 部分节点已经 Commit 的情况。
2. 它可以发送一个符合最高视图锁对应值的提案，并将此锁与提案一起发送。
3. 获得了视图切换的线性通信复杂度。然而，由于 O(Delta) 的等待——**协议丧失响应性**。

* HotSuff
1. 从接受 2f + 1 个 PrepareQC 开始，响应性和复杂度都很好，但是增加了一个 PrepareQC 阶段。

## HotSuff-2 Model

HotSuff-2 在 HotSuff 的基础上借鉴了 Tendermint BFT 的设计。
1. 三阶段改成两阶段，没有 PrepareQC 阶段，就是 LockQC -> CommitQC 两轮投票。
2. Happy Path，新 leader 收到了“法定数量数量的投票”，然后继续以线性的方式继续（与 HotSuff 流程一样）。
3. Unhappy Path，新 leader 知道前一个视图中必定已经过了一个 O(Delta) 的定时器延迟（view change 的原因是前面某个流程超时）。在这种情况下，反正也没有响应性了，因此它在保证所有方进入视图后，再等待额外的 O(Delta) 延迟（类似 Tendermint BFT 的方式），以获得系统中的最大锁定值。

“法定数量数量的投票” 这点其实是比较值得讨论的，他与 HotSuff 的 2f+1 PrepareQC 是不同的，因为 2f+1 不能解决隐藏锁（在两阶段情况下）的问题，所以这个“法定数量数量的投票”在工程实践中是可以根据实际情况变动的，比如 66%，80%，100% 等等，都是通过概率来解决实际的工程问题，社区中很多项目也针对这块有不同的实现。 

## Summary

HotSuff-2 并没有新的创新，而是基于观察做出的简单改动获取了大的效果，HotSuff-2 和 Tendermint BFT 更像是理论完备的协议，但是是工程实践和线上运行的实际情况下，Unhappy Path 毕竟不算事经常出现，所以多数情况下 2 轮投票足够，及时出现了网络异常 HotSuff 等待 2f+1 的 PrepareQC 可能也是一个系统不可接受的时间（所以才有 TC 的工程方案），此时不如就使用 Tendermint BFT 的等 O(Delta) 的方案。

## Reference
[HotStuff-2: Optimal Two-Phase Responsive BFT](https://eprint.iacr.org/2023/397.pdf)

[What is the difference between PBFT, Tendermint, HotStuff, and HotStuff-2?](https://decentralizedthoughts.github.io/2023-04-01-hotstuff-2/)

[HotStuff-2phase详解](https://blog.csdn.net/ganzr/article/details/144076516)