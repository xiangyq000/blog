---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：请求处理
date: 2017-12-19 01:01:58
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

## 1. 会话创建请求

ZooKeeper 服务端对于会话创建的处理，大体可以分为请求接收、会话创建、预处理、事务处理、事务应用和会话响应 6 大环节，其大体流程如图。

![](https://user-images.githubusercontent.com/12514722/34207103-ffdb7a80-e5c3-11e7-85e4-f5bb3b99bc40.png)

<!--more-->

### 1.1 	请求接收

#### 1.1.1 I/O 层接收来自客户端的请求

在 ZooKeeper 中，`NIOServerCnxnFactory` 会在运行过程中为客户端连接创建对应的  `NIOServerCnxn` 实例，客户端与服务端的所有通信都是由`NIOServerCnxn` 负责的 —— 其负责统一接收来自客户端的所有请求，并将请求内容从底层网络 I/O 中完整地读取出来，一个客户端连接就对应了一个 `NIOServerCnxn` 的实例。注意刚创建时 `NIOServerCnxn` 实例的 `initialized` 字段为 false。

`org.apache.zookeeper.server.NIOServerCnxnFactory#run`

```java
    public void run() {
        while (!ss.socket().isClosed()) {
            try {
                selector.select(1000);
                Set<SelectionKey> selected;
                synchronized (this) {
                    selected = selector.selectedKeys();
                }
                ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                        selected);
                Collections.shuffle(selectedList);
                for (SelectionKey k : selectedList) {
                    if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                        SocketChannel sc = ((ServerSocketChannel) k
                                .channel()).accept();
                        InetAddress ia = sc.socket().getInetAddress();
                        int cnxncount = getClientCnxnCount(ia);
                        if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns){
                            LOG.warn("Too many connections from " + ia
                                     + " - max is " + maxClientCnxns );
                            sc.close();
                        } else {
                            LOG.info("Accepted socket connection from "
                                     + sc.socket().getRemoteSocketAddress());
                            sc.configureBlocking(false);
                            SelectionKey sk = sc.register(selector,
                                    SelectionKey.OP_READ);
                            NIOServerCnxn cnxn = createConnection(sc, sk);
                            sk.attach(cnxn);
                            addCnxn(cnxn);
                        }
                    } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                        NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                        c.doIO(k);
                    } else {
                        if (LOG.isDebugEnabled()) {
                            LOG.debug("Unexpected ops in select "
                                      + k.readyOps());
                        }
                    }
                }
                selected.clear();
            } catch (RuntimeException e) {
                LOG.warn("Ignoring unexpected runtime exception", e);
            } catch (Exception e) {
                LOG.warn("Ignoring exception", e);
            }
        }
        closeAll();
        LOG.info("NIOServerCnxn factory exited run method");
    }
```

#### 1.1.2 判断是否是客户端会话创建请求

当底层 I/O 有数据可读时，`NIOServerCnxnFactory` 找到绑定的 `NIOServerCnxn` 实例，调用其 `doIO` 方法。这里会做一个判断，若 `initialized` 字段为 false，则这一定是客户端的第一个请求会话创建请求。

`org.apache.zookeeper.server.NIOServerCnxn#readPayload`

```java
    /** Read the request payload (everything following the length prefix) */
    private void readPayload() throws IOException, InterruptedException {
        if (incomingBuffer.remaining() != 0) { // have we read length bytes?
            int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from client sessionid 0x"
                        + Long.toHexString(sessionId)
                        + ", likely client has closed socket");
            }
        }

        if (incomingBuffer.remaining() == 0) { // have we read length bytes?
            packetReceived();
            incomingBuffer.flip();
            if (!initialized) {
                readConnectRequest();
            } else {
                readRequest();
            }
            lenBuffer.clear();
            incomingBuffer = lenBuffer;
        }
    }
```

#### 1.1.3 反序列化 ConnectRequest 请求

一旦确定客户端请求是会话创建请求，那么服务端就可以对其进行反序列化，并生成一个 `ConnectRequest` 请求实体。

`org.apache.zookeeper.server.NIOServerCnxn#readConnectRequest`

```java
    private void readConnectRequest() throws IOException, InterruptedException {
        if (!isZKServerRunning()) {
            throw new IOException("ZooKeeperServer not running");
        }
        zkServer.processConnectRequest(this, incomingBuffer);
        initialized = true;
    }
```

`org.apache.zookeeper.server.ZooKeeperServer#processConnectRequest`

```java
    public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
        BinaryInputArchive bia = BinaryInputArchive.getArchive(new ByteBufferInputStream(incomingBuffer));
        ConnectRequest connReq = new ConnectRequest();
        connReq.deserialize(bia, "connect");
        boolean readOnly = false;
        try {
            readOnly = bia.readBool("readOnly");
            cnxn.isOldClient = false;
        } catch (IOException e) {
            // this is ok -- just a packet from an old client which
            // doesn't contain readOnly field
            LOG.warn("Connection request from old client "
                    + cnxn.getRemoteSocketAddress()
                    + "; will be dropped if server is in r-o mode");
        }
        if (readOnly == false && this instanceof ReadOnlyZooKeeperServer) {
            String msg = "Refusing session request for not-read-only client "
                + cnxn.getRemoteSocketAddress();
            LOG.info(msg);
            throw new CloseRequestException(msg);
        }
        if (connReq.getLastZxidSeen() > zkDb.dataTree.lastProcessedZxid) {
            String msg = "Refusing session request for client "
                + cnxn.getRemoteSocketAddress()
                + " as it has seen zxid 0x"
                + Long.toHexString(connReq.getLastZxidSeen())
                + " our last zxid is 0x"
                + Long.toHexString(getZKDatabase().getDataTreeLastProcessedZxid())
                + " client must try another server";

            LOG.info(msg);
            throw new CloseRequestException(msg);
        }
        int sessionTimeout = connReq.getTimeOut();
        byte passwd[] = connReq.getPasswd();
        int minSessionTimeout = getMinSessionTimeout();
        if (sessionTimeout < minSessionTimeout) {
            sessionTimeout = minSessionTimeout;
        }
        int maxSessionTimeout = getMaxSessionTimeout();
        if (sessionTimeout > maxSessionTimeout) {
            sessionTimeout = maxSessionTimeout;
        }
        cnxn.setSessionTimeout(sessionTimeout);
        // We don't want to receive any packets until we are sure that the
        // session is setup
        cnxn.disableRecv();
        long sessionId = connReq.getSessionId();
        if (sessionId != 0) {
            long clientSessionId = connReq.getSessionId();
            serverCnxnFactory.closeSession(sessionId);
            cnxn.setSessionId(sessionId);
            reopenSession(cnxn, sessionId, passwd, sessionTimeout);
        } else {
            createSession(cnxn, passwd, sessionTimeout);
        }
    }
```

#### 1.1.4 判断是否 ReadOnly 客户端

如果当前 ZooKeeper 客户端是以 ReadOnly 模式启动的，那么所有来自非 ReadOnly 客户端的请求将无法被处理。因此，服务端需要先检查其是否是 ReadOnly 客户端，并以此来决定是否接受该会话创建请求。

#### 1.1.5 检查客户端 ZXID

在正常情况下，同一个 ZooKeeper 集群中，服务端的 ZXID 必定大于客户端的 ZXID，因此如果发现客户端的 ZXID 大于服务端的 ZXID，那么服务端不接受该客户端的会话创建请求。

#### 1.1.6 协商 sessionTimeout

客户端在构造 ZooKeeper 实例时，会有一个 `sessionTimeout` 参数用于指定会话的超时时间。客户端向服务器发送这个超时时间后，服务器会根据自己的超时时间限制最终确定该会话的超时时间。

默认情况下，ZooKeeper 服务器对超时时间的限制介于 2 个 `tickTime` 到 20 个 `tickTime` 之间。

#### 1.1.7 判断是否需要重新创建会话

服务端根据客户端请求中是否包含 `sessionID` 来判断该客户端是否需要重新创建会话。如果客户端请求中已经包含了 `sessionID`，那么就认为该客户端正在进行会话重连。这种情况下，服务端只需要重新打开这个会话，否则需要重新创建。

### 1.2 会话创建

`org.apache.zookeeper.server.ZooKeeperServer#createSession`

```java
    long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
        long sessionId = sessionTracker.createSession(timeout);
        Random r = new Random(sessionId ^ superSecret);
        r.nextBytes(passwd);
        ByteBuffer to = ByteBuffer.allocate(4);
        to.putInt(timeout);
        cnxn.setSessionId(sessionId);
        submitRequest(cnxn, sessionId, OpCode.createSession, 0, to, null);
        return sessionId;
    }
```

`org.apache.zookeeper.server.SessionTrackerImpl#createSession`

```java
    synchronized public long createSession(int sessionTimeout) {
        addSession(nextSessionId, sessionTimeout);
        return nextSessionId++;
    }
```

#### 1.2.1 为客户端生成 sessionId

在为客户端创建会话之前，服务端首先会为每个客户端都分配一个 `sessionId`。 分配方式是通过 `SessionTracker` 对基准 `sessionId` 做自增操作。无论客户端连的是哪台服务器，生成的 `sessionId` 都是全局唯一的。

#### 1.2.2 注册会话

在会话创建初期，会将客户端会话的相关信息保存到 `SessionTracker` 的 `sessionWithTimeout` 和 `sessionById` 中。

`org.apache.zookeeper.server.SessionTrackerImpl#addSession`

```java
    synchronized public void addSession(long id, int sessionTimeout) {
        sessionsWithTimeout.put(id, sessionTimeout);
        if (sessionsById.get(id) == null) {
            SessionImpl s = new SessionImpl(id, sessionTimeout, 0);
            sessionsById.put(id, s);
        }
        touchSession(id, sessionTimeout);
    }
```

#### 1.2.3 激活会话

为会话安排一个区块，方便会话清理程序能够快速高效地进行会话清理。

`org.apache.zookeeper.server.SessionTrackerImpl#touchSession`

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
```

#### 1.2.4 生成会话密钥

服务端在创建一个客户端会话时，会同时为客户端生成一个会话密码，连同 `sessionId` 一起发送给客户端，作为会话在集群中不同机器间转移的凭证。

#### 1.2.5 将请求交给 firstProcessor

`org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.ServerCnxn, long, int, int, java.nio.ByteBuffer, java.util.List<org.apache.zookeeper.data.Id>)`

```java
    private void submitRequest(ServerCnxn cnxn, long sessionId, int type,
            int xid, ByteBuffer bb, List<Id> authInfo) {
        Request si = new Request(cnxn, sessionId, xid, type, bb, authInfo);
        submitRequest(si);
    }
```

这里的 `type` 为 `createSession`。

`org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.Request)`

```java
    public void submitRequest(Request si) {
        if (firstProcessor == null) {
            synchronized (this) {
                try {
                    // Since all requests are passed to the request
                    // processor it should wait for setting up the request
                    // processor chain. The state will be updated to RUNNING
                    // after the setup.
                    while (state == State.INITIAL) {
                        wait(1000);
                    }
                } catch (InterruptedException e) {
                    LOG.warn("Unexpected interruption", e);
                }
                if (firstProcessor == null || state != State.RUNNING) {
                    throw new RuntimeException("Not started");
                }
            }
        }
        try {
            touch(si.cnxn);
            boolean validpacket = Request.isValid(si.type);
            if (validpacket) {
                firstProcessor.processRequest(si);
                if (si.cnxn != null) {
                    incInProcess();
                }
            } else {
                LOG.warn("Received packet at server of unknown type " + si.type);
                new UnimplementedRequestProcessor().processRequest(si);
            }
        } 
        // ...
    }
```

`firstProcessor` 是一个 `RequestProcessor` 类型的变量。在提交给 `firstProcessor` 处理器之前，Zookeeper 会根据该请求所属的会话，进行一次激活会话操作，以确保当前会话处于激活状态，完成会话激活后，则提交请求至 `firstProcessor` 处理器，放入待处理请求队列中。

到这里 `createSession` 方法结束，后续流程由 `firstProcessor` 线程异步处理。

在会话创建请求的处理中，无论客户端连接的是 Leader 还是 Learner，到目前为止的处理流程都是相同的。接下来的差别在于：

- 对于 Leader 服务器，其 `firstProcessor` 的实现为 `PrepRequestProcessor`。
- 对于 Follower 服务器，其 `firstProcessor` 的实现为 `FollowerRequestProcessor`。
- 对于 Observer 服务器，其 `firstProcessor` 的实现为 `ObserverRequestProcessor`。

`FollowerRequestProcessor` 和 `ObserverRequestProcessor`会将事务请求以 REQUEST 消息的形式转发给 Leader 处理。Leader 的 `LearnerHandler` 在接收到这个消息后，会解析出客户端的原始请求，然后提交到自己的请求处理链中开始进行事务请求的处理。

### 1.3 事务预处理

#### 1.3.1 异步处理请求

`org.apache.zookeeper.server.PrepRequestProcessor#run`

```java
    @Override
    public void run() {
        try {
            while (true) {
                Request request = submittedRequests.take();
                long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
                if (request.type == OpCode.ping) {
                    traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
                }
                if (Request.requestOfDeath == request) {
                    break;
                }
                pRequest(request);
            }
        } 
        // ...
    }
```

`org.apache.zookeeper.server.PrepRequestProcessor#pRequest` 方法根据请求的类型，将事务类请求交由 `org.apache.zookeeper.server.PrepRequestProcessor#pRequest2Txn` 方法处理。对一些类型的事务请求，还要生成变更记录放入 `outstandingChanges` 队列中。

```java
        // ... 
        request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid, Time.currentWallTime(), type);        
        switch (type) {
            // ...
            case OpCode.createSession:
                request.request.rewind();
                int to = request.request.getInt();
                request.txn = new CreateSessionTxn(to);
                request.request.rewind();
                zks.sessionTracker.addSession(request.sessionId, to);
                zks.setOwner(request.sessionId, request.getOwner());
                break;
            // ...
        }
        // ...
```

#### 1.3.2 创建请求事务头

对于事务请求，ZooKeeper 首先会为其创建请求事务头。请求事务头包含了一个事务请求最基本的一些信息，包括 `sessionId`、ZXID、CXID（客户端的操作序列号） 和请求类型等。

```java
public class TxnHeader implements Record {
  private long clientId;
  private int cxid;
  private long zxid;
  private long time;
  private int type;
}
```

#### 1.3.3 创建请求事务体

对于事务请求，ZooKeeper 还会为其创建请求事务体。对应到会话创建请求，对应的事务体实现为 `CreateSessionTxn`。

#### 1.3.4 注册与激活会话

此处进行会话注册与激活的目的是处理由非 Leader 服务器转发过来的会话创建请求，在这种情况下，其尚未在 Leader 的 `SessionTracker` 中进行会话的注册，因此需要在此处进行一次注册与激活。

### 1.4 事务处理

在 `pRequest` 方法最后，会将请求提交给 `RequestProcessor` 类型变量 `nextProcessor` 处理。对于 Leader，这个变量的实现类为 `ProposalRequestProcessor`。

`ProposalRequestProcessor` 顾名思义是一个与提案相关的处理器。所谓的提案，是 ZooKeeper 中针对事务请求所展开的一个投票流程中对事务操作的包装。从 `ProposalRequestProcessor` 处理器开始，请求的处理将会同时进入三个子处理流程，分别是 Sync 流程、Proposal 流程和 Commit 流程。

![image](https://user-images.githubusercontent.com/12514722/34207238-a43f0c04-e5c4-11e7-872c-34123aba3b83.png)

`org.apache.zookeeper.server.quorum.ProposalRequestProcessor#processRequest`

```java
    public void processRequest(Request request) throws RequestProcessorException {
        /* In the following IF-THEN-ELSE block, we process syncs on the leader. 
         * If the sync is coming from a follower, then the follower
         * handler adds it to syncHandler. Otherwise, if it is a client of
         * the leader that issued the sync command, then syncHandler won't 
         * contain the handler. In this case, we add it to syncHandler, and 
         * call processRequest on the next processor.
         */
        
        if(request instanceof LearnerSyncRequest){
            zks.getLeader().processSync((LearnerSyncRequest)request);
        } else {
                nextProcessor.processRequest(request);
                if (request.hdr != null) {
                // We need to sync and get consensus on any transactions
                try {
                    zks.getLeader().propose(request);
                } catch (XidRolloverException e) {
                    throw new RequestProcessorException(e.getMessage(), e);
                }
                syncProcessor.processRequest(request);
            }
        }
    }
```

对 Leader 而言：

- `ProposalRequestProcessor` 会首先将请求提交给 `nextProcessor`，其具体实现是 `CommitProcessor`。请求被放入 `CommitProcessor` 的 队列 `queuedRequests`，等待 `CommitProcessor` 的线程异步处理（即等待投票完成），此即 Commit 流程。
- 调用 Leader 的 `propose` 方法，生成 `Proposal` 并广播给 Follower，统计 Follower 返回的投票结果并通知各个 Learner 最终提交事务，此即 Proposal 流程。这个流程会在完成后唤醒 Commit 流程。
- 由 `SyncRequestProcessor` 进行事务日志的记录，并调用 `AckRequestProcessor` 处理 Leader 自己的投票，此即 Sync 流程。这个流程会流向 Proposal 流程。

当 Leader 对非事务请求的处理流程到达此处时，由于不包含请求事务头，因此仅仅只是把请求提交给 `CommitProcessor`。

#### 1.4.1 Proposal 流程

`org.apache.zookeeper.server.quorum.Leader#propose`

```java
    /**
     * create a proposal and send it out to all the members
     * 
     * @param request
     * @return the proposal that is queued to send to all the members
     */
    public Proposal propose(Request request) throws XidRolloverException {
        /**
         * Address the rollover issue. All lower 32bits set indicate a new leader
         * election. Force a re-election instead. See ZOOKEEPER-1277
         */
        if ((request.zxid & 0xffffffffL) == 0xffffffffL) {
            String msg = "zxid lower 32 bits have rolled over, forcing re-election, and therefore new epoch start";
            shutdown(msg);
            throw new XidRolloverException(msg);
        }

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
        try {
            request.hdr.serialize(boa, "hdr");
            if (request.txn != null) {
                request.txn.serialize(boa, "txn");
            }
            baos.close();
        } catch (IOException e) {
            LOG.warn("This really should be impossible", e);
        }
        QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, baos.toByteArray(), null);
        
        Proposal p = new Proposal();
        p.packet = pp;
        p.request = request;
        synchronized (this) {
            lastProposed = p.packet.getZxid();
            outstandingProposals.put(lastProposed, p);
            sendPacket(pp);
        }
        return p;
    }
```

##### 1.4.1.1 发起投票

如果当前请求是事务请求，那么 Leader 服务器就会发起一轮事务投票。在发起事务投票之前，会首先检查当前服务器的 ZXID 是否可用。

##### 1.4.1.2 生成提案 Proposal

若 ZXID 可用，ZooKeeper 会将之前创建的请求头和事务体，以及 ZXID 和请求本身序列化到 `Proposal` 对象中 —— 此 `Proposal` 对象就是一个提案，即针对 ZooKeeper 服务器状态的一次变更申请。

##### 1.4.1.3 广播提案

更新 `lastProposed`，以 ZXID 作为 key 将该提案放入投票箱 `outstandingProposals` 中，同时将该提案广播给所有 Follower。

`org.apache.zookeeper.server.quorum.Leader#sendPacket`

```java
    void sendPacket(QuorumPacket qp) {
        synchronized (forwardingFollowers) {
            for (LearnerHandler f : forwardingFollowers) {                
                f.queuePacket(qp);
            }
        }
    }
```

##### 1.4.1.4 Follower 接收提案（Follower Sync 流程）

Follower 启动后，会通过 `followLeader` 方法不断从与 Leader 之间的连接中读取数据并作相应处理。

`org.apache.zookeeper.server.quorum.Follower#processPacket`

```java
    /**
     * Examine the packet received in qp and dispatch based on its contents.
     * @param qp
     * @throws IOException
     */
    protected void processPacket(QuorumPacket qp) throws IOException{
        switch (qp.getType()) {
        case Leader.PING:            
            ping(qp);            
            break;
        case Leader.PROPOSAL:            
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            if (hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
            }
            lastQueued = hdr.getZxid();
            fzk.logRequest(hdr, txn);
            break;
        case Leader.COMMIT:
            fzk.commit(qp.getZxid());
            break;
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Follower started");
            break;
        case Leader.REVALIDATE:
            revalidate(qp);
            break;
        case Leader.SYNC:
            fzk.sync();
            break;
        default:
            LOG.error("Invalid packet type: {} received by Observer", qp.getType());
        }
    }
```

`org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#logRequest`

```java
    public void logRequest(TxnHeader hdr, Record txn) {
        Request request = new Request(null, hdr.getClientId(), hdr.getCxid(),
                hdr.getType(), null, null);
        request.hdr = hdr;
        request.txn = txn;
        request.zxid = hdr.getZxid();
        if ((request.zxid & 0xffffffffL) != 0) {
            pendingTxns.add(request);
        }
        syncProcessor.processRequest(request);
    }
```

到这里，Follower 将这个事务记录到 `pendingTxns` 中，并将事务请求提交给 `syncProcessor` 作异步处理，在 Follower 的 Sync 流程中对提案做响应并向 Leader 提交 ACK 信息。

##### 1.4.1.5 Leader 统计投票

Leader 的 `LearnerHandler` 会接收来自各个 Follower 的 ACK 信息，并调用 Leader 的 `org.apache.zookeeper.server.quorum.Leader#processAck` 对投票做处理。

```java
    /**
     * Keep a count of acks that are received by the leader for a particular proposal
     * 
     * @param zxid the zxid of the proposal sent out
     * @param followerAddr
     */
    synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {
        if ((zxid & 0xffffffffL) == 0) {
            return;
        }
        if (outstandingProposals.size() == 0) {
            return;
        }
        if (lastCommitted >= zxid) {
            // The proposal has already been committed
            return;
        }
        Proposal p = outstandingProposals.get(zxid);
        if (p == null) {
            LOG.warn("Trying to commit future proposal: zxid 0x{} from {}", Long.toHexString(zxid), followerAddr);
            return;
        }
        
        p.ackSet.add(sid);
        if (self.getQuorumVerifier().containsQuorum(p.ackSet)){             
            if (zxid != lastCommitted+1) {
                LOG.warn("Commiting zxid 0x{} from {} not first!", Long.toHexString(zxid), followerAddr);
                LOG.warn("First is 0x{}", Long.toHexString(lastCommitted + 1));
            }
            outstandingProposals.remove(zxid);
            if (p.request != null) {
                toBeApplied.add(p);
            }

            if (p.request == null) {
                LOG.warn("Going to commmit null request for proposal: {}", p);
            }
            commit(zxid);
            inform(p);
            zk.commitProcessor.commit(p.request);
            if(pendingSyncs.containsKey(zxid)){
                for(LearnerSyncRequest r: pendingSyncs.remove(zxid)) {
                    sendSync(r);
                }
            }
        }
    }
```

当提案获得了集群中过半 PARTICIPANT 的投票，那么就认为该提案通过。

##### 1.4.1.6 处理通过的提案

1. 将提案的 ZXID 从 `outstandingProposals` 中移除。
2. 将提案添加到 `toBeApplied` 队列。
3. 向所有 Follower 发送 `COMMIT` 消息。由于 Follower 已经保存了所有关于该提案的信息，这里只需向其发送 ZXID 即可。
4. 向所有 Observer 发送 `INFORM` 消息。由于 Observer 并未参与之前的投票阶段，因此 Observer 服务器并未保存任何关于该提案的信息。`INFORM` 消息中会包含当前提案的内容。
5. 向 `CommitProcessor` 提交这个被通过的事务，进入 Leader 的 Commit 流程。

#### 1.4.2 Sync 流程

Leader 在生成事务提案和 Follower 接收到事务提案时，都会将提案放入 `SyncRequestProcessor` 的提案队列 `queuedRequests`，等待 `SyncRequestProcessor` 线程异步处理。 `SyncRequestProcessor` 处理器会记录事务日志，并提交给 `nextProcessor` 做后续处理。但是，Leader 和 Follower 的  `SyncRequestProcessor` 具有不同的 `nextProcessor` 实现。

##### 1.4.2.1 Leader 的 Sync 流程

对于 Leader，其 `SyncRequestProcessor` 的 `nextProcessor` 是 `AckRequestProcessor`。由于 Leader 自己也需要对事务进行投票，`AckRequestProcessor` 会用事务请求本身作为 ACK，并调用 Leader 的方法处理该 ACK。因此，Leader 的 Sync 流程最终会流向 Proposal 流程。

`org.apache.zookeeper.server.quorum.AckRequestProcessor#processRequest`

```java
    /**
     * Forward the request as an ACK to the leader
     */
    public void processRequest(Request request) {
        QuorumPeer self = leader.self;
        if(self != null)
            leader.processAck(self.getId(), request.zxid, null);
        else
            LOG.error("Null QuorumPeer");
    }
```

##### 1.4.2.2 Follower 的 Sync 流程

对于 Follower，其 `SyncRequestProcessor` 的 `nextProcessor` 是 `SendAckRequestProcessor`。`syncProcessor` 进行事务日志的记录后，由 `SendAckRequestProcessor` 向 Leader 回复一个 ACK 消息。

`org.apache.zookeeper.server.quorum.SendAckRequestProcessor#processRequest`

```java
    public void processRequest(Request si) {
        if(si.type != OpCode.sync){
            QuorumPacket qp = new QuorumPacket(Leader.ACK, si.hdr.getZxid(), null,
                null);
            try {
                learner.writePacket(qp, false);
            } catch (IOException e) {
                LOG.warn("Closing connection to leader, exception during packet send", e);
                try {
                    if (!learner.sock.isClosed()) {
                        learner.sock.close();
                    }
                } catch (IOException e1) {
                    // Nothing to do, we are shutting things down, so an exception here is irrelevant
                    LOG.debug("Ignoring error closing the connection", e1);
                }
            }
        }
    }
```

##### 1.4.2.3 Observer 的 Sync 流程

虽然 Observer 会初始化 `SyncRequestProcessor`，但由于 Leader 不会向 Observer 转发事务提案，因此 Observer 不存在 Sync 流程。

#### 1.4.3 Commit 流程

`org.apache.zookeeper.server.quorum.CommitProcessor#run`

```java
    @Override
    public void run() {
        try {
            Request nextPending = null;            
            while (!finished) {
                int len = toProcess.size();
                for (int i = 0; i < len; i++) {
                    nextProcessor.processRequest(toProcess.get(i));
                }
                toProcess.clear();
                synchronized (this) {
                    if ((queuedRequests.size() == 0 || nextPending != null) && committedRequests.size() == 0) {
                        wait();
                        continue;
                    }

                    if ((queuedRequests.size() == 0 || nextPending != null) && committedRequests.size() > 0) {
                        Request r = committedRequests.remove();
                        if (nextPending != null && nextPending.sessionId == r.sessionId && nextPending.cxid == r.cxid) {
                            nextPending.hdr = r.hdr;
                            nextPending.txn = r.txn;
                            nextPending.zxid = r.zxid;
                            toProcess.add(nextPending);
                            nextPending = null;
                        } else {
                            toProcess.add(r);
                        }
                    }
                }
                if (nextPending != null) {
                    continue;
                }
                synchronized (this) {
                    while (nextPending == null && queuedRequests.size() > 0) {
                        Request request = queuedRequests.remove();
                        switch (request.type) {
                        case OpCode.create:
                        case OpCode.delete:
                        case OpCode.setData:
                        case OpCode.multi:
                        case OpCode.setACL:
                        case OpCode.createSession:
                        case OpCode.closeSession:
                            nextPending = request;
                            break;
                        case OpCode.sync:
                            if (matchSyncs) {
                                nextPending = request;
                            } else {
                                toProcess.add(request);
                            }
                            break;
                        default:
                            toProcess.add(request);
                        }
                    }
                }
            }
        } catch (InterruptedException e) {
            LOG.warn("Interrupted exception while waiting", e);
        } catch (Throwable e) {
            LOG.error("Unexpected exception causing CommitProcessor to exit", e);
        }
        LOG.info("CommitProcessor exited loop!");
    }
```

##### 1.4.3.1 Leader 的 Commit 流程

1. 将请求交付给 `CommitProcessor` 处理器。

   如前所述，Leader 在生成提案之前，会首先将生成的提案放到 `CommitProcessor` 的 `queuedRequests` 队列中。

2. 处理 `queuedRequests` 队列请求。

   `CommitProcessor` 会有一个单独的线程来处理 `queuedRequests` 队列中的请求。

3. 标记 `nextPending`。

   若从 `queuedRequests` 中取出的请求是一个事务请求，则需要在集群中进行投票处理，同时将`nextPending` 标记为当前请求。

4. 等待 Proposal 投票。

   在进行 Commit 流程的同时，Leader 会生成 `Proposal` 并广播给所有 Follower 服务器，此时，Commit 流程等待，直到投票结束。

5. 投票通过。

   若提案获得过半 PARTICIPANT 认可，那么进入请求提交阶段。Leader 会将该请求放入 `commitedRequests` 队列中，同时唤醒 Commit 流程。

6. 提交请求。

   若 `commitedRequests` 队列中存在可以提交的请求，那么 Commit 流程将请求放入 `toProcess` 队列中。在这个过程中为了保证事务请求的顺序执行，Commit 流程还会对比之前标记的 `nextPending` 和 `commitedRequests` 队列中的第一个请求是否一致。在下一次循环中，`toProcess` 队列中的请求将被取出交付下一个请求处理器。对于 Leader 而言，下一个请求处理器是 `ToBeAppliedRequestProcessor`。

##### 1.4.3.2 Follower 的 Commit 流程

`org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#commit`

```java
    public void commit(long zxid) {
        if (pendingTxns.size() == 0) {
            LOG.warn("Committing " + Long.toHexString(zxid)
                    + " without seeing txn");
            return;
        }
        long firstElementZxid = pendingTxns.element().zxid;
        if (firstElementZxid != zxid) {
            LOG.error("Committing zxid 0x" + Long.toHexString(zxid) + " but next pending txn 0x" + Long.toHexString(firstElementZxid));
            System.exit(12);
        }
        Request request = pendingTxns.remove();
        commitProcessor.commit(request);
    }
```

Follower 在收到 `COMMIT` 消息时，会首先将该事务的 ZXID 与 `pendingTxns` 队列中缓存的事务对比，然后放入 `CommitProcessor` 的 `committedRequests` 队列。

Follower 的 `CommitProcessor` 将在两个队列中整理事务信息，在后续循环中提交给下一个请求处理器，即 `FinalRequestProcessor`。

##### 1.4.3.3 Observer 的 Commit 流程

`org.apache.zookeeper.server.quorum.Observer#processPacket`

```java
    /**
     * Controls the response of an observer to the receipt of a quorumpacket
     * @param qp
     * @throws IOException
     */
    protected void processPacket(QuorumPacket qp) throws IOException{
        switch (qp.getType()) {
        case Leader.PING:
            ping(qp);
            break;
        case Leader.PROPOSAL:
            LOG.warn("Ignoring proposal");
            break;
        case Leader.COMMIT:
            LOG.warn("Ignoring commit");            
            break;            
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Observer started");
            break;
        case Leader.REVALIDATE:
            revalidate(qp);
            break;
        case Leader.SYNC:
            ((ObserverZooKeeperServer)zk).sync();
            break;
        case Leader.INFORM:            
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            Request request = new Request (null, hdr.getClientId(), 
                                           hdr.getCxid(),
                                           hdr.getType(), null, null);
            request.txn = txn;
            request.hdr = hdr;
            ObserverZooKeeperServer obs = (ObserverZooKeeperServer)zk;
            obs.commitRequest(request);            
            break;
        default:
            LOG.error("Invalid packet type: {} received by Observer", qp.getType());
        }
    }
```

Observer 收到来自 Leader 的 `INFORM` 消息后的处理过程类似于 Follower。

### 1.5 事务应用

对于 Leader，事务由 `CommitProcessor` 提交给 `ToBeAppliedRequestProcessor`，再由 `ToBeAppliedRequestProcessor` 提交给 `FinalRequestProcessor`；对于 Follower 和 Observer，事务由 `CommitProcessor`  提交给 `FinalRequestProcessor`。

1. 有效性检查

   `FinalRequestProcessor` 处理器检查 `outstandingChanges` 队列中请求的有效性，如果发现这些请求已经落后于当前正在处理的请求，那么直接从 `outstandingChanges` 队列中移除。

2. 事务应用

   之前的请求处理仅仅是将事务请求记录到了事务日志中去，而内存数据库中的状态尚未变更。因此，在这个环节，需要将事务变更应用到内存数据库中。

   `org.apache.zookeeper.server.ZooKeeperServer#processTxn`

   ```java
       public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
           ProcessTxnResult rc;
           int opCode = hdr.getType();
           long sessionId = hdr.getClientId();
           rc = getZKDatabase().processTxn(hdr, txn);
           if (opCode == OpCode.createSession) {
               if (txn instanceof CreateSessionTxn) {
                   CreateSessionTxn cst = (CreateSessionTxn) txn;
                   sessionTracker.addSession(sessionId, cst.getTimeOut());
               } else {
                   LOG.warn("*****>>>>> Got " + txn.getClass() + " " + txn.toString());
               }
           } else if (opCode == OpCode.closeSession) {
               sessionTracker.removeSession(sessionId);
           }
           return rc;
       }
   ```

   对于会话创建这类事务请求，需要向 `SessionTracker` 进行会话注册。此时，一个客户端的会话被保存到了集群中的所有服务器上（但是注意，Leader 和 Learner 的 `SessionTracker` 具有不同实现）。

3. 将事务放入 `commitProposal` 队列

   一旦完成事务请求的内存数据库应用，就可以将该请求放入 `commitProposal` 队列中。 `commitProposal` 队列用来保存最近被提交的事务请求，以便集群间机器进行数据的快速同步。

### 1.6 会话响应

`FinalRequestProcessor` 继续处理对会话请求的响应。

1. 统计处理

   ZooKeeper 计算请求在服务端处理所花费的时间，统计客户端连接的基本信息，如 `lastZxid`（最新的 ZXID）、`lastOp`（最后一次和服务端的操作）和 `lastLatency`（最后一次请求处理所花费的时间）等。

2. 创建响应 `ConnectResponse`

   `ConnectResponse` 就是一个会话创建成功后的响应，包含了当前客户端与服务端之间的通信协议版本号 `protocolVersion`、会话超时时间、`sessionId` 和会话密码。


3. 序列化 `ConnectResponse`
4. I/O 层发送响应给客户端

### 1.7 客户端处理请求响应

对于会话创建请求，客户端会调用 `org.apache.zookeeper.ClientCnxn.SendThread#onConnected` 方法，生成一个 `None-SyncConnected` 事件，交由 `EventThread` 处理：

```java
        void onConnected(int _negotiatedSessionTimeout, long _sessionId,
                byte[] _sessionPasswd, boolean isRO) throws IOException {
            negotiatedSessionTimeout = _negotiatedSessionTimeout;
            if (negotiatedSessionTimeout <= 0) {
                state = States.CLOSED;

                eventThread.queueEvent(new WatchedEvent(
                        Watcher.Event.EventType.None,
                        Watcher.Event.KeeperState.Expired, null));
                eventThread.queueEventOfDeath();

                String warnInfo;
                warnInfo = "Unable to reconnect to ZooKeeper service, session 0x"
                    + Long.toHexString(sessionId) + " has expired";
                LOG.warn(warnInfo);
                throw new SessionExpiredException(warnInfo);
            }
            if (!readOnly && isRO) {
                LOG.error("Read/write client got connected to read-only server");
            }
            readTimeout = negotiatedSessionTimeout * 2 / 3;
            connectTimeout = negotiatedSessionTimeout / hostProvider.size();
            hostProvider.onConnected();
            sessionId = _sessionId;
            sessionPasswd = _sessionPasswd;
            state = (isRO) ? States.CONNECTEDREADONLY : States.CONNECTED;
            seenRwServerBefore |= !isRO;
            KeeperState eventState = (isRO) ?
                    KeeperState.ConnectedReadOnly : KeeperState.SyncConnected;
            eventThread.queueEvent(new WatchedEvent(
                    Watcher.Event.EventType.None,
                    eventState, null));
        }
```

## 2. SetData 请求

服务端对于 SetData 请求的处理大致可以分为 4 步，分别是请求的预处理、事务处理、事务应用和请求响应。

### 2.1 预处理

1. I/O 层接收来自客户端的请求。

2. 判断是否是客户端会话创建请求。对于 SetData 请求，由于已经完成了会话创建，因此按照正常事务请求进行处理。

3. 将请求交给 `PrepRequestProcessor` 处理器进行处理。

   `org.apache.zookeeper.server.PrepRequestProcessor#pRequest2Txn`

   ```java
           // ... 
           request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid, Time.currentWallTime(), type);        
           switch (type) {
               // ...
               case OpCode.setData:
                   zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                   SetDataRequest setDataRequest = (SetDataRequest)record;
                   if(deserialize)
                       ByteBufferInputStream.byteBuffer2Record(request.request, setDataRequest);
                   path = setDataRequest.getPath();
                   validatePath(path, request.sessionId);
                   nodeRecord = getRecordForPath(path);
                   checkACL(zks, nodeRecord.acl, ZooDefs.Perms.WRITE,
                           request.authInfo);
                   version = setDataRequest.getVersion();
                   int currentVersion = nodeRecord.stat.getVersion();
                   if (version != -1 && version != currentVersion) {
                       throw new KeeperException.BadVersionException(path);
                   }
                   version = currentVersion + 1;
                   request.txn = new SetDataTxn(path, setDataRequest.getData(), version);
                   nodeRecord = nodeRecord.duplicate(request.hdr.getZxid());
                   nodeRecord.stat.setVersion(version);
                   addChangeRecord(nodeRecord);
                   break;
               // ...
           }
           // ...
   ```

4. 创建请求事务头。

5. 会话检查。

   检查该会话是否有效，即是否已经超时。

6. 反序列化请求，并创建 `ChangeRecord` 记录。

   ZooKeeper 首先会对请求反序列化并生成特定的 `SetDataRequest` 请求，请求中包含了数据节点路径 path、更新的内容 data 和期望的数据节点版本 version。同时根据请求对应的 path，Zookeeper 会生成一个 `ChangeRecord` 记录。

7. ACL检查。检查客户端是否具有数据更新的权限。

8. 数据版本检查。

   ZooKeeper 通过 version 属性来实现乐观锁机制的写入校验。

9. 创建请求事务体 `SetDataTxn`。

10. 保存 `ChangeRecord` 记录到 `outstandingChanges` 队列中。

### 2.2 事务处理

参见会话创建的事务处理阶段。

### 2.3 事务应用

1. 交付给 `FinalRequestProcessor` 处理器。

2. 事务应用。

   将请求事务头和事务体直接交给内存数据库 `ZKDatabase` 进行事务应用，同时返回 `ProcessTxnResult` 对象，包含了数据节点内容更新后的 stat。

3. 将事务请求放入 `commitProposal` 队列。

### 2.4 请求响应

1. 统计处理。

2. 创建响应体 `SetDataResponse`。

   其包含了当前数据节点的最新状态 stat。

3. 创建响应头。

   包含当前响应对应的事务 ZXID 和请求处理是否成功的标识。

4. 序列化响应。

5. I/O层发送响应给客户端。

## 3. GetData 请求

服务端对于 `GetData` 请求的处理，大致分为 3 步，分别是请求的预处理、非事务处理和请求响应。

### 3.1 预处理

1. I/O 层接收来自客户端的请求。
2. 判断是否是客户端会话创建请求。
3. 会话检查。
4. 将请求提交给 `firstProcessor`。
   - 对于 Leader，`PreRequestProcessor` 再次检查会话，然后交给 `ProposalRequestProcessor`。由于这种情况下请求事务头为 null，Leader 将提交请求给 `CommitProcessor` 并忽略 Proposal 和 Sync 阶段。
   - 对于 Learner，提交请求给 `CommitProcessor`。

### 3.2 非事务处理

1. 反序列化 `GetDataRequest` 请求。
2. 获取数据节点。
3. ACL检查。
4. 获取数据内容和 stat，注册 `Watcher`。

### 3.3 请求响应

1. 创建响应体 `GetDataResponse`。响应体包含当前数据节点的内容和状态 stat。
2. 创建响应头。
3. 统计处理。
4. 序列化响应。
5. I/O层发送响应给客户端。

## 4. 自问自答

### 4.1  Learner 如何处理事务请求？

当一个 Learner 收到客户端的事务请求时，会通过 REQUEST 消息转发给 Leader。Leader 的 `LearnerHandler` 收到消息后，会提交给 `PreRequestProcessor`，进入预处理阶段。由于该请求不是来自于与 Leader 相连的客户端的，因此相比于完整流程，跳过了前面的会话创建阶段。

### 4.2 在事务处理的过程中，Follower 会收到哪些消息？

如何客户端连接的是一个 Follower，整个流程中该 Follower 会收到：

- 来自客户端的事务请求。由 `FollowerRequestProcessor` 处理，发送 REQUEST 消息给 Leader，并添加到 `CommitProcessor` 的 `queuedRequests` 队列。

不论客户端是向哪台服务器提交的事务请求，所有 Follower 都会收到：

1. 来自 Leader 的事务提案。由 `FollowerZooKeeperServer` 交给 `SyncRequestProcessor` 处理，提交到 `SendAckRequestProcessor`，向 Leader 回复 ACK。
2. 来自 Leader 的 `COMMIT` 消息。由 `FollowerZooKeeperServer` 添加事务请求到 `CommitProcesser` 的 `committedRequests` 队列，并在接下来提交到 `FinalRequestProcessor`。

### 4.2 在事务处理的过程中，Observer 会收到哪些消息？

如何客户端连接的是一个 Observer，整个流程中该 Observer 会收到：

- 来自客户端的事务请求。由 `ObserverRequestProcessor` 处理，发送 REQUEST 消息给 Leader，并记录到 `CommitProcessor` 的 `queuedRequests` 队列。

不论客户端是向哪台服务器提交的事务请求，所有 Observer 都会收到：

- 来自 Leader 的 `INFORM` 消息。由 `ObserverZooKeeperServer` 添加事务请求到 `CommitProcesser` 的 `committedRequests` 队列，并在接下来提交到 `FinalRequestProcessor`。

### 4.3 Leader 是否回复来自 Learner 的 REQUEST 消息？

Leader 不会对 Learner 的 REQUEST 消息做回复。请求处理结果由 Leader 向所有 Learner 发送确认信息（`COMMIT` 或 `INFORM`）传达。

### 4.4 如何保证只由接收客户端事务请求的那台服务器来对客户端发送响应？

1. 对于接收客户端事务请求的服务器，在流程中流转时，会创建一个 `Request` 对象，其 `cnxn` 属性被设置为处理该客户端请求的 `NIOServerCnxn` 实例。这个对象最终被添加到 `CommitProcessor` 的 `queuedRequests` 队列中，等待 Leader 确认事务处理结果。其他服务器不会执行这一个步骤。

2. 对于 Leader：

   1. 如果客户端请求是直接发送给 Leader 的，如前所述，Leader 会创建一个 `Request` 对象，其 `cnxn` 属性被设置为处理该客户端请求的 `NIOServerCnxn` 实例，然后调用 `org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.Request)`。

   2. 如果客户端请求不是直接发送给 Leader 的，那么 Leader 会收到来自某一个 Learner 的 REQUEST 请求。Leader 会创建一个 `Request` 对象，其 `cnxn` 属性为 null，然后调用 `org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.Request)`。

      `org.apache.zookeeper.server.quorum.LearnerHandler#run`

      ```java
          @Override
          public void run() {
              // ...
              while (true) {
                  switch (qp.getType()) {
                      case Leader.REQUEST:                    
                          bb = ByteBuffer.wrap(qp.getData());
                          sessionId = bb.getLong();
                          cxid = bb.getInt();
                          type = bb.getInt();
                          bb = bb.slice();
                          Request si;
                          if(type == OpCode.sync){
                              si = new LearnerSyncRequest(this, sessionId, cxid, type, bb, qp.getAuthinfo());
                          } else {
                              si = new Request(null, sessionId, cxid, type, bb, qp.getAuthinfo());
                          }
                          si.setOwner(this);
                          leader.zk.submitRequest(si);
                          break;  
                   }
               }
           }
      ```

   3. 无论哪种情况，这个 `Request` 将在 Leader 的 Commit 流程中被 Leader 添加到 `CommitProcessor` 的 `queuedRequests` 队列中，等待集群投票结果。

   4. 无论哪种情况，这个 `Request` 将在 Leader 的 Proposal 流程中被 Leader 添加到 `CommitProcessor` 的 `committedRequests` 队列中，等待事务应用。

3. 对于 Follower：

   1. 在 Leader 向 Follower 提交事务提案后，也会创建一个  `Request` 对象，但其 `cnxn` 属性被设置为 null，然后将其添加到 `FollowerZooKeeperServer` 的 `pendingTxn` 队列中。
   2. 在 Leader 向 Follower 正式提交事务（COMMIT）后，会从 `pendingTxn` 队列取出该 `Request` 对象，放入 `CommitProcesser` 的 `committedRequests` 队列中。

4. 对于 Observer：

   - 在 Leader 向 Follower 正式提交事务（INFORM）后，会创建一个  `Request` 对象，但其 `cnxn` 属性被设置为 null，放入 `CommitProcesser` 的 `committedRequests` 队列中。

5. 综上所述，对于服务器：

   - 如果自己是收到客户端请求的那个服务器，那么自己的 `CommitProcesser` 的 `queuedRequests` 队列中都会包含一个待提交的事务请求，其 `cnxn` 属性为客户端连接对应的 `NIOServerCnxn` 实例。
   - 如果自己不是收到客户端请求的那个服务器，那么自己的 `CommitProcesser` 的 `committedRequests` 队列中都会包含一个待提交的事务请求，其 `cnxn` 属性为 null。

6. 在 `CommitProcesser` 整理请求信息的过程中，会优先考虑 `queuedRequests` 队列中的 `Request` 对象。因此，如果自己是收到客户端请求的那个服务器，那么提交给 `FinalRequestProcessor` 的 `Requet` 对象的 `cnxn` 属性不为 null；反之则为 null。

7. 在 `FinalRequestProcessor` 的处理过程中，各服务器首先完成事务应用。这是将做一次判断，只有当传入的 `Request` 对象的 `cnxn` 参数不为 null 时，才会继续进行后续的会话响应操作。

8. 最终，集群中的所有服务器都提交并应用了事务，但只有客户端所连接的那个服务器才会对客户端进行响应。