# Sui Storage Table

本文档详细记录了 Sui 区块链中所有 RocksDB 表（列族）的结构、用途和存储内容。

## 目录

1. [AuthorityPerpetualTables - 持久化存储](#1-authorityperpetualTables)
2. [AuthorityPrunerTables - 裁剪器表](#2-authorityprunertables)
3. [AuthorityEpochTables - 每轮次存储](#3-authorityepochtables)
4. [CheckpointStoreTables - 检查点存储](#4-checkpointstoretables)
5. [CommitteeStoreTables - 委员会存储](#5-committeestoretables)
6. [IndexStoreTables - RPC 索引](#6-indexstoretables-rpc-索引)
7. [IndexStoreTables - JSON-RPC 索引（遗留）](#7-indexstoretables-json-rpc-索引遗留)
8. [RocksDBStore - 共识存储](#8-rocksdbstore-共识存储)
9. [WritePathPendingTransactionTable - 待处理交易](#9-writepathpendingtransactiontable)
10. [BridgeOrchestratorTables - 跨链桥存储](#10-bridgeorchestatortables)
11. [PackageStoreTables - 包存储](#11-packagestoretables)
12. [Indexer-Alt Consistent Store - 新索引器](#12-indexer-alt-consistent-store)

---

## 1. AuthorityPerpetualTables

**位置：** `crates/sui-core/src/authority/authority_store_tables.rs`

**用途：** 存储必须在轮次（epoch）之间保留的持久化数据。

### 表结构

#### `objects` - 对象存储
- **类型：** `DBMap<ObjectKey, StoreObjectWrapper>`
- **键：** `ObjectKey` (ObjectID + SequenceNumber)
- **值：** `StoreObjectWrapper` (Move 模块或 Move 对象)
- **说明：** 存储所有版本的对象，只裁剪在 TransactionEffects 中作为输入的版本
- **优化：** 使用 5GB 自定义块缓存（可通过 `OBJECTS_BLOCK_CACHE_MB` 配置）

#### `live_owned_object_markers` - 活跃对象标记
- **类型：** `DBMap<ObjectRef, Option<LockDetailsWrapperDeprecated>>`
- **键：** `ObjectRef` (ObjectID + SequenceNumber + ObjectDigest)
- **值：** 可选的交易锁定详情（旧轮次已弃用）
- **说明：** 跟踪当前可变更的活跃对象
- **优化：** 使用 1GB 块缓存（可通过 `LOCKS_BLOCK_CACHE_MB` 配置）

#### `transactions` - 交易存储
- **类型：** `DBMap<TransactionDigest, TrustedTransaction>`
- **键：** 交易摘要
- **值：** 完整交易数据
- **说明：** 存储可执行交易（本地执行或通过状态同步）
- **优化：** 针对点查询优化（512MB 块缓存，通过 `TRANSACTIONS_BLOCK_CACHE_MB` 配置）

#### `effects` - 交易效果
- **类型：** `DBMap<TransactionEffectsDigest, TransactionEffects>`
- **键：** 交易效果摘要
- **值：** 交易效果
- **说明：** 存储交易执行效果（同步或本地执行时存储），可能存储两次
- **优化：** 针对点查询优化（1GB 块缓存，通过 `EFFECTS_BLOCK_CACHE_MB` 配置）

#### `executed_effects` - 已执行效果
- **类型：** `DBMap<TransactionDigest, TransactionEffectsDigest>`
- **键：** 交易摘要
- **值：** 交易效果摘要
- **说明：** 跟踪在本节点本地执行的交易，用于等待交易执行完成

#### `events` - 事件存储（已弃用）
- **类型：** `DBMap<(TransactionEventsDigest, usize), Event>`
- **说明：** 旧的事件存储方式，已弃用

#### `events_2` - 事件存储 V2
- **类型：** `DBMap<TransactionDigest, TransactionEvents>`
- **键：** 交易摘要
- **值：** 交易产生的所有事件
- **说明：** 当前使用的事件存储，按交易摘要索引

#### `unchanged_loaded_runtime_objects` - 未变更的运行时对象
- **类型：** `DBMap<TransactionDigest, Vec<ObjectKey>>`
- **键：** 交易摘要
- **值：** 对象键列表
- **说明：** 跟踪加载但未变更的运行时对象引用

#### `executed_transactions_to_checkpoint` - 已执行交易到检查点映射（已弃用）
- **类型：** `DBMap<TransactionDigest, (EpochId, CheckpointSequenceNumber)>`
- **说明：** 已移至每轮次存储

#### `root_state_hash_by_epoch` - 轮次根状态哈希
- **类型：** `DBMap<EpochId, (CheckpointSequenceNumber, GlobalStateHash)>`
- **键：** 轮次 ID
- **值：** 最后检查点序号 + 全局状态哈希
- **说明：** 每个轮次的最终根状态哈希，每轮次写入一次，永不更改

#### `epoch_start_configuration` - 轮次启动配置
- **类型：** `DBMap<(), EpochStartConfiguration>`
- **键：** 单元类型（单条记录）
- **值：** 轮次配置
- **说明：** 单例表，存储轮次启动参数

#### `pruned_checkpoint` - 已裁剪检查点
- **类型：** `DBMap<(), CheckpointSequenceNumber>`
- **说明：** 单例表，跟踪最新裁剪的检查点，用于对象裁剪器进度跟踪

#### `expected_network_sui_amount` - 预期网络 SUI 总量
- **类型：** `DBMap<(), u64>`
- **说明：** 单例表，存储网络中预期的 SUI 总量，用于轮次结束时的昂贵检查

#### `expected_storage_fund_imbalance` - 预期存储基金不平衡
- **类型：** `DBMap<(), i64>`
- **说明：** 存储基金和存储退款之间的预期不平衡，用于处理早期协议版本的错误

#### `object_per_epoch_marker_table` - 每轮次对象标记表
- **类型：** `DBMap<(EpochId, ObjectKey), MarkerValue>`
- **说明：** 跟踪每轮次接收/删除的对象，防止接收对象时的竞态条件，按轮次裁剪

#### `object_per_epoch_marker_table_v2` - 每轮次对象标记表 V2
- **类型：** `DBMap<(EpochId, FullObjectKey), MarkerValue>`
- **说明：** 使用完整对象键的 V2 版本

#### `executed_transaction_digests` - 已执行交易摘要
- **类型：** `DBMap<(EpochId, TransactionDigest), ()>`
- **说明：** 跨轮次跟踪已执行的交易，轮次前缀键支持高效范围删除，支持地址余额 gas 支付功能

---

## 2. AuthorityPrunerTables

**位置：** `crates/sui-core/src/authority/authority_store_tables.rs`

**用途：** 为裁剪器组件跟踪已裁剪的对象。

### 表结构

#### `object_tombstones` - 对象墓碑
- **类型：** `DBMap<ObjectID, SequenceNumber>`
- **键：** 对象 ID
- **值：** 对象被删除时的序列号
- **说明：** 记录已裁剪对象的墓碑

---

## 3. AuthorityEpochTables

**位置：** `crates/sui-core/src/authority/authority_per_epoch_store.rs`

**用途：** 每轮次存储，在轮次边界重置。

### 表结构

#### `signed_transactions` - 已签名交易
- **类型：** `DBMap<TransactionDigest, TrustedEnvelope<SenderSignedData, AuthoritySignInfo>>`
- **键：** 交易摘要
- **值：** 带权威签名的已签名交易数据
- **说明：** 在 transaction_lock 中找到的交易

#### `owned_object_locked_transactions` - 拥有对象锁定交易
- **类型：** `DBMap<ObjectRef, LockDetailsWrapper>`
- **键：** 对象引用
- **值：** 锁定详情
- **说明：** 将对象引用映射到锁定它们的交易

#### `effects_signatures` - 效果签名
- **类型：** `DBMap<TransactionDigest, AuthoritySignInfo>`
- **说明：** 交易效果的签名，存储以避免重复签名相同效果

#### `signed_effects_digests` - 已签名效果摘要
- **类型：** `DBMap<TransactionDigest, TransactionEffectsDigest>`
- **说明：** 记录效果摘要以检测等价性

#### `transaction_cert_signatures` - 交易证书签名
- **类型：** `DBMap<TransactionDigest, AuthorityStrongQuorumSignInfo>`
- **说明：** 本地执行的交易证书的签名

#### `next_shared_object_versions_v2` - 下一个共享对象版本 V2
- **类型：** `DBMap<ConsensusObjectSequenceKey, SequenceNumber>`
- **说明：** 每个共享对象的下一个可用版本

#### `consensus_message_processed` - 共识消息已处理
- **类型：** `DBMap<SequencedConsensusTransactionKey, bool>`
- **说明：** 跟踪已处理的共识交易，确保 next_shared_object_versions 每个交易只推进一次

#### `pending_consensus_transactions` - 待处理共识交易
- **类型：** `DBMap<ConsensusTransactionKey, ConsensusTransaction>`
- **说明：** 此权威提交到共识的待处理交易

#### `last_consensus_stats` - 最后共识统计
- **类型：** `DBMap<u64, ExecutionIndicesWithStats>`
- **说明：** 单例表，存储最新共识消息索引和运行哈希

#### `reconfig_state` - 重配置状态
- **类型：** `DBMap<u64, ReconfigState>`
- **说明：** 验证者的当前重配置状态

#### `end_of_publish` - 发布结束
- **类型：** `DBMap<AuthorityName, ()>`
- **说明：** 在此轮次发送 EndOfPublish 消息的验证者

#### `builder_digest_to_checkpoint` - 构建器摘要到检查点
- **类型：** `DBMap<TransactionDigest, CheckpointSequenceNumber>`
- **说明：** 检查点构建器的内部交易→检查点映射

#### `transaction_key_to_digest` - 交易键到摘要
- **类型：** `DBMap<TransactionKey, TransactionDigest>`
- **说明：** 执行后将非摘要交易键映射到摘要

#### `pending_checkpoint_signatures` - 待处理检查点签名
- **类型：** `DBMap<(CheckpointSequenceNumber, u64), CheckpointSignatureMessage>`
- **说明：** 存储待处理的检查点签名

#### `builder_checkpoint_summary_v2` - 构建器检查点摘要 V2
- **类型：** `DBMap<CheckpointSequenceNumber, BuilderCheckpointSummary>`
- **说明：** 轮次内构建的检查点摘要

#### `state_hash_by_checkpoint` - 检查点状态哈希
- **类型：** `DBMap<CheckpointSequenceNumber, GlobalStateHash>`
- **说明：** 每个检查点的累积状态哈希（仅追加）

#### `running_root_state_hash` - 运行中根状态哈希
- **类型：** `DBMap<CheckpointSequenceNumber, GlobalStateHash>`
- **说明：** 运行中（未最终确定）的根状态累加器

#### `authority_capabilities` - 权威能力（仅 tidehunter）
- **类型：** `DBMap<AuthorityName, AuthorityCapabilitiesV1>`
- **说明：** 权威能力 V1

#### `authority_capabilities_v2` - 权威能力 V2
- **类型：** `DBMap<AuthorityName, AuthorityCapabilitiesV2>`
- **说明：** 每个权威公布的能力

#### `override_protocol_upgrade_buffer_stake` - 覆盖协议升级缓冲质押
- **类型：** `DBMap<u64, u64>`
- **说明：** 单例表，覆盖协议升级缓冲质押

#### `executed_transactions_to_checkpoint` - 已执行交易到检查点
- **类型：** `DBMap<TransactionDigest, CheckpointSequenceNumber>`
- **说明：** 通过检查点执行器执行的交易

#### `pending_jwks` - 待处理 JWKs
- **类型：** `DBMap<(AuthorityName, JwkId, JWK), ()>`
- **说明：** 已投票但尚未激活的 JWKs

#### `active_jwks` - 活跃 JWKs
- **类型：** `DBMap<(u64, (JwkId, JWK)), ()>`
- **说明：** 当前用于 zklogin 认证的活跃 JWKs

#### `deferred_transactions_v2` - 延迟交易 V2
- **类型：** `DBMap<DeferralKey, Vec<TrustedExecutableTransaction>>`
- **说明：** 延迟到未来时间的交易

#### `dkg_processed_messages_v2` - DKG 已处理消息 V2
- **类型：** `DBMap<PartyId, VersionedProcessedMessage>`
- **说明：** 从其他节点处理的 DKG 消息

#### `dkg_used_messages_v2` - DKG 已使用消息 V2
- **类型：** `DBMap<u64, VersionedUsedProcessedMessages>`
- **说明：** 用于生成 DKG 确认的消息

#### `dkg_confirmations_v2` - DKG 确认 V2
- **类型：** `DBMap<PartyId, VersionedDkgConfirmation>`
- **说明：** 从其他节点接收的 DKG 确认

#### `dkg_output` - DKG 输出
- **类型：** `DBMap<u64, dkg_v1::Output<PkG, EncG>>`
- **说明：** DKG 完成后的最终输出

#### `randomness_next_round` - 随机性下一轮
- **类型：** `DBMap<u64, RandomnessRound>`
- **说明：** 要生成的下一个随机性轮次

#### `randomness_highest_completed_round` - 随机性最高完成轮次
- **类型：** `DBMap<u64, RandomnessRound>`
- **说明：** 最高完成的随机性轮次

#### `randomness_last_round_timestamp` - 随机性最后轮次时间戳
- **类型：** `DBMap<u64, TimestampMs>`
- **说明：** 最近随机性轮次的时间戳

#### `congestion_control_object_debts` - 拥塞控制对象债务
- **类型：** `DBMap<ObjectID, CongestionPerObjectDebt>`
- **说明：** 拥塞控制的每对象债务

#### `congestion_control_randomness_object_debts` - 拥塞控制随机性对象债务
- **类型：** `DBMap<ObjectID, CongestionPerObjectDebt>`
- **说明：** 随机性特定的对象债务

#### `execution_time_observations` - 执行时间观察
- **类型：** `DBMap<(u64, AuthorityIndex), Vec<(ExecutionTimeObservationKey, Duration)>>`
- **说明：** 用于拥塞控制的执行时间观察

#### `deferred_transactions_with_aliases_v2` - 带别名的延迟交易 V2
- **类型：** `DBMap<DeferralKey, Vec<TrustedExecutableTransactionWithAliases>>`
- **说明：** 带对象别名的延迟交易

---

## 4. CheckpointStoreTables

**位置：** `crates/sui-core/src/checkpoints/mod.rs`

**用途：** 存储检查点相关数据。

### 表结构

#### `checkpoint_content` - 检查点内容
- **类型：** `DBMap<CheckpointContentsDigest, CheckpointContents>`
- **说明：** 将检查点内容摘要映射到内容

#### `checkpoint_sequence_by_contents_digest` - 按内容摘要的检查点序号
- **类型：** `DBMap<CheckpointContentsDigest, CheckpointSequenceNumber>`
- **说明：** 将内容摘要映射到序列号

#### `full_checkpoint_content` - 完整检查点内容（已弃用）
- **类型：** `DBMap<CheckpointSequenceNumber, FullCheckpointContents>`
- **说明：** 来自状态同步的完整检查点内容，状态累积完成后删除

#### `certified_checkpoints` - 已认证检查点
- **类型：** `DBMap<CheckpointSequenceNumber, TrustedCheckpoint>`
- **说明：** 按序列号存储已认证的检查点

#### `checkpoint_by_digest` - 按摘要的检查点
- **类型：** `DBMap<CheckpointDigest, TrustedCheckpoint>`
- **说明：** 将检查点摘要映射到已认证检查点

#### `locally_computed_checkpoints` - 本地计算的检查点
- **类型：** `DBMap<CheckpointSequenceNumber, CheckpointSummary>`
- **说明：** 本地计算的检查点摘要，用于分叉检测，验证后可裁剪

#### `epoch_last_checkpoint_map` - 轮次最后检查点映射
- **类型：** `DBMap<EpochId, CheckpointSequenceNumber>`
- **说明：** 每个轮次的最后检查点序列号

#### `watermarks` - 水位线
- **类型：** `DBMap<CheckpointWatermark, (CheckpointSequenceNumber, CheckpointDigest)>`
- **说明：** 跟踪最高已验证、已同步和已执行的检查点

#### `transaction_fork_detected` - 检测到交易分叉
- **类型：** `DBMap<u8, (TransactionDigest, TransactionEffectsDigest, TransactionEffectsDigest)>`
- **说明：** 存储交易分叉检测信息

#### `full_checkpoint_content_v2` - 完整检查点内容 V2
- **类型：** `DBMap<CheckpointSequenceNumber, VersionedFullCheckpointContents>`
- **说明：** V2 版本的完整检查点内容存储

---

## 5. CommitteeStoreTables

**位置：** `crates/sui-core/src/epoch/committee_store.rs`

**用途：** 存储每轮次的委员会信息。

### 表结构

#### `committee_map` - 委员会映射
- **类型：** `DBMap<EpochId, Committee>`
- **键：** 轮次 ID
- **值：** 委员会信息
- **优化：** 针对点查询优化（64MB 块缓存）

---

## 6. IndexStoreTables (RPC 索引)

**位置：** `crates/sui-core/src/rpc_index.rs`

**用途：** 为 RPC 查询提供索引（所有者查找、余额等）。

### 表结构

#### `meta` - 元数据
- **类型：** `DBMap<(), MetadataInfo>`
- **说明：** 单例表，存储数据库元数据和版本

#### `watermark` - 水位线
- **类型：** `DBMap<Watermark, CheckpointSequenceNumber>`
- **说明：** 跟踪最高已索引和已裁剪的检查点

#### `epochs` - 轮次
- **类型：** `DBMap<EpochId, EpochInfo>`
- **说明：** 轮次的额外元数据

#### `transactions` - 交易
- **类型：** `DBMap<TransactionDigest, TransactionInfo>`
- **说明：** 交易的额外元数据

#### `owner` - 所有者索引
- **类型：** `DBMap<OwnerIndexKey, OwnerIndexInfo>`
- **键：** (所有者地址, 对象类型, 倒序余额, 对象 ID)
- **值：** 对象版本
- **说明：** 对象所有权索引

#### `dynamic_field` - 动态字段
- **类型：** `DBMap<DynamicFieldKey, ()>`
- **说明：** 动态字段（子对象）索引

#### `coin` - 代币
- **类型：** `DBMap<CoinIndexKey, CoinIndexInfo>`
- **说明：** 带元数据的代币类型索引

#### `balance` - 余额
- **类型：** `DBMap<BalanceKey, BalanceIndexInfo>`
- **说明：** 每个所有者和代币类型的余额索引，使用合并操作符高效更新余额

#### `package_version` - 包版本
- **类型：** `DBMap<PackageVersionKey, PackageVersionInfo>`
- **说明：** 包版本索引

#### `events_by_stream` - 按流的事件
- **类型：** `DBMap<EventIndexKey, ()>`
- **说明：** 按流的已认证事件索引

---

## 7. IndexStoreTables (JSON-RPC 索引，遗留)

**位置：** `crates/sui-core/src/jsonrpc_index.rs`

**用途：** 为 JSON-RPC 提供遗留索引（正在逐步淘汰）。

### 表结构

#### `meta` - 元数据
- **类型：** `DBMap<(), MetadataInfo>`
- **说明：** 数据库元数据

#### `transactions_from_addr` - 来自地址的交易
- **类型：** `DBMap<(SuiAddress, TxSequenceNumber), TransactionDigest>`
- **说明：** 由地址发起的交易

#### `transactions_to_addr` - 发送到地址的交易
- **类型：** `DBMap<(SuiAddress, TxSequenceNumber), TransactionDigest>`
- **说明：** 发送到地址的交易

#### `transactions_by_input_object_id` - 按输入对象 ID 的交易（已弃用）
- **类型：** `DBMap<(ObjectID, TxSequenceNumber), TransactionDigest>`

#### `transactions_by_mutated_object_id` - 按变更对象 ID 的交易（已弃用）
- **类型：** `DBMap<(ObjectID, TxSequenceNumber), TransactionDigest>`

#### `transactions_by_move_function` - 按 Move 函数的交易
- **类型：** `DBMap<(ObjectID, String, String, TxSequenceNumber), TransactionDigest>`
- **说明：** 按 Move 函数调用的交易

#### `transaction_order` - 交易顺序
- **类型：** `DBMap<TxSequenceNumber, TransactionDigest>`
- **说明：** 所有已索引交易的顺序

#### `transactions_seq` - 交易序列
- **类型：** `DBMap<TransactionDigest, TxSequenceNumber>`
- **说明：** 交易摘要到序列号的映射

#### `owner_index` - 所有者索引
- **类型：** `DBMap<OwnerIndexKey, ObjectInfo>`
- **说明：** 对象所有权索引

#### `coin_index_2` - 代币索引 2
- **类型：** `DBMap<CoinIndexKey2, CoinInfo>`
- **说明：** 带余额排序的代币索引

#### `address_balances` - 地址余额
- **类型：** `DBMap<(SuiAddress, TypeTag), ()>`
- **说明：** 地址余额的简单存在性检查

#### `dynamic_field_index` - 动态字段索引
- **类型：** `DBMap<DynamicFieldKey, DynamicFieldInfo>`
- **说明：** 动态字段索引

#### `event_order` - 事件顺序
- **类型：** `DBMap<EventId, EventIndex>`
- **说明：** 事件排序

#### `event_by_move_module` - 按 Move 模块的事件
- **类型：** `DBMap<(ModuleId, EventId), EventIndex>`
- **说明：** 按 Move 模块的事件

#### `event_by_move_event` - 按 Move 事件的事件
- **类型：** `DBMap<(StructTag, EventId), EventIndex>`
- **说明：** 按 Move 事件类型的事件

#### `event_by_event_module` - 按事件模块的事件
- **类型：** `DBMap<(ModuleId, EventId), EventIndex>`
- **说明：** 按事件模块的事件

#### `event_by_sender` - 按发送者的事件
- **类型：** `DBMap<(SuiAddress, EventId), EventIndex>`
- **说明：** 按发送者的事件

#### `event_by_time` - 按时间的事件
- **类型：** `DBMap<(u64, EventId), EventIndex>`
- **说明：** 按时间戳的事件

#### `pruner_watermark` - 裁剪器水位线
- **类型：** `DBMap<(), TxSequenceNumber>`
- **说明：** 裁剪器水位线

---

## 8. RocksDBStore (共识存储)

**位置：** `consensus/core/src/storage/rocksdb_store.rs`

**用途：** Mysticeti 共识协议存储。

### 表结构

#### `blocks` - 区块
- **类型：** `DBMap<(Round, AuthorityIndex, BlockDigest), Bytes>`
- **键：** (轮次, 权威索引, 区块摘要)
- **值：** 序列化的区块
- **说明：** 按引用存储 SignedBlock
- **优化：** 针对大区块的写入吞吐量优化（128KB）

#### `digests_by_authorities` - 按权威的摘要
- **类型：** `DBMap<(AuthorityIndex, Round, BlockDigest), ()>`
- **说明：** 按作者排序引用的二级索引，从 "digests" 重命名

#### `commits` - 提交
- **类型：** `DBMap<(CommitIndex, CommitDigest), Bytes>`
- **说明：** 将提交索引映射到提交数据

#### `commit_votes` - 提交投票
- **类型：** `DBMap<(CommitIndex, CommitDigest, BlockRef), ()>`
- **说明：** 收集提交的投票

#### `commit_info` - 提交信息
- **类型：** `DBMap<(CommitIndex, CommitDigest), CommitInfo>`
- **说明：** 用于恢复的提交相关信息

#### `finalized_commits` - 已最终确定的提交
- **类型：** `DBMap<(CommitIndex, CommitDigest), BTreeMap<BlockRef, Vec<TransactionIndex>>>`
- **说明：** 将已最终确定的提交映射到被拒绝的交易

---

## 9. WritePathPendingTransactionTable

**位置：** `crates/sui-storage/src/write_path_pending_tx_log.rs`

**用途：** 去重交易提交处理。

### 表结构

#### `logs` - 日志
- **类型：** `DBMap<TransactionDigest, TrustedEnvelope<SenderSignedData, EmptySignInfo>>`
- **说明：** 用于崩溃恢复的待处理交易

---

## 10. BridgeOrchestratorTables

**位置：** `crates/sui-bridge/src/storage.rs`

**用途：** 跨链桥协调器状态。

### 表结构

#### `pending_actions` - 待处理操作
- **类型：** `DBMap<BridgeActionDigest, BridgeAction>`
- **说明：** 尚未执行的待处理跨链桥操作

#### `sui_syncer_cursors` - Sui 同步器游标
- **类型：** `DBMap<Identifier, EventID>`
- **说明：** 模块标识符到最后处理的 EventID

#### `eth_syncer_cursors` - ETH 同步器游标
- **类型：** `DBMap<AlloyAddressSerializedAsEthers, u64>`
- **说明：** 合约地址到最后处理的区块

#### `sui_syncer_sequence_number_cursor` - Sui 同步器序列号游标
- **类型：** `DBMap<(), u64>`
- **说明：** 要处理的下一个跨链桥记录的序列号

---

## 11. PackageStoreTables

**位置：** `crates/sui-analytics-indexer/src/package_store/mod.rs`

**用途：** 分析索引器的包存储。

### 表结构

#### `packages` - 包
- **类型：** `DBMap<ObjectID, Object>`
- **说明：** 按 ID 存储包对象

---

## 12. Indexer-Alt Consistent Store

**位置：** `crates/sui-indexer-alt-consistent-store/src/db/mod.rs`

**用途：** 支持基于快照读取的新索引器。

### 特殊列族

#### `$watermark` - 水位线
- **说明：** 管理每个管道的检查点水位线

#### `$restore` - 恢复
- **说明：** 跟踪恢复进度

**注意：** 这是一个通用数据库包装器，允许管道动态定义自己的列族。

---

## 关键架构模式

### 1. 存储分离

- **持久化存储：** 在轮次变更中存活的数据（AuthorityPerpetualTables）
- **每轮次存储：** 在轮次边界重置（AuthorityEpochTables）
- **检查点存储：** 检查点特定数据
- **索引存储：** 查询优化索引

### 2. 优化策略

#### 块缓存调优
不同的表根据访问模式使用不同的缓存大小：
- **Objects:** 5GB（读密集型）
- **Locks:** 1GB
- **Transactions:** 512MB（点查询）
- **Effects:** 1GB（点查询）

#### 写入优化
- 共识存储针对写入吞吐量优化

#### 压缩过滤器
- Objects 表使用自定义压缩过滤器进行裁剪

### 3. 裁剪支持

- **轮次前缀键：** 支持高效范围删除
- **水位线跟踪：** 跟踪裁剪进度
- **墓碑表：** 跟踪已删除的对象

### 4. 一致性机制

- **原子批量写入：** 带水位线
- **基于快照的读取：** indexer-alt
- **分叉检测表：** 检测和记录分叉

### 5. 特殊功能

#### 合并操作符
- Balance 索引使用合并操作符进行高效更新

#### Tidehunter 支持
- 某些表支持替代存储后端

#### 双重实现
- 某些表有 V2 版本用于迁移

---

## 表统计

Sui 在其各个组件中使用约 **80+ 个不同的 RocksDB 列族**，每个都针对其特定的访问模式和数据生命周期需求进行了精心优化。

### 按组件分类

1. **AuthorityPerpetualTables:** 17 个表
2. **AuthorityPrunerTables:** 1 个表
3. **AuthorityEpochTables:** 30+ 个表
4. **CheckpointStoreTables:** 10 个表
5. **CommitteeStoreTables:** 1 个表
6. **IndexStoreTables (RPC):** 10 个表
7. **IndexStoreTables (JSON-RPC 遗留):** 17 个表
8. **RocksDBStore (共识):** 6 个表
9. **WritePathPendingTransactionTable:** 1 个表
10. **BridgeOrchestratorTables:** 4 个表
11. **PackageStoreTables:** 1 个表
12. **Indexer-Alt Consistent Store:** 动态列族

---

## 相关文件路径

### 核心存储
- `crates/sui-core/src/authority/authority_store_tables.rs` - 权威存储表
- `crates/sui-core/src/authority/authority_per_epoch_store.rs` - 每轮次存储
- `crates/sui-core/src/checkpoints/mod.rs` - 检查点存储
- `crates/sui-core/src/epoch/committee_store.rs` - 委员会存储

### 索引
- `crates/sui-core/src/rpc_index.rs` - RPC 索引
- `crates/sui-core/src/jsonrpc_index.rs` - JSON-RPC 索引（遗留）

### 共识
- `consensus/core/src/storage/rocksdb_store.rs` - 共识存储

### 其他
- `crates/sui-storage/src/write_path_pending_tx_log.rs` - 待处理交易日志
- `crates/sui-bridge/src/storage.rs` - 跨链桥存储
- `crates/sui-analytics-indexer/src/package_store/mod.rs` - 包存储
- `crates/sui-indexer-alt-consistent-store/src/db/mod.rs` - 新索引器

---

## 总结

Sui 的 RocksDB 存储架构展示了以下关键特性：

1. **分层设计：** 持久化、每轮次、检查点和索引层分离
2. **性能优化：** 针对不同访问模式的块缓存调优
3. **可扩展性：** 支持裁剪和高效范围删除
4. **一致性：** 原子写入、快照读取和分叉检测
5. **灵活性：** 支持多个存储后端和动态列族

这种设计使 Sui 能够高效处理高吞吐量的交易处理，同时保持数据一致性和可查询性。
