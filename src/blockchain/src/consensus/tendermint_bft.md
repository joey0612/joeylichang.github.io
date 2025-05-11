# Tendermint BFT
![tendermint_bft](../../images/tendermint_bft.png)

## 结论
Tenderment BFT 与 PBFT 相比最大的特点是不会产生分叉，极限情况下会卡住在某一个 Block 不出块。

## 流程
与 PBFT 同属于 BFT 系列，仍然使用三阶段，如上图所示：propose，prevote，precommit 类比 BFT 的 pre-prepare，prepare，commit 三个阶段，第一个阶段都是广播消息，后两个阶段都是广播签名进行验证，保证最终出块一致。

巩固：第二个阶段确保收到的消息相同，避免因为切主导致同一个 epoch 和 index 位置的 message 不同。因为第二阶段之后切主，新 leader 会从第一阶段开始此时之前第二阶段的签名就是有用的，第三阶段是确认第二阶段。

1. Propose

    1. roposer 出块并向所有 validator 发送 block x 消息。 
    2. Proposer 采用轮训机制，validator 是根据质押的资产决定.
    __注意：后面所有轮次的投票权重（2/3）都是指质押资产的维度，并不是节点个数。__

2. PreVote

    1. validator 收到 block 消息后向全部其他 validator 发送 prevote block x 消息。
    2. 超时或者block失效（校验失败），此时可以 prevote nil。
    3. 当收到 +2/3 投票（prevote 消息）之后，向其他 validator 发送 precommit 消息。
    4. **锁定机制（重点）:**
        1. 如果收到 +2/3 prevote 投票，此时当前 validator 锁定 block x（锁定信息：block-消息，x-高度，r-轮数），+2/3 prevote 投票组成 PoLC（proof-of-lock-change）。
        2. 如果在 precommit 阶段没有达成共识，会重复这三个阶段，每次循环都用轮数 r 记录。
        3. 如果 block 被锁定，那么下一轮无论 proposer（应该会换新的）发布什么 block y，当前节点只能 prevote block x。如果 validator 在轮次 R1 被锁定，在轮次 R2 收到了其他块的 PoLC，则它可以在轮次 R3 解锁，R1<R2<R3。

3. PreCommit
    1. 该阶段只能 precommit 锁定的 block，此时可能 precommit nil（PoLC nil 情况下）。
    2. 收到 +2/3 以上的投票（precommit 消息）则对block 进行 commit。
    3. 其他情况（未达到共识）维持锁定状态并  precommit nil。


### 理解锁定机制

锁定机制是保证不分叉的协议保证，保证了只要 +2/3以上节点认证了 block x，在下一个轮次中一定还是提交该消息（类似 Paxos）。

#### 两个轮次以上可以解锁

**为什么不是一轮？**

因为解锁阶段在 propose 阶段也就是发送 prevote 之前不能确定前一轮的 block 一定失效。

**为什么是两轮以上？**

因为经过两轮针对当前锁定的消息都没有达成共识说明经过多次尝试全网都没有达成共识，之前 +2/3 以上的认证应该因为某些原因失效了（节点异常导致无法重新达成 +2/3 以上共识）。

### 换主

正常情况下就是根据质押进行轮换（这里还会设计 vote power计算），在异常情况下通过 prevote nil 和 procommit nil 使主流程正常运行保证 Proposer 进行切换。

### Commit 失效原因
如果某轮没有成功 commit 被提议的区块，那么就需要进行一个新的轮次，可能造成 commit 失败的原因有：

- 该轮指定的 proposer 离线
- 该轮 proposer 提议的 block 无效
- 该轮 proposer 提议的 block 未能在规定时间内广播出去
- 该轮 proposer 提议的 block 有效，但是2/3+的 validators 在进入 pre-commit 阶段时没能及时收到其他节点足够的投票
- 该轮 proposer 提议的 block 有效，且2/3+的 validators 及时收到了其他节点足够的投票，但是对区块的 pre-commit 未能及时被有效数量（2/3+）的节点接收到

## Tendermint BFT VS. PBFT

### 相同点

1. 同属 BFT 体系
2. 可抗 1/3 拜占庭节点攻击
3. 三阶段提交，第一阶段广播交易（区块），后两阶段广播签名（确认）
4. 两者都需要达到法定人数才能提交块

### 不同点

1. 拜占庭节点概念不同，PBFT 指的是节点数，而 Tendermint 代表的是节点的权益数，也就是投票权力。
2. PBFT 需要预设一组固定的验证人，而 Tendermint 是通过要求超过2/3法定人数的验证人员批准会员变更，从而支持验证人的动态变化。
3. 容错情况优于 PBFT
    
    1. 当拜占庭节点数量在验证者数量的1/3和2/3之间时，PBFT 算法无法提供保障，使得攻击者可以将任意结果返回给客户端。
    2. 而 Tendermint 只有达到 +2/3 投票才能出块，所以只有达到 +2/3 才会出现作恶节点，否则不出块。
    3. 简单说，PBFT +1/3 就可以分叉，Tendermint BFT +2/3 才可以分叉，1/3 -2/3 之间最多时阻止出块。