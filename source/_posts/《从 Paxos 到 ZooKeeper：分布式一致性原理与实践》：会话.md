---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：会话
date: 2017-12-26 20:38:35
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

## 1. 会话状态

在 ZooKeeper 客户端与服务端成功完成连接创建后，就创建了一个会话。ZooKeeper 会话在整个运行期间的生命周期中，会在不同的会话状态之间进行切换，这些状态可以一般可以分为 `CONNECTING`、`CONNECTED`、`RECONNECTING`、`RECONNECTED`、`CLOSE` 等。

一旦客户端开始创建 ZooKeeper 对象，那么客户端状态就会变成 `CONNECTING`，同时客户端开始从服务器地址列表中逐个选取 IP 地址来尝试进行网络连接，直到成功连接上服务器，然后将客户端状态变更为 `CONNECTED`。

通常情况下，伴随着网络闪断或是其他原因，客户端与服务端之间的连接会出现断开情况，一旦碰到这种情况，ZooKeeper 客户端会自动进行重连操作，同时客户端的状态再次变为 `CONNCTING`，直到重新连接上服务器后，客户端状态又会再次转变成 `CONNECTED`。因此，通常情况下，在 ZooKeeper 运行期间，客户端的状态总是介于 `CONNECTING` 和 `CONNECTED` 两者之一。

另外，如果出现诸如会话超时、权限检查失败或是客户端主动退出程序等情况，那么客户端的状态就会直接变更为 `CLOSE`。

<!--more-->

## 2. 会话创建

### 2.1 Session

`Session` 是ZooKeeper 中会话的实体，代表了一个客户端会话。其包含以下 4 个基本属性：

- `sessionlD`：会话 ID，用来唯一标识一个会话，每次客户端创建新会话的时候，ZooKeeper 都会为其分配一个全局唯一的 `sessionID`。
- `TimeOut`：会话超时时间。客户端在构造 ZooKeeper 实例的时候，会配置一个 `sessionTimeout` 参数用于指定会话的超时时间。ZooKeeper 客户端向服务器发送这个超时时间后，服务器会根据自己的超时时间限制最终确定会话的超时时间。
- `TickTime`：下次会话超时时间点。为了便于 ZooKeeper 对会话实行 “分桶策略” 管理，同时也是为了高效低耗地实现会话的超时检测与清理，ZooKeeper 会为每个会话标记一个下次会话超时时间点。`TickTime` 是一个 13 位的 `long` 型数据，其值接近于当前时间加上 `TimeOut`，但不完全相等。
- `isClosing`：该属性用于标记一个会话是否已经被关闭。通常当服务端检测到一个会话巳经超时失效的时候，会将该会话的 `isClosing` 属性标记为 “已关闭”，这样就能确保不再处理来自该会话的新请求了。

### 2.2 生成 sessionId

`org.apache.zookeeper.server.SessionTrackerImpl#initializeNextSession`

```java
public static long initializeNextSession(long id) {  
    long nextSid;  
    nextSid = (Time.currentElapsedTime() << 24) >>> 8;  
    nextSid =  nextSid | (id <<56);  
    return nextSid;  
}  
```

### 2.3 SessionTracker

`SessionTracker` 是 ZooKeeper 服务端的会话管理器。Leader 的 `SessionTracker` 会保存整个集群中的所有会话信息，负责会话的创建、管理和清理工作，其实现是 `SessionTrackerImpl`。Learner 的  `SessionTracker` 的实现是 `LearnerSessionTracker`，仅做记录客户端会话激活信息用。

每一个会话在 `SessionTrackerImpl` 内部都保留了三份：

- `sessionById`：这是一个 `HashMap<Long, SessionImpl>` 类型的数据结构，用于根据 `sessionid` 来管理 `Session` 实体。
- `sessionWithTimeout`：这是一个 `ConcurrentHashMap<Long, Integer>` 类型的数据结构，用于根据 `sessionId` 来管理会话的超时时间。该数据结构和 ZooKeeper 内存数据库相连通，会被定期持久化到快照文件中去。
- `sessionSets`：这是一个 `HashMap<Long, SessionSet>` 类型的数据结构，用于根据下次会话超时时间点来归档会话，便于进行会话管理和超时检查。

## 3. 会话管理

ZooKeeper 集群的会话管理由 Leader 统一处理。

### 3.1 分桶策略

ZooKeeper 采用了一种特殊的会话管理方式，我们称之为 “分桶策略”。所谓分桶策略，是指将类似的会话放在同一区块中进行管理，以便于 ZooKeeper 对会话进行不同区块的隔离处理以及同一区块的统一处理。

ZooKeeper 将所有的会话都分配在了不同的区块之中，分配的原则是每个会话的 “下次超时时间点”（ExpirationTime)。ExpirationTime 是指该会话最近一次可能超时的时间点，对于一个新创建的会话而言，其会话创建完毕后， ZooKeeper 就会为其计算 ExpirationTime，计算方式如下：
$$
ExpirationTime = CurrentTime + SessionTimeout
$$
在 ZooKeeper 的实际实现中，还做了一个处理。ZooKeeper 的 Leader 服务器在运行期间会定时地进行会话超时检査，其时间间隔是 ExpirationInterval，单位是毫秒，默认值是 `tickTime` 的值，即默认情况下，每隔 2000 毫秒进行一次会话超时检查。为了方便对多个会话同时进行超时检査，完整的 ExpirationTime 的计算方式如下：
$$
\begin{equation}
ExpirationTime\_  =  CurrentTime + SessionTimeout \\
ExpirationTime  = (ExpirationTime\_ /Expirationlnterval +1)*Expirationlnterval
\end{equation}
$$
最终计算出的 ExpirationTime 即为客户端会话下次超时时间所对应的 ”桶“（即，大于会话下次超时时间的最小 ExpirationInterval 整数倍）。同时，会话的 `tickTime` 属性被设置为 ExpirationTime，`SessionTracker` 以这个 ExpirationTime 为 key 将会话保存到 `sessionSets` 中。

`org.apache.zookeeper.server.SessionTrackerImpl`

```java
    synchronized public boolean touchSession(long sessionId, int timeout) {
        SessionImpl s = sessionsById.get(sessionId);
        // Return false, if the session doesn't exists or marked as closing
        if (s == null || s.isClosing()) {
            return false;
        }
        long expireTime = roundToInterval(Time.currentElapsedTime() + timeout);
        if (s.tickTime >= expireTime) {
            // Nothing needs to be done
            return true;
        }
        SessionSet set = sessionSets.get(s.tickTime);
        if (set != null) {
            set.sessions.remove(s);
        }
        s.tickTime = expireTime;
        set = sessionSets.get(s.tickTime);
        if (set == null) {
            set = new SessionSet();
            sessionSets.put(expireTime, set);
        }
        set.sessions.add(s);
        return true;
    }

    private long roundToInterval(long time) {
        // We give a one interval grace period
        return (time / expirationInterval + 1) * expirationInterval;
    }

    @Override
    synchronized public void run() {
        try {
            while (running) {
                currentTime = Time.currentElapsedTime();
                if (nextExpirationTime > currentTime) {
                    this.wait(nextExpirationTime - currentTime);
                    continue;
                }
                SessionSet set;
                set = sessionSets.remove(nextExpirationTime);
                if (set != null) {
                    for (SessionImpl s : set.sessions) {
                        setSessionClosing(s.sessionId);
                        expirer.expire(s);
                    }
                }
                nextExpirationTime += expirationInterval;
            }
        } catch (InterruptedException e) {
            handleException(this.getName(), e);
        }
        LOG.info("SessionTrackerImpl exited loop!");
    }
```

### 3.2 会话激活

在 ZooKeeper 的实际设计中，只要客户端有请求发送到服务端，那么就会触发一次会话激活。会话激活大体分为以下两种情况：

1. 客户端向服务端发送请求，包括读写请求，就么就会触发一次会话激活。

   这是通过在 `org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.Request)`  中调用 `touchSession` 来实现的。

2. 客户端发现在 sessionTimeout/3 时间内尚未和服务端进行过任何通信，那么就会主动发起一个 `PING` 请求，服务端收到该请求后，就会触发第一种情况下的会话激活，俗称 “心跳检测”。

会话激活的过程，不仅能够使服务端检测到对应客户端的存活性，同时也能让客户端自己保持连接状态。

- 如果客户端连接的是 Leader，那么由 Leader 的 `SessionTrackerImpl` 直接处理会话激活操作。

- 如果客户端连接的是 Learner，那么 Learner 将客户端的激活信息保存在 `touchTable` 中：

  `org.apache.zookeeper.server.quorum.LearnerSessionTracker#touchSession`

  ```java
      HashMap<Long, Integer> touchTable = new HashMap<Long, Integer>();
      synchronized public boolean touchSession(long sessionId, int sessionTimeout) {
          touchTable.put(sessionId, sessionTimeout);
          return true;
      }
  ```

  Leader 会定时通过 `PING` 消息从 Learner 那儿获取客户端激活信息，这个时候，Learner 会将 `touchTable` 的内容发送给 Leader，同时清空 `touchTable`。

  `org.apache.zookeeper.server.quorum.Learner#ping`

  ```java
      protected void ping(QuorumPacket qp) throws IOException {
          // Send back the ping with our session data
          ByteArrayOutputStream bos = new ByteArrayOutputStream();
          DataOutputStream dos = new DataOutputStream(bos);
          HashMap<Long, Integer> touchTable = zk
                  .getTouchSnapshot();
          for (Entry<Long, Integer> entry : touchTable.entrySet()) {
              dos.writeLong(entry.getKey());
              dos.writeInt(entry.getValue());
          }
          qp.setData(bos.toByteArray());
          writePacket(qp, true);
      }
  ```

  `org.apache.zookeeper.server.quorum.LearnerZooKeeperServer#getTouchSnapshot`

  ```java
      protected HashMap<Long, Integer> getTouchSnapshot() {
          if (sessionTracker != null) {
              return ((LearnerSessionTracker) sessionTracker).snapshot();
          }
          return new HashMap<Long, Integer>();
      }
  ```

  `org.apache.zookeeper.server.quorum.LearnerSessionTracker#snapshot`

  ```java
      synchronized HashMap<Long, Integer> snapshot() {
          HashMap<Long, Integer> oldTouchTable = touchTable;
          touchTable = new HashMap<Long, Integer>();
          return oldTouchTable;
      }
  ```

  Leader 会从 `PING` 消息的回复中，逐个取出 `sessionId` 及其 `sessionTimeout` 时间，做激活操作。

  `org.apache.zookeeper.server.quorum.LearnerHandler#run`

  ```java
      @Override
      public void run() {
          // ...
          while (true) {
              switch (qp.getType()) {
                  case Leader.PING:
                      // Process the touches
                      ByteArrayInputStream bis = new ByteArrayInputStream(qp.getData());
                      DataInputStream dis = new DataInputStream(bis);
                      while (dis.available() > 0) {
                          long sess = dis.readLong();
                          int to = dis.readInt();
                          leader.zk.touch(sess, to);
                      }
                      break;
               }
           }
       }
  ```

Leader 的会话激活操作分为以下几个步骤：

1. 检验该会话是否已经被关闭。
2. 计算该会话新的超时时间 ExpirationTime_New。
3. 获取该会话上次超时时间 ExpirationTime_Old。
4. 迁移会话。将该会话从老的区块中取出，放入 ExpirationTime_New 对应的新区块中。

### 3.3 会话超时检查

在 ZooKeeper 中，会话超时检査由 Leader 的  `SessionTrackerImpl` 负责。`SessionTrackerImpl` 中有一个单独的线程专门进行会话超时检査，其工作机制的核心思路非常简单：逐个依次地对会话桶中剩下的会话进行清理。

`ZooKeeperServer` 实现了 `SessionExpirer` 接口：

`org.apache.zookeeper.server.ZooKeeperServer`

```java
    public void expire(Session session) {
        long sessionId = session.getSessionId();
        LOG.info("Expiring session 0x" + Long.toHexString(sessionId)
                + ", timeout of " + session.getTimeout() + "ms exceeded");
        close(sessionId);
    }

    private void close(long sessionId) {
        submitRequest(null, sessionId, OpCode.closeSession, 0, null, null);
    }
```

可见，会话超时是通过提交一条 `closeSession`  的情求来实现的。

## 4. 会话清理

当 `SessionTrackerImpl` 的会话超时检查线程整理出一些已经过期的会话后，就要开始进行会话清理了。会话清理的步骤大致可以分为以下 7 步。

### 4.1 标记会话状态为已关闭

由于整个会话清理过程需要一段时间，为了保证在此期间不再处理来自该客户端的新请求，`SessionTrackerImpl` 会首先将该会话的 `isClosing` 属性标记为 true，这样，即使在会话清理期间接收到该客户端的新请求（虽然目前只是在 Leader 上标记了，但客户端的事务请求会被提交到 Leader），也无法继续处理了。

### 4.2 发起会话关闭请求

为了使对该会话的清理和关闭操作在整个服务端集群中都生效，ZooKeeper 使用了提交会话关闭请求的方式，将其作为一个事务处理。事务被提交到 Leader 的 `PreRequestProcessor`。

### 4.3 收集需要清理的临时节点

一旦某个会话失效后，那么和该会话相关的临时节点都需要被一并清除掉。因此，在清理临时节点之前，首先需要将服务器上所有和该会话相关的临时节点都整理出来。

在 ZooKeeper 的内存数据库中，为每个会话都单独保存了一份由该会话维护的所有临时节点集合，因此在会话清理阶段，只需要根据当前即将关闭的会话的 `sessionId` 从内存数据库中获取到这份临时节点列表即可。

在 Leader 处理会话关闭请求之前，可能正好有以下两类请求到达了服务端并正在处理中。

- 节点删除请求，删除的目标节点正好是上述临时节点中的一个。
- 临时节点创建请求，创建的目标节点正好是上述临时节点中的一个。

假定当前获取到的临时节点列表是 `ephemerals`。对于第一类请求，需要将所有这些请求对应的数据节点路径从 `ephemerals` 中移除，以避免重复删除。对于第二类请求，需要将所有这些请求对应的数据节点路径添加到 `ephemerals` 中去，以删除这些即将被创建但是尚未保存到内存数据库中去的临时节点。

`org.apache.zookeeper.server.PrepRequestProcessor#pRequest2Txn`

```java
        // ... 
        request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid, Time.currentWallTime(), type);        
        switch (type) {
            // ...
            case OpCode.closeSession:
                // We don't want to do this check since the session expiration thread
                // queues up this operation without being the session owner.
                // this request is the last of the session so it should be ok
                // zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                HashSet<String> es = zks.getZKDatabase().getEphemerals(request.sessionId);
                synchronized (zks.outstandingChanges) {
                    for (ChangeRecord c : zks.outstandingChanges) {
                        if (c.stat == null) {
                            // Doing a delete
                            es.remove(c.path);
                        } else if (c.stat.getEphemeralOwner() == request.sessionId) {
                            es.add(c.path);
                        }
                    }
                    for (String path2Delete : es) {
                        addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path2Delete, null, 0, null));
                    }

                    zks.sessionTracker.setSessionClosing(request.sessionId);
                }

                LOG.info("Processed session termination for sessionid: 0x"
                        + Long.toHexString(request.sessionId));
                break;
            // ...
        }
        // ...
```

### 4.4 添加节点删除事务变更

完成该会话相关的临时节点收集后，Leader 会逐个将对这些临时节点创建变更记录，并放入事务变更队列 `outstandingChanges` 中。

### 4.5 删除临时节点

节点删除事务被提交到整个集群后，各服务器的 `FinalRequestProcessor` 会触发内存数据库，删除该会话对应的所有临时节点。

### 4.6 移除会话

各服务器的 `FinalRequestProcessor` 在处理会话关闭事务请求时，会将会话从 `SessionTracker` 中移除。

### 4.7 关闭 NIOServerCnxn

最后，客户端所连接的那个服务器会向 `ServerCnxnFactory` 发送一个关闭连接的请求数据，从 `NIOServerCnxnFactory` 找到该会话对应的 `NIOServerCnxn`，将其关闭。

## 5. 重连

当客户端与服务端之间的网络连接断开时，ZooKeeper 客户端会自动进行反复的重连，直到最终成功连接上 ZooKeeper 集群中的一台机器。在这种情况下，再次连接上服务端的客户端有可能处于以下两种状态之一。

1. CONNECTED。如果在会话超时时间内重新连接上了集群中任意一台机器，那么被视为重连成功。
2. EXPIRED。如果在会话超时时间以外重新连接上，那么服务端其实已经对该会话进行了会话清理操作，因此再次连接上的会话将被视为非法会话。

在客户端与服务端之间维持的是一个长连接，在 sessionTimeout 时间内，服务端会不断地检测该客户端是否还处于正常连接——服务端会将客户端的每次操作视为一次有效的心跳检测来反复地进行会话激活。因此，在正常情况下，客户端会话是一直有效的。然而，当客户端与服务端之间的连接断开后，用户在客户端可能主要会看到两类异常：CONNECTION_LOSS（连接断开）和 SESSION_EXPIRED（会话过期）。

### 5.1 连接断开 CONNECTION_LOSS

有时会因为网络闪断导致客户端与服务器断开连接，或是因为客户端当前连接的服务器出现问题导致连接断开，我们统称这类问题为 “客户端与服务器连接断开” 现象，即 CONNECTION_LOSS。在这种情况下，ZooKeeper 客户端会自动从地址列表中重新逐个选取新的地址并尝试进行重新连接，直到最终成功连接上服务器。 

举个例子，假设某应用在使用 ZooKeeper 客户端进行 `setData` 操作的时候，正好出现了 CONNECTION_LOSS 现象，那么客户端会立即接收到事件 None-Disconnected 通知，同时会抛出异常：`org.apache.zookeeper.KeeperException.Code#CONNECTIONLOSS`。在这种情况下，应用需要做的事情就是捕获住 ConnectionLossException，然后等待 ZooKeeper 的客户端自动完成重连。一旦客户端成功连接上一台 ZooKeeper 机器后，那么客户端就会收到事件 None-SyncConnected 通知，之后就可以重试刚刚出错的 `setData` 操作。

### 5.2 会话失效 SESSION_EXPIRED

客户端与服务端断开连接后，如果重连期间耗时过长，超过了会话超时时间限制后还没有成功连接上服务器，那么服务器认为这个会话已经结束了，就会开始进行会话清理。但是另一方面，该客户端本身不知道会话已经失效，并且其客户端状态还是 DISCONNECTED。之后，如果客户端重新连上了服务器，服务器会告诉客户端该会话已经失效（SESSION_EXPIRED）。在这种情况下，用户就需要重新实例化一个 ZooKeeper 对象，并且看应用的复杂情况，重新恢复临时数据。

### 5.3 会话转移 SESSION_MOVED

ZooKeeper 明确提出了会话转移的概念，同时封装了 `SessionMovedException` 异常。之后，在处理客户端请求的时候，会首先检查会话的所有者（Owner)：如果客户端请求的会话 Owner 不是当前服务器的话，那么就会直接抛出 `SessionMovedException` 异常。当然，由于客户端已经和这个服务器断开了连接，因此无法收到这个异常的响应。只有多个客户端使用相同的 sessionld/sessionPasswd 创建会话时，才会收到这样的异常。因为一旦有一个客户端会话创建成功，那么 ZooKeeper 服务器就会认为该 `sessionld` 对应的那个会话已经发生了转移，于是，等到第二个客户端连接上服务器后，就被认为是 “会话转移” 的情况了。

## 6. 自问自答

### 6.1 会话重连如何实现？

1. 客户端的会话创建请求作为一个事务，会被同步到整个 ZooKeeper 集群中，各个服务器的 `SessionTracker` 中记录了整个集群中的会话信息。

2. 客户端会与所连接的服务器保持会话存活（通过读写请求或 `PING` 请求）。

3. Leader 会定时地向 Learner 服务器发送 `PING` 消息，Learner 服务器在接收到 `PING` 消息后，会将这段时间内保持心跳检测的客户端列表，同样以 `PING` 消息的形式反馈给 Leader 服务器，由 Leader 服务器来负责逐个对这些客户端进行会话激活。

4. 当客户端重连到另一台服务器上时，`org.apache.zookeeper.server.NIOServerCnxn#readConnectRequest` 调用 `org.apache.zookeeper.server.ZooKeeperServer#processConnectRequest`。这里会做一个判断，如果请求中附带了 `sessionId`，就不执行创建会话操作，而是重新打开会话。

5. 服务器首先执行密码校验工作，如果校验通过，则调用 `org.apache.zookeeper.server.ZooKeeperServer#revalidateSession` 验证会话有效性。

6. 如果客户端重连上的是 Leader，会话的有效性验证在本地即可操作。

7. 如果客户端重连上的是 Learner，`LearnerZooKeeperServer` 中重写了 `revalidateSession` 方法：

   `org.apache.zookeeper.server.quorum.LearnerZooKeeperServer#revalidateSession`

   ```java
       @Override
       protected void revalidateSession(ServerCnxn cnxn, long sessionId,
               int sessionTimeout) throws IOException {
           getLearner().validateSession(cnxn, sessionId, sessionTimeout);
       }
   ```

   `org.apache.zookeeper.server.quorum.Learner#validateSession`

   ```java
       void validateSession(ServerCnxn cnxn, long clientId, int timeout)
               throws IOException {
           LOG.info("Revalidating client: 0x" + Long.toHexString(clientId));
           ByteArrayOutputStream baos = new ByteArrayOutputStream();
           DataOutputStream dos = new DataOutputStream(baos);
           dos.writeLong(clientId);
           dos.writeInt(timeout);
           dos.close();
           QuorumPacket qp = new QuorumPacket(Leader.REVALIDATE, -1, baos.toByteArray(), null);
           pendingRevalidations.put(clientId, cnxn);
           writePacket(qp, true);
       } 
   ```

   Learner 会向 Leader 发送 `REVALIDATE` 消息，由 Leader 来完成客户端会话有效性的验证工作，并向 Learner 返回验证结果。

8. 注意到 `SessionTracker` 的 `sessionWithTimeout` 是和 ZooKeeper 的内存数据库相连通的。这样，即使在 ZooKeeper 集群运行过程中发生了重新选举，新的 Leader 也可以在从快照文件和事务日志中恢复出内存数据库后，执行会话有效性验证工作。

9. Learner 在接收到 Leader 的会话验证结果后，即使会话有效，也不会做一次 `addSession` 或 `touchSession` 操作。这是因为如果 Leader 判定该会话有效，那么该会话一定存在于重连上的 Learner 的 `LearnerSessionTracker` 中，不需要重新添加。同时，这样的会话验证过程会在 Leader 上对该会话进行一次激活。