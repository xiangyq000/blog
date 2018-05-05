---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：角色与消息
date: 2017-12-25 12:25:36
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

![image](https://user-images.githubusercontent.com/12514722/34335956-073d451a-e98e-11e7-8626-df89e4efd676.png)

<!--more-->

## 1. Leader

Leader服务器是 ZooKeeper 集群工作的核心，其主要工作如下：

- 事务请求的唯一调度和处理者，保证集群事务处理的顺序性。
- 集群内部各服务器的调度者。

ZooKeeper 使用责任链模式来处理每一个客户端的请求。

### 1.1 PrepRequestProcessor

Leader 服务器的请求预处理器。在 ZooKeeper 中，那些会改变服务器状态的请求被称为事务请求（创建节点、更新数据、删除节点、创建会话等）。`PrepRequestProcessor` 能够识别出当前客户端请求是否是事务请求。对于事务请求，`PrepRequestProcessor` 处理器会对其进行一系列预处理，如创建请求事务头、事务体、会话检查、ACL 检查和版本检查等。

### 1.2 ProposalRequestProcessor

Leader 服务器的事务投票处理器。Leader 服务器事务处理流程的发起者，对于非事务请求，`ProposalRequestProcessor` 会直接将请求转发到 `CommitProcessor` 处理器，不再做其他处理；而对于事务请求，除了将请求转发到 `CommitProcessor` 外，还会根据请求类型创建对应的 Proposal 提议，并发送给所有的 Follower 服务器来发起一次集群内的事务投票。同时，`ProposalRequestProcessor` 还会将事务请求交付给 `SyncRequestProcessor` 进行事务日志的记录。

### 1.3 SyncRequestProcessor

事务日志记录处理器，用来将事务请求记录到事务日志文件中，同时还会触发 ZooKeeper 进行数据快照。

### 1.4 AckRequestProcessor

Leader 特有的处理器，负责在 `SyncRequestProcessor` 完成事务日志记录后，向 Proposal 的投票收集器发送 ACK 反馈，以通知投票收集器当前服务器已经完成了对该 Proposal 的事务日志记录。

### 1.5 CommitProcessor

事务提交处理器。对于非事务请求，该处理器会直接将其交付给下一级处理器处理；对于事务请求，其会等待集群内针对 Proposal 的投票直到该 Proposal 可被提交。利用 `CommitProcessor`，每个服务器都可以很好地控制对事务请求的顺序处理。

### 1.6 ToBeAppliedRequestProcessor

该处理器中有一个 `toBeApplied` 队列，用来存储那些已经被 `CommitProcessor` 处理过的可被提交的 Proposal。`ToBeAppliedRequestProcessor` 处理器将这些请求逐个交付给 `FinalRequestProcessor` 处理器进行处理 —— 等到 `FinalRequestProcessor` 处理器处理完之后，再将其从 `toBeApplied队列` 中移除。

### 1.7 FinalRequestProcessor

用来进行客户端请求返回之前的收尾工作，包括创建客户端请求的响应；针对事务请求，该处理还会负责将事务应用到内存数据库中去。

## 2. Follower

Follower 是 ZooKeeper 集群的跟随者，其主要工作如下：

- 处理客户端非事务请求，转发事务请求给 Leader 服务器。
- 参与事务请求 Proposal 的投票。
- 参与 Leader 选举投票。

Follower 也采用了责任链模式组装的请求处理链来处理每一个客户端请求。

### 2.1 FollowerRequestProcessor

Follower 服务器的第一个请求处理器，其主要工作就是识别出当前请求是否是事务请求。如果是事务请求，那么 Follower 就会将该请求转发给 Leader 服务器，Leader 服务器在接收到这个事务请求后，就会将其提交到请求处理链，按照正常事务请求进行处理。

### 2.2 SendAckRequestProcessor

该处理器承担了事务日志记录反馈的角色，在完成事务日志记录后，会向 Leader 服务器发送 ACK 消息以表明自身完成了事务日志的记录工作。它与 `AckRequestProcessor` 的区别在于，`AckRequestProcessor` 处理器和 Leader 在同一个服务器上，因此它的 ACK 仅仅是一个本地的方法调用；而 `SendAckRequestProcessor` 处理器由于在 Follower 服务器上，因此需要通过以 ACK 消息的形式来向 Leader 服务器进行反馈。

## 3. Observer

Observer 充当观察者角色，观察 ZooKeeper 集群的最新状态变化并将这些状态同步过来。Observer 服务器再工作原理上和 Follower 是基本一致的，对于非事务请求，都可以进行独立的处理，而对于事务请求，则会转发给 Leader 服务器进行处理。区别在于，Observer 不参与任何形式的投票，包括事务请求 Proposal 的投票和 Leader 选举投票。简单地讲，Observer 服务器只提供非事务服务，通常用于在不影响集群事务处理能力的前提下提升集群的非事务处理能力。

需要注意的一点是，Observer 服务器再初始化阶段会将 `SyncRequestProcessor` 处理器也组装上去，但是在实际运行过程中，Leader 服务器不会将事务请求的投票发送给 Observer 服务器。

## 4. 集群间消息通信

ZooKeeper 的消息类型大体分为数据同步型、服务器初始化型、请求处理型和会话管理型。

### 4.1 数据同步型

数据同步型消息是指在 Learner 和 Leader 服务器进行数据同步时，网络通信所用到的消息，通常有DIFF、TRUNC、SNAP 和 UPTODATE 四种。

| 消息类型        | 发送方 → 接收方      | 说 明                                      |
| ----------- | -------------- | ---------------------------------------- |
| DIFF，13     | Leader→Learner | 用于通知 Learner 服务器，Leader 即将与其进行 DIFF 方式的数据同步 |
| TRUNC，14    | Leader→Learner | 用于触发 Learner 服务器进行内存数据库的回滚操作             |
| SNAP，15     | Leader→Learner | 用于通知 Learner 服务器，Leader 即将与其进行 “全量”  方式的数据同步 |
| UPTODATE，12 | Leader→Learner | 用来告诉 Learner 服务器，已经完成了数据同步，可以开始对外提供服务了   |

### 4.2 服务器初始化型

服务器初始化型消息是指在整个集群或是某些新机器初始化的时候，Leader 和 Learner 之间相互通信所使用的消息类型，常见的有 OBSERVERINFO、FOLLOWERINFO、 LEADERINFO、ACKEPOCH 和 NEWLEADER 五种。

| 消悤类型            | 发送方一接收方         | 说 明                                      |
| --------------- | --------------- | ---------------------------------------- |
| OBSERVERINFO，16 | Observer→Leader | 该信息通常是由 Observer 服务器在启动的时候发送给 Leader 的，用于向 Leader 服务器注册自己，同时向 Leader 服务器表明当前 Learner 服务器的角色是 Observer。消息中包含了当前 Observer 服务器的 SID 和已经处理的最新 ZXID |
| FOLLOWERINFO，11 | Follower→Leader | 该信息通常是由 Follower 服务器在启动的时候发送 给 Leader 的，用于向 Leader 服务器注册自己，同时向 Leader 服务器表明当前 Learner 服务器的角色是 Follower。消息中包含了当前 Follower 服务器的 SID 和已经处理的最新 ZXID |
| LEADERINFO，17   | Leader→Learner  | 在上面已经提到，在 Learner 连接上 Leader 后， 会向 Leader 发送 LearnerInfo 消息（包含了 OBSERVERINFO 和 FOLLOWERINFO 两类消息），Leader 服务器在接收到该消息后，也会将 Leader 服务器的基本信息发送给这些 Learner，这个消息就是 LEADERINFO， 通常包含了当前 Leader 服务器的最新 EPOCH 值 |
| ACKEPOCH，18     | Learner→Leader  | Learner 在接收到 Leader 发来的 LEADERINFO 消息后，会将自已更新后的 ZXID 和 EPOCH 以 ACKEPOCH 消息的形式发送给 Leader |
| NEWLEADER，10    | Leader→Learner  | 该消息通常用于 Leader 服务器向 Learner 发送一个阶段性的标识消息 —— Leader 会在和 Learner 完成一个交互流程后，向 Learner 发送 NEWLEADER 消息， 同时带上当前 Leader 服务器处理的最新 ZXID。这一系列交互流程包括：足够多的 Follower 服务器连接上 Leader 或是完成数据同步 |

### 4.3 请求处理型

请求处理型消息是指在进行请求处理的过程中，Leader 和 Learner 服务器之间互相通信所使用的消息，常见的有 REQUEST、PROPOSAL、ACK、COMMIT、INFORM 和 SYNC 六种。

| 消息类型       | 发送方—接收方         | 说 明                                      |
| ---------- | --------------- | ---------------------------------------- |
| REQUEST，1  | Learner→Leader  | 该消息是 ZooKeeper 的请求转发消息。在 ZooKeeper 中，所有的事务请求必由 Leader 服务器来处理。当 Learner 服务器接收到客户端的事务请求后，就会将请求以 REQUEST 消息的形式转发给 Leader 服务器来处理 |
| PROPOSAL，2 | Leader→Follower | 该消息是 ZooKeeper 实现 ZAB 算法的核心所在，即 ZAB  协议中的提议。在处理事务请求的时候，Leader 服务器会将事务请求以 PROPOSAL 消息的形式创建投票发送给集群中所有的 Follower 服务器来进行事务日志的记录 |
| ACK,3      | Follower→Leader | Follower 服务器在接收到来自 Leader 的 PROPOSAL 消息后，会进行事务日志的记录。如果完成了事务日志的记录，那么就会以 ACK 消息的形式反馈给 Leader |
| COMMITS    | Leader→Follower | 该消息用于通知集群中所有的 Follower 服务器，可以进行事务请求的提交了。Leader 服务器在接收到过半的 Follower 服务器发来的 ACK 消息后，就进入事务请求的最终提交流程 —— 生成 COMMIT 消息，告知所有的 Follower 服务器进行事务请求的提交 |
| INFORM，8   | Leade→Observer  | 在事务请求提交阶段，针对 Follower 服务器，Leader 仅仅只需要发送一个 COMMIT 消息，Follower 服务器就可以完成事务请求的提交了，因为在这之前的事务请求投票阶段，Follower 已经接收过 PROPOSAL 消息，该消息中包含了事务请求的内容，因此 Follower 可以从之前的 Proposal 缓存中再次获取到事务请求。而对于 Observer 来说，由于之前没有参与事务请求的投票，因此没有该事务请求的上下文，显然， 如果 Leader 同样对其发送一个简单的 COMMIT 消息， Observer 服务器是无法完成事务请求的提交的。为了解决这个问題，ZooKeeper 特别设计了 INFORM 消息，该消息不仅能够通知 Observer 已经可以提交事务请求，同时还会在消息中携带事务请求的内容 |
| SYNC，7     | Leader→Learner  | 该消息用于通知 Learner 服务器已经完成了 Sync 操作         |

### 4.4 会话管理型

会话管理型消息是指 ZooKeeper 在进行会话管理的过程中，和 Learner 服务器之间互相通信所使用的消息，常见的有 PING 和 REVALIDATE 两种。

| 消息类型         | 发送方一接收方        | 说 明                                      |
| ------------ | -------------- | ---------------------------------------- |
| PING，5       | Leader→Learner | 该消息用于 Leader 同步 Learner 服务器上的客户端心跳检测，用以激活存活的客户端。ZooKeeper 的客户端往往会随机地和任意一个 ZooKeeper 服务器保持连接，因此Leader 服务器无法直接接收到所有客户端的心跳检测，需要委托给 Learner 来保存这些客户端的心跳检测记录。Leader 会定时地向 Learner 服务器发送 PING 消息，Learner 服务器在接收到 PING 消息后，会将这段时间内保持心跳检测的客户端列表，同样以 PING 消息的形式反馈给 Leader 服务器，由 Leader 服务器来负责逐个对这些客户端进行会话激活。 |
| REVALIDATE，6 | Learner→Leader | 该消息用于 Learner 校验会话是否有效，同时也会激活会话，这通常发生在客户端重连的过程中，新的服务器需要向 Leader 发送 REVALIDATE 消息以确定该会话是否已经超时。 |