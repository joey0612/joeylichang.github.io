crates/storage/provider/
├── src/
│   ├── lib.rs                    # 主入口文件，导出公共 API
│   ├── traits/                   # 核心 trait 定义
│   │   ├── mod.rs               # trait 模块入口
│   │   ├── full.rs              # FullProvider trait
│   │   └── static_file_provider.rs # 静态文件提供者 trait
│   ├── providers/               # 具体提供者实现
│   │   ├── mod.rs               # 提供者模块入口
│   │   ├── blockchain_provider.rs # 区块链提供者（主要实现）
│   │   ├── consistent.rs        # 一致性提供者
│   │   ├── consistent_view.rs   # 一致性视图
│   │   ├── database/            # 数据库相关提供者
│   │   │   ├── mod.rs           # 数据库模块入口
│   │   │   ├── provider.rs      # 数据库提供者实现
│   │   │   ├── builder.rs       # 提供者构建器
│   │   │   ├── chain.rs         # 链相关功能
│   │   │   └── metrics.rs       # 指标收集
│   │   ├── state/               # 状态提供者
│   │   │   ├── mod.rs           # 状态模块入口
│   │   │   ├── latest.rs        # 最新状态提供者
│   │   │   ├── historical.rs    # 历史状态提供者
│   │   │   └── macros.rs        # 状态相关宏
│   │   └── static_file/         # 静态文件提供者
│   │       ├── mod.rs           # 静态文件模块入口
│   │       ├── manager.rs       # 静态文件管理器
│   │       ├── writer.rs        # 静态文件写入器
│   │       ├── jar.rs           # JAR 文件处理
│   │       └── metrics.rs       # 静态文件指标
│   ├── writer/                  # 统一写入器
│   │   └── mod.rs               # UnifiedStorageWriter
│   ├── bundle_state/            # Bundle 状态处理
│   │   ├── mod.rs               # Bundle 状态模块入口
│   │   └── state_reverts.rs     # 状态回滚处理
│   └── test_utils/              # 测试工具

数据输入 → codecs (编码) → db (存储) → provider (访问) → storage-api (接口)

storage-api (顶层抽象)
↓
provider (统一接口)
↓
db-api (数据库抽象)
↓
db (MDBX 实现)
↓
libmdbx-rs (底层绑定)

storage-api - 存储 API 抽象层
定义存储系统的核心 trait 和类型
主要组件：
    Traits: 定义数据访问接口（BlockReader, AccountReader, StateProvider 等）
    Types: 定义存储相关的数据类型
    Abstractions: 提供存储抽象，支持多种后端
主要 trait 包括：
  - BlockReader: 区块数据读取
  - AccountReader: 账户数据读取
  - StateProvider: 状态数据提供
  - HeaderProvider: 区块头数据提供
  - TransactionsProvider: 交易数据提供
  - ReceiptProvider: 收据数据提供

provider - 数据提供者
统一的数据访问接口实现
主要组件：
    BlockchainProvider: 区块链数据提供者
    DatabaseProvider: 数据库数据提供者
    StateProvider: 状态数据提供者
    StaticFileProvider: 静态文件数据提供者
支持多种数据源：
  - 数据库存储
  - 静态文件存储
  - 内存缓存
  - 统一访问接口

db-api - 数据库抽象层
提供数据库操作的抽象接口
主要组件：
    Database: 数据库抽象
    Transaction: 事务抽象（DbTx, DbTxMut）
    Cursor: 游标抽象（DbCursorRO, DbCursorRW）
    Table: 表抽象
    Walker: 数据遍历抽象
支持两种表类型：
    - 简单 KV 表：一个键对应一个值
    - Dup 表：一个键对应多个值
事务保证：
    - 原子性操作
    - 强一致性保证
    - 支持读写分离

db - MDBX 数据库实现
功能：基于 MDBX 的具体数据库实现
主要组件：
    MDBX 实现: 高性能的键值存储引擎
    数据库管理: 数据库创建、打开、关闭
    事务管理: 读写事务处理
    测试工具: 数据库测试辅助工具
主要功能：
  - 数据库环境管理
  - 事务处理
  - 游标操作
  - 性能指标收集
  - 锁文件管理

libmdbx-rs - MDBX 绑定
Rust 语言的 MDBX 数据库绑定
主要组件：
    Environment: 数据库环境
    Transaction: 事务处理
    Cursor: 游标操作
    Database: 数据库管理
MDBX 特性：
  - 高性能键值存储
  - ACID 事务支持
  - 内存映射
  - 并发访问支持

db-models - 数据库模型
功能：定义数据库中的数据结构
主要组件：
    Accounts: 账户数据模型
    Blocks: 区块数据模型
    Client Version: 客户端版本信息
数据模型：
  - 账户状态模型
  - 区块数据模型
  - 交易数据模型
  - 收据数据模型

db-common - 数据库通用操作
功能：提供数据库操作的通用工具
主要组件：
    Database Tools: 数据库工具
    Initialization: 数据库初始化
    Common Operations: 通用操作
通用功能：
  - 数据库初始化
  - 数据导入导出
  - 数据库维护工具
  - 通用查询操作


codecs - 数据编码
功能：提供高效的数据序列化和反序列化
主要组件：
    Compact: 紧凑编码格式
    Compression: 数据压缩
    Serialization: 序列化支持
nippy-jar - 不可变数据存储格式
功能：专门为不可变数据设计的存储格式
主要组件：
    NippyJar: 主要存储容器
    Compression: 支持多种压缩算法
    Columnar Storage: 列式存储
    Memory Mapping: 内存映射访问
zstd-compressors - ZSTD 压缩器
功能：提供 ZSTD 压缩和解压缩功能
主要组件：
    Transaction Compressor: 交易压缩器
    Receipt Compressor: 收据压缩器
    Dictionary Compression: 字典压缩




crates/trie/
├── common/          # 通用类型和工具
├── db/             # 数据库集成
├── trie/           # 核心 MPT 实现
├── sparse/         # 稀疏 MPT 实现
├── sparse-parallel/ # 并行稀疏 MPT
└── parallel/       # 并行 trie 计算

  common
  /  |  \
 /   |   \
/    |    \
db    trie  sparse
|     |     |
|     |     |
|     |     |
parallel  sparse-parallel

crates/trie/trie/src/
├── lib.rs              # 主模块入口
├── trie.rs             # 核心 MPT 实现 (StateRoot, StorageRoot)
├── trie_cursor/        # Trie 游标实现
│   ├── mod.rs          # 游标工厂和接口
│   ├── in_memory.rs    # 内存游标实现
│   ├── subnode.rs      # 子节点游标
│   ├── noop.rs         # 空操作游标
│   └── mock.rs         # 测试用模拟游标
├── hashed_cursor/      # 哈希状态游标
│   ├── mod.rs          # 哈希游标工厂和接口
│   ├── post_state.rs   # 后状态游标实现
│   ├── noop.rs         # 空操作游标
│   └── mock.rs         # 测试用模拟游标
├── walker.rs           # Trie 遍历器
├── node_iter.rs        # 节点迭代器
├── proof/              # 证明生成
│   ├── mod.rs          # 证明生成器
│   └── trie_node.rs    # Trie 节点证明
├── witness.rs          # 见证生成
├── forward_cursor.rs   # 前向内存游标
├── progress.rs         # 进度跟踪
├── stats.rs            # 统计信息
├── metrics.rs          # 监控指标
└── test_utils.rs       # 测试工具

StateRoot/StorageRoot (根计算器)
↓ 使用
TrieWalker (遍历器)
↓ 使用
TrieCursor (游标接口)
↓ 实现
├── InMemoryTrieCursor (内存游标)
├── SubNodeCursor (子节点游标)
└── NoopTrieCursor (空操作游标)

TrieNodeIter (节点迭代器)
↓ 使用
├── TrieWalker (遍历器)
└── HashedCursor (哈希游标)
↓ 实现
├── PostStateHashedCursor (后状态游标)
└── NoopHashedCursor (空操作游标)

Proof (证明生成器)
↓ 使用
├── TrieCursorFactory (游标工厂)
└── HashedCursorFactory (哈希游标工厂)

StateRoot::root()
↓
TrieWalker::new() 创建遍历器
↓
TrieNodeIter::new() 创建迭代器
↓
TrieNodeIter::try_next() 迭代节点
↓
TrieWalker::advance() 遍历 trie
↓
TrieCursor::seek/next() 游标操作
↓
HashBuilder 构建哈希
↓
返回状态根

Proof::account_proof()
↓
Proof::multiproof() 生成多证明
↓
TrieWalker 遍历 trie
↓
ProofRetainer 保留证明节点
↓
返回 AccountProof


walker

1. 初始化：创建根节点，压入栈
2. 循环遍历：
   a. 检查是否可以跳过当前节点
   b. 如果不能跳过且子节点在 trie 中：
    - 如果是父分支节点：移动到下一个兄弟
    - 如果是子节点：消费下一个节点（深度扩展）
      c. 如果可以跳过：移动到下一个兄弟
      d. 更新跳过标志
3. 直到栈为空或遍历完成

节点跳过：如果节点没有变更且哈希已知，跳过详细遍历


TrieNodeIter

1. 检查 walker 键：
    - 如果可以跳过当前节点，返回分支节点
    - 否则继续

2. 检查哈希条目：
    - 如果有当前哈希条目，检查键的顺序
    - 如果 walker 键小于哈希条目键，继续
    - 否则返回叶子节点

3. 处理查找和推进：
    - 如果有上次哈希键，从该位置恢复
    - 否则获取下一个未处理的键
    - 推进 walker 并查找哈希条目

4. 重复直到没有更多节点

// HashBuilder 内部自动处理：
// - 分支节点 → RLP 编码的分支节点
// - 叶子节点 → RLP 编码的叶子节点  
// - 扩展节点 → 通过路径压缩自动生成
// - 哈希节点 → 通过缓存机制处理
6. 总结
   正确的理解：
   HashBuilder 不构建节点结构体（如 BranchNode、ExtensionNode）
   HashBuilder 直接构建 RLP 编码（如 [v0, v1, ..., v15, value]）
   RLP 编码格式完全符合以太坊 MPT 规范
   通过 RLP 编码保证以太坊兼容性
   关键点：
   HashBuilder 是一个 RLP 编码构建器，不是节点结构构建器
   它跳过了节点结构这一层，直接构建最终的 RLP 编码
   这样既保证了兼容性，又提高了性能
   你的第二个理解完全正确！

crates/trie/sparse/src/
├── lib.rs              # 主模块入口
├── state.rs            # 稀疏状态 trie 实现 (SparseStateTrie)
├── trie.rs             # 稀疏 trie 核心实现 (SparseTrie, SerialSparseTrie)
├── traits.rs           # 稀疏 trie 接口定义 (SparseTrieInterface)
├── provider.rs         # trie 节点提供者
└── metrics.rs          # 性能指标

SparseStateTrie
├── state: SparseTrie<A> (账户 trie)
│   ├── Blind(Option<Box<SerialSparseTrie>>)
│   └── Revealed(Box<SerialSparseTrie>)
│       └── SerialSparseTrie
│           ├── nodes: HashMap<Nibbles, SparseNode>
│           ├── values: HashMap<Nibbles, Vec<u8>>
│           └── prefix_set: PrefixSetMut
├── storage: StorageTries<S> (存储 trie 集合)
│   └── tries: B256Map<SparseTrie<S>>
└── revealed_account_paths: HashSet<Nibbles>


ParallelSparseTrie
├── upper_subtrie: SparseSubtrie (深度 0-1)
│   ├── path: Nibbles
│   ├── nodes: HashMap<Nibbles, SparseNode>
│   └── inner: SparseSubtrieInner
│       ├── values: HashMap<Nibbles, Vec<u8>>
│       └── buffers: SparseSubtrieBuffers
├── lower_subtries: [LowerSparseSubtrie; 256] (深度 2+)
│   ├── LowerSparseSubtrie::Blind(Option<Box<SparseSubtrie>>)
│   └── LowerSparseSubtrie::Revealed(Box<SparseSubtrie>)
│       └── SparseSubtrie (同上)
└── 共享状态
├── prefix_set: PrefixSetMut
├── updates: Option<SparseTrieUpdates>
└── update_actions_buffers: Vec<Vec<SparseTrieUpdatesAction>>





/// Helper function to find the common prefix length between two byte slices
fn prefix_len(a: &[u8], b: &[u8]) -> usize {
let min_len = a.len().min(b.len());
for i in 0..min_len {
if a[i] != b[i] {
return i;
}
}
min_len
}
