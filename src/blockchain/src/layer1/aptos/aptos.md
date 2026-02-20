# Aptos

## Overview

Aptos 是一个 Layer 1 区块链，由 Meta（前 Facebook）的 Diem 项目核心团队成员创建。它使用 Rust 语言编写，采用 Move 智能合约语言，旨在通过更好的技术和用户体验为 Web3 带来范式转变。核心亮点：

1. Pipeline：共识和执行分离，流水线并行处理。
2. Quorum Store：提前传播交易，共识只需要对顺序达成一致，不需要再传输交易，避免[Double Transmission](../sui/sui_consensus.md#double-transmission).
3. Block-STM：并行执行引擎，充分利用多核 CPU。
4. MoveVM：Move 语言提供资源安全性，和对象级别的并发粒度。

理论 TPS 可达 160,000+，实际生产环境可达数万 TPS，亚秒级交易确认（通常 < 1 秒）。

## Conensus: AptosBFT

AptosBFT 本质上是 [HotStuff-2](../../consensus/hotstuff2.md) 的工程实践版本，采用 2-chain 提交规则替代传统 HotStuff 的 3-chain，在保持 Happy Path 响应性的同时，通过牺牲 Unhappy Path 的响应性换取更快的 finalized 速度。接下来介绍一下 AptosBFT 在 HotStuff-2 的工程实践中一些要点。

#### View Change

HotStuff-2 与 HotStuff 的本质区别是在 view change 的时候分成 Happy Path 和 Unhappy Path，在 Happy Path 下收到 2f+1 投票的时候继续轮换 leader，在 Unhappy Path 的情况下需要等待 O(Delta) 收集更多的信息，AptosBFT 在工程实践的时候针对 Unhappy Path 使用了常见的 TC，下面具体看一下。

1. 本地超时触发
    1. 节点在当前 round 的超时时间内未收到有效的 QC。
    2. 默认情况下： 
        1. Round 0（最后一次 commit 后）: 1000ms
        2. Round 1: 1200ms (1000 * 1.2¹)
        3. Round 2: 1440ms (1000 * 1.2²)
        4. Round 3: 1728ms (1000 * 1.2³)
        5. Round 4: 2074ms (1000 * 1.2⁴)
        6. Round 5: 2488ms (1000 * 1.2⁵)
        7. Round 6+: 2986ms (1000 * 1.2⁶，封顶)

    注意：
        1. Round 0 恢复 1000ms 的条件是 Block Commit 之后，由于
        2. 每轮的超时时间在 1-3s 之间。
2. 发送超时消息
    1. RoundTimeout 消息
        ```rust
            RoundTimeout {
                timeout: TwoChainTimeout {
                    epoch: u64,
                    round: Round,
                    quorum_cert: QuorumCert,  // 本节点已知的最高 QC
                },
                author: Author,
                timeout_reason: RoundTimeoutReason,  // 超时原因（新增）
                signature: Signature,
            }
        ```
        1. 创建 `TwoChainTimeout` 对象
        2. 通过 SafetyRules 签名（防止双重超时）
        3. 计算超时原因（`compute_timeout_reason`）
        4. 广播 `RoundTimeoutMsg` 到所有节点
        5. 附带 `SyncInfo`（包含最高 QC/TC）

        注意：
        1. 附带最高 QC/TC，这个消息非常重要，当新的 leader 收集到 2f+1 的超时消息之后，在这些超时消息中选择最高的 QC/TC 就能获取全局最高的 QC 作为 TC 的起始，后面涉及到安全性详细介绍。
        2. 超时消息是广播模式，而不是 HotStuff 的主从模式。

    2. Echo Timeout 机制（f+1 触发）
        1. 节点收到 f+1 个超时消息（而不是等待 2f+1）
        2. 节点立即触发本地超时（即使自己的超时计时器未到期），优化机制。
3. 形成 TC (Timeout Certificate)
    1. 聚合 2f+1 个超时签名
    2. 形成 TwoChainTimeoutCertificate（会选择其中最高的 QC（highest quorum cert, HQC））
    3. 更新本地状态，准备进入下一轮

    注意：此时是广播模式，所有的节点进入下一轮，并不是主从模式。
4. 新 Leader 的状态收集（O(Δ) 等待）
    1. Leader 是基于底层 PoS 可预测的轮换模式，新 leader 在形成 TC 之后会等待 O(Δ)，这期间从其他节点获取缺失的区块。
    2. 具体逻辑：
        1. 并行请求：同时向 3 个节点请求
        2. 快速响应：任意一个节点响应即可
        3. 重试机制：失败后等待 500ms 重试
        4. 最大等待：理论最大 25 秒（5 次 * 5 秒），实际通常 < 5 秒
    
    注意：
    1. 每个节点发送超时消息时，都附带自己已知的最高 QC。
    2. TC 聚合时，会选择其中最高的 QC（highest quorum cert, HQC）—— 安全性保证。
    3. 新 Leader 从 TC 中直接获取这个 HQC。
    4. 新 Leader 从 TC 获取目标，向正确的节点请求。

#### Satety(safe_to_vote)

引入 TC 之后，投票的安全性规则就会与传统的 HotStuff 有一些区别，接下来看一下，在真实介绍规则之前需要理解一些概念：

| 术语 | 含义 | 示例 |
|------|------|------|
| `block.round` | 当前区块的轮次 | Block_5 的 round = 5 |
| `block.qc.round` | 当前区块的 QC 所认证的区块轮次（父区块轮次） | Block_5 包含 QC_4，则 block.qc.round = 4 |
| `tc.round` | TC 所在的轮次（超时发生的轮次） | Round 5 超时，tc.round = 5 |
| `tc.highest_hqc.round` | TC 中最高 QC 所认证的区块轮次 | TC_5 包含 QC_4，则 tc.highest_hqc.round = 4 |

1. block.round == block.qc.round + 1 （连续轮次）
2. block.round == tc.round + 1 && block.qc.round >= tc.highest_hqc.round （超时后的恢复）

规则 1 比较好理解，在 Happy Path 情况下，2 轮完成 finalized，所以保证当前投票的区块与父区块QC严格单调递增即可。

规则 2 的基本思想是要 TC 的 HQC 之后进行投票，防止"回退"到更早的 QC，因为 TC 里面的 HQC 已经是收集到 2f+1 节点中最高的 QC （前面已经介绍过—— 防止隐藏锁，即使部分节点"锁定"了某个区块，TC 的 HQC 会包含这个信息，新 Leader 必须基于 >= HQC，不会跳过已经被部分节点认可的区块）。

block.round == tc.round + 1 表示新的轮次是基于超时轮次之后的第一个新轮次，block.qc.round >= tc.highest_hqc.round 表示父区块的 QC 大于等于 TC 已知的 HQC(全网最高的 QC)。

**AptosBFT 牺牲 Unhappy Path 响应性，换取 Happy Path 更快的 finalized**

### Quorum Store

AptosBFT 共识也是类似 [Sui Consensus: Mysticeti](../sui/sui_consensus.md) 对交易顺序进行共识，也想尝试解决 [Double Transmission](../sui/sui_consensus.md#double-transmission) 问题。但是它的设计却是不同的思路，基本的思想：

1. 全局 Mempool，而不是类似 Mysticeti 的 DAG-Base 共识的局部 Mempool。
2. 每个节点在全局 Mempool 中间提交的是 Batch（一批交易的顺序），leader 打包全块交易的时候选择若干个 Batch（根据 Batch Gas，既要保证公平性又要平衡激励）。

Quorum Store 就是完成 Batch 快速同步的子系统，它与共识在设计上也是分离的，但是却是共识的基础所以在共识部分介绍。

Quorum Store 步骤如下：
1. 阶段 1: Batch 创建 && 广播
    1. BatchGenerator 定期从 Mempool 拉取交易
        1. 触发时机：每 25ms 轮询一次（batch_generation_poll_interval_ms）
        2. 拉取限制：
            1. 单个 Batch：最多 50 笔交易（sender_max_batch_txns）
            2. 单个 Batch：最多 1MB（sender_max_batch_bytes）
            3. 单次拉取：最多 1500 笔交易（sender_max_total_txns）
            4. 单次拉取：最多 4MB（sender_max_total_bytes）
        3. 按 Gas 价格分桶排序
        4. 创建 Batch 对象
            ```rust
            Batch {
                batch_info: BatchInfo {
                    author: 本节点 PeerId,
                    batch_id: 递增的唯一 ID,
                    epoch: 当前 epoch,
                    expiration: 当前时间 + 60秒,  // batch_expiry_gap_when_init_usecs
                    digest: 交易内容的哈希,
                    num_txns: 交易数量,
                    num_bytes: 字节数,
                    gas_bucket_start: Gas 价格桶,
                },
                payload: Vec<SignedTransaction>,  // 完整交易数据
            }
            ```
        5. 本地签名：SignedBatchInfo = BatchInfo + 本节点签名
        6. 广播 Batch 给所有节点
            1. 发送内容：完整的 Batch（包含交易数据）
            2. 发送方式：通过 NetworkSender 广播
            3. 目标：所有其他 Validator 节点
            ```rust
            BatchMsg {
                batches: Vec<Batch>,  // 可以包含多个 Batch
            }
            ```
        7. 其他节点的 BatchCordinator 接收 BatchCordinatorCommand::NewBatches
            1. 验证限制（同上）
            2. 验证交易过滤规则（transaction_filter）
        8. 接受者持久化到 QuorumStoreDB
            1. 内存存储 + 持久化存储
            2. 设置过期时间：500ms（remote_batch_expiry_gap_when_init_usecs）
        9. 节点对 BatchInfo 签名： 生成 SignedBatchInfo，发送回 Batch 创建者
2. 阶段 2：ProfOfStore 收集 && 广播
    1. 创建者收集签名（ProofCordinator）
        1. 当达到 2f+1 投票权时，生成 ProofOfStore（聚合签名）
        2. 超时时间：10秒（proof_timeout_ms）
        ```rust
             ProfOfStore {
                info: BatchInfo,
                multi_signature: AggregateSignature,  // 2f+1 签名
            }
        ```
    2. 广播 ProofOfStore
    3. 所有节点的 ProofManager 接收 ProofOfStore，插入到 BatchProfQueue（优先级队列）
3. 阶段 3：Leader 选择 ProfOfStore
    1. Leader 从 ProfManager 拉取 ProfOfStore
        1. 拉取策略：Round-Robin + Gas 价格优先
        2. 队列组织：按 author 分组，每个 Validator 独立队列
            1. Validator A: [PoS_A1(gas=500), PoS_A2(gas=300)]
            2. Validator B: [PoS_B1(gas=400), PoS_B2(gas=200)]
            3. Validator C: [PoS_C1(gas=600), PoS_C2(gas=100)]
        3. 拉取顺序：
            1. 随机打乱 Validator 顺序
            2. Round-robin 轮询每个 Validator
            3. 每个 Validator 内部按 Gas 价格从高到低
    2. Leader 打包进 Block
        ```rust
        Block {
            round: N,
            quorum_cert: QC_{N-1},
            payload: [PoS_C1, PoS_A1, PoS_B1, ...],  // 只有证明
        }
        ```
    3. 节点发现缺失 Batch
        1. 场景 1：收到 ProfOfStore，但本地没有对应的 Batch
        2. 场景 2：执行时需要 Batch，但本地没有
        3. 触发 BatchRequester
            1. 请求对象：ProfOfStore 中的签名者（signers）
            2. 请求策略：
                1. 同时向 5 个节点请求（batch_request_num_peers = 5）
                2. 重试间隔：500ms（batch_request_retry_interval_ms）
                3. 最大重试：10 次（batch_request_retry_limit）
                4. RPC 超时：5000ms（batch_request_rpc_timeout_ms）。

注意：
1. BatchInfo 中的 expiration 字段是创建时设置的，默认 60 秒。
2. 远程节点的 500ms 只是本地存储的临时过期时间，一旦生成 ProofOfStore，所有节点都使用原始的 60 秒过期时间。
3. 本地创建的 Batch 60秒后清理。
4. 避免了内存 + 持久化存储的膨胀。
5. 交易去重复在上述流程中多个阶段，比如：Batch 创建（同一个节点内），PoofOfStore 插入队列（不同节点间）等不在此赘述。

## Execute: BlockSTM

### Account Model

Aptos 采用**混合数据模型**，同时支持传统的账户模型和创新的对象模型。Aptos 在 2023 年引入了对象模型（Object Model），作为账户模型的补充和增强。

| 特性 | 账户模型 | 对象模型 |
|------|---------|---------|
| **地址** | 用户控制的地址 | 程序生成的地址 |
| **所有权** | 账户拥有资源 | 对象可以被账户或对象拥有 |
| **转移** | 资源不可直接转移 | 对象可以整体转移 |
| **删除** | 资源可删除 | 对象可删除（如果允许） |
| **嵌套** | 不支持 | 支持（最多 8 层） |
| **用途** | 用户账户、基础代币 | NFT、复杂资产、可组合对象 |
| **存储优化** | 独立资源 | 资源组（Resource Group） |

### BlockSTM

Block-STM（Software Transactional Memory）是 Aptos 区块链的核心创新之一，它实现了确定性并行事务执行。与传统区块链的串行执行不同，Block-STM 能够并行执行多个事务，同时保证执行结果与串行执行完全一致。其核心思想是乐观并行，如果发生冲突，冲突的交易再次重新执行知道所有的交易执行完，其结果是和串行执行的结果一致。支撑乐观冲突检测的基础是 MVHashMap（多版本数据结构）。

#### MVHashMap
```rust
    // aptos-move/mvhashmap/src/lib.rs
    pub struct MVHashMap<K, T, V, I> {
        // 简单数据的多版本存储
        data: VersionedData<K, V>,

        // 资源组的多版本存储
        group_data: VersionedGroupData<K, T, V>,

        // 延迟字段的多版本存储
        delayed_fields: VersionedDelayedFields<I>,

        // 模块缓存
        module_cache: SyncModuleCache<...>,

        // 脚本缓存
        script_cache: SyncScriptCache<...>,
    }

    // 版本化数据
    pub struct VersionedData<K, V> {
        // Key → BTreeMap<Version, Entry>
        data: DashMap<K, BTreeMap<Version, Entry<V>>>,
    }

    // 版本 = (事务索引, 化身编号)
    pub struct Version {
        txn_idx: TxnIndex,      // 事务索引（0, 1, 2, ...）
        incarnation: u32,       // 化身编号（0, 1, 2, ...）
    }

    // 条目可以是数据或估计标记
    pub enum Entry<V> {
        Value(V),           // 实际值
        Estimate,           // 估计标记（表示可能被覆盖）
    }
```

核心数据结构逻辑： MVHashMap->VersionedData(Version, Entry)，其中 Version.incarnation 和 Entry.Estimate 是设计的灵魂。Version.incarnation 表示交易执行的次数，因为冲突的原因一个交易可以执行多次，需要进行标记（后面会详细介绍）。Entry.Estimate 表示数据正在被更新，其他需要访问该数据的交易应该立刻停止等待最新值。

#### 3-phase execution model

1. 执行阶段：乐观地并行执行所有事务，假设没有冲突。
2. 验证阶段：检查执行结果是否与串行执行一致。
3. 提交阶段：按照串行顺序提交验证成功的事务。

下面看一个完整的示例：
```rust
// 初始状态
Alice.balance = 100
Bob.balance = 50
Charlie.balance = 30

// 事务块
Tx1: Alice.balance -= 30        // Alice = 70
Tx2: Bob.balance = Alice.balance + 10   // Bob = 80
Tx3: Charlie.balance += 20      // Charlie = 50
Tx4: Alice.balance += Bob.balance      // Alice = 150

// 依赖关系
Tx2 依赖 Tx1（读取 Alice.balance）
Tx4 依赖 Tx1 和 Tx2（读取 Alice.balance 和 Bob.balance）
Tx3 独立
```

执行时间线
```
时间 | Thread 1 | Thread 2 | Thread 3 | Thread 4 | MVHashMap 状态
-----|----------|----------|----------|----------|------------------
t0   | 启动     | 启动     | 启动     | 启动     | 初始状态
-----|----------|----------|----------|----------|------------------
t1   | Tx1 执行 |          |          |          | Alice(1,0)=70
     | 开始     |          |          |          |
-----|----------|----------|----------|----------|------------------
t2   |          | Tx2 执行 |          |          | Alice(1,0)=70
     |          | 开始     |          |          | Bob(2,0)=110 ✗
     |          | 读 Alice |          |          | (读到初始值100)
     |          | =100     |          |          |
-----|----------|----------|----------|----------|------------------
t3   |          |          | Tx3 执行 |          | Alice(1,0)=70
     |          |          | 开始     |          | Bob(2,0)=110
     |          |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t4   | Tx1 完成 |          |          |          | Alice(1,0)=70
     | 写 Alice |          |          |          | Bob(2,0)=110
     | =70      |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t5   |          | Tx2 完成 |          |          | Alice(1,0)=70
     |          | 写 Bob   |          |          | Bob(2,0)=110
     |          | =110     |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t6   |          |          | Tx3 完成 |          | Alice(1,0)=70
     |          |          | 写       |          | Bob(2,0)=110
     |          |          | Charlie  |          | Charlie(3,0)=50
     |          |          | =50      |          |
-----|----------|----------|----------|----------|------------------
t7   | Tx1 验证 |          |          |          | Alice(1,0)=70
     | ✓ 成功   |          |          |          | Bob(2,0)=110
     |          |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t8   |          | Tx2 验证 |          |          | Alice(1,0)=70
     |          | ✗ 失败！ |          |          | Bob(2,0)=ESTIMATE
     |          | (Alice   |          |          | Charlie(3,0)=50
     |          | 版本变了)|          |          |
-----|----------|----------|----------|----------|------------------
t9   |          |          | Tx3 验证 |          | Alice(1,0)=70
     |          |          | ✓ 成功   |          | Bob(2,0)=ESTIMATE
     |          |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t10  | Tx1 提交 |          |          |          | 全局状态:
     |          |          |          |          | Alice=70
-----|----------|----------|----------|----------|------------------
t11  |          | Tx2 重新 |          |          | Alice(1,0)=70
     |          | 执行     |          |          | Bob(2,1)=80 ✓
     |          | 读 Alice |          |          | Charlie(3,0)=50
     |          | =70      |          |          |
-----|----------|----------|----------|----------|------------------
t12  |          | Tx2 完成 |          |          | Alice(1,0)=70
     |          | 写 Bob   |          |          | Bob(2,1)=80
     |          | =80      |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t13  |          | Tx2 验证 |          |          | Alice(1,0)=70
     |          | ✓ 成功   |          |          | Bob(2,1)=80
     |          |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t14  |          | Tx2 提交 |          |          | 全局状态:
     |          |          |          |          | Alice=70, Bob=80
-----|----------|----------|----------|----------|------------------
t15  |          |          | Tx3 提交 |          | 全局状态:
     |          |          |          |          | Alice=70, Bob=80
     |          |          |          |          | Charlie=50
-----|----------|----------|----------|----------|------------------
t16  |          |          |          | Tx4 执行 | Alice(1,0)=70
     |          |          |          | 开始     | Bob(2,1)=80
     |          |          |          | 读 Alice | Charlie(3,0)=50
     |          |          |          | =70      | Alice(4,0)=150
     |          |          |          | 读 Bob   |
     |          |          |          | =80      |
-----|----------|----------|----------|----------|------------------
t17  |          |          |          | Tx4 完成 | Alice(4,0)=150
     |          |          |          | 写 Alice | Bob(2,1)=80
     |          |          |          | =150     | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t18  |          |          |          | Tx4 验证 | Alice(4,0)=150
     |          |          |          | ✓ 成功   | Bob(2,1)=80
     |          |          |          |          | Charlie(3,0)=50
-----|----------|----------|----------|----------|------------------
t19  |          |          |          | Tx4 提交 | 全局状态:
     |          |          |          |          | Alice=150, Bob=80
     |          |          |          |          | Charlie=50
```

##### Q1：ESTIMATE 标记的作用？

没有 ESTIMATE，Tx2 读取旧值 → 执行完成 → 验证失败 → 重新执行， 浪费资源。
有 ESTIMATE，Tx2 读取 ESTIMATE → 立即暂停 → 等待 Tx1 完成 → 继续执行

##### Q2：为什么要多版本？

Tx1 和 Tx2 并行，如果是单版本，Tx1 的更新看不到了，无法验证 Tx1 是否正确（每个交易的验证是必要的，可以保证并行的结果与串行的结果相同），所以保留每个交易的更新对于验证阶段是必要的，多版本可以保证每个事务的写都被保留。

Tx2 冲突之后需要重新执行，多版本可以保证 Tx2 重新执行时拿到 Tx1 的执行结果。

##### Q3：化身的必要性？

Tx4 依赖 Tx1 和 Tx2，Tx2 执行了两次，Tx4 使用 Tx2 哪次执行的结果？验证阶段使用 Tx2 那次直接过节？incarnation 就可以解决这个问题。

##### Summary

多版本存储的本质是保留所有更新，给验证阶段保留足够的信息用于确定并行的结果与串行的结果一致。同时，由于多版本的设计，可以保证：读历史版本 + 写新版本 + 并行验证（前两步锁的粒度小，性能好）。

明显的优势：
1. 不需要依赖声明，运行时检测冲突，对开发者透明。
2. 运行结果是确定性的，与串行结果一致。
3. 并行提升资源利用率和吞吐。


### Oreder Vote && Commit Vote

AptosBFT 只是对交易顺序进行了共识，对于交易结果仍然需要共识才能提交。类似 [Sui Checkpoint](../sui/sui.md#workflow)。Aptos 在这里有两轮投票 Oreder Vote 和 Commit Vote，其中 Oreder Vote 对于执行结果达成共识（收集 2f+1 形成 Order Proof）然后 PreCommit，Commit Vote 会作为共识的内容之一伴随共识中的网络交互完成共识（收集 2f+1 形成 Commit Proof）最终完成 Block Commit，相当于 Finalized。


#### WorkFlow

假设有 4 个验证者：V1, V2, V3, V4，每个权重 25%，2f+1 = 75% (需要 3 个验证者)

1. Round 100
    1. V1 (Leader) 提出 B100，QC99
    2. V2, V3, V4 收到 B100，开始处理（pipeline 处理，后续介绍）
    3. V2, V3, V4 验证通过，投票给 V2 (下一轮 Leader)
    4. V2 收集到 3 个投票，形成 QC100
        1. vote_data: { proposed: B100, parent: B99 }
        2. commit_info: B99  ← 可以提交 B99 (2-chain)
        3. signatures: [V2, V3, V4] (75% 权重)
    
    注意：
    1. 重点是 vote_data 内容，以及 commit_info，后面重点介绍 commit_info = B100 的形成过程。
    2. 此时 B100 正在执行，执行完之后等待 Order Vote。

2. Round 101 
    1. V2 (Leader) 提出 B101，QC100
    2. V1, V3, V4 收到 B101，开始处理（pipeline 处理，后续介绍）
    3. 此时，B100 执行完成！触发 Order Vote。
        1. V2, V3, V4 广播 Order Vote (针对 B100)
            1. OrderVote(B100, state_root=0xabc123, version=1000, sig_V2/3/4)
        2. 所有节点收集到 3 个 Order Vote，形成 Order Proof
        3. B100 收到 Order Proof，触发 Pre-commit
    4. 与 3 并行，发送 CommitVote，commit_info = B99。
    5. V1, V3, V4 投票给 V3 (下一轮 Leader)
    6. V3 形成 QC101

    注意：
    1. 触发 Order Vote 是立刻的，不需要与共识的网络交互复用，也不需要等待任何共识中的 QC。
    2. Order Vote 包含了执行的状态信息。
    3. Round 100 中 commit_info = B99 其实是 Commit Vote(CommitQC)，告诉节点可以提交 B99.
    4. 所以，执行结果其实也是类似 HotStuff2 的两轮共识(Order + Commit)。

3. Round 102
    1. V3 (Leader) 提出 B102， QC101
    2. 所有节点收到 QC101， QC101 的 commit_info 指向 B100。
    3. B100 收到 Commit Proof (QC101)，开始提交。

**总结：Aptos 对于执行结果仍然是两轮共识，Order + Commit，有一些工程优化，Order 阶段不需要依赖 AptosBFT 共识的 QC，所以可以做一个独立的网络交互提升性能，并且针对相对较快的 Order Vote 做了一个 PreCommit，加快Commit 的性能。**
    
#### Pipeline 

交易顺序共识达成，Aptos 将执行阶段划分了若干个子阶段进行 pipeline 并行处理：
1. Materialize: 等待 QuorumStore 数据
2. Prepare: 签名验证 (16线程并行)
3. Execute: Block-STM 执行交易
4. Ledger Update: 生成状态根
5. Sign Commit Vote: 等待触发
6. Pre-commit: 等待 Order Proof
7. Commit: 等待 Commit Proof

### Back Pressure

本质上 Aptos 是 2-chain 的确认块，将 Order Vote 并行加快了速度，Commit Vote（上述的 Commit Info）与前一块的 QC 结合到一轮投票中。

仍然存在速度不匹配的问题：QuorumStore > AptosBFT > Pipeline 执行，所以仍然背压系统。

1. QuorumStore Backpressure
    ```rust
    back_pressure_total_txn_limit: u64,      // 总交易数限制
    remaining_total_txn_num: u64,            // 剩余容量

    back_pressure_total_proof_limit: u64,    // 总证明数限制
    remaining_total_proof_num: u64,          // 剩余容量
    ```
    1. 防止内存溢出：限制队列大小
    2. 自适应：根据执行速度调整生成速度
    3. 平滑：通过多级背压避免突变
2. Execution Backpressure
    1. 如果执行慢，减少区块中的交易数量
    2. 如果执行快，增加区块中的交易数量
3. Pipeline Backpressure
    1. "最老的未提交区块"在流水线中的时间
    2. 减少区块大小（交易数量和字节数）
    3. 增加提议延迟（减慢共识速度）
    ```
    流水线延迟 < 1.2秒：正常（5MB 区块，无延迟）
    流水线延迟 1.2-1.5秒：轻微反压（5MB 区块，延迟 50ms）
    流水线延迟 1.5-1.9秒：中等反压（5MB 区块，延迟 100ms）
    流水线延迟 > 6秒：严重反压（只允许 5 个交易，延迟 300ms）
    ```
4. Vote Backpressure
    1. 如果 ordered_round - commit_round > 12
    2. 拒绝投票，强制减慢共识
    3. 最后防线

## Storage: JMT

Aptos 使用 [JMT](../../state_mode/jmt.md) 组织状态数据并计算 state root 用于共识。其状态数据有两部分，一部分是 JMT 还有一部分是叶子节点（即帐户或者对象的数据）使用扁平 kv。JMT 的一个特点是 key 带有版本号（Diem 性能提升的关键），Apots 中的版本号是全局的交易序号而不是 Block Number，支持交易级别的状态证明（对 DeFi 等应用很重要），方便调试和问题定位（开发者友好），为未来的更细粒度的功能预留空间等优点。

#### Data Struct

```rust
// NodeKey 结构：(Version, NibblePath)
pub struct NodeKey {
    version: Version,           // 节点创建时的版本号
    nibble_path: NibblePath,   // 从根到该节点的路径
}

JellyfishMerkleNodeSchema:
Key:   (Version, num_nibbles, nibble_path)
Value: Node (Internal/Leaf/Null)
```

JMT 使用 分 shard 存储，并行度和性能表现更好：
1. `state_merkle_metadata_db`：存储根节点和顶层节点
2. `state_merkle_db_shard_0` ~ `shard_15`：存储各分片的节点

```rust
// 分片模式（推荐）
StateValueByKeyHashSchema:
Key:   (Hash(StateKey), Version)
Value: Option<StateValue>
```

1. 最新版本排在前面
2. 范围查询时可以快速找到最新值---**rocksdb seek(比get操作性能更差一些)**
3. 支持高效的版本回溯

#### Snapshot
```rust
pub struct BufferedState {
    // 最后一个持久化的快照
    base: StateWithSummary,

    // 当前最新状态（可能未持久化）
    current: Arc<Mutex<LedgerStateWithSummary>>,

    // 目标：每 TARGET_SNAPSHOT_INTERVAL_IN_VERSION 个版本创建一个快照
}
```
1. 默认每 10,000 个版本创建一个快照
2. 快照包含完整的 Merkle 树根节点
3. 支持从任意快照恢复状态

current 表示内存中缓存的节点，base 表示持久化的db实例。

#### Cache


| 缓存 | 缓存的数据 | 数据结构 | 是否分片 |
|------|--------|------|------|
| **BufferedState** | State KV（状态键值对） | `HashMap<StateKey, StateValue>` | ❌ 不分片 |
| **PersistedState** | State KV（元数据） | `{ version, root_hash, usage }` | ✅ 数据已分片 |
| **LRU Cache** | JMT Node（Merkle 树节点） | `LruCache<NodeKey, Node>` | ✅ 16 个分片 |
| **Versioned Cache** | JMT Node（按版本批量） | `LruCache<Version, HashMap<NodeKey, Node>>` | ❌ 不分片 |

```
┌────────────────────────────────────────────────────────┐
│  Transaction Execution                                 │
│  ↓                                                     │
│  WriteSet (状态变更)                                   │
│  ↓                                                     │
│  BufferedState (内存缓存，不分片) ← State KV           │
│  ↓                                                     │
│  Versioned Cache (全局缓存，不分片) ← JMT Node         │
│  ↓                                                     │
├────────────────────────────────────────────────────────┤
│  分片层（Sharding Layer）                              │
├────────────────────────────────────────────────────────┤
│  ↓                                                     │
│  LRU Cache (16 个分片，每个分片独立) ← JMT Node        │
│  ↓                                                     │
│  State KV DB (16 个分片) ← State KV                    │
│  State Merkle DB (16 个分片) ← JMT Node                │
│  ↓                                                     │
│  RocksDB (磁盘存储)                                    │
└────────────────────────────────────────────────────────┘
```

##### Hot State

针对 State KV 数据（非 JMT 节点），进行缓存。
1. 写入立即变热。
2. 读不写的 key，在 区块结尾（block epilogue） 时批量提升为热数据，每个区块最多提升 10,240 个 key。
3. 如果一个 key 已经是热数据，但距离上次访问超过 10w 次，再次读取时会刷新其 hot_since_version，防止被淘汰。
4. 总共 16 个 shard，每个 shard 最多存储 250,000 个热数据项，每个 shard 使用 LRU 缓存。

#### Pruning

Aptos 使用三种独立的剪枝器来管理历史数据，每种剪枝器负责不同类型的数据清理：

1. **State Merkle Pruner** - 剪枝过期的 JMT 节点
2. **State KV Pruner** - 剪枝过期的状态键值对
3. **Epoch Snapshot Pruner** - 剪枝跨 epoch 的过期快照

这三种剪枝器都遵循相同的设计模式：
1. 保留最近 N 个版本的数据
2. 使用索引表追踪过期数据
3. 支持分片并行剪枝
4. 独立的进度追踪

##### State Merkle Pruner

```rust
pub struct StaleNodeIndex {
    pub stale_since_version: Version,  // 节点从哪个版本开始过期
    pub node_key: NodeKey,              // 过期节点的键
}

|<--------------key-------------->|  |<-value->|
| stale_since_version | node_key |  |   ()    |

旧节点: NodeKey(version=90, nibble_path=0x1234)
新节点: NodeKey(version=100, nibble_path=0x1234)

写入索引:
StaleNodeIndex {
    stale_since_version: 100,
    node_key: NodeKey(version=90, nibble_path=0x1234)
}

当前版本: 5000
剪枝目标: 4000 (保留 4001-5000)

索引表内容:
stale_since_version | node_key
--------------------|------------------
3500                | NodeKey(v=3400, path=0x01)
3800                | NodeKey(v=3700, path=0x02)
4000                | NodeKey(v=3900, path=0x03)
4200                | NodeKey(v=4100, path=0x04)  <- 不删除

剪枝操作:
1. 扫描 stale_since_version <= 4000 的记录
2. 删除 3 个过期节点
3. 删除对应的索引记录
4. 更新剪枝进度到 4000
```

1. Key: `stale_since_version` (大端序) + `node_key`
2. Value: 空（仅用作索引）
3. 排序: 按 `stale_since_version` 升序排列

##### State KV Pruner

```rust
pub struct StaleStateValueIndex {
    pub stale_since_version: Version,  // 值从哪个版本开始过期
    pub version: Version,               // 值的版本号
    pub state_key: StateKey,            // 状态键
}

|<-------------------key------------------->|  |<-value->|
| stale_since_version | version | state_key |  |   ()    |

StateKey("account_balance") 的历史:
- version 100: value = 1000
- version 200: value = 2000
- version 300: value = 3000

当 version 300 写入时，version 200 过期:
StaleStateValueIndex {
    stale_since_version: 300,
    version: 200,  // 需要这个字段定位具体版本
    state_key: "account_balance"
}

StateKey: "0x1::account::balance"

历史版本:
- v100: 1000 APT
- v200: 2000 APT
- v300: 3000 APT
- v400: 4000 APT

索引表:
stale_since_version | version | state_key
--------------------|---------|------------------
200                 | 100     | "0x1::account::balance"
300                 | 200     | "0x1::account::balance"
400                 | 300     | "0x1::account::balance"

当前版本: 500
剪枝目标: 400 (保留 v401-v500)

剪枝操作:
1. 删除 (state_key, v100)
2. 删除 (state_key, v200)
3. 删除 (state_key, v300)
4. 保留 (state_key, v400) - 最新值
```

与 StaleNodeIndex 不同，StaleStateValueIndex 包含 `version` 字段，原因是：
1. 同一个 key 可能有多个历史版本
2. 没有 version 字段，无法区分同一 key 的不同历史版本

##### Epoch Snapshot Pruner

Aptos 的 epoch 是重要的检查点：
1. Epoch 边界：验证器集合变更、配置更新
2. 状态快照：epoch 结束时的完整状态
3. 验证需求：需要保留 epoch 边界的状态用于验证

双索引表设计：
1. StaleNodeIndexSchema：普通过期节点
    1. 用于 epoch 内的节点更新
    2. 剪枝策略：保留最近 N 个版本
2. StaleNodeIndexCrossEpochSchema：跨 epoch 过期节点
    1. 用于 epoch 边界的节点
    2. 剪枝策略：保留 epoch 快照

如何判断一个节点是否属于 epoch 边界？
```
当前版本: 1000
上一个 epoch 结束版本: 800

过期节点: NodeKey(version=750, path=0x01)

判断:
750 <= 800 ?  -> Yes
-> 这个节点属于上一个 epoch
-> 写入 StaleNodeIndexCrossEpochSchema
```

#### Config
```rust
// 配置剪枝窗口
pub struct PrunerConfig {
    // 保留最近 10,000 个版本的状态
    pub state_store_prune_window: Option<u64> = Some(10_000),

    // 保留最近 2 个 epoch 的快照
    pub epoch_snapshot_prune_window: Option<u64> = Some(2),

    // 每次剪枝最多处理 1000 条记录
    pub pruning_batch_size: usize = 1000,
}
```

#### summary
| 特性 | State Merkle Pruner | State KV Pruner | Epoch Snapshot Pruner |
|------|---------------------|-----------------|----------------------|
| 剪枝对象 | JMT 节点 | 状态键值对 | Epoch 边界节点 |
| 索引表 | StaleNodeIndex | StaleStateValueIndex | StaleNodeIndexCrossEpoch |
| 索引字段 | version + node_key | version + version + state_key | version + node_key |
| 分片支持 | ✓ (16 shards) | ✓ (16 shards) | ✓ (16 shards) |
| 保留策略 | 最近 N 个版本 | 最近 N 个版本 | 最近 N 个 epoch |