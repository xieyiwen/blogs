---
layout: post
title: Level存储引擎
categories: ["后端技术", "Storage Engine", "LevelDB"]
date: 2022-07-11
keywords: levelDB
---

LevelDB是一个磁盘上的Key-value存储数据库,本文讨论其存储引擎LSM Tree在LevelDB中的应用。

## 1 LevelDB介绍
LevelDB 使用 LSM(Log structured merge) Tree作为其数据结构。和B+ Tree等数据存储结构不同，LSM tree是一种文件存储结构。

特点是：分层、有序、针对块存储设备特点而设计的存储结构。通过牺牲部分读性能来提升写性能。 这意味着它在内存中执行大部分工作，并偶尔将数据同步到磁盘。写入速度很快，但读取速度很慢。

RocksDB是LevelDB的进化版，增加了事务、bloomFilter、列族、TTL、合并运算等功能。

RocksDB使用LSM Tree作为其数据结构，是LevelDB的升级版。

## 2 LSM

**核心理论基础**: 利用顺序写来提升写性能。磁盘的顺序写速度比随机写速度快非常多，即使是SSD，由于块擦除和垃圾回收的影响，顺序写速度还是比随机写速度快很多。

<img src="/assets/images/20220711/lsmt.png" width="60%">

##### 三个重要组成部分
1. MemTable
> 存在于内存中的数据存储结构，用于缓存更新的数据。用于内存不可靠，所以需要通过WAL的辅助来实现数据的可靠性。 MemTable内置排序，有不同实现，LevelDB用的跳表（SkipList）.
   
2. Immutable MemTable
> 不可变的MemTable，用于MemTable落盘的中间状态。目的是释放MemTable，使主程序的写操作不被阻塞。MemTable在一定时间或者存储的数据size达到阈值将转变为Immutable MemTable，等待落盘。

3. SSTable
> MemTable在磁盘上的有序存储（Sorted String Table）。SSTable有层级的设定，其中level0是immutable写入的层级，所以虽然SSTable中的key有序，但是level0整体无序，且可能重叠。level0之外的层级通过SSTable的合并产生，且之后层级的key不重复，整体有序。

LSM更新数据，不修改原来的数据，而是将新数据写到一个新的SSTable(Sorted String Table)中；删除数据，也是将为数据创建一个删除标志写入到新的SSTable中，保证数据是顺序写入到磁盘中。

同时，LSM Tree会合并多个SSTable，删除重叠数据，从而减少SSTable的数量和大小（这是Merge的由来）。为了防止合并操作影响主线程的读写性能，LevelDB采用一个单一的异步线程来完成。
合并的过程会产生新的SSTable，新的SSTable创建成功后，会删除被合并的SSTable文件。由于设计到sst文件的变更，所以每次合并会生成新的manifest文件，记录sst文件的新路径。

WAL是对LSM Tree的补充，相当于binlog，用于当程序崩溃时，将内存中的数据重放，以防数据丢失。

LSM Tree数据的写入顺序：WAL—> 内存MemTable。

当MemTable大小达到上限时，MemTable变为不可变的Immutable MemTable, ，`LSM Tree`一般会有一些与写入线程（或进程）相独立的背景线程（或进程）负责将`Immutable MemTable` flush到磁盘中，将数据持久化。写入到磁盘后`Immutable MemTable` 对应的WAL就可以从磁盘中删除了。

由于WAL是追加写入，SSTable是顺序写入，所以LSM Tree的写入速度是恒定的，和数据规模无关。

LSM Tree的数据查找顺序为：先在内存`MemTable`中查找，然后在内存中的`Immutable MemTable`中查找，然后在`level 0 SSTable`中查找，最后在`level N SSTable`中查找。确定到SSTable后，读取SSTable的Filter Block到内存中，然后通过BloomFilter快速判断数据是否在SSTable中，如果存在，则采用二分法确定数据在哪个数据block，然后将相应数据block读到内存中进行精确查找。

由上可知，LSM Tree查找性能有严重影响，有几种优化手段：

1. 压缩Block。同样的大小，Block放更多数据，减少读取次数
2. BloomFilter。通过读取block的元数据，快速定位数据是否在SSTable中，而不用读取Block了。
3. Cache。因为SSTable是不可变的，所以缓存到内存中，可以减少热点数据的反复读取磁盘。
4. SSTable合并。通过删除重叠数据，减少SSTable的大小和数量，从而减少查询规模。

其中前三种为数据库通用处理方式。

## 3 LMDB
LMDB可称为：闪电内存映射数据库（Lightning Memory-Mapped Database），是一种高度优化的事务数据库，采用键值存储形式，具有闪电般的读取速度和良好的写入性能。

其特点是：执行key和value的物理地址映射成内存的逻辑地址，所以读取时减少了数据的拷贝，从而性能快很多。

B+ 树是 LMDB 的数据结构.

## 4 参考
1. [Log-structured merge tree](https://www.wikiwand.com/en/Log-structured_merge-tree)
2. [LSM Tree原理详解](https://www.jianshu.com/p/b43b856e09bb)