---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：客户端
date: 2017-12-13 15:55:52
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---
ZooKeeper 客户端主要由以下几个核心组件组成：

- `ZooKeeper` 实例：客户端的入口
- `ClientWatchManager`：客户端Watcher管理器
- `HostProvider`：客户端地址列表管理器
- `ClientCnxn`：客户端核心线程

<!--more-->

ZooKeeper 客户端的构造方法有以下几种：

```Java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly)
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd)
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
```
这些构造方法的最终根据是否重用 session 有两种实现：

```Java
private final ZKWatchManager watchManager = new ZKWatchManager();

    //不重用session
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly)
        throws IOException
    {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

        watchManager.defaultWatcher = watcher;

        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        HostProvider hostProvider = new StaticHostProvider(
                connectStringParser.getServerAddresses());
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        cnxn.start();
    }

    //重用session
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
        throws IOException
    {
        watchManager.defaultWatcher = watcher;

        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        HostProvider hostProvider = new StaticHostProvider(
                connectStringParser.getServerAddresses());
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), sessionId, sessionPasswd, canBeReadOnly);
        cnxn.seenRwServerBefore = true; // since user has provided sessionId
        cnxn.start();
    }

    //ClientCnxnSocket 的默认实现是 ClientCnxnSocketNIO
    private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
        String clientCnxnSocketName = System
                .getProperty(ZOOKEEPER_CLIENT_CNXN_SOCKET);
        if (clientCnxnSocketName == null) {
            clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
        }
        try {
            return (ClientCnxnSocket) Class.forName(clientCnxnSocketName).getDeclaredConstructor()
                    .newInstance();
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + clientCnxnSocketName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```

## 1. 一次会话的创建过程

### 1.1 初始化阶段

客户端的初始化过程分为以下几个步骤：

1. 初始化 ZooKeeper 对象。

   通过调用 ZooKeeper 的构造方法来实例化一个 ZooKeeper 对象，在初始化过程中，会创建一个客户端的 Watcher 管理器：`ClientWatchManager`。

2. 设置会话默认 Watcher。

   如果在构造方法中传入了一个 Watcher 对象，那么客户端会将这个对象作为默认 Watcher 保存在 `ClientWatchManager` 中。

3. 构造 ZooKeeper 服务器地址列表管理器：`HostProvider`。

   对于构造方法中传入的服务器地址，客户端会将其存放在服务器地址列表管理器 `HostProvider` 中。

4. 创建并初始化客户端网络连接器：`ClientCnxn`。

   ZooKeeper 客户端首先会创建一个网络连接器 `ClientCnxn`，用来管理客户端与服务器的网络交互。另外，客户端在创建 `ClientCnxn` 的同时，还会初始化两个核心队列 `outgoingQueue` 和 `pendingQueue`，分别作为客户端请求的发送队列和服务端响应的等待队列。

   `ClientCnxn` 的构造和启动方法如下：

   ```java
   private final LinkedList<Packet> pendingQueue = new LinkedList<Packet>();
   private final LinkedList<Packet> outgoingQueue = new LinkedList<Packet>();
       
   public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
           ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
           long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
       this.zooKeeper = zooKeeper;
       this.watcher = watcher;
       this.sessionId = sessionId;
       this.sessionPasswd = sessionPasswd;
       this.sessionTimeout = sessionTimeout;
       this.hostProvider = hostProvider;
       this.chrootPath = chrootPath;

       connectTimeout = sessionTimeout / hostProvider.size();
       readTimeout = sessionTimeout * 2 / 3;
       readOnly = canBeReadOnly;

       sendThread = new SendThread(clientCnxnSocket);
       eventThread = new EventThread();
   }

   //启动 SendThread 和 EventThread
   public void start() {
       sendThread.start();
       eventThread.start();
   }
   ```

5. 初始化 `SendThread` 和 `EventThread`。

   `ClientCnxn` 还会创建两个核心网络线程 `SendThread` 和 `EventThread`，前者用于管理客户端和服务端之间的所有网络 I/O，后者则用于进行客户端的事件处理。同时，客户端还会将 `ClientCnxnSocket` 分配给 `SendThread` 作为底层网络 I/O 处理器，并初始化 `EventThread` 的待处理事件队列 `waitingEvents`，用于存放所有等待被客户端处理的事件。

   `SendThread`：

   ```java
   SendThread(ClientCnxnSocket clientCnxnSocket) {
       super(makeThreadName("-SendThread()"));
       //设置当前状态为 CONNECTING
       state = States.CONNECTING;
       this.clientCnxnSocket = clientCnxnSocket;
       setDaemon(true);
   }
   ```

   `EventThread`：

   ```java
   private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
   EventThread() {
       super(makeThreadName("-EventThread"));
       setDaemon(true);
   }
   ```

![image](https://user-images.githubusercontent.com/12514722/34354083-da7d4c58-ea66-11e7-863a-d988843d43f8.png)

### 1.2 会话创建阶段

1. 启动 `SendThread` 和 `EventThread`。

   `SendThread` 首先会判断当前客户端的状态，进行一系列请理性工作。

2. 获取一个服务器地址。

   在开始创建 TCP之前，`SendThread` 首先需要获取一个 ZooKeeper 服务器的目标地址，这通常是从 `HostProvider` 中随机获取出一个地址，然后委托给 `ClientCnxnSocket` 去创建与 ZooKeeper 服务器之间的 TCP 连接。

3. 创建TCP连接。

   获取一个服务器地址后，`ClientCnxnSocket` 负责和服务器创建一个 TCP 长连接。

4. 构造 `ConnectRequest` 请求。

   以上步骤后，`ClientCnxnSocket` 和服务器之间创建了一个 TCP 长连接，但和 ZooKeeper 服务器之间的会话创建尚未完成。`ClientCnxnSocket` 会进一步调用 `SendThread` 的 `primeConnection()` 方法，构造一个 `ConnectRequest` 请求。该请求代表了客户端试图和服务端之间创建一个会话。同时，将该请求包装成 `Packet` 对象，放入请求发送队列 `outgoingQueue` 中去。

5. 发送请求。

   `ClientCnxnSocket` 负责从 `outgoingQueue` 中取出一个待发送的 `Packet` 对象，将其序列化成 `ByteBuffer` 后，向服务端进行发送。

### 1.3 响应处理阶段

1. 接受服务器端响应。

   `ClientCnxnSocket` 接受到服务端响应后，会首先判断当前的客户端状态是否是 “已初始化”，如果尚未完成初始化，那么就认为该响应一定是会话创建请求的响应，直接交由 `readConnectResult` 方法来处理该响应。

2. 处理 Response。

   `ClientCnxnSocket` 会对接受到的服务端响应进行反序列化，得到 `ConnectResponse` 对象，并从中获取到 ZooKeeper 服务端分配的会话 `sessionId`。

3. 连接成功。

   连接成功后，一方面需要通知 `SendThread` 线程，进一步对客户端进行会话参数的设置，包括 `readTimeout` 和 `connectTimeout` 等，并更新客户端状态，另一方面，需要通知地址管理器 `HostProvider` 当前成功连接的服务器地址。

4. 生成事件：`SyncConnected-None`。

   为了能够让上层应用感知到会话的成功创建，`SendThread` 会生成一个事件 `SyncConnected-None`，代表客户端与服务器会话创建成功，并将该事件传递给 `EventThread` 线程。

5. 查询 `Watcher`。

   `EventThread` 线程收到事件后，会从 `ClientWatchManager` 管理器中查询出对应的 `Watcher`，针对 `SyncConnected-None` 事件，那么就直接找出存储的默认 `Watcher`，然后将其放到 `EventThread` 的 `watingEvents` 队列中去。

6. 处理事件。

   `EventThread` 不断的从 `watingEvents` 队列中取出待处理的 `Watcher` 对象，然后直接调用该对象的 `process` 接口方法，以达到触发 `Watcher` 的目的。

![image](https://user-images.githubusercontent.com/12514722/34354127-22dace76-ea67-11e7-80dc-e0649ae0c778.png)

## 2. 服务器地址列表

### 2.1 Chroot：客户端隔离命名空间

在 3.2.0 之后版本的 ZooKeeper 中，添加了 “Chroot” 特性，该特性允许每个客户端为自己设置一个命名空间。如果一个 ZooKeeper 客户端设置了 `Chroot`，那么该客户端对服务器的任何操作，都将会被限制在自己的命名空间下。

客户端可以通过在 `connectString` 中添加后缀的方式来设置 `Chroot`，如下所示：

```
192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181/apps/X
```

将这样一个 `connectString` 传入客户端的 `ConnectStringParser` 后就能够解析出 `Chroot` 并保存在 `chrootPath` 属性中。

### 2.2 HostProvider：地址列表管理器

`HostProvider` 的默认实现是 `StaticHostProvider`。通过调用 `staticHostProvider` 的 `next()` 方法，能够从 `StaticHostProvider` 中获取一个可用的服务器地址。这个 `next()` 方法并非简单地从 `serverAddresses` 中一次获取一个服务器地址，而是先将随机打散后的服务器地址列表拼装成一个环形的循环队列。注意这个随机过程是一次性的，也就是说，之后的使用过程中一直是按照这样的顺利来获取服务器地址的。

## 3. ClientCnxn：网络 I/O

### 3.1 Packet

`Packet` 是 `ClientCnxn` 内部定义的一个堆协议层的封装，用作 ZooKeeper 中请求和响应的载体。`Packet` 包含了请求头（`requestHeader`）、响应头（`replyHeader`）、请求体（`request`）、响应体（`response`）、节点路径（`clientPath/serverPath`）、注册的 `Watcher`（`watchRegistration`）等信息，然而，并非 `Packet` 中所有的属性都在客户端与服务端之间进行网络传输，只会将 `requestHeader`、`request`、`readOnly` 三个属性序列化，并生成可用于底层网络传输的 `ByteBuffer`，其他属性都保存在客户端的上下文中，不会进行与服务端之间的网络传输。

`org.apache.zookeeper.ClientCnxn.Packet#createBB`

```java
        public void createBB() {
            try {
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                boa.writeInt(-1, "len"); // We'll fill this in later
                if (requestHeader != null) {
                    requestHeader.serialize(boa, "header");
                }
                if (request instanceof ConnectRequest) {
                    request.serialize(boa, "connect");
                    // append "am-I-allowed-to-be-readonly" flag
                    boa.writeBool(readOnly, "readOnly");
                } else if (request != null) {
                    request.serialize(boa, "request");
                }
                baos.close();
                this.bb = ByteBuffer.wrap(baos.toByteArray());
                this.bb.putInt(this.bb.capacity() - 4);
                this.bb.rewind();
            } catch (IOException e) {
                LOG.warn("Ignoring unexpected exception", e);
            }
        }
```

### 3.2 outgoingQueue 和 pendingQueue

`ClientCnxn` 维护着 `outgoingQueue`（客户端的请求发送队列）和 `pendingQueue`（服务端响应的等待队列），`outgoingQueue` 专门用于存储那些需要发送到服务端的 `Packet` 集合，`pendingQueue` 用于存储那些已经从客户端发送到服务端的，但是需要等待服务端响应的 `Packet` 集合。

### 3.3 ClientCnxnSocket：底层 Socket 通信层

在 ZooKeeper 中，`ClientCnxnSocket`  的默认实现是 `ClientCnxnSocketNIO`，该实现类使用 Java 原生的 NIO 接口，其核心是 `doIO` 逻辑，主要负责对请求的发送和响应接收过程。

`SendThread` 线程中会循环调用 `org.apache.zookeeper.ClientCnxnSocketNIO#doTransport`

```java
    @Override
    void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn) throws IOException, InterruptedException {
        selector.select(waitTimeOut);
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();
        }
        // Everything below and until we get back to the select is
        // non blocking, so time is effectively a constant. That is
        // Why we just have to do this once, here
        updateNow();
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    sendThread.primeConnection();
                }
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                doIO(pendingQueue, outgoingQueue, cnxn);
            }
        }
        if (sendThread.getZkState().isConnected()) {
            synchronized(outgoingQueue) {
                if (findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    enableWrite();
                }
            }
        }
        selected.clear();
    }
```

`org.apache.zookeeper.ClientCnxnSocketNIO#doIO`

```java
    void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
      throws InterruptedException, IOException {
        SocketChannel sock = (SocketChannel) sockKey.channel();
        if (sock == null) {
            throw new IOException("Socket is null!");
        }
        if (sockKey.isReadable()) {
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from server sessionid 0x"
                                + Long.toHexString(sessionId)
                                + ", likely server has closed socket");
            }
            if (!incomingBuffer.hasRemaining()) {
                incomingBuffer.flip();
                if (incomingBuffer == lenBuffer) {
                    recvCount++;
                    readLength();
                } else if (!initialized) {
                    readConnectResult();
                    enableRead();
                    if (findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                        // Since SASL authentication has completed (if client is configured to do so),
                        // outgoing packets waiting in the outgoingQueue can now be sent.
                        enableWrite();
                    }
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                    initialized = true;
                } else {
                    sendThread.readResponse(incomingBuffer);
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                }
            }
        }
        if (sockKey.isWritable()) {
            synchronized(outgoingQueue) {
                Packet p = findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress());

                if (p != null) {
                    updateLastSend();
                    // If we already started writing p, p.bb will already exist
                    if (p.bb == null) {
                        if ((p.requestHeader != null) &&
                                (p.requestHeader.getType() != OpCode.ping) &&
                                (p.requestHeader.getType() != OpCode.auth)) {
                            p.requestHeader.setXid(cnxn.getXid());
                        }
                        p.createBB();
                    }
                    sock.write(p.bb);
                    if (!p.bb.hasRemaining()) {
                        sentCount++;
                        outgoingQueue.removeFirstOccurrence(p);
                        if (p.requestHeader != null
                                && p.requestHeader.getType() != OpCode.ping
                                && p.requestHeader.getType() != OpCode.auth) {
                            synchronized (pendingQueue) {
                                pendingQueue.add(p);
                            }
                        }
                    }
                }
                if (outgoingQueue.isEmpty()) {
                    // No more packets to send: turn off write interest flag.
                    // Will be turned on later by a later call to enableWrite(),
                    // from within ZooKeeperSaslClient (if client is configured
                    // to attempt SASL authentication), or in either doIO() or
                    // in doTransport() if not.
                    disableWrite();
                } else if (!initialized && p != null && !p.bb.hasRemaining()) {
                    // On initial connection, write the complete connect request
                    // packet, but then disable further writes until after
                    // receiving a successful connection response.  If the
                    // session is expired, then the server sends the expiration
                    // response and immediately closes its end of the socket.  If
                    // the client is simultaneously writing on its end, then the
                    // TCP stack may choose to abort with RST, in which case the
                    // client would never receive the session expired event.  See
                    // http://docs.oracle.com/javase/6/docs/technotes/guides/net/articles/connection_release.html
                    disableWrite();
                } else {
                    // Just in case
                    enableWrite();
                }
            }
        }
    }
```

#### 3.3.1 请求发送

客户端提交请求：

`org.apache.zookeeper.ClientCnxn#submitRequest`

```java
    public ReplyHeader submitRequest(RequestHeader h, Record request,
            Record response, WatchRegistration watchRegistration)
            throws InterruptedException {
        ReplyHeader r = new ReplyHeader();
        Packet packet = queuePacket(h, r, request, response, null, null, null,
                    null, watchRegistration);
        synchronized (packet) {
            while (!packet.finished) {
                packet.wait();
            }
        }
        return r;
    }
```

`org.apache.zookeeper.ClientCnxn#queuePacket`

```java
    Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
            Record response, AsyncCallback cb, String clientPath,
            String serverPath, Object ctx, WatchRegistration watchRegistration)
    {
        Packet packet = null;

        // Note that we do not generate the Xid for the packet yet. It is
        // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
        // where the packet is actually sent.
        synchronized (outgoingQueue) {
            packet = new Packet(h, r, request, response, watchRegistration);
            packet.cb = cb;
            packet.ctx = ctx;
            packet.clientPath = clientPath;
            packet.serverPath = serverPath;
            if (!state.isAlive() || closing) {
                conLossPacket(packet);
            } else {
                // If the client is asking to close the session then
                // mark as closing
                if (h.getType() == OpCode.closeSession) {
                    closing = true;
                }
                outgoingQueue.add(packet);
            }
        }
        sendThread.getClientCnxnSocket().wakeupCnxn();
        return packet;
    }
```

在正常情况下，`ClientCnxnSocket` 会从 `outgoingQueue` 中取出一个可发送的 `Packet` 对象，同时生成一个客户端请求序号 XID 并将其设置到 `Packet` 请求头中去，然后将其序列化后进行发送。

请求发送完毕后，会立即将该 `Packet` 保存到 `pendingQueue` 中，以便等待服务端响应返回后进行相应的处理。

#### 3.3.2 响应接收

客户端获取到来自服务端的完整响应数据后，根据不同的客户端请求类型，会进行不同的处理。

- 如果检测到当前客户端尚未进行初始化，那么说明当前客户端与服务端之间正在进行会话创建，那么就直接将接收到的 `ByteBuffer`（`incomingBuffer`）序列化成 `ConnectResponse` 对象。
- 如果当前客户端已经处于正常的会话周期，并且接收到的服务端响应是一个事件，那么客户端将接收到的 `ByteBuffer` 序列化成 `WatcherEvent` 对象，并将该事件放入待处理队列中。
- 如果是一个常规的请求响应（`Create`、`GetData`、`Exist` 等），那么会从 `pendingQueue` 队列中取出一个 `Packet` 来进行相应的处理。客户端首先会通过检验服务端响应中的 XID 来确保请求处理的顺序性，然后再将接收到的 `ByteBuffer` 序列化成 `Response` 对象。
- 最后，会在 `finishPacket` 方法中处理 `Watcher` 注册等逻辑。

### 3.4 SendThread

`SendThread` 是客户端 `ClientCnxn` 内部一个核心的 I/O 调度线程，用于管理客户端和服务端之间的所有网络 I/O 操作。在 ZooKeeper 客户端的实际运行过程中，一方面，`SendThread` 维护了客户端与服务端之间的会话生命周期，其通过在—定的周期频率内向服务端发送一个 `PING` 包来实现心跳检测。同时，在会话周期内，如果客户端与服务端之间出现 TCP 连接断开的怙况，那么就会自动且透明化地完成重连操作。

另一方面，`SendThread` 管理了客户端所有的请求发送和响应接收操作，其将上层客户端 API 操作转换成相应的请求协议并发送到服务端，并完成对同步调用的返回和异步调用的回调。同时，`SendThread` 还负责将来自服务端的事件传递给 `EventThread` 去处理。

### 3.5 EventThread

`EventThread` 是客户端 `ClientCnxn` 内部的另一个核心线程，负责客户端的事件处理，并触发客户端注册的 `Watcher` 监听。`EventThread` 中有一个 `waitingEvents` 队列，用于临时存放那些需要被触发的 Object，包括那些客户端注册的 `Watcher` 和异步接口中注册的回调器 `AsyncCallback`。同时，`EventThread` 会不断地从 `waitingEvents` 这个队列中取出 Object，识别出具体类型（`Watcher` 或者 `AsyncCallback`），并分别调用 `process` 和 `processResult` 接口方法来实现对事件的触发和回调。