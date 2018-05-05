---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：数据与存储
date: 2017-12-25 11:53:16
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

## 1. 内存数据

ZooKeeper 的数据模型是一棵树，而从使用角度看， Zookeeper 就像一个内存数据库一样。在这个内存数据库中，存储了整棵树的内容，包括所有的节点路径、节点数据及其 ACL 信息等，Zookeeper 会定时将这个数据存储到磁盘上。

<!--more-->

### 1.1 DataTree

`DataTree` 是内存数据存储的核心，是一个树结构，代表了内存中一份完整的数据。`DataTree` 不包含任何与网络、客户端连接及请求处理相关的业务逻辑，是一个独立的组件。

![diagram](https://user-images.githubusercontent.com/12514722/34336360-6306a5b4-e991-11e7-89ff-6d2e3e7e993e.jpg)

### 1.2 DataNode

`DataNode` 是数据存储的最小单元，其内部除了保存了节点的数据内容、ACL 列表、节点状态之外，还记录了父节点的引用和子节点列表两个属性。同时，`DataNode` 也提供了对子节点列表进行操作的接口。

### 1.3 nodes

`DataTree` 用于存储所有 ZooKeeper 节点的路径、数据内容及其 ACL 信息等，底层的数据结构其实是一个典型的 `ConcurrentHashMap` 键值对结构：

```java
private final ConcurrentHashMap<String, DataNode> nodes = new ConcurrentHashMap<String, DataNode>();
```

在 `nodes` 这个 Map 中，存放了 ZooKeeper 服务器上所有的数据节点，可以说，对于 ZooKeeper 数据的所有操作，底层都是对这个 Map 结构的操作。`nodes` 以数据节点的路径（path）为 key，value 则是节点的数据内容：`DataNode`。

另外，对于所有的临时节点，为了便于实时访问和及时清理，`DataTree` 中还单独将临时节点保存起来：

```java
private final Map<Long, HashSet<String>> ephemerals = new ConcurrentHashMap<Long, HashSet<String>>();
```

### 1.4 ZKDatabase

ZooKeeper 的内存数据库，负责管理 ZooKeeper 的所有会话、`DataTree` 存储和事务日志。`ZKDatabase` 会定时向磁盘 dump 快照数据，同时在 ZooKeeper 启动时，会通过磁盘的事务日志和快照文件恢复成一个完整的内存数据库。

## 2. 事务日志

### 2.1 日志格式

可以使用 `org.apache.zookeeper.server.LogFormatter` 将事务日志文件转换成可视化的事务操作日志。在转换结果中，每个事务日志第一行为文件头信息：

```
ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
```

之后每一行记录一次事务操作。

```
16-11-19 下午01时12分48秒 session 0x15554779e59002c cxid 0x0 zxid 0x600000075 createSession 40000
```

这一行就是一次客户端会话创建的事务操作日志，其中我们不难看出，从左向右分别记录事物操作时间、客户端会话 ID、CXID (客户端的操作序列号）、ZXID、操作类型和会话超时时间。

```
16-11-19 下午12时04分00秒 session 0x15554779e59000f cxid 0x1 zxid 0x600000027 create '/zookeeper/test,#2f7a6f6f6b65657065722f74657374,v{s{31,s{'digest,'foo:Jfg7TYUBs/6KEtdDWd5OB6bdD2Q=}}},F,2  
```

这一行是节点创建操作的事务操作日志，从左向右分別记录了事物操作时间、客户端会话 ID、CXID、ZXID、操作类型、节点路径、节点数据内容（#2f7a6f6f6b65657065722f74657374，在 `LogFormatter` 中使用如下格式输出节点内容：#+内容的 ASCII 码值）、节点的 ACL 信息、是否是临时节点（F 代表持久节点，T 代表临时节点）和父节点的子节点版本号。

注意，转换结果中没有显示每条事务操作记录的 Checksum 信息。

### 2.2 日志写入

`FileTxnLog` 负责维护事务日志对外的接口，包括事务日志的写入和读取等。将事务操作写入事务日志的工作主要由 `append` 方法来负责：

```java
    /**
     * append an entry to the transaction log
     * @param hdr the header of the transaction
     * @param txn the transaction part of the entry
     * returns true iff something appended, otw false 
     */
    public synchronized boolean append(TxnHeader hdr, Record txn)
        throws IOException
    {
        if (hdr == null) {
            return false;
        }

        if (hdr.getZxid() <= lastZxidSeen) {
            LOG.warn("Current zxid " + hdr.getZxid() + " is <= " + lastZxidSeen + " for " + hdr.getType());
        } else {
            lastZxidSeen = hdr.getZxid();
        }

        if (logStream==null) {
            logFileWrite = new File(logDir, ("log." + Long.toHexString(hdr.getZxid())));
            fos = new FileOutputStream(logFileWrite);
            logStream=new BufferedOutputStream(fos);
            oa = BinaryOutputArchive.getArchive(logStream);
            FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
            fhdr.serialize(oa, "fileheader");
            // Make sure that the magic number is written before padding.
            logStream.flush();
            currentSize = fos.getChannel().position();
            streamsToFlush.add(fos);
        }
        padFile(fos);
        byte[] buf = Util.marshallTxnEntry(hdr, txn);
        if (buf == null || buf.length == 0) {
            throw new IOException("Faulty serialization for header and txn");
        }
        Checksum crc = makeChecksumAlgorithm();
        crc.update(buf, 0, buf.length);
        oa.writeLong(crc.getValue(), "txnEntryCRC");
        Util.writeTxnBytes(oa, buf);

        return true;
    }
```

从方法定义中可以看到，ZooKeeper 在进行事务日志写入的过程中，会将事务头和事务体传给该方法。事务日志的写入过程大体可以分为如下 6 个步骤。

#### 2.2.1 确定是否有事务日志可写

当 ZooKeeper 服务器启动完成需要进行第一次事务日志的写入，或是上一个事务日志文件写满时，都会处于与事务日志文件断开的状态，即 ZooKeeper 服务器没有和任意一个日志文件相关联。因此在进行事务日志写入前，ZooKeeper 首先会判断 `FileTxnLog` 组件是否已经关联上一个可写的事务日志文件。若没有，则会使用该事务操作关联的 ZXID 作为后缀创建一个事务日志文件，同时构建事务日志的文件头信息，并立即写入这个事务日志文件中去。同时，将该文件的文件流放入 `streamToFlush` 集合，该集合用来记录当前需要强制进行数据落盘（将数据强制刷入磁盘上）的文件流。

#### 2.2.2 确定事务日志文件是否需要扩容（预分配）

当检测到当前务日志文件剩余空间不足 4096 字节（4KB）时，就会开始进行文件空间扩容，即，在现有文件大小的基础上，将文件大小增加 65536KB （64MB），然后使用 “0”（\0）填充这些被扩容的文件空间。

那么 ZooKeeper 为什么要进行事务日志文件的磁盘空间预分配呢？对客户端的每一次事务操作，ZooKeeper 都会将其写入事务日志文件中。因此，事物日志的写入性能直接决定了 ZooKeeper 服务器对事务请求的响应，也就是说，事务写入近似可以被看作是一个磁盘 I/O 的过程。严格地讲，文件的不断追加写入操作会触发底层磁盘 I/O 为文件开辟新的磁盘块，即磁盘 Seek。因此，为了避免磁盘 Seek 的频率，提高磁盘 I/O 的效率，ZooKeeper 在创建事物日志的时候就会进行文件空间 “预分配” —— 在文件创建之初就向操作系统预分配一个很大的磁盘块，默认是 64MB，而一旦已分配的文件空间不足 4KB 时，将会再次 “预分配”，以避免随着每次事务的写入过程中文件大小增长带来的 Seek 开销，直至创建新的事务日志。事务日志预分配的大小可以通过系统属性 `zookeeper.preAllocSize` 来进行设置。

#### 2.2.3 事务序列化

事务序列化包括对事务头和事务体的序列化，分别是对 `TxnHeader` （事务头）和 `Record` （事务体）的序列化。其中事务体又可分为会话创建事务（`CreateSessionTxn`）、节点创建事务（`CreateTxn`）、节点删除事务（`DeleteTxn`）和节点数据更新事务 （`SetDataTxn`）等。

#### 2.2.4 生成 Checksum

为了保证事务日志文件的完整性和数据的准确性，ZooKeeper 在将事务日志写入文件前，会根据事务序列化产生的字节数组来计算 Checksum。ZooKeeper 默认使用 Adler32 算法来计算 Checksum 值。

#### 2.2.5 写入事务日志文件流

将序列化后的事务头、事务体及 Checksum 值写入到文件流中去。此时由于 ZooKeeper 使用的是 `BufferedOutputStream`，因此写入的数据并非真正被写入到磁盘文件上。

#### 2.2.6 事务日志刷入磁盘

到此，已经将事务操作写入文件流中，但是由于缓存的原因，无法实时地写入磁盘文件中，因此我们需要将缓存数据强制刷入磁盘中。这里会从 `streamsToFlush` 中提取出文件流，并调用 `FileChannel.force(boolean metaData)` 接口来强制将数据刷入磁盘文件中去。`force` 接口对应的其实是底层的 `fsync` 接口，是一个比较耗费磁盘 I/O 资源的接口，因此 ZooKeeper 允许用户控制是否需要主动调用该接口，可以通过系统属性 `zookeeper.forceSync` 来设置。

### 2.3 日志截断

在 ZooKeeper 运行过程中，可能出现非 Leader 机器上记录的事务 ID（`peerLastZxid`） 比 Leader 服务器大的情况，这是一个非法的运行时状态。

一旦某台机器碰到这样的情况，Leader 会发送 `TRUNC` 命令给这个机器，要求进行日志截断。Learner 服务器在接收到该命令后，就会删除所有包含或大于 `peerLastZxid` 的事务日志文件。

## 3. snapshot——数据快照

数据快照用来记录 ZooKeeper 服务器上某一时刻的全量内存数据内容，并将其写入指定的磁盘文件中。快照文件也是使用 ZXID 的十六进制表示来作为文件名后缀，该后缀标识了本次数据快照开始时刻的服务器最新 ZXID。

可以使用 `org.apache.zookeeper.server.SnapshotFormatter` 将快照数据文件转换成可视化的数据内容。

### 3.1 数据快照

`FileSnap` 负责维护快照数据对外的接口，包括快照数据的写入和读取等。针对客户端的每一次事务操作，ZooKeeper 都会将它们记录到事务日志中，同时也会将数据变更应用到内存数据库中。ZooKeeper 在进行若干次事务日志记录之后，将内存数据库的全量数据 Dump 到本地文件中，这个过程就是数据快照。

#### 3.1.1 确定是否需要进行数据快照

每进行一次事务日志记录之后，ZooKeeper 都会检测当前是否需要进行数据快照。理论上进行 snapCount 次事务操作后就会开始数据快照，但是考虑到数据快照对 ZooKeeper 所在机器的整体性能的影响，需要尽量避免 ZooKeeper 集群中的所有机器在同一时刻进行数据快照。因此 ZooKeeper 在具体的实现中，并不是严格地按照这个策略执行的，而是采取 “过半随机” 策略，即符合如下条件就进行数据快照：
$$
logCount> (snapCount / 2 + randRoll)
$$
其中 logCount 代表了当前已经记录的事务日志数量，randRoll 为 1~snapCount/2 之间的随机数，因此上面的条件就相当于：如果我们配置的 snapCount 值为默认的 100000，那么 ZooKeeper 会在 50000~100000 次事务日志记录后进行一次数据快照。

#### 3.1.2 切换事务日志文件

满足上述条件之后，ZooKeeper 就要开始进行数据快照了。首先是进行事务日志文件的切换。所谓的事务日志文件切换是指当前的事务日志已经 “写满”（已经写入了 snapCount 个事务日志），需要重新创建一个新的事务日志。

#### 3.1.3 创建数据快照异步线程

为了保证数据快照过程不影响 ZooKeeper 的主流程，这里需要创建一个单独的异步线程来进行数据快照。

#### 3.1.4 获取全最数据和会话信息

数据快照本质上就是将内存中的所有数据节点信息（`DataTree`）和会话信息保存到本地磁盘中去。因此这里会先从 `ZKDatabase` 中获取到 `DataTree` 和会话信息。

#### 3.1.5 生成快照数据文件名

在这一步中， ZooKeeper 会根据当前已提交的最大 ZXID 来生成数据快照文件名。

#### 3.1.6 数椐序列化

接下来就开始真正的数据序列化了。在序列化时，首先会序列化文件头信息，这里的文件头和事务日志中的一致，同样也包含了魔数、版本号和 dbid 信息。然后再对会话信息和 `DataTree` 分別进行序列化，同时生成一个 Checksum，—并写入快照数据文件中去。

## 4. 初始化

### 4.1 初始化流程

#### 4.1.1 初始化FileTxnSnapLog

`FileTxnSnapLog` 是 ZooKeeper 事务日志和快照数据访问层，用于衔接上层业务与底层数据存储。底层数据包含了事务日志和快照数据两部分，因此 `FileTxnSnapLog` 内部又分为 `FileTxnLog` 和 `FileSnap` 的初始化，分别代表事务日志管理器和快找数据管理器的初始化。

#### 4.1.2 初始化 ZKDatabase

在初始化过程中，首先会构建一个初始化的 `DataTree`，同时将 `FileTxnSnapLog` 交付 `ZKDatabase`，以便内存数据库能够对事务日志和快照数据进行访问。在 `ZKDatabase` 初始化的时候，`DataTree` 也会进行相应的初始化工作，如创建一些 ZooKeeper 的默认节点，包括 `/`、`/zookeeper`、`/zookeeper/quota` 三个节点的创建。

除了 ZooKeeper 的数据节点，在 `ZKDatabase` 的初始化阶段还会创建一个用于保存所有客户端会话超时时间的记录器：`sessionWithTimeouts`。

#### 4.1.3 创建 PlayBackListener

`PlayBackListener` 监听器主要用来接收事务应用过程中的回调。在 ZooKeeper 数据恢复后期，会有一个事务订正的过程，在这个过程中，会回调 `PlayBackListener` 来进行对应的数据订正。

#### 4.1.4 处理快照文件

此时，ZooKeeper 可以开始从磁盘中恢复数据了，首先从快照文件开始加载。

#### 4.1.5 获取最新的 100 个快照文件

获取最新的至多 100 个快照文件。

#### 4.1.6 解析快照文件

ZooKeeper 逐个对快照文件进行解析，此时需要对其进行反序列化，生成 `DataTree` 和 `sessionsWithTimeouts`，同时还会进行文件的 `checksum` 校验以确定快照文件的正确性。

虽然获取到的是至多 100 个快照文件，但其实在这里的逐个解析过程中，如果正确性校验通过的话，那么通常只会解析最新的那个快照文件。换句话说，只有当最新的快照文件不可用的时候，才会逐个进行解析，直到将这 100 个文件全部解析完。如果将获取到的所有快照文件都解析完后还是无法成功恢复一个完整的 `DataTree` 和 `sessionWithTimeouts`，则认为无法从磁盘中加载数据，服务器启动失败。

#### 4.1.7 获取最新的 ZXID

此时根据快照文件的文件名就可以解析出一个最新的 ZXID：zxid_for_snap，该 ZXID 代表了 ZooKeeper 开始进行数据快照的时刻。

#### 4.1.8 处理事务日志

此时 ZooKeeper 服务器内存中已经有了一份近似全量的数据，现在开始通过事务日志来更新增量数据。

#### 4.1.9 获取所有 zxid_for_snap 之后提交的事务

从事务日志中获取所有 ZXID 比 zxid_for_snap 大的事务操作。

#### 4.1.10 事务应用

获取到所有 ZXID 大于 zxid_for_snap 的事务后，将其逐个应用到之前基于快照数据文件恢复出来的 `DataTree` 和 `sessionsWithTimeouts` 中去。每当有一个事务被应用到内存数据库中去后，ZooKeeper 同时会回调 `PlayBackListener`，将这一事务操作记录转换成 Proposal，并保存到 `ZKDatabase` 的 `committedLog` 中，以便 Follower 进行快速同步。

#### 4.1.11 再次获取最新的 ZXID

待所有的事务都被完整地应用到内存数据库中之后，基本上也就完成了数据的初始化过程，此时再次获取一个 ZXID，用来标识上次服务器正常运行时提交的最大事务 ID。

#### 4.1.12 校验 epoch

完成数据加载后，ZooKeeper 会从最新 ZXID 中解析出事务处理的 Leader 周期：epochOfZxid。同时也会从磁盘的 `currentEpoch` 和 `acceptedEpoch` 文件中读取上次记录的最新的 epoch 值，进行校验。

### 4.2 PlayBackListener

`PlayBackListener` 是一个事务应用监听器，用于在事务应用过程中的回调：每当成功将一条事务日志应用到内存数据库中后，就会调用这个监听器。其接口定义非常简单，只有一个方法：

```java
    public interface PlayBackListener {
        void onTxnLoaded(TxnHeader hdr, Record rec);
    }
```

用于对单条事务进行处理。在完成 `ZKDatabase` 的初始化后，ZooKeeper 会立即创建一个 `PlayBackListener` 监听器，并将其置于 `FileTxnSnapLog` 。

## 5. 数据同步

### 5.1 获取 Learner 状态

在注册 Learner 的最后阶段，Learner 服务器会发送给 Leader 服务器一个 `ACKEPOCH` 数据包，Leader 会从这个数据包中解析出该 Learner 的 `currentEpoch` 和 `lastZxid`。然后 Learner 调用 `org.apache.zookeeper.server.quorum.Learner#syncWithLeader` 等待同步开始。

### 5.2 数据同步初始化

在开始数据同步之前，Leader 服务器会进行数据同步初始化，首先会从 ZooKeeper 的内存数据库中提取出事务请求对应的提议缓存队列，同时完成对以下三个 ZXID 值的初始化。

-  `peerLastZxid`：该 Learner 服务器最后处理的 ZXID。
-  `minCommittedLog`：Leader 服务器提议缓存队列 `commitedLog` 中的最小 ZXID。
-  `maxCommittedLog`：Leader 服务器提议缓存队列 `commitedLog` 中的最大 ZXID。

ZooKeeper 集群数据同步通常分为四类，分别是直接差异化同步（DIFF 同步）、先回滚再差异化同步（TRUNC + DIFF 同步）、仅回滚同步（TRUNC 同步）、全量同步（SNAP 同步）。在初始化阶段，Leader 服务器会优先以全量同步方式来同步数据。同时，会根据 Leader 和 Learner 之间的数据差异情况来决定最终的数据同步方式。

### 5.3 同步模式

#### 5.3.1 直接差异化同步（DIFF 同步）

场景：`peerLastZxid` 介于 `minCommittedLog` 和 `maxCommittedLog` 之间。

Leader 服务器会首先向这个 Learner 发送一个 `DIFF` 指令，用于通知 Learner “进入差异化数据同步阶段，Leader 服务器即将把一些 Proposal 同步给自己”。在实际 Proposal 同步过程中，针对每个 Proposal，Leader 服务器都会通过发送两个数据包来完成，分别是 PROPOSAL 内容数据包和 COMMIT 指令数据包——这和 ZooKeeper 运行时 Leader 和 Follower 之间的事务提交过程是一致的。Leader 会将 PROPOSAL 数据包和 COMMIT 指令包暂时先放入 `queuedPackets` 队列中。

#### 5.3.2 先回滚再差异化同步（TRUNC + DIFF 同步）

场景：`peerLastZxid` 介于 `minCommittedLog` 和 `maxCommittedLog` 之间，但 Leader 发现某个 Learner 包含了一条自己没有的事务记录。

对于这个特殊场景，就使用先回滚再差异化同步（TRUNC + DIFF 同步）的方式。Leader 通过 `TRUNC` 指令让该 Learner 进行事务回滚，回滚到 Leader 服务器上存在的，同时也是最接近于 `peerLastzxid` 的 ZXID ，随后发送余下的差异数据。

#### 5.3.3 仅回滚同步（TRUNC 同步）

场景：`peerLastzxid` 大于 `maxCommittedLog`。

这种场景其实就是上述先回滚再差异化同步的简化模式，Leader 会要求 Learner 回滚到 ZXID 值为 `maxCommittedLog` 对应的事务操作。

#### 5.3.4 全量同步（SNAP 同步）

- 场景1：`peerLastZxid` 小于 `minCommittedLog`。
- 场景2：Leader 服务器上没有提议缓存队列，`peerLastZxid` 不等于 `lastProcessedZxid`（Leader 服务器数据恢复后得到的最大 ZXID）

上述这两个场景非常类似，在这两种场景下, Leader服务器都无法直接使用提议缓存队列和 Learner进行数据同步，因此只能进行全量同步（SNAP 同步）。

所谓全量同步就是 Leader 服务器将本机上的全量内存数据都同步给 Learner。Leader 服务器将会向 Learner发送一个 SNAP 指令，通知 Learner 即将进行全量数据同步。随后， Leader 会从内存数据库中获取到全量的数据节点和会话超时时间记录器，将它们序列化后传输给 Learner 。Learner 服务器接收到该全最数据后，会对其反序列化后载入到内存数据库中。

### 5.4 后续处理

在上面的步骤之后，Leader 已经完成了同步模式的选择。接下来：

1. Leader 将 Learner 加入到 `forwardingFollowers` 或 `observingLearners` 队列中。
2. Leader 将 `NEWLEADER` 指令添加到 `queuedPackets` 队列中。
3. Leader 将选出的同步模式发送给 Learner，其中包含了一个标志 ZXID：
   - 对于 DIFF 同步，该 ZXID 为 `maxCommittedLog`。
   - 对于 TRUNC 同步，该 ZXID 为 Learner 需要回滚到的那个 ZXID。
   - 对于 SNAP 同步，该 ZXID 为 `DataTree` 的最新事务 ZXID（这里不能设置为  `maxCommittedLog`，因为 `commitedLog` 队列可能为空）。此时，Leader 还会马上将全量数据序列化后发送给 Learner。
4. 然后，在一个新线程中，发送 `queuedPackets` 队列中的数据，包括 DIFF 和 DIFF + TRUNC 模式下要用到的差异化数据，以及最后的 `NEWLEADER` 指令。
5. Leader 和各个 LearnerHandler 在 `org.apache.zookeeper.server.quorum.Leader#waitForNewLeaderAck` 方法处同步等待半数以上 PARTICIPANT 对 `NEWLEADER` 指令做出响应。
6. 对于 Learner 而言，在同步数据之后，还会接收到来自 Leader 的 `NEWLEADER` 指令，此时 Learner 就会反馈给 Leader —个 `ACK` 消息，表明自己确实完成了对提议缓存队列中 Proposal 的同步。
7. Leader 在接收到来 Learner 的这个 `ACK` 消息以后，就认为当前 Learner 已经完成数据同步，同时进入 “过半策略” 等待阶段 —— Leader 会和其他 Learner 服务器进行上述同样的数据同步流程，直到集群中有过半的 PARTICIPANT 机器响应了 Leader 这个 `NEWLEADER` 消息。注意这里不考虑 Observer。
8. 一但满足 “过半策略” 后，Leader 服务器就会向所有已经完成数据同步的 Learner 发送一个 `UPTODATE` 指令，用来通知 Learner 已经完成了数据同步，同时集群中已经有过半机器完成了数据同步，集群已经具备了对外服务的能力了。
9. Learner 在接收到这个来自 Leader 的 `UPTODATE` 指令后，会终止数据同步流程，然后向 Leader 再次反馈一个 `ACK` 消息。

`org.apache.zookeeper.server.quorum.Leader#waitForNewLeaderAck`

```java
    /**
     * Process NEWLEADER ack of a given sid and wait until the leader receives
     * sufficient acks.
     *
     * @param sid
     * @param learnerType
     * @throws InterruptedException
     */
    public void waitForNewLeaderAck(long sid, long zxid, LearnerType learnerType)
            throws InterruptedException {
        synchronized (newLeaderProposal.ackSet) {
            if (quorumFormed) {
                return;
            }
            long currentZxid = newLeaderProposal.packet.getZxid();
            if (zxid != currentZxid) {
                LOG.error("NEWLEADER ACK from sid: " + sid
                        + " is from a different epoch - current 0x"
                        + Long.toHexString(currentZxid) + " receieved 0x"
                        + Long.toHexString(zxid));
                return;
            }
            if (learnerType == LearnerType.PARTICIPANT) {
                newLeaderProposal.ackSet.add(sid);
            }

            if (self.getQuorumVerifier().containsQuorum(
                    newLeaderProposal.ackSet)) {
                quorumFormed = true;
                newLeaderProposal.ackSet.notifyAll();
            } else {
                long start = Time.currentElapsedTime();
                long cur = start;
                long end = start + self.getInitLimit() * self.getTickTime();
                while (!quorumFormed && cur < end) {
                    newLeaderProposal.ackSet.wait(end - cur);
                    cur = Time.currentElapsedTime();
                }
                if (!quorumFormed) {
                    throw new InterruptedException("Timeout while waiting for NEWLEADER to be acked by quorum");
                }
            }
        }
    }
```

## 6. 自问自答

### 6.1 Observer 是否不接收 PROPOSAL 和 COMMIT 消息？

在同步阶段，如果 Learner 适用于 DIFF 同步，则 Leader 会把差量数据以 `PROPOSAL` 和 `COMMIT` 消息对的形式发送给 Learner，无论该 Learner 是 Follower 还是 Observer。

### 6.2 每个事务日志文件一定是 64M 大吗？

不一定。ZooKeeper 只会在需要进行数据快照时切换事务日志文件。假定 snapCount 为默认的 100000，且 randRoll 为50000，平均每条事务记录大小为 1KB。那么，在新的一次数据快照之前，事务日志文件已经记录了 $1KB * (100000 / 2 + 50000)=100MB$ 的数据，当前事务日志文件扩展到了 128MB。那么当执行数据快照、创建新的事务日志文件后，上一个事务日志文件大小就为 128MB。可见，事务日志文件的大小，归根结底取决于平均事务记录的大小与 randRoll 的值。