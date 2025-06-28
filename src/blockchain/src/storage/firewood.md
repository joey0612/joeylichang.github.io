# Firewood

Embedded key-value store, optimized to store recent Merkleized blockchain state with minimal overhead

## Introduction

1. It not built on top of the generic of KV store such as LevelDB/PebbleDB 
2. It's more like a B+-tree based database and directly uses the merkle trie structure as the index on-disk. 
3. It only store the latest state on-disk. In-place update rather than COW(copy-on-write)
4. It provide OS-level crash recovery via WAL(write-ahead log). WAL also offers "reversibility", means it can allow a fast in-memory rollback to recover some past versions of the entire store back in memory.

## Key Features

1. Merkle Trie Index

   Firewood uses the Merkle Patricia Trie (MPT) structure as its on-disk index, ensuring efficient storage and retrieval of the latest blockchain state. Firewood employs in-place updates rather than the copy-on-write (COW) method, minimizing overhead.

2. Write-Ahead Log (WAL)
   
    The Write-Ahead Log (WAL) provides OS-level crash recovery, ensuring data integrity even in the event of a system crash. The WAL also supports "reversibility," allowing fast in-memory rollbacks to recover past versions of the entire store.

3. Validation Storage
   
    Firewood optimizes for storage footprint and the performance of operations on the latest or recent states. It is ideal for validators and other use cases where access to the latest blockchain state is crucial. Firewood retains only the current trie and a changelog of historical account/storage (WAL) in files.

## Architecture

1. Proposal: An atomic batch of changes proposed against the latest committed revision or any existing Proposal.  (similar to difflayer  in PBSS)
    1. Supports multiple proposals created against the latest committed revision at the same time.
    2. Immutable, not allowed to be updated after creation
    3. Once a proposal committed, all other proposals that are not children of the committed one will be invalidated
2. Current Trie
    1. Only keep the latest version of the trie index on disk
    2. Instead of COW(copy on write), firewood use in-place updates for trie nodes of MPT.
3. Revision
    1. Keeps some configurable number of previous states in memory
    2. Keeps old WAL on disk to allow fast rollback

## Storage Model

Firewood's storage model is built on three layers of abstraction that completely decouple the layout of data on disk from its underlying logical structure. These layers include a linear, memory-like space, a persistent item storage stash, and the data structure itself. Through this abstraction, the actual data influencing the state of the data structure is represented by a byte vector in the linear space. This separation of logical data from its physical representation simplifies storage management, facilitates code reuse, and maintains flexibility.

## Page-based Shadowing and Revisions

Frequent writes to a continuous linear space lead to significant overhead. To improve write efficiency, Firewood divides the 64-bit virtual space into pages(4k) and tracks and records dirty pages in CachedStore.

The normal operation flow follows a Read-Modify-Write (RMW) style, different to the Copy-On-Write(COW) in Geth.
1. Traverse the trie, inducing access to nodes.
2. Bring necessary pages containing accessed nodes into memory and cache them (storage::CachedSpace).
3. Make changes to the trie, inducing writes to nodes. Nodes are either already cached or brought into memory.
4. Writes to nodes are converted into interval writes to the staging StoreRevMut space overlaying CachedSpace. Dirty pages during the write batch are captured in StoreRevMut.
5. Finally, the interval writes will be bundled into a single WAL record and persist into WAL file. then dirty pages are writtern into the store files.

## Conclusion

1. Rust, Current State, B+-Tree-like page-based storage
2. Use MPT as the index of database
3. Use 64-bits memory-like address space to store the two-layer MPT trie.
4. Aggregated interval writes on contiguous address space into pages and then persisted into store file  
5. Every state changSet for a block will be persisted into WAL and support )S-level recovery.