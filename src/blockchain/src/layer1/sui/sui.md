# Sui

## Overview

Sui 的设计目标是高性能 Layer1，为此展开设计，从结果上看有三个主要的特点：
1. MoveVM：Move 语言，资源导向--天然适配安全性（默认不可复制/丢弃）。
2. 对象模型：结合 Move 的特点，摒弃了常见的帐户模型，该用对象模型，更好的支持并发。
3. 共识与执行解耦：降低了共识瓶颈，带来更高吞吐、更低延迟、更好的资源利用率。

MoveVM 不是本文的介绍重点（详情见[官网](https://docs.sui.io/guides/developer/packages/move-package-management)）。本文从 Infrastructure 角度，通过介绍对象模型，共识，执行，状态模型等理解 Sui 的高性能。

## Object Model

对象模型由于天然对高并发友好，因此是 Sui 完成高性能的重要基石之一，也理解后续的基础，所以先从对象模型开始介绍。

### Object Struct

```rust
pub struct Object {
    pub object_id: ObjectID,
    pub version: SequenceNumber,
    pub type_: ObjectType,
    pub owner: Owner,
    pub data: Data,
    pub storage_rebate: u64,
}
```
*代码实际的结构比这复杂，这里是在源码 ObjetInfo, ObjectInner 基础上进行了抽象和合并，方便理解*。

* ObjectID
    1. 32 字节，对象的唯一表示，可以与 SuiAddress 转换实现对象转移（对象所有权部分介绍）。
* SequenceNumber
    1. Lamport 时间戳，版本号 = max(所有输入对象版本) + 1。
    2. 所有的输入对象在操作之后统一变成一个新的版本号，标记最后一次更新。
    3. 不是每个对象的顺序版本号（递增但不单调）。
    4. 目的：
        1. 防止双花：交易必须引用精确版本
        2. 乐观并发：版本不匹配则交易失败
* ObjectType
    1. 主要分为两类，一类是 Move 结构的对象，一类是合约对象。
    2. 针对 Move 结构的对象，对一些特定场景的类型进行了存储优化（后续对象类型部分介绍）。
* Owner
    1. 对象的所有权类型，是一个枚举有 5 种类型（后续对象所有权部分详细介绍）。
    2. 它是 Sui 对象模型的精髓，重点通过它来理解执行路径（快速路径 or 共识路径）。
* Data
    1. 对象的数据内容，BCS 编码。
* StorageRebate
    1. 根据对象大小预留部分 GasCoin，当删除对象之后归还用户，激励用户删除数据。

对象模型与帐户模型相比，对于并发执行更友好，比如：A 有两个对象分别代表 100 Token 和 50 Token，可以并行转发给 B 和 C。对象模型粒度更小，MoveVM 在执行之前要求将需要的对象作为输入，可以基于对象力度进行并发，而不是帐户模型下拥有所的资源，并发粒度比较粗。

### Object Ownership

Sui 中的对象由 5 中所有权类型，对象所有权在 Sui 的设计中之冠重要，正是所有权的涉及使得 80% 的交易可以快速执行并返回客户端（共识部分介绍）。

* AddressOwner
    1. 对象由单个 Sui 地址（用户账户）独占拥有（读、写、删除、转移），无需要共识排序。
    2. 最常见的所有权类型，例如：用户拥有的 NFT、代币，个人配置、游戏角色等。
* ObjectOwner
    1. 对象由另一个对象拥有（父子关系），实现"转移到对象"机制。
    2. 权限由父对象控制，是否需要共识排序也由父对象决定。
    3. 典型的应用: Kiosk NFT 交易市场，用户将 NFT 上架 Kiosk（属于其子对象）进行展示售卖。
* Shard
    1. 对象是共享的，任何人都可以访问和修改，需要共识排序。
    2. 必须在创建时共享，不能被转移或变为不可变。
    3. 典型应用：DeFi 协议的共享池。
* ConsensusAddressOwner
    1. 对象由单个地址独占拥有，但需要共识排序。
    2. 典型应用：DAO 基金管理，质押提取需要共识。

##### Object Comapre

| 所有权类型 | 读取 | 写入 | 删除 | 转移 | 谁可以操作 | 备注 |
|-----------|------|------|------|------|-----------|------|
| **AddressOwner** | ✓ | ✓ | ✓ | ✓ | 仅所有者 | 最常见的所有权类型 |
| **ObjectOwner** | ✓ | ✓ | ✓ | ✓ | 通过父对象 | 必须通过父对象的 UID 访问 |
| **Shared** | ✓ | ✓ | ✓ | ✗ | 任何人 | 不能转移，一旦共享永远共享 |
| **Immutable** | ✓ | ✗ | ✓ | ✗ | 任何人 | 完全不可变，永久状态 |
| **ConsensusAddressOwner** | ✓ | ✓ | ✓ | ✓ | 仅所有者 | 需要共识排序 |

##### Object Transfer

| 从 \ 到 | AddressOwner | ObjectOwner | Shared | Immutable | ConsensusAddressOwner |
|---------|--------------|-------------|--------|-----------|----------------------|
| **AddressOwner** | ✓ | ✓ | ❌ | ✓ | ✓ |
| **ObjectOwner** | ✓ | ✓ | ❌ | ✓ | ✓ |
| **Shared** | ❌ | ❌ | ✓ | ❌ | ❌ |
| **Immutable** | ❌ | ❌ | ❌ | ✓ | ❌ |
| **ConsensusAddressOwner** | ✓ | ✓ | ❌ | ✓ | ✓ |

##### Consensus Path

| 所有权类型 | 执行路径 | 需要共识 | 版本控制 | 性能 |
|-----------|---------|---------|---------|------|
| **AddressOwner** | 快速路径 | ❌ | 手动 | 高 |
| **ObjectOwner** | 由父对象决定 | 取决于父对象 | 自动（通过父对象）| 取决于父对象 |
| **Shared** | 仅共识 | ✅ | 自动 | 较低（需要共识）|
| **Immutable** | 快速路径 | ❌ | 手动（最终版本）| 高 |
| **ConsensusAddressOwner** | 仅共识 | ✅ | 自动 | 较低（需要共识）|

### Object Type

Sui 对象主要分两大类：
1. MoveObject - 用户数据和状态
    1. Other(StructTag)：普通类型（不是 Coin），Move 数据结构。
        1. StructTag 基于对象的一些元数据，比如 address/module/name 名等。
    2. GasCoin：SUI 代币。
        1. 本质上也是 StructTag 结构，但是由于 GasCoin 比较场景，在压缩的时候进行了优化，只有 1 个字节，因为其元数据都是可预测的。
    3. StakedSui：质押的 SUI。
        1. 同上 GasCoin 也是 1 字节优化。
        2. 与 GasCoin 的区别是，GasCoin 是可转移的 SUI 代币，用于支付 gas，元数据记录的内容和普通 Coin 一样。StakedSui 是质押记录对象，元数据记录的是 pool_id、激活 epoch，principal 等质押元数据，不是 coin 类型，也不能直接当 gas 用。
    4. Coin(TypeTag)：非 SUI 代币，压缩优化存储为 20 字节。
    5. SuiBalanceAccumulatorField：SUI 余额累加器字段。
        1. 余额累加器，地址级余额聚合，例如：池子的总余额，避免大量对象遍历。
    6. BalanceAccumulatorField(TypeTag)：非 SUI 余额累加器字段。
        1. 同上。
2. Package - 编译后的 Move 代码
    1. 系统包（System Packages）
        1. 不受大小限制
        2. 可以在特殊情况下修改（版本递增）
        3. 所有版本存在于同一个 ID
        4. 总是加载最新版本
    2. 普通包（Regular Packages）
        1. 受大小限制
        2. 创建后不能修改
        3. 升级会创建新的包对象（新 ID）
        4. 每个 ID 只有一个版本

##### Object Type Compare

| 特性 | MoveObject | Package |
|------|-----------|---------|
| **用途** | 用户数据和状态 | 编译后的 Move 代码 |
| **内容** | BCS 编码的 Move 结构体 | 模块字节码映射 |
| **可变性** | 可变或不可变 | **总是不可变** |
| **所有权** | 所有 5 种 Owner 类型 | **总是 Immutable** |
| **版本控制** | Lamport 时间戳 | 顺序递增（系统包）|
| **大小限制** | 有限制 | 系统包无限制 |
| **创建方式** | Move 代码执行 | 包发布交易 |
| **升级** | 修改现有对象 | 创建新包对象 |
| **存储键** | (ID, Version) | (ID) |
| **典型示例** | NFT, Coin, 游戏对象 | Sui Framework, 用户包 |

##### Object Arch

```
Object (包装结构)
├── data: Data
│   ├── Move(MoveObject)      ← 用户数据
│   │   ├── GasCoin          ← SUI 代币
│   │   ├── StakedSui        ← 质押 SUI
│   │   ├── Coin<T>          ← 其他代币
│   │   └── Other            ← 普通对象
│   └── Package(MovePackage)  ← Move 代码
│       ├── System Packages  ← 系统包
│       └── User Packages    ← 用户包
├── owner: Owner
│   ├── AddressOwner
│   ├── ObjectOwner
│   ├── Shared
│   ├── Immutable
│   └── ConsensusAddressOwner
├── previous_transaction
└── storage_rebate
```

## Consensus

Sui 的共识是一个基于 DAG 的拜占庭容错共识协议，具体内容见[Mysticeti](./sui_consensus.md)。其主要特点如下：

1. 与其他 BFT 类共识不同，Mysticeti 只共识交易的顺序，并未执行交易。
2. 本质是拜占庭容错共识协议，也是两轮投票。
3. 投票轮次中中的网络交互次数达到了理论下限 3 次，并且交互的消息附带子 DAG，提升吞吐量。

在这里要重点介绍的是 Mysticeti-FPC，即快速路径共识。在对象所有部分介绍了 AddressOwner 和 Immutable 的执行路径是快速路径而非共识路径。共识路径就是 [Mysticeti](./sui_consensus.md) 中介绍的完整流程。快速路径是在第二轮收集到 2f+1 的投票之后，就会在本地立即执行，然后返回给客户端结果，本地会缓存结果（注意此时未 finalized），继续走共识路径，在共识结果出来之后执行期间会查阅本地缓存跳过执行，否则仍然执行。

快速路径其实是用户体验的糖衣，因为交易并没有确定最终上链，但是由于所有权的约束，其结果是正确的（如果上链就是之前执行的结果不会有分歧）。

## Execute

Mysticeti 只共识了交易顺序并没有执行交易，本部分介绍从共识结束到执行再到 finalized 的余下部分，并针对共识与执行速度不匹配情况下 Sui 的做法。 

### Workflow

1. Mysticeti 之后的结果会形成 CommittedSubDag（经过[线性化 DAG 处理](./sui_consensus.md#linearizer-dag)），并提交给共识处理器。
2. 共识处理器：提取子 DAG 中所有的交易，去重，排序，处理系统交易，创建 PendingCheckpoint。
3. 检查点构建：执行交易，收集交易结果，计算状态 Hash，创建 CheckpointSummary。
4. 检查点认证：节点签名并广播 CheckpointSummary，收集 2f+1 以上的签名，形成 CertifiedCheckpointSummary，交易被最终 Finalized。

注意：
1. Checkpoint 其实才是传统意义上的区块的概念，共识中本质上是子 DAG（Sui 中称作共识区块，其实容易混淆造成模糊）。
2. CertifiedCheckpointSummary 仍然需要一轮共识，所以严格上说 Sui 的共识是分两层的，一层共识交易顺序，一层共识区块执行的最终确定层。
    1. 共识层
        - 高频率共识（50-250ms）
        - DAG 结构提高吞吐量
        - 快速排序交易
        - 容错性强
    2. 最终确定层
        - 交易执行和验证
        - 状态哈希计算
        - 2f+1 签名确保安全
        - 状态同步点

### Traffic limiting

MoveVM 配合对象模型，使用乐观并行的方式执行交易，极大的提升了执行速度。但是有一点不能忽略的是，共识层相对仍然是轻量级，执行层依然是瓶颈，在流量较大的时候执行速度会跟不上共识的速度，所以需要一套机制保证他们的匹配，下面看一下 Sui 是怎么做的。

* 背压
    1. 待提交的交易（共识提交的交易）大于 10w 时，停止 CommittedSubDag。
    2. 可以保证整个系统不被压垮。
* 全局队列限制
    1. 待执行交易的总数（执行的交易）大于 10w 是，拒绝新交易。
    2. 快速失败，而不是慢慢崩溃，给用户明确的错误信息，控制内存，CPU 等资源快速增长。
* 单对象队列限制
    1. 某个对象的待执行交易大于 2000 时，拒绝针对该对象的新交易。
    2. 隔离热点对象的影响，防止单个热点拖垮整个系统。
* 交易年龄限制
    1. 交易在队列中等待时间 > 1s，拒绝新交易。
    2. 防止雪崩效应，避免用户长时间等待后失败。
* 负载削减：优雅降级，而不是崩溃
    1. 正常模式（延迟 < 1 秒）：正常接受交易。
    2. 软限制模式（延迟 1-10 秒）：开始负载削减，0% → 95%。
    3. 硬限制模式（延迟 > 10 秒）：拒绝率: 50% → 95%。

在实际能力范围之外（10w）开始服务降级，保证系统不垮，并在限制的情况下尝试尽量处理更多新交易。

## State Model

Sui 并没有使用类似 MPT 的树形结构，而是使用了扁平 KV 存储（RocksDB）存储对象，状态 Hash 使用的是累加式的 ECMH，支持 Union 操作与顺序无关，简单理解就是将对象旧版本+新版本的摘要输入，可以得到全局对象（最新版本）的 Hash 状态。

之前介绍了子 DAG（共识区块），Checkpoint（类比传统意义的区块），在介绍 tate Root 计算流程中还需要介绍一个新的逻辑概念 Epoch。Epoch 是验证者集合的时间周期，类似于其他区块链的"时代"或"轮次"，目前主网是 24 小时。主要作用：

1. 验证者轮换流程：
    1. 计算 Epoch N 的最终状态哈希
    2. 确定 Epoch N+1 的验证者集合
    3. 在 Epoch N 的最后一个 Checkpoint 中包含 EndOfEpochData
    4. 所有验证者签名确认
    5. 切换到 Epoch N+1
2. 协议升级
    1. 所有验证者同步到相同状态
    2. 可以安全地改变协议规则
    3. 避免分叉
3. Staking 和奖励
    1. 验证者获得 staking 奖励
    2. 委托者获得收益
    3， 可以更改 stake 分配

##### Compare

| 概念 | 时间跨度 | 频率 | 主要作用 |
|------|---------|------|---------|
| **Epoch** | ~24 小时 | 每天 1 次 | 验证者集合轮换、协议升级 |
| **Checkpoint** | ~3-5 秒 | 每秒多次 | 交易最终确定、状态快照 |
| **状态哈希** | 每个 Checkpoint | 每秒多次 | 状态承诺、验证一致性 |

### State Root

1. 每个 Checkpoint 都计算状态哈希。
    ```
    state_hash_by_checkpoint[N] = ECMH({ 所有新建/修改的对象摘要 }) - ECMH({ 所有旧版本对象摘要 })
    ```
2. 运行根状态哈希
    ```
    running_root[0] = state_hash[0]
    running_root[1] = running_root[0] ∪ state_hash[1]
    running_root[2] = running_root[1] ∪ state_hash[2]
    ...
    running_root[N] = running_root[N-1] ∪ state_hash[N]
    ```
3. Epoch 级别的状态哈希
    ```
    Epoch 状态哈希 = 最后一个 Checkpoint 的运行根哈希
    ```

**Checkpoint 级别**（`state_hash_by_checkpoint`）：
- 记录每个 Checkpoint 的状态变更
- 用于增量验证
- 支持并行计算

**运行根级别**（`running_root_state_hash`）：
- 记录从 Epoch 开始的累积状态
- 用于快速验证整体状态
- 支持状态同步

**Epoch 级别**（`root_state_hash_by_epoch`）：
- 记录 Epoch 的最终状态
- 用于跨 Epoch 验证
- 支持历史查询

### Storage

Sui 采用 RocksDB 作为底层存储，详细的数组组织结构见[Sui Storage Table](./sui_storage_table.md)。

### Prune

状态增长
对象模型可能导致更多状态
需要有效的修剪策略——如何修剪的？
存储成本：按对象大小计费

## Reference

[Sui Source Code](https://github.com/MystenLabs/sui.git)