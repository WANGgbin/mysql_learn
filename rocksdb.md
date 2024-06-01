rocksdb 是 facebook 开源的一款基于 leveldb 的存储引擎。mysql 也接受 rocksdb 为存储引擎。本文描述 rocksdb 的相关内容。

# 事务

rocksdb 支持两种事务：悲观事务、乐观事务。

## 悲观事务

即悲观的认为存在冲突。因此在事务中的每个写操作，都会占据对应 key 的锁。如果获取不到锁，要么一直阻塞，要么超时。

### key 锁

rocksdb 维护了一个全局的锁信息，本质就是 key 到 Lock 的映射。

## 乐观事务

即乐观任务不存在冲突，直到事务提交的时候才进行冲突检查。这个冲突检查流程具体是什么样的呢？<br>

因为 rocksdb 通过追加的方式记录 key 的变化，因此同一个 key 只要有变更，就会记录一条数据。因为，在 rocksdb 中是天然支持 mvcc 的。具体实现中，rocksdb 会维护一个全局的 sequence(逻辑时钟，类似 etcd 中的 revision)，当进行写操作的时候，就会分配一个 sequence。key 的实际数据结构为：key + sequence。<br>

在事务中，对于变更的 key，就会记录原有的 sequence 信息，这样在提交事务的时候，就会判断 key 最新的 sequence 是否根之前记录的 sequence 一致，如果不一样，说明在此期间，该 key 有新变更，即发生了冲突，因此事务回滚，否则成功提交。<br>

## 原子性

类似于 etcd 的 STM 框架，rocksdb 中对于事务会维护一个 WriteBatch，存储当前事务的所有写操作，事务读取的时候也会先从 WriteBatch 读取，在最后提交的时候，才会将整个 WriteBatch 原子性的写入到 WAL 和 memtable 中。

## 隔离性

rocksdb 支持 read commited 和 repeatable read 两种隔离级别。

- 读提交

    如果是读提交，则每次读的时候，就会获取系统当前最新的 sequence number，然后获取 key sequence <= sequence 的记录，这就是读提交。

- 可重复读
 
    在创建事务的时候，就会获取系统当前的 sequence number，后续的每次读操作都是基于此 sequence number 读取，从而保证了可重复读。

# 数据结构

rocksdb 是一个适合**写多读少**的场景。我们通过解析 rocksdb 的数据结构来说明为什么。

## memtable

memtable 即内存数据结构。rocksdb 的写操作并不会直接落盘，而是写入到一个内存的数据结构中。该数据结构需支持快速查询。rocksdb 采用的 skip list 来实现该结构。<br>

因为写操作是直接写入到内存中的，所以 rocks 的写性能是很不错的。

## WAL

既然是直接写入到内存的，那就有数据丢失的风险。rocksdb 是如何解决的呢？通过 WAL。<br>

WAL 即 Write Ahead Log，是一个追加的 log，用于数据恢复。rocksdb 在执行写操作的时候，会先在 WAL 持久化一条日志，然后再写入到内存数据结构。<br>

## SSTable

memtable 毕竟是存放在内存中的，大小是有限制的。因此当 memtable 长度达到特定大小的时候，就会触发 flush 操作。将 memtable 内容 flush 到磁盘上。而磁盘上对应的数据结构就是 SSTable(Sorted String Table)。<br>

SSTable 中的 key 就是按照大小排序的，这很容易实现。因为 memtable 也是个排序结构，只需按序 flush 即可。<br>

可是如何在 SSTable 中查找数据呢？一种思路是在内存中维护每个 key 到 SSTable 文件中偏移量的映射，这样就可以通过二分查找定位到 key，然后根据偏移量从 SSTable 中读取数据。<br>

但内存大小是有限的，如果 SSTable 很大，这个内存索引也会占据很大内存，那怎么办呢？<br>

rocksdb 的思路是采用稀疏索引，即并不对每个 key 都维护一个索引，而是每一部分 key，只记录一个索引。在 rocksdb 中，这里的每一部分 key 就是一个 block，然后再 SSTable 有一个 data index block，该 block 每一项便是每个 data block 最后一个 key 及其偏移量的映射。这样通过该 data index block，就可以很快速的定位目标 key 所在的 block，然后将 block 加载进内存，再采用二分查找即可。

## Level Compaction

分层压缩要解决的问题是：通过一定程度的写放大，来提升读性能。<br>

想象两种极端情况：

- 磁盘中存放很多 SSTable，不进行任何 Compaction

    写性能很好，顶多 flush memtable 即可。没有额外的写放大。但是对于读操作性能就很差，需要依次遍历每一个 SSTable。

- 磁盘中只存放一个文件，每次 flush memtable 的时候，都需要 Compactin 到这个文件

    读性能较好，只需要从这个文件读取即可。但是对于写，每次 flush memtable 的时候，还需要执行 Compaction。写放大比较厉害导致写性能较差。

Level 指的是：将磁盘上的 SSTable 分成若干层，每往下一层容量都是指数级别增长的。除了 level0 层外，所有的层中的数据都是按序排序的，这样的好处当然是方便二分查找。<br>

当进行写操作的时候，只有当前层的容量超过限制的时候，才会通过某种策略，从当前层选择一个 SSTable，然后 Compaction 到下一层。所以写放大是有条件的。而在读的角度看，因为每一层都是有序的，因此可以根据二分查找快速查找到数据。

# DB 一致性

一致性是很重要的，不管是业务系统还是基建系统，很多时候就是在处理一致性，保证系统从一个一致的状态迁移到另一个一致的状态，一致性保证了系统的正确性。<br>

作为一个 db，rocks 也需要维护一致性状态，如何做的呢？<br>

rocks 提出了一个 MANIFEST 的概念。包括 manifest log 文件 和 一个指针文件，指针文件指向当前的 manifest log 文件。所谓 manifest log 文件，就是记录了 rocks 一系列 meta 信息的变更，同样通过追加的方式记录。manifest log 文件大小也是有限制的，当超过一定大小的时候，就需要压缩，压缩本质上就是创建个新的 manifest 文件，并记录当前 db 的 所有 meta 信息。当新的 manifest 文件落盘后，修改指针文件内容，此时就可以删除旧的 manifest log 文件。<br>

元信息就包括：
- db 的逻辑时钟：sequence number
- 每个 SSTable 文件元数据包括：最大 key，最小 key，文件所在的层

# 读/写流程

## 写

先在 WAL 中写入一条记录，然后更改 memtable。当 memtable 容量达到限制的时候，就会将 memtable flush 到磁盘上，可能还会触发 level compaction.

## 读

先从 memtable 读取，如果读取不到，则从磁盘读取。首先通过 manifest 文件获取当前层每一个 SStable 的最大 key、最小 key。然后通过二分查找的方式，定位到所在的 SStable 文件，然后读取 SSTable 的 data index block，通过二分查找定位到所在的 block，然后将 block 加载到内存中，再进行二分查找，如此循环，直到查找到数据或者数据不存在。


# 2pc

rocksdb 也支持 2pc(two phased commit)。两阶段分别为：prepared + commit。

- prepared

    此阶段，会将变更日志存储到 WAL 中，但并会将变更写入到 memtable 中。

- commit

    此阶段，首先会在 WAL 中追加 Commit 记录，然后将变更写入到 memtable 中。
