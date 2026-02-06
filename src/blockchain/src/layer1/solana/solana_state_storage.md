# Solana State Storage

## Overview

Solana 的存储系统由两个主要部分组成：
1. **Blockstore（区块存储）** - 存储区块链的历史数据（交易、区块、分片）
2. **AccountsDB（账户数据库）** - 存储账户状态数据

Solana 中 Slot 类比区块，在网络中 Slot 使用纠删码将数据非为数据块和编码块（单位是 Shred，大小为 MTU），通过 Turbine 在节点之间传输。内存进行缓存，组织 Shred 还原 Slot，并维护 Fork 等，底层使用 rocksdb 存储，这不是本文的重点，至此不在赘述。

```
Tick (最小时间单位)
  ↓ 64 ticks = 1 slot (~400ms)
Slot (一个 leader 的出块时间)
  ↓ 432,000 slots = 1 epoch (~2 days)
Epoch (leader 调度周期)
```

### Slot 数据与 State 数据分离
1. **不同的访问模式**:
   - Blockstore: 顺序写入，历史查询
   - AccountsDB: 随机读写，状态查询

2. **不同的生命周期**:
   - Blockstore: 永久保存（或归档）
   - AccountsDB: 需要定期清理旧版本

3. **不同的优化目标**:
   - Blockstore: 吞吐量和可靠性
   - AccountsDB: 延迟和并发性

## AccountDB

AccountDB 维护状态数据（即帐户数据，solana 使用帐户模型）。
核心设计理念：
1. 使用内存映射文件（memory-mapped files）存储账户数据，追加写最新版本，一步删除旧版本。
2. 全内存索引，租金模式使得全量索引规模可控，128G 内存完全 hold。
3. 支持并发单线程追加写入和多线程并发读取，只有索引需要写锁，数据写入是顺序的。

### AppendVec

AppendVec 实际存储账户数据的磁盘文件，内部是顺序写入的帐户数据。
```rust
pub struct AppendVec {
    path: PathBuf,                    // 磁盘文件路径（例如: /data/0.123）
    map: MmapMut,                     // 内存映射（将文件映射到内存地址空间）
    append_offset: Mutex<usize>,      // 追加偏移量（单线程写入）
    current_len: AtomicUsize,         // 当前已使用长度
    file_size: u64,                   // 文件总大小
}
```

AppendVec 文件内部布局
```
偏移量 0:    [StoredMeta]      - 元数据（write_version, pubkey, data_len）
偏移量 X:    [AccountMeta]     - 账户元数据（lamports, owner, executable, rent_epoch）
偏移量 Y:    [Hash]            - 账户哈希（32 字节）
偏移量 Z:    [账户数据]         - 实际的账户数据（data_len 字节）
偏移量 W:    [下一个账户...]   - 8 字节对齐后继续
```

1. 每个 AppendVec 只存储单个 slot 的账户
    1. 默认文件大小: 4MB
    2. 最大文件大小: 16GB 
    3. 数据对齐: 8 字节边界对齐
2. 写入是追加式的（append-only），不会修改已写入的数据
3. 通过内存映射实现零拷贝读取，所有的读都是引用文件的内存映射，必要时才进行 copy。

```rust
pub struct StoredMeta {
    pub write_version: u64,  // 全局写入版本号
    pub pubkey: Pubkey,      // 账户公钥
    pub data_len: u64,       // 数据长度
}
```

```rust
pub struct AccountMeta {
    pub lamports: u64,       // 账户余额（lamports）
    pub owner: Pubkey,       // 拥有者程序
    pub executable: bool,    // 是否可执行
    pub rent_epoch: Epoch,   // 下次支付租金的 epoch
}
```

#### StoredAccount

StoredAccount 是内存中帐户的表现形式，指向内存映射文件中账户数据的**引用集合**（零拷贝视图）。

```rust
/// References to Memory Mapped memory
/// The Account is stored separately from its data, so getting the actual account requires a clone
pub struct StoredAccount<'a> {
    pub meta: &'a StoredMeta,           // 引用：存储元数据
    pub account_meta: &'a AccountMeta,  // 引用：账户元数据
    pub data: &'a [u8],                 // 引用：账户数据（切片，不拥有数据）
    pub offset: usize,                  // 在 AppendVec 中的偏移量
    pub hash: &'a Hash,                 // 引用：账户哈希
}
```

**关键特性**:
- ✅ 所有字段都是**引用**（`&'a`），不拥有数据
- ✅ 数据实际存储在 AppendVec 的内存映射文件中
- ✅ **零拷贝**：直接指向 mmap 的内存，不需要复制
- ✅ 生命周期 `'a` 绑定到 AppendVec，确保内存安全
- ✅ 轻量级：只是一组指针，创建和传递非常快

### AccountsIndex
AccountsIndex 是内存中的索引结构，用于快速查找账户。
```rust
// accounts_index.rs:8-15
pub struct AccountsIndex<T> {
    // 核心索引：Pubkey → 该账户在不同 slot 的版本列表
    pub account_maps: HashMap<Pubkey, RwLock<SlotList<T>>>,

    pub roots: HashSet<Slot>,     // 已确认的根 slot 集合
    pub last_root: Slot,          // 最后的根 slot
}

type SlotList<T> = Vec<(Slot, T)>;  // 每个账户在不同 slot 的版本列表
```

**具体示例**:
```rust
// 假设账户 Pubkey_A 在多个 slot 中被更新过
account_maps = {
    Pubkey_A → RwLock<[
        (Slot_100, AccountInfo{store_id: 5, offset: 0, lamports: 1000}),
        (Slot_105, AccountInfo{store_id: 7, offset: 200, lamports: 1500}),
        (Slot_110, AccountInfo{store_id: 9, offset: 400, lamports: 2000}),
    ]>,

    Pubkey_B → RwLock<[
        (Slot_102, AccountInfo{store_id: 6, offset: 100, lamports: 500}),
    ]>,
    // ...
}
```

**AccountInfo**:
AccountInfo 是 AppendVec 的索引内容
```rust
// accounts_db.rs:71-81
pub struct AccountInfo {
    store_id: AppendVecId,  // 指向哪个 AppendVec 文件（文件标识符）
    offset: usize,          // 在该 AppendVec 文件中的字节偏移量
    lamports: u64,          // 余额（用于优化零余额账户的清理）
}

pub type AppendVecId = usize;  // AppendVec 的唯一标识符
```

1. `AccountInfo` 才是真正的"位置信息"
2. `store_id` 告诉你账户数据在哪个 AppendVec 文件中
3. `offset` 告诉你在该文件中的具体位置（字节偏移）
4. 通过 `(store_id, offset)` 可以直接定位并读取账户数据

### Compare

**AccountStorage（物理维度 - 按 Slot 组织）**:
```rust
AccountStorage {
    Slot_100 → {
        AppendVecId_5 → AccountStorageEntry { AppendVec: /data/slot_100/5.123 },
        AppendVecId_7 → AccountStorageEntry { AppendVec: /data/slot_100/7.456 },
    },
    Slot_105 → {
        AppendVecId_9 → AccountStorageEntry { AppendVec: /data/slot_105/9.789 },
    },
}
```

1. **按 Slot 组织** - 每个 slot 有自己的 AppendVec 文件集合
2. **物理存储** - 代表磁盘上的实际文件
3. **增量更新** - 每个 slot 只包含该 slot 中更新的账户
4. **用于计算 State Root** - 可以针对特定 slot 计算状态哈希

| 维度 | AccountStorage | AccountsIndex |
|------|----------------|---------------|
| **组织方式** | 按 Slot 组织 | 按 Pubkey 组织 |
| **数据范围** | 每个 slot 的增量更新 | 所有账户的全量历史 |
| **物理/逻辑** | 物理存储（文件） | 逻辑索引（内存） |
| **用途** | 存储实际数据 | 快速查找 |
| **State Root** | ✅ 可以针对特定 slot 计算 | ❌ 需要结合 ancestors 查询 |
| **持久化** | ✅ 磁盘文件 | ❌ 重启后重建 |

### Global Write Version

每个帐户有一个全局唯一的版本号，在更新的时候变更。

**特点**:
1. **每更新一个账户就加 1**（不是每个 slot 或 block）
2. 全局唯一且单调递增
3. 用于确定账户的写入顺序
4. 用于从持久化存储恢复索引时确定最新版本
5. 使用原子操作（`AtomicUsize`）保证线程安全

**用途**:
1. **版本控制**: 每个账户更新都有唯一的版本号
2. **恢复索引**: 从 AppendVec 文件恢复时，通过 write_version 确定哪个是最新版本

### Write/Read Flow

#### 写入流程
```
1. 账户更新 → AccountsDB
2. 分配 write_version（全局原子计数器）
3. 追加到 AppendVec（单线程，持有 Mutex）
4. 更新 AccountsIndex（需要写锁）
5. 返回 AccountInfo（store_id + offset）
```

#### 读取流程
```
1. 通过 Pubkey 查询 AccountsIndex
2. 根据 ancestors（祖先 slot 集合）找到最新版本
3. 获取 AccountInfo（store_id + offset）
4. 从对应的 AppendVec 读取数据（无锁，内存映射）
5. 反序列化并返回 Account
```

### Prune

1. **只回收已 rooted（类比 finalized block） 的旧版本** - 已确认的历史版本可以安全删除
2. **保留非 root slot 的版本** - 可能需要回滚到分叉
3. **零余额账户优先回收** - 不再活跃的账户
4. **整个 AppendVec 为空才删除文件** - 避免碎片化

### State Root

Solana 使用 XOR Hash，是一种累加式计算，需要输入帐户旧版本和新版本，可以维护全局状态，其中在计算时引入 BanchHash（PoH 相关，不是本文重点）。特点就是计算快，但是安全性没有那么高。

## Summary

**AccountsDB**:
- AppendVec 写入使用 Mutex（单线程追加）
- 读取完全无锁（内存映射 + 不可变数据）
- AccountsIndex 使用 RwLock（读多写少）