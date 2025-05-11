# MPT

## 优点（对比）
1. HashTable：为了防止篡改，需要计算 rootHash ，每次账户信息的更新（写入和更新）都需要扫描一遍全量账户信息，计算 rootHash。
2. Merkel Tree：主要是与 BTC 对比，BTC 是 Block 内，ETH 是世界状态，全局的，二者的量级是不一样的，这就需要局部更新（每个 Block 只更新部分账户的信息）影响范围足够小。

## 缺点
1. 空间浪费，中间节点需要存储
2. 磁盘IO方法，一次读写会涉及中间节点的读写

## HashNode的引入
在内存中操作节点的数据显然是比较浪费资源的（主要是memory），使用指针比较划算，但是跨节点之间指针又是不起作用的（因为同样的数据在不同节点内指针是不一样的，没有办法被唯一标识），所以引入了 HashNode，即hash指针，可以跨节点唯一标识一个节点，同时只有在使用的时候才从磁盘中解析出来。

## MPT 的组成概念
1. Radis Tree：压缩的前缀树。
2. Patricia Tree：插入的 key 数量 == 节点数量，在 MPT 中的体现是最后一个flag（17th）标识如果没有子节点可以合并到父亲节点，其他分支依然可以有子节点（这是也 PT + RadisTree 结合的体现）。
3. Merkel Tree：叶子节点是数据，中间节点是子节点计算的 hash

MPT = Radis + Patricia + Merkel Tree，上述三种 Tree 在 MPT  中都有所体现

## ETH-MPT

### 逻辑概念
1. extensionNode：公共前缀部分。
2. branchNode：分支节点，一定是 extensionNode 的子节点，如果 extensionNode 已经是一个完整的 key，那么最后一个分支（16进制应该是 16分叉，但是会有 17个分叉）记录了 extensionNode 对应的 value。
3. leafNode：叶子节点存储的是数据。

### 物理概念
物理概念即代码中的实现：
1. extensionNode 对应代码中的 shortNode
2. branchNode 对应代码中的 fullNode
3. leafNode 对应代码中的 valueNode
如果 extensionNode 是一个完整的 key，那么子节点的最后一个分支是 shortNode，shortNode 是 valueNode

代码中还有一个 hashNode，可以理解为节点的hash指针，fullNode 和 shortNode 中的子节点其实存储在磁盘都是 hashNode，需要的时候通过 hashNode（hash指针）去磁盘读到相应的数据。

shortNode 的子节点可能是 valueNode 或者 hashNode（可能是 shortNode 或者 fullNode）

_注意：valueNode 存储的是真正的数据，不是 hash 指针。_

### 编码
上述介绍的 node （在内存和磁盘上，物理实现上）都是有编码格式的，即从磁盘上读出来之后解析的时候都能够明确是那种node（prefix）是不同的。

除了内存和磁盘之间有编码，用户 account （即 用户侧 key）与内存中的key 也是有编码的。

raw（用户key） → hex编码（内存，扩展2倍，同时如果是叶子节点有 16 的结束符，保证了每一个 Byte 都可以用 16 进制表示没有益出）→ HP 编码（磁盘编码，加了前缀-奇偶等，方便 hash 计算和存储空间）

hex编码：保证了每一个 Byte 都可以用 16 进制表示没有益出

HP 编码：Hex 格式下的 Key，长度会变成是原来 keybytes 格式下的两倍。这一点对于节点的哈希计算，比如计算 hashNode，影响很大。HP 编码是一种将任意长度的nibble(4bit为一个nibble) 编码成byte数组的方法，也就是将最小粒度为4bit的数组编码成最小粒度为8bit数组。另一个问题，例如："5b3"->"5b30" 在 HP 编码中需要区分，因为在 tree 中是不一样的。

1. 扩展节点且nibbles数量为偶数，prefixes为0，二进制为:00000000;
2. 扩展节点且nibbles数量为奇数，prefixes为1，二进制为:0001;
3. 叶子节点且nibbles数量为偶数，prefixes为2，二进制为:00100000;
4. 叶子节点且nibbles数量为奇数，prefixes为3，二进制为:0011;

前缀之后，将 Hex 你编码即可获得原先的 raw 追加到 prefix

奇偶加的长短不一样，主要是为了保证最终的结果是偶数。

**另外：storageTree 和 accountTree 的 key 都是是 hash 值，即 32字节 256位，statedb 中的 trie 是 SecureTrie 类型，内部会将 account 的 key（20字节的 account） 进行 hash**

### 参考
1. https://zhuanlan.zhihu.com/p/46702178
2. https://www.jianshu.com/p/8b6d2a7fb6b5