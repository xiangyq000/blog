---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：集群启动与选举
date: 2017-12-29 15:55:52
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

## 1. QuorumPeerMain

`QuorumPeerMain` 类的 `Main` 函数较为简单，直接调用了 `initializeAndRun` 方法，参数就是 `zkServer.sh` 转入的参数，这里是 “start”。在 `initializeAndRun` 方法内部，首先启动的是定时清除镜像任务 `DatadirCleanupManager`，默认设置为保留 3 份。由于 `purgeInterval` 这个参数默认设置为 0，所以不会启动镜像定时清除机制。

`org.apache.zookeeper.server.DatadirCleanupManager#start`

```java
    public void start() {
        if (PurgeTaskStatus.STARTED == purgeTaskStatus) {
            LOG.warn("Purge task is already running.");
            return;
        }
        // Don't schedule the purge task with zero or negative purge interval.
        if (purgeInterval <= 0) {
            LOG.info("Purge task is not scheduled.");
            return;
        }

        timer = new Timer("PurgeTask", true);
        TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
        timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));

        purgeTaskStatus = PurgeTaskStatus.STARTED;
    }
```

<!--more-->

接下来，如果配置的 ZooKeeper 服务器大于 1 台，调用 `runFromConfig` 方法进行集群信息配置，并启动 `QuorumPeer` 线程。

`org.apache.zookeeper.server.quorum.QuorumPeerMain#runFromConfig`

```java
          // ...
          ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
          cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns());

          quorumPeer = getQuorumPeer();

          quorumPeer.setQuorumPeers(config.getServers());
          quorumPeer.setTxnFactory(new FileTxnSnapLog(
                  new File(config.getDataDir()),
                  new File(config.getDataLogDir())));
          quorumPeer.setElectionType(config.getElectionAlg());
          quorumPeer.setMyid(config.getServerId());
          quorumPeer.setTickTime(config.getTickTime());
          quorumPeer.setInitLimit(config.getInitLimit());
          quorumPeer.setSyncLimit(config.getSyncLimit());
          quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
          quorumPeer.setCnxnFactory(cnxnFactory);
          quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
          quorumPeer.setClientPortAddress(config.getClientPortAddress());
          quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
          quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
          quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
          quorumPeer.setLearnerType(config.getPeerType());
          quorumPeer.setSyncEnabled(config.getSyncEnabled());

          // sets quorum sasl authentication configurations
          quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
          if(quorumPeer.isQuorumSaslAuthEnabled()){
              quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
              quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
              quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
              quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
              quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
          }

          quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
          quorumPeer.initialize();

          quorumPeer.start();
          quorumPeer.join();
          
          //...
```

## 2. ServerCnxnFactory

每个 `QuorumPeer` 线程启动之前都会先启动一个 `cnxnFactory` 线程，首先初始化 `ServerCnxnFactory`，这个是用来接收来自客户端的连接的，也就是这里启动的是一个 TCP 服务器。在 ZooKeeper 里提供两种 TCP 服务器的实现，一个是使用 Java 原生 NIO 的方式，另外一个是使用 NETTY。默认是 NIO 的方式，一个典型的 Reactor 模型。

`org.apache.zookeeper.server.ServerCnxnFactory#createFactory()`

```java
    static public ServerCnxnFactory createFactory() throws IOException {
        String serverCnxnFactoryName = System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
        if (serverCnxnFactoryName == null) {
            serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
        }
        try {
            ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName).newInstance();
            LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
            return serverCnxnFactory;
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate " + serverCnxnFactoryName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```

## 3. QuorumPeer

接下来会开始针对 `QuorumPeer` 实例进行参数配置，`QuorumPeer` 类代表了 ZooKeeper 集群内的一个节点，参数较多，比较关键的是 `setQuorumPeers`、`setMyid`（每一个 ZooKeeper 节点对应有一个 `MyId`）、`setCnxnFactory`（TCP 服务）、`setZKDatabase`（ZooKeeper 自带的内存数据库）、`setTickTime`（ZooKeeper 服务端和客户端的会话控制）等等。注意到 `QuorumPeer` 在初始化时 `ServerState` 被设置为 LOOKING。

接下来调用同步方法 `start`，正式进入 `QuorumPeer 类`。`start` 方法主要包括四个方法，即读取内存数据库、启动 TCP 服务、选举 ZooKeeper 的 Leader 角色、启动自己线程。

`org.apache.zookeeper.server.quorum.QuorumPeer#start`

```java
    @Override
    public synchronized void start() {
        loadDataBase();
        cnxnFactory.start();        
        startLeaderElection();
        super.start();
    }
```

### 3.1 读取内存数据库

`org.apache.zookeeper.server.quorum.QuorumPeer#loadDataBase`

```java
    private void loadDataBase() {
        File updating = new File(getTxnFactory().getSnapDir(), UPDATING_EPOCH_FILENAME);
        try {
            zkDb.loadDataBase();

            // load the epochs
            long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
    	    long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
            try {
                currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
                if (epochOfZxid > currentEpoch && updating.exists()) {
                    setCurrentEpoch(epochOfZxid);
                }
            // ...
            }
        // ...
    }
```

`loadDataBase` 方法用于恢复数据，即从磁盘读取数据到内存，调用了 `ZKDatabase` 实例的 `addCommittedProposal` 方法，该方法维护了一个提交日志的队列，用于快速同步 follower 角色的节点信息，日志信息默认保存 500 条，所以选用了 `LinkedList` 队列用于快速删除数据溢出时的第一条信息。

`org.apache.zookeeper.server.ZKDatabase#addCommittedProposal`

```java
    public void addCommittedProposal(Request request) {
        WriteLock wl = logLock.writeLock();
        try {
            wl.lock();
            if (committedLog.size() > commitLogCount) {
                committedLog.removeFirst();
                minCommittedLog = committedLog.getFirst().packet.getZxid();
            }
            if (committedLog.size() == 0) {
                minCommittedLog = request.zxid;
                maxCommittedLog = request.zxid;
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
                LOG.error("This really should be impossible", e);
            }
            QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, baos.toByteArray(), null);
            Proposal p = new Proposal();
            p.packet = pp;
            p.request = request;
            committedLog.add(p);
            maxCommittedLog = p.packet.getZxid();
        } finally {
            wl.unlock();
        }
    }
```

为了保证事务的顺序一致性，ZooKeeper 采用了递增的事务 id 号（ZXID）来标识事务。所有的提议（Proposal）都在被提出的时候加上了 ZXID。实现中 ZXID 是一个 64 位的数字，高 32 位是 EPOCH 用来标识 Leader 节点是否改变，每次一个 Leader 被选出来以后它都会有一个新的 EPOCH 值，标识当前属于哪个 Leader 的统治，低 32 位用于递增计数。

如果当前保存的 EPOCH 和最新获取的不一样，那就说明 Leader 重新选举过了，用最新的值替换。

### 3.2 选举准备工作

`startLeaderElection` 方法调用了 `createElectionAlgorithm` 方法进行选举，目前仅用 `electionType` 为 3，即使用 FastLeaderElection 算法。

`org.apache.zookeeper.server.quorum.QuorumPeer#createElectionAlgorithm`

```java
        case 3:
            qcm = createCnxnManager();
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
                listener.start();
                le = new FastLeaderElection(this, qcm);
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
```

#### 3.2.1 监听选举端口

在 `QuorumCnxManager.Listener` 中启动 I/O 线程，默认绑定 3888 端口，等待集群其他机器连接：

`org.apache.zookeeper.server.quorum.QuorumCnxManager.Listener#run`

```java
        @Override
        public void run() {
            int numRetries = 0;
            InetSocketAddress addr;
            while((!shutdown) && (numRetries < 3)){
                try {
                    ss = new ServerSocket();
                    ss.setReuseAddress(true);
                    if (listenOnAllIPs) {
                        int port = view.get(QuorumCnxManager.this.mySid).electionAddr.getPort();
                        addr = new InetSocketAddress(port);
                    } else {
                        addr = view.get(QuorumCnxManager.this.mySid).electionAddr;
                    }
                    LOG.info("My election bind port: " + addr.toString());
                    setName(view.get(QuorumCnxManager.this.mySid).electionAddr.toString());
                    ss.bind(addr);
                    while (!shutdown) {
                        Socket client = ss.accept();
                        setSockOpts(client);
                        LOG.info("Received connection request " + client.getRemoteSocketAddress());

                        // Receive and handle the connection request
                        // asynchronously if the quorum sasl authentication is
                        // enabled. This is required because sasl server
                        // authentication process may take few seconds to finish,
                        // this may delay next peer connection requests.
                        if (quorumSaslAuthEnabled) {
                            receiveConnectionAsync(client);
                        } else {
                            receiveConnection(client);
                        }

                        numRetries = 0;
                    }
                // ...
            // ...
        // ...          
```

#### 3.2.2 接收连接请求

`receiveConnection` 会调用 `handleConnection` 方法，对 Socket 做一次读操作，接收对方发送过来的 sid。为了避免 peer 之间重复建立连接，这里仅允许高 sid 的实例向低 sid 的实例发起连接请求。

对于合法连接请求，`QuorumCnxManager` 根据 sid 分配独立的 `SendWorker` 和 `RecvWorker`，负责读写 Socket。`QuorumCnxManager` 中以 sid 为 key 保存了来自各个 peer 的连接对应的一些数据结构：

```java
    final ConcurrentHashMap<Long, SendWorker> senderWorkerMap;
    final ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>> queueSendMap;
    final ConcurrentHashMap<Long, ByteBuffer> lastMessageSent;

    public final ArrayBlockingQueue<Message> recvQueue;
```

`org.apache.zookeeper.server.quorum.QuorumCnxManager#handleConnection`

```java
private void handleConnection(Socket sock, DataInputStream din)
            throws IOException {
        Long sid = null;
        try {
            // Read server id
            sid = din.readLong();
            if (sid < 0) { // this is not a server id but a protocol version (see ZOOKEEPER-1633)
                sid = din.readLong();

                // next comes the #bytes in the remainder of the message
                // note that 0 bytes is fine (old servers)
                int num_remaining_bytes = din.readInt();
                if (num_remaining_bytes < 0 || num_remaining_bytes > maxBuffer) {
                    LOG.error("Unreasonable buffer length: {}", num_remaining_bytes);
                    closeSocket(sock);
                    return;
                }
                byte[] b = new byte[num_remaining_bytes];

                // remove the remainder of the message from din
                int num_read = din.read(b);
                if (num_read != num_remaining_bytes) {
                    LOG.error("Read only " + num_read + " bytes out of " + num_remaining_bytes + " sent by server " + sid);
                }
            }
            if (sid == QuorumPeer.OBSERVER_ID) {
                /*
                 * Choose identifier at random. We need a value to identify
                 * the connection.
                 */
                sid = observerCounter.getAndDecrement();
                LOG.info("Setting arbitrary identifier to observer: " + sid);
            }
        } catch (IOException e) {
            closeSocket(sock);
            LOG.warn("Exception reading or writing challenge: " + e.toString());
            return;
        }

        // do authenticating learner
        LOG.debug("Authenticating learner server.id: {}", sid);
        authServer.authenticate(sock, din);

        //If wins the challenge, then close the new connection.
        if (sid < this.mySid) {
            /*
             * This replica might still believe that the connection to sid is
             * up, so we have to shut down the workers before trying to open a
             * new connection.
             */
            SendWorker sw = senderWorkerMap.get(sid);
            if (sw != null) {
                sw.finish();
            }

            /*
             * Now we start a new connection
             */
            LOG.debug("Create new connection to server: " + sid);
            closeSocket(sock);
            connectOne(sid);

            // Otherwise start worker threads to receive data.
        } else {
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            
            if(vsw != null)
                vsw.finish();
            
            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return;
        }
    }
```

#### 3.2.3 主动发起连接请求

若收到的连接请求的源服务器的 sid 更小，则关闭连接并调用 `connectOne`  方法主动向对方发起连接。这里的 `connectOne` 方法会根据是否已经与目标 peer 建立连接来判断是否需要建立连接。判断方法是检查 `senderWorkerMap` 里是否有 sid 对应的 `SendWorker`。若没有，则调用 `startConnection`  方法发起连接。

这里 `startConnection` 和前面的 `handleConnection` 的作用很相似，只不过一个用于主动发起连接请求，一个用于处理收到连接请求。相应地，这里仅允许向更低 sid 的 peer 发起连接。

对应地，在 `startConnection` 方法中，建立 TCP 连接后会将自己的 sid 发送给对方，供对方的 `handleConnection`  方法读取。

`org.apache.zookeeper.server.quorum.QuorumCnxManager#startConnection`

```java
    private boolean startConnection(Socket sock, Long sid)
            throws IOException {
        DataOutputStream dout = null;
        DataInputStream din = null;
        try {
            // Sending id and challenge
            dout = new DataOutputStream(sock.getOutputStream());
            dout.writeLong(this.mySid);
            dout.flush();

            din = new DataInputStream(new BufferedInputStream(sock.getInputStream()));
        } catch (IOException e) {
            LOG.warn("Ignoring exception reading or writing challenge: ", e);
            closeSocket(sock);
            return false;
        }

        // authenticate learner
        authLearner.authenticate(sock, view.get(sid).hostname);

        // If lost the challenge, then drop the new connection
        if (sid > this.mySid) {
            LOG.info("Have smaller server identifier, so dropping the " +
                     "connection: (" + sid + ", " + this.mySid + ")");
            closeSocket(sock);
            // Otherwise proceed with the connection
        } else {
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            
            if(vsw != null)
                vsw.finish();
            
            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return true;    
            
        }
        return false;
    }
```

但在刚启动时，还没有向其他实例发起连接。

- `RecvWorker` 负责从 Socket 中读出数据封装成 `Message` 放入 `recvQueue` 中。
- `SendWorker` 负责从 `queueSendMap` 中取出数据写入 Socket 并放入 `lastMessageSent`。这里有个细节：一旦 ZK 发现针对当前远程服务器的发送队列为空，会从 `lastMessageSent` 中取出一个最近发送过的消息再次发送。

**总结一下：`Listener` 启动后，会监听选举端口上的连接请求，对每个连接请求，从其 Socket 中读取对方的 sid，并与自己的 sid 比较，判断连接发起流程是否合法。若不合法，则断开连接，由自己主动向对方的选举端口发起连接，并发送自己的 sid。对于每个合法连接请求，双方都会为其分配单独的 `SendWorker` 和 `RecvWorker`。那么这里有一个问题，最初的连接请求是如何发起的？后文可以看到，在 `QuorumPeer` 主线程启动后，每个 peer 都会根据集群的配置，向所有选举的 PARTICIPANT（非 OBSERVER）发起连接请求，并发送选票。**

#### 3.2.2 准备选举算法

然后调用基于 TCP 的选举算法 FastLeaderElection。这里已经通过 FastLeaderElection 的构造函数初始化了一个 `Messenger` 实例，启动了 `WorkerSender` 和 `WorkerReceiver` 线程。

`org.apache.zookeeper.server.quorum.FastLeaderElection`

```java
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }

    LinkedBlockingQueue<ToSend> sendqueue;
    LinkedBlockingQueue<Notification> recvqueue;

    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;

        sendqueue = new LinkedBlockingQueue<ToSend>();
        recvqueue = new LinkedBlockingQueue<Notification>();
        this.messenger = new Messenger(manager);
    }

    protected class Messenger {
        Messenger(QuorumCnxManager manager) {

            this.ws = new WorkerSender(manager);

            Thread t = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();

            this.wr = new WorkerReceiver(manager);

            t = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();
        }
    }
```

- `WorkerSender`：不断从 `sendqueue` 中获取待发送的选票，并将其传递给 `QuorumCnxManager` 的 `queueSendMap`。若还未与选票的目标服务器建立连接，则发起连接请求。

   `org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger.WorkerSender#run`

   ```java
               public void run() {
                   while (!stop) {
                       try {
                           ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                           if(m == null) continue;

                           process(m);
                       } catch (InterruptedException e) {
                           break;
                       }
                   }
                   LOG.info("WorkerSender is down");
               }

               void process(ToSend m) {
                   ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch);
                   manager.toSend(m.sid, requestBuffer);
               }
   ```

   `org.apache.zookeeper.server.quorum.QuorumCnxManager#toSend`

   ```java
       /**
        * Processes invoke this message to queue a message to send. Currently, 
        * only leader election uses it.
        */
       public void toSend(Long sid, ByteBuffer b) {
           /*
            * If sending message to myself, then simply enqueue it (loopback).
            */
           if (this.mySid == sid) {
                b.position(0);
                addToRecvQueue(new Message(b.duplicate(), sid));
               /*
                * Otherwise send to the corresponding thread to send.
                */
           } else {
                /*
                 * Start a new connection if doesn't have one already.
                 */
                ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
                ArrayBlockingQueue<ByteBuffer> bqExisting = queueSendMap.putIfAbsent(sid, bq);
                if (bqExisting != null) {
                    addToSendQueue(bqExisting, b);
                } else {
                    addToSendQueue(bq, b);
                }
                connectOne(sid);
           }
       }
   ```

- `WorkerReceiver`：不断从 `QuorumCnxManager` 的 `recvQueue` 中拉取收到的选票。

   在该过程中如果当前服务器是 LOOKING 状态，将选票保存到 `recvQueue` 队列中。如果发现外部选票的选举轮次（逻辑时钟）小于自己的，则忽略该选票并立即发出自己的内部选票。

   如果当前服务器不是 LOOKING 状态，则忽略选票并将 Leader 信息以选票的形式发送出去。

   `org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger.WorkerReceiver#run`

   ```java
   public void run() {

       Message response;
       while (!stop) {
           // Sleeps on receive
           try{
               response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
               if(response == null) continue;

               if(!self.getVotingView().containsKey(response.sid)){
                   Vote current = self.getCurrentVote();
                   ToSend notmsg = new ToSend(ToSend.mType.notification, current.getId(), current.getZxid(), logicalclock.get(), self.getPeerState(), response.sid, current.getPeerEpoch());

                   sendqueue.offer(notmsg);
               } else {
                   // ...

                   // Instantiate Notification and set its attributes
                   Notification n = new Notification();

                   // State of peer that sent this message
                   QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                   switch (response.buffer.getInt()) {
                       case 0:
                           ackstate = QuorumPeer.ServerState.LOOKING;
                           break;
                       case 1:
                           ackstate = QuorumPeer.ServerState.FOLLOWING;
                           break;
                       case 2:
                           ackstate = QuorumPeer.ServerState.LEADING;
                           break;
                       case 3:
                           ackstate = QuorumPeer.ServerState.OBSERVING;
                           break;
                       default:
                           continue;
                   }

                   n.leader = response.buffer.getLong();
                   n.zxid = response.buffer.getLong();
                   n.electionEpoch = response.buffer.getLong();
                   n.state = ackstate;
                   n.sid = response.sid;
                   if(!backCompatibility){
                       n.peerEpoch = response.buffer.getLong();
                   } else {
                       n.peerEpoch = ZxidUtils.getEpochFromZxid(n.zxid);
                   }

                   n.version = (response.buffer.remaining() >= 4) ? response.buffer.getInt() : 0x0;
                   // ...
                   if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                       recvqueue.offer(n);

                       if((ackstate == QuorumPeer.ServerState.LOOKING) && (n.electionEpoch < logicalclock.get())){
                           Vote v = getVote();
                           ToSend notmsg = new ToSend(ToSend.mType.notification, v.getId(), v.getZxid(), logicalclock.get(), self.getPeerState(), response.sid, v.getPeerEpoch());
                           sendqueue.offer(notmsg);
                       }
                   } else {
                       Vote current = self.getCurrentVote();
                       if(ackstate == QuorumPeer.ServerState.LOOKING){
                           ToSend notmsg;
                           if(n.version > 0x0) {
                               notmsg = new ToSend(ToSend.mType.notification, current.getId(), current.getZxid(), current.getElectionEpoch(), self.getPeerState(), response.sid, current.getPeerEpoch());
                           } else {
                               Vote bcVote = self.getBCVote();
                               notmsg = new ToSend(ToSend.mType.notification, bcVote.getId(), bcVote.getZxid(), bcVote.getElectionEpoch(), self.getPeerState(), response.sid, bcVote.getPeerEpoch());
                          }
                       sendqueue.offer(notmsg);
                   }
               }
           } catch (InterruptedException e) {
               System.out.println("Interrupted Exception while waiting for new message" + e.toString());
           }
       }
   }
   ```

### 3.3 启动自己线程

在等待其他节点提交自己申请的过程中，进入了 `QuorumPeer` 的线程：

```java
@Override
public void run() {
    // ...
      
    try {
        while (running) {
            switch (getPeerState()) {
            case LOOKING:
                LOG.info("LOOKING");
                // ...
                try {
                     setBCVote(null);
                     setCurrentVote(makeLEStrategy().lookForLeader());
                } catch (Exception e) {
                     LOG.warn("Unexpected exception", e);
                     setPeerState(ServerState.LOOKING);
                }
                break;
            case OBSERVING:
                try {
                    LOG.info("OBSERVING");
                    setObserver(makeObserver(logFactory));
                    observer.observeLeader();
                }
                // ...
                break;
            case FOLLOWING:
                try {
                    LOG.info("FOLLOWING");
                    setFollower(makeFollower(logFactory));
                    follower.followLeader();
                }
                // ...
                break;
            case LEADING:
                LOG.info("LEADING");
                try {
                    setLeader(makeLeader(logFactory));
                    leader.lead();
                    setLeader(null);
                }
                // ...
                break;
            }
        }
    }
    // ...
}
```

## 4. Leader 选举

`QuorumPeer` 是 ZooKeeper 服务器实例的托管者，在运行期间，`QuorumPeer` 的核心工作就是不断地检测当前服务器的状态，并做出相应的处理。在正常情况下，ZooKeeper 服务器的状态在 LOOKING、LEADING 和 FOLLOWING / OBSERVING 之间进行切换。而在启动阶段，`QuorumPeer` 的初始状态是 LOOKING，因此开始进行 Leader 选举。

在 LOOKING 状态下，会调用 `org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader` 方法进行 Leader 选举：

```java
// ...

HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
int notTimeout = finalizeWait;

synchronized(this){
    logicalclock.incrementAndGet();
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}
sendNotifications();

/*
 * Loop in which we exchange notifications until we find a leader
 */
while ((self.getPeerState() == ServerState.LOOKING) && (!stop)){
    /*
     * Remove next notification from queue, times out after 2 times
     * the termination time
     */
    Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

    /*
     * Sends more notifications if haven't received enough.
     * Otherwise processes new notification.
     */
    if(n == null){
        if(manager.haveDelivered()){
            sendNotifications();
        } else {
            manager.connectAll();
        }

        /*
         * Exponential backoff
         */
        int tmpTimeOut = notTimeout*2;
        notTimeout = (tmpTimeOut < maxNotificationInterval?tmpTimeOut:maxNotificationInterval);
        LOG.info("Notification time out: " + notTimeout);
    } else if(self.getVotingView().containsKey(n.sid)) {
        /*
         * Only proceed if the vote comes from a replica in the
         * voting view.
         */
        switch (n.state) {
            case LOOKING:
                // If notification > current, replace and send messages out
                if (n.electionEpoch > logicalclock.get()) {
                    logicalclock.set(n.electionEpoch);
                    recvset.clear();
                    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                    } else {
                        updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                    }
                    sendNotifications();
                } else if (n.electionEpoch < logicalclock.get()) {
                    break;
                } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                    updateProposal(n.leader, n.zxid, n.peerEpoch);
                    sendNotifications();
                }

                recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                if (termPredicate(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch))) {

                    // Verify if there is any change in the proposed leader
                    while((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null){
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)){
                            recvqueue.put(n);
                            break;
                        }
                    }

                    /*
                     * This predicate is true once we don't read any new
                     * relevant message from the reception queue
                     */
                    if (n == null) {
                        self.setPeerState((proposedLeader == self.getId()) ? ServerState.LEADING: learningState());

                        Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                }
                break;
                case OBSERVING:
                     LOG.debug("Notification from observer: " + n.sid);
                     break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if(n.electionEpoch == logicalclock.get()){
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                           
                        if(ooePredicate(recvset, outofelection, n)) {
                            self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING: learningState());

                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify
                     * a majority is following the same leader.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
           
                    if(ooePredicate(outofelection, outofelection, n)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);
                            self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING: learningState());
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)", n.state, n.sid);
                    break;
            }
        } else {
            LOG.warn("Ignoring notification from non-cluster member " + n.sid);
        }
    }
    return null;
}
```

### 4.1 自增选举轮次

ZK 在进行新一轮的投票时，会首先对 `logicalClock` 进行自增操作。

### 4.2 初始化投票

在 `updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch())` 语句中，会设置初始化选票。这里要注意，对于 PARTICIPANT，选票中的 Leader SID 为服务器自己的 SID；而对于 OBSERVER，选票中的 Leader SID 为 `Long.MIN_VALUE`。类似地，选票中的 ZXID 和 peerEpoch 也为 `Long.MIN_VALUE`。

```java
    private long getInitId(){
        if(self.getLearnerType() == LearnerType.PARTICIPANT)
            return self.getId();
        else return Long.MIN_VALUE;
    }

    private long getInitLastLoggedZxid(){
        if(self.getLearnerType() == LearnerType.PARTICIPANT)
            return self.getLastLoggedZxid();
        else return Long.MIN_VALUE;
    }

    private long getPeerEpoch(){
        if(self.getLearnerType() == LearnerType.PARTICIPANT)
        	try {
        		return self.getCurrentEpoch();
        	} catch(IOException e) {
        		RuntimeException re = new RuntimeException(e.getMessage());
        		re.setStackTrace(e.getStackTrace());
        		throw re;
        	}
        else return Long.MIN_VALUE;
    }
```

数据结构 `Vote` 如下：

```java
class Vote {
    private int version;
    private long id; // 当前服务器自身的 SID
    private long zxid; // 当前服务器的最新 ZXID 值
    private long electionEpoch; // 当前服务器的逻辑时钟
    private long peerEpoch; // 被推举的服务器的选举轮次
    private ServerState state; // LOOKING
}
```

### 4.3 发送初始化投票

在 `sendNotifications` 方法中，会根据配置信息，向所有其他参与投票的 PARTICIPANT （即非 OBSERVER）发送 LOOKING 状态的投票。

`org.apache.zookeeper.server.quorum.FastLeaderElection#sendNotifications`

```java
    /**
     * Send notifications to all peers upon a change in our vote
     */
    private void sendNotifications() {
        for (QuorumServer server : self.getVotingView().values()) {
            long sid = server.id;

            ToSend notmsg = new ToSend(ToSend.mType.notification,
                    proposedLeader,
                    proposedZxid,
                    logicalclock.get(),
                    QuorumPeer.ServerState.LOOKING,
                    sid,
                    proposedEpoch);
            }
            sendqueue.offer(notmsg);
        }
    }
```

回忆前述 `WorkerSender` 的作用，这里也即是向配置文件中的其他 PARTICIPANT 发起连接的时机（但可能因为 sid 的规则限制被拒绝并由对方再次发起连接，或者该 PARTICIPANT 对应的实例尚未启动）。连接请求是发往其他所有 PARTICIPANT 的，因此服务器的启动顺序不影响整个流程。最终 PARTICIPANT 两两之间会建立连接，Observer 会与所有 PARTICIPANT 建立连接。

### 4.4 接受外部投票

如果发出选票的服务器 sid 不在集群配置的 PARTICIPANT 范围内，则 `WorkerReceiver` 立即用服务器当前选票作回应，该选票不会被添加到 `recvQueue` 中。这也就是说，虽然 Observer 也在 LOOKING 状态下向其他 PARTICIPANT 发出了自己的选票，但是会被其他 PARTICIPANT 忽略。

每台服务器会通过 `lookForLeader` 方法不断从 `recvQueue` 队列中获取外部投票。如果服务器发现无法获取到任何投票，那么就会立即确认自己是否和集群中其他服务器保持着有效连接。如果发现没有建立连接，那么就会马上建立连接。如果已经建立了连接，那么就再次发送自己当前的内部投票。

### 4.5 判断选票状态

- 如果发送选票的服务器状态是 OBSERVING，则忽略该选票。
- 如果发送选票的服务器状态是 FOLLOWING 或者 LEADING（回忆前面说的 `WorkerReceiver` 的逻辑，服务器在非 LOOKING 状态下收到了来自 LOOKING 状态服务器的选票，则以内部投票进行响应），说明当前集群中已经完成了选举，则根据选票中的 LEADER 和 EPOCH  等信息更新自身状态。
- 如果发送选票的服务器状态是 LOOKING，进入下面的流程。

### 4.6 LOOKING 时选票处理流程

#### 4.6.1 比较逻辑时钟

在处理外部投票的时候，会根据逻辑时钟来进行不同的处理。

- 外部投票的逻辑时钟大于内部投票。此时立即更新自己的逻辑时钟，并且清空所有已经收到的投票，然后使用初始化的投票来进行 PK 已确定是否变更内部投票。
- 外部投票的逻辑时钟小于内部投票。此时忽略该外部投票。
- 外部投票的逻辑时钟和内部投票一致。此时进行选票 PK。

#### 4.6.2 选票 PK

- 如果外部投票中被推举的 Leader 服务器的选举轮次（epoch）大于内部投票，那么就需要进行投票变更。
- 如果选举轮次一致，那么就对比两者的 ZXID，如果外部投票的 ZXID 大于内部投票，那么就需要进行投票变更。
- 如果两者的 ZXID 一致，那么就对比两者的 SID。如果外部投票的 SID 大于内部投票，那么就需要进行投票变更。

`org.apache.zookeeper.server.quorum.FastLeaderElection#totalOrderPredicate`

```java
    protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
        if(self.getQuorumVerifier().getWeight(newId) == 0){
            return false;
        }
        
        /*
         * We return true if one of the following three cases hold:
         * 1- New epoch is higher
         * 2- New epoch is the same as current epoch, but new zxid is higher
         * 3- New epoch is the same as current epoch, new zxid is the same
         *  as current zxid, but server id is higher.
         */
        
        return ((newEpoch > curEpoch) || 
                ((newEpoch == curEpoch) &&
                ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
    }
```

#### 4.6.3 变更投票

使用外部投票的选票信息来覆盖内部投票，变更完成后，再次将这个变更后的内部投票发送出去。

注意到 OBSERVER 同样会做选票 PK、选票变更等操作，只不过其后续选票也会被忽略。

#### 4.6.4 选票归档

无论是否进行了投票变更，都会将刚刚收到的那份外部投票放入 `recvset` 中进行归档。`recvset` 用于记录当前服务器再本轮次的 Leader 选举中收到的所有外部投票，并按 SID 分组。

#### 4.6.5 统计投票

统计集群中是否有过半的服务器认可了当前的内部投票。如果确定已经有过半的服务器认可了该内部投票，则终止投票。

`org.apache.zookeeper.server.quorum.FastLeaderElection#termPredicate`

```java
/**
     * Termination predicate. Given a set of votes, determines if
     * have sufficient to declare the end of the election round.
     *
     *  @param votes    Set of votes
     *  @param l        Identifier of the vote received last
     *  @param zxid     zxid of the the vote received last
     */
    protected boolean termPredicate(HashMap<Long, Vote> votes, Vote vote) {
        HashSet<Long> set = new HashSet<Long>();

        /*
         * First make the views consistent. Sometimes peers will have
         * different zxids for a server depending on timing.
         */
        for (Map.Entry<Long,Vote> entry : votes.entrySet()) {
            if (vote.equals(entry.getValue())){
                set.add(entry.getKey());
            }
        }

        return self.getQuorumVerifier().containsQuorum(set);
    }
```

#### 4.6.6 更新服务器状态

服务器会首先判断当前被过半服务器认可的投票所对应的 Leader 服务器是否是自己，如果是自己的话，那么就会将自己的服务器状态更新为 LEADING，否则根据具体情况来确定自己是 FOLLOWING（自己是 PARTICIPANT） 或是 OBSERVING（自己不是 PARTICIPANT）。

## 5. 选举时序图

![image](https://user-images.githubusercontent.com/12514722/34102275-74faa05e-e423-11e7-8984-e2ec3023d32b.png)

![image](https://user-images.githubusercontent.com/12514722/34352667-e375ad4e-ea5e-11e7-91d3-db13db78fca2.png)

## 6. Follower 启动

回到 `QuorumPeer` 的主线程，当服务器状态变为非 LOOKING 时，会根据自己的角色创建相应的服务器实例，并开始进入各自角色的主流程。

`org.apache.zookeeper.server.quorum.Follower#followLeader`

```java
    /**
     * the main method called by the follower to follow the leader
     *
     * @throws InterruptedException
     */
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {}", electionTimeTaken);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
            QuorumServer leaderServer = findLeader();            
            try {
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                syncWithLeader(newEpochZxid);                
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }
```

### 6.1 创建服务器实例

创建 `Follower` 和 `FollowerZooKeeperServer` 实例。

### 6.2 和 Leader 建立连接

所有的 Learner 服务器在启动完毕后，会从 Leader 选举的投票结果中找到当前集群中的 Leader 服务器，然后与其建立连接。

### 6.3 向 Leader 注册

将 Learner 服务器自己的基本信息发送给 Leader 服务器，即 `LearnerInfo`，包括 SID 和最新的 ZXID。

### 6.4 发送 ACK 信息

Learner 在收到来自 Leader 的 `LEADERINFO` 消息后，解析出 epoch 和 ZXID，然后向 Leader 反馈一个 `ACKEPOCH` 响应。

`org.apache.zookeeper.server.quorum.Learner#registerWithLeader`

```java
    /**
     * Once connected to the leader, perform the handshake protocol to
     * establish a following / observing connection. 
     * @param pktType
     * @return the zxid the Leader sends for synchronization purposes.
     * @throws IOException
     */
    protected long registerWithLeader(int pktType) throws IOException{
        /*
         * Send follower info, including last zxid and sid
         */
    	long lastLoggedZxid = self.getLastLoggedZxid();
        QuorumPacket qp = new QuorumPacket();                
        qp.setType(pktType);
        qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
        
        /*
         * Add sid to payload
         */
        LearnerInfo li = new LearnerInfo(self.getId(), 0x10000);
        ByteArrayOutputStream bsid = new ByteArrayOutputStream();
        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
        boa.writeRecord(li, "LearnerInfo");
        qp.setData(bsid.toByteArray());
        
        writePacket(qp, true);
        readPacket(qp);        
        final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
        if (qp.getType() == Leader.LEADERINFO) {
            // we are connected to a 1.0 server so accept the new epoch and read the next packet
            leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
            byte epochBytes[] = new byte[4];
            final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);
            if (newEpoch > self.getAcceptedEpoch()) {
                wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
                self.setAcceptedEpoch(newEpoch);
            } else if (newEpoch == self.getAcceptedEpoch()) {
                // since we have already acked an epoch equal to the leaders, we cannot ack
                // again, but we still need to send our lastZxid to the leader so that we can
                // sync with it if it does assume leadership of the epoch.
                // the -1 indicates that this reply should not count as an ack for the new epoch
                wrappedEpochBytes.putInt(-1);
        	} else {
        	    throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
        	}
            QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
            writePacket(ackNewEpoch, true);
            return ZxidUtils.makeZxid(newEpoch, 0);
        } else {
            if (newEpoch > self.getAcceptedEpoch()) {
                self.setAcceptedEpoch(newEpoch);
            }
            if (qp.getType() != Leader.NEWLEADER) {
                LOG.error("First packet should have been NEWLEADER");
                throw new IOException("First packet should have been NEWLEADER");
            }
            return qp.getZxid();
        }
    } 
```

### 6.5 数据同步

参见 [《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：数据与存储](https://xyq000.github.io/2017/12/25/%E3%80%8A%E4%BB%8E%20Paxos%20%E5%88%B0%20ZooKeeper%EF%BC%9A%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E3%80%8B%EF%BC%9A%E6%95%B0%E6%8D%AE%E4%B8%8E%E5%AD%98%E5%82%A8/)。

这里会将自己持有的 `SessionTracker` 设置为 `LearnerSessionTracker`。

### 6.6 启动 FollowerZooKeeperServer

### 6.7 处理与 Leader 的后续交互

## 7. Leader 启动

`org.apache.zookeeper.server.quorum.Leader#lead`

```java
    /**
     * This method is main function that is called to lead
     * 
     * @throws IOException
     * @throws InterruptedException
     */
    void lead() throws IOException, InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("LEADING - LEADER ELECTION TOOK - {}", electionTimeTaken);
        self.start_fle = 0;
        self.end_fle = 0;

        zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);

        try {
            self.tick.set(0);
            zk.loadData();
            
            leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

            // Start thread that waits for connection requests from 
            // new followers.
            cnxAcceptor = new LearnerCnxAcceptor();
            cnxAcceptor.start();
            
            readyToStart = true;
            long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());
            
            zk.setZxid(ZxidUtils.makeZxid(epoch, 0));
            
            synchronized(this){
                lastProposed = zk.getZxid();
            }
            
            newLeaderProposal.packet = new QuorumPacket(NEWLEADER, zk.getZxid(), null, null);

            if ((newLeaderProposal.packet.getZxid() & 0xffffffffL) != 0) {
                LOG.info("NEWLEADER proposal has Zxid of " + Long.toHexString(newLeaderProposal.packet.getZxid()));
            }
            
            waitForEpochAck(self.getId(), leaderStateSummary);
            self.setCurrentEpoch(epoch);

            // We have to get at least a majority of servers in sync with
            // us. We do this by waiting for the NEWLEADER packet to get
            // acknowledged
            try {
                waitForNewLeaderAck(self.getId(), zk.getZxid(), LearnerType.PARTICIPANT);
            } catch (InterruptedException e) {
                shutdown("Waiting for a quorum of followers, only synced with sids: [ " + getSidSetString(newLeaderProposal.ackSet) + " ]");
                HashSet<Long> followerSet = new HashSet<Long>();
                for (LearnerHandler f : learners)
                    followerSet.add(f.getSid());
                    
                if (self.getQuorumVerifier().containsQuorum(followerSet)) {
                    LOG.warn("Enough followers present. "
                            + "Perhaps the initTicks need to be increased.");
                }
                Thread.sleep(self.tickTime);
                self.tick.incrementAndGet();
                return;
            }
            
            startZkServer();
            
            /**
             * WARNING: do not use this for anything other than QA testing
             * on a real cluster. Specifically to enable verification that quorum
             * can handle the lower 32bit roll-over issue identified in
             * ZOOKEEPER-1277. Without this option it would take a very long
             * time (on order of a month say) to see the 4 billion writes
             * necessary to cause the roll-over to occur.
             * 
             * This field allows you to override the zxid of the server. Typically
             * you'll want to set it to something like 0xfffffff0 and then
             * start the quorum, run some operations and see the re-election.
             */
            String initialZxid = System.getProperty("zookeeper.testingonly.initialZxid");
            if (initialZxid != null) {
                long zxid = Long.parseLong(initialZxid);
                zk.setZxid((zk.getZxid() & 0xffffffff00000000L) | zxid);
            }
            
            if (!System.getProperty("zookeeper.leaderServes", "yes").equals("no")) {
                self.cnxnFactory.setZooKeeperServer(zk);
            }
            // Everything is a go, simply start counting the ticks
            // WARNING: I couldn't find any wait statement on a synchronized
            // block that would be notified by this notifyAll() call, so
            // I commented it out
            //synchronized (this) {
            //    notifyAll();
            //}
            // We ping twice a tick, so we only update the tick every other
            // iteration
            boolean tickSkip = true;
    
            while (true) {
                Thread.sleep(self.tickTime / 2);
                if (!tickSkip) {
                    self.tick.incrementAndGet();
                }
                HashSet<Long> syncedSet = new HashSet<Long>();

                // lock on the followers when we use it.
                syncedSet.add(self.getId());

                for (LearnerHandler f : getLearners()) {
                    // Synced set is used to check we have a supporting quorum, so only
                    // PARTICIPANT, not OBSERVER, learners should be used
                    if (f.synced() && f.getLearnerType() == LearnerType.PARTICIPANT) {
                        syncedSet.add(f.getSid());
                    }
                    f.ping();
                }

                // check leader running status
                if (!this.isRunning()) {
                    shutdown("Unexpected internal error");
                    return;
                }

              if (!tickSkip && !self.getQuorumVerifier().containsQuorum(syncedSet)) {
                //if (!tickSkip && syncedCount < self.quorumPeers.size() / 2) {
                    // Lost quorum, shutdown
                    shutdown("Not sufficient followers synced, only synced with sids: [ "
                            + getSidSetString(syncedSet) + " ]");
                    // make sure the order is the same!
                    // the leader goes to looking
                    return;
              } 
              tickSkip = !tickSkip;
            }
        } finally {
            zk.unregisterJMX(this);
        }
    }
```

### 7.1 创建服务器实例

创建 `Leader` 和 `LeaderZooKeeperServer` 实例。

### 7.2 启动 LearnerCnxAcceptor

创建并启动 Learner 接收器 `LearnerCnxAcceptor`，负责接收所有非 Leader 服务器的连接请求，创建并启动对应的 `LearnerHandler`。

`org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor#run`

```java
                while (!stop) {
                    try{
                        Socket s = ss.accept();
                        // start with the initLimit, once the ack is processed
                        // in LearnerHandler switch to the syncLimit
                        s.setSoTimeout(self.tickTime * self.initLimit);
                        s.setTcpNoDelay(nodelay);

                        BufferedInputStream is = new BufferedInputStream(s.getInputStream());
                        LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                        fh.start();
                    } catch (SocketException e) {
                        if (stop) {
                            stop = true;
                        } else {
                            throw e;
                        }
                    } catch (SaslException e){
                        LOG.error("Exception while connecting to quorum learner", e);
                    }
                }
```

### 7.3 创建 LearnerHandler

Leader 接收到来自其他机器的连接创建请求后，会为每一个 Learner 创建一个 `LearnerHandler` 实例，以 TCP 长连接的形式负责 Leader 和 Learner 之间几乎所有的消息通信和数据同步。

### 7.4 解析 Learner 信息，计算新的 epoch

`org.apache.zookeeper.server.quorum.Leader#getEpochToPropose`

```java
    private HashSet<Long> connectingFollowers = new HashSet<Long>();
    public long getEpochToPropose(long sid, long lastAcceptedEpoch) throws InterruptedException, IOException {
        synchronized(connectingFollowers) {
            if (!waitingForNewEpoch) {
                return epoch;
            }
            if (lastAcceptedEpoch >= epoch) {
                epoch = lastAcceptedEpoch+1;
            }
            connectingFollowers.add(sid);
            QuorumVerifier verifier = self.getQuorumVerifier();
            if (connectingFollowers.contains(self.getId()) && 
                                            verifier.containsQuorum(connectingFollowers)) {
                waitingForNewEpoch = false;
                self.setAcceptedEpoch(epoch);
                connectingFollowers.notifyAll();
            } else {
                long start = Time.currentElapsedTime();
                long cur = start;
                long end = start + self.getInitLimit()*self.getTickTime();
                while(waitingForNewEpoch && cur < end) {
                    connectingFollowers.wait(end - cur);
                    cur = Time.currentElapsedTime();
                }
                if (waitingForNewEpoch) {
                    throw new InterruptedException("Timeout while waiting for epoch from quorum");        
                }
            }
            return epoch;
        }
    }
```

Leader 服务器在接收到 Learner 的基本信息后，会解析出该 Learner 的 SID 和 ZXID，然后根据该 Learner 的 ZXID 解析出其对应的 epoch_of_learner，和当前 Leader 的 epoch_of_leader 进行比较，如果 epoch_of_learner 更大，则更新 epoch_of_learner：

$$
epoch\_of\_learner = epoch\_of\_learner + 1
$$
Leader 的 `lead` 方法和各个 `LearnerHandler` 线程会阻塞在 `getEpochToPropose` 方法处，直到过半的 Quorum 已经向 Leader 进行了注册，Leader 就可以确定当前集群的 epoch 了，并将 `waitingForEpoch` 标记设置为 false。

### 7.5 发送 Leader 状态

```java
    private HashSet<Long> electingFollowers = new HashSet<Long>();
    private boolean electionFinished = false;
    public void waitForEpochAck(long id, StateSummary ss) throws IOException, InterruptedException {
        synchronized(electingFollowers) {
            if (electionFinished) {
                return;
            }
            if (ss.getCurrentEpoch() != -1) {
                if (ss.isMoreRecentThan(leaderStateSummary)) {
                    throw new IOException("Follower is ahead of the leader, leader summary: " + leaderStateSummary.getCurrentEpoch() + " (current epoch), " + leaderStateSummary.getLastZxid() + " (last zxid)");
                }
                electingFollowers.add(id);
            }
            QuorumVerifier verifier = self.getQuorumVerifier();
            if (electingFollowers.contains(self.getId()) && verifier.containsQuorum(electingFollowers)) {
                electionFinished = true;
                electingFollowers.notifyAll();
            } else {                
                long start = Time.currentElapsedTime();
                long cur = start;
                long end = start + self.getInitLimit()*self.getTickTime();
                while(!electionFinished && cur < end) {
                    electingFollowers.wait(end - cur);
                    cur = Time.currentElapsedTime();
                }
                if (!electionFinished) {
                    throw new InterruptedException("Timeout while waiting for epoch to be acked by quorum");
                }
            }
        }
    }
```

计算出新的 epoch 之后，各个 `LearnerHandler` 会将该信息以一个 `LEADERINFO` 消息的形式发送给 Learner，同时等待 Learner 以 `ACKEPOCH`  消息进行响应。此时，Leader 的 `lead` 方法和各个 `LearnerHandler` 线程会阻塞在 `waitForEpochAck` 方法处，直到有过半的 Learner 确认了新的 epoch，然后将 `electionFinished` 标记设置为 true。

这里存在和计算 epoch 时一样的问题。

### 7.8 数据同步

Leader 服务器收到 Learner 的 `ACKEPOCH` 消息后，就可以开始与 Learner 进行数据同步了。同步完成后根据 Learn 的类型将 `LearnerHandler` 添加到 `forwardingFollowers` 或 `observingLearners` 集合中。

参见 [《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：数据与存储](https://xyq000.github.io/2017/12/25/%E3%80%8A%E4%BB%8E%20Paxos%20%E5%88%B0%20ZooKeeper%EF%BC%9A%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E3%80%8B%EF%BC%9A%E6%95%B0%E6%8D%AE%E4%B8%8E%E5%AD%98%E5%82%A8/)

### 7.9 启动 LeaderZooKeeperServer

### 7.10 处理与 Learner 的后续交互

## 8. Observer 启动

当服务器状态变为 OBSERVING 时，服务器创建 `Observer` 和 `ObserverZooKeeperServer` 实例，并调用 `org.apache.zookeeper.server.quorum.Observer#observeLeader` 方法处理与 Leader 的后续流程。

```java
    /**
     * the main method called by the observer to observe the leader
     *
     * @throws InterruptedException
     */
    void observeLeader() throws InterruptedException {
        zk.registerJMX(new ObserverBean(this, zk), self.jmxLocalPeerBean);

        try {
            QuorumServer leaderServer = findLeader();
            LOG.info("Observing " + leaderServer.addr);
            try {
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                long newLeaderZxid = registerWithLeader(Leader.OBSERVERINFO);

                syncWithLeader(newLeaderZxid);
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);                   
                }
            } catch (Exception e) {
                LOG.warn("Exception when observing the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX(this);
        }
    }
```

## 9. ZooKeeperServer 的启动与请求链初始化

![diagram](https://user-images.githubusercontent.com/12514722/34108731-2851130a-e43c-11e7-8f19-e2b8cb1dab52.jpg)

`org.apache.zookeeper.server.ZooKeeperServer#startup`

```java
    public synchronized void startup() {
        if (sessionTracker == null) {
            createSessionTracker();
        }
        startSessionTracker();
        setupRequestProcessors();

        registerJMX();

        setState(State.RUNNING);
        notifyAll();
    }
```

1. 创建并启动会话管理器。

2. 初始化 ZooKeeper 的请求处理链。

   `org.apache.zookeeper.server.quorum.LeaderZooKeeperServer#setupRequestProcessors`

   ```java
       @Override
       protected void setupRequestProcessors() {
           RequestProcessor finalProcessor = new FinalRequestProcessor(this);
           RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(
                   finalProcessor, getLeader().toBeApplied);
           commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                   Long.toString(getServerId()), false,
                   getZooKeeperServerListener());
           commitProcessor.start();
           ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                   commitProcessor);
           proposalProcessor.initialize();
           firstProcessor = new PrepRequestProcessor(this, proposalProcessor);
           ((PrepRequestProcessor)firstProcessor).start();
       }
   ```

   `org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#setupRequestProcessors`

   ```java
       @Override
       protected void setupRequestProcessors() {
           RequestProcessor finalProcessor = new FinalRequestProcessor(this);
           commitProcessor = new CommitProcessor(finalProcessor,
                   Long.toString(getServerId()), true,
                   getZooKeeperServerListener());
           commitProcessor.start();
           firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
           ((FollowerRequestProcessor) firstProcessor).start();
           syncProcessor = new SyncRequestProcessor(this,
                   new SendAckRequestProcessor((Learner)getFollower()));
           syncProcessor.start();
       }
   ```

   `org.apache.zookeeper.server.quorum.ObserverZooKeeperServer#setupRequestProcessors`

   ```java
       @Override
       protected void setupRequestProcessors() {      
           // We might consider changing the processor behaviour of 
           // Observers to, for example, remove the disk sync requirements.
           // Currently, they behave almost exactly the same as followers.
           RequestProcessor finalProcessor = new FinalRequestProcessor(this);
           commitProcessor = new CommitProcessor(finalProcessor,
                   Long.toString(getServerId()), true,
                   getZooKeeperServerListener());
           commitProcessor.start();
           firstProcessor = new ObserverRequestProcessor(this, commitProcessor);
           ((ObserverRequestProcessor) firstProcessor).start();

           /*
            * Observer should write to disk, so that the it won't request
            * too old txn from the leader which may lead to getting an entire
            * snapshot.
            *
            * However, this may degrade performance as it has to write to disk
            * and do periodic snapshot which may double the memory requirements
            */
           if (syncRequestProcessorEnabled) {
               syncProcessor = new SyncRequestProcessor(this, null);
               syncProcessor.start();
           }
       }
   ```

3. 注册 JMX 服务。


## 10. 自问自答

### 10.1 Observer 如何获知选举结果？

Observer 在启动时向 PARTICIPANT 发送的初始选票会被 PARTICIPANT 忽略，且 PARTICIPANT 不会在选举过程中向 Observer 发送选票。此时，根据 `org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader`，Observer 将不断向 PARTICIPANT 发送初始选票，直到集群选举完毕，某一个变更为 LEADING 或 FOLLOWING 状态的服务器收到了它的选票并且向它回复了选举结果。

### 10.2 QuorumVerifier 如何检查一个集合是否包含 Quorum？

`QuorumPeerConfig` 在读取配置文件时，会将各服务器以 SID 为 key，对应的 `QuorumPeer` 实例为 value 放入对应的哈希表中。其中，PARTICIPANT 放入 `servers`，Observer 放入 `observers`。然后以 `servers` 的大小初始化 `QuorumVerifier`，其默认实现是 `QuorumMaj`。

`org.apache.zookeeper.server.quorum.flexible.QuorumMaj#containsQuorum`

```java
    /**
     * Verifies if a set is a majority.
     */
    public boolean containsQuorum(HashSet<Long> set){
        return (set.size() > half);
    }
```

该方法会将入参集合的大小与 PARTICIPANT 数量的一半做比较，但不会检查入参集合中的元素是否属于某一个 PARTICIPANT。

最后，`observers` 中的元素会被移入 `servers`。

### 10.3 选举完成后的新 epoch 计算过程存在什么问题？

这里存在两个问题：

1. `connectingFollowers` 名称与作用不符。它至少应包含 Leader 的 SID，除 Follower 以外，还可能包含 Observer 的 SID。

2. 在这里，由于 `connectingFollowers` 集合可能包含了 Observer 的 SID，以它为参数作 Quorum 检查是不合适的（没有 PARTIPANT 检查）。

   假定集群当前由一个 Leader、两个 Follower 和一个 Observer 组成。这里，PARTICIPANT 总数为 3，则 Quorum 的一半是 2。即使只有 Leader 和 Observer 提交了 epoch 并且 SID 被添加到 `connectingFollowers` 中，条件也被满足了。

   在这个地方，要么应该忽略来自 Observer 提交的 epoch，要么应当要求已提交 epoch 的成员总数（Leader + Learner）超过集群所有成员数的一半（而不是 PARTICIPANT 的一半），逻辑才是一致的。

新 epoch 发送给集群后，Leader 对 ACK 消息的确认逻辑存在类似的问题。