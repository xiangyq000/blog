---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：Watcher
date: 2017-12-07 21:04:27
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

ZooKeeper 允许客户端向服务端注册一个 Watcher 监听，当服务端的一些指定事件触发了这个 Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。

ZooKeeper 的 Watcher 机制主要包括客户端线程、客户端 WatchManager 和 ZooKeeper 服务器三部分。在具体工作流程上，简单地讲，客户端在向 ZooKeeper 服务器注册 Watcher 的同时，会将 Watcher 对象存储在客户端的 WatchManager 中。当 ZooKeeper 服务器端触发 Watcher 事件后，会向客户端发送通知，客户端线程从 WatchManager 中取出对应的 Watcher 对象来执行回调逻辑。

<!--more-->

## 1. Watcher 接口

在 ZooKeeper 中，接口类 `Watcher` 用于表示一个标准的事件处理器，其定义了事件通知相关的逻辑，包含 `KeeperState` 和 `EventType` 两个枚举类，分别代表了通知状态和事件类型，同时定义了事件的回调方法：`process(WatchedEvent event)`。

### 1.1 Watcher 事件

| KeeperState      | EventType           | 触发条件                                     | 说明                                       |
| ---------------- | ------------------- | ---------------------------------------- | ---------------------------------------- |
|                  | None（-1）            | 客户端与服务端成功建立连接                            |                                          |
|                  | NodeCreated（1）      | Watcher 监听的对应数据节点被创建                     |                                          |
| SyncConnected（0） | NodeDeleted（2）      | Watcher 监听的对应数据节点被删除                     | 此时客户端和服务器处于连接状态                          |
|                  | NodeDataChanged（3）  | Watcher 监听的对应数据节点的数据内容发生变更               |                                          |
|                  | NodeChildChanged（4） | Wather 监听的对应数据节点的子节点列表发生变更               |                                          |
| Disconnected（0）  | None（-1）            | 客户端与 ZooKeeper 服务器断开连接                   | 此时客户端和服务器处于断开连接状态                        |
| Expired（-112）    | Node（-1）            | 会话超时                                     | 此时客户端会话失效，通常同时也会受到 SessionExpiredException 异常 |
| AuthFailed（4）    | None（-1）            | 通常有两种情况。（1）使用错误的 schema 进行权限检查 （2）SASL 权限检查失败 | 通常同时也会收到 AuthFailedException 异常          |

### 1.2 回调方法 process()

`process` 方法是 `Watcher` 接口中的一个回调方法，当 ZooKeeper 向客户端发送一个 `Watcher` 事件通知时，客户端就会对相应的 `process` 方法进行回调，从而实现对事件的处理。

`org.apache.zookeeper.Watcher#process`

```java
    abstract public void process(WatchedEvent event);
```

`WatchedEvent` 包含了每一个事件的三个基本属性：通知状态（`keeperState`），事件类型（`EventType`）和节点路径（`path`）。

在这里提一下 `WathcerEvent` 实体。笼统地讲，两者表示的是同一个事物，都是对一个服务端事件的封装。不同的是，`WatchedEvent` 是一个逻辑事件，用于服务端和客户端程序执行过程中所需的逻辑对象，而 `WatcherEvent` 因为实现了序列化接口，因此可以用于网络传输。

`org.apache.zookeeper.proto.WatcherEvent`

```java
public class WatcherEvent implements Record {
    private int type;
    private int state;
    private String path;
}
```

服务端在生成 `WatchedEvent` 事件之后，会调用 `getWrapper` 方法将自己包装成一个可序列化的 `WatcherEvent` 事件，以便通过网络传输到客户端。客户端在接收到服务端的这个事件对象后，首先会将 `WatcherEvent` 还原成一个 `WatchedEvent` 事件，并传递给 `process` 方法处理，回调方法 `process` 根据入参就能够解析出完整的服务端事件了。

## 2. 工作机制

### 2.1 客户端注册 Watcher

在创建一个 ZooKeeper 客户端的实例时可以向构造方法中传入一个默认的 Watcher：

```java
public ZooKeeper(String connectString，int sessionTimeout,Watcher watcher);
```

这个 Watcher 将作为整个 ZooKeeper 会话期间的默认 Watcher，会一直被保存在客户端 `ZKWatchManager` 的 `defaultWatcher` 中。另外，ZooKeeper 客户端也可以通过 `getData`，`getChildren` 和 `exist` 三个接口来向 ZooKeeper 服务器注册 Watcher，无论使用哪种方式，注册 Watcher 的工作原理都是一致的。

以 `org.apache.zookeeper.ZooKeeper#getData(java.lang.String, org.apache.zookeeper.Watcher, org.apache.zookeeper.data.Stat)` 为例：

```java
    public byte[] getData(final String path, Watcher watcher, Stat stat)
        throws KeeperException, InterruptedException
     {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        // the watch contains the un-chroot path
        WatchRegistration wcb = null;
        if (watcher != null) {
            wcb = new DataWatchRegistration(watcher, clientPath);
        }

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.getData);
        GetDataRequest request = new GetDataRequest();
        request.setPath(serverPath);
        request.setWatch(watcher != null);
        GetDataResponse response = new GetDataResponse();
        ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        if (stat != null) {
            DataTree.copyStat(response.getStat(), stat);
        }
        return response.getData();
    }
```

在向 `getData` 接口注册 Watcher 后，客户端首先会对当前客户端请求 `request` 进行标记，将其设置为 “使用 Watcher 监听”，同时会封装一个 Watcher 的注册信息 `WatchRegistration` 对象，用于暂时保存数据节点的路径和 Watcher 的对应关系。

在 ZooKeeper 中，`Packet` 可以被看作一个最小的通信协议单元，用于进行客户端与服务端之间的网络传输，任何需要传输的对象都需要包装成一个 `Packet` 对象。因此，在 `ClientCnxn` 中 `WatchRegistration` 又会被封装到 `Packet` 中，然后放入发送队列中等待客户端发送：

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

随后，ZooKeeper 客户端就会向服务端发送这个请求，同时等待请求的返回。完成请求发送后，会由客户端 `SendThread` 线程的 `readResponse` 方法负责接收来自服务端的响应，`finishPacket` 方法会从 `Packet` 中取出对应的 Watcher 并注册到 `ZkWatchManager` 中去：

`org.apache.zookeeper.ClientCnxn#finishPacket`

```java
    private void finishPacket(Packet p) {
        if (p.watchRegistration != null) {
            p.watchRegistration.register(p.replyHeader.getErr());
        }

        if (p.cb == null) {
            synchronized (p) {
                p.finished = true;
                p.notifyAll();
            }
        } else {
            p.finished = true;
            eventThread.queuePacket(p);
        }
    }
```

从上面的内容中，我们已经了解到客户端已经将 Watcher 暂时封装在了 `WatchRegistration` 对象中，现在就需要从这个封装对象中再次提取出 Watcher 来：

`org.apache.zookeeper.ZooKeeper.WatchRegistration#register`

```java
        abstract protected Map<String, Set<Watcher>> getWatches(int rc);
        public void register(int rc) {
            if (shouldAddWatch(rc)) {
                Map<String, Set<Watcher>> watches = getWatches(rc);
                synchronized(watches) {
                    Set<Watcher> watchers = watches.get(clientPath);
                    if (watchers == null) {
                        watchers = new HashSet<Watcher>();
                        watches.put(clientPath, watchers);
                    }
                    watchers.add(watcher);
                }
            }
        }
```

`org.apache.zookeeper.ZooKeeper.ZKWatchManager`

```java
    private static class ZKWatchManager implements ClientWatchManager {
        private final Map<String, Set<Watcher>> dataWatches =
            new HashMap<String, Set<Watcher>>();
        private final Map<String, Set<Watcher>> existWatches =
            new HashMap<String, Set<Watcher>>();
        private final Map<String, Set<Watcher>> childWatches =
            new HashMap<String, Set<Watcher>>();

        private volatile Watcher defaultWatcher;
    }
```

在 `register` 方法中，客户端会将之前暂时保存的 Watcher 对象转交给 `ZKWatchManager`，并最终保存到 `dataWatches` 中去。`ZKWatchManager.dataWatches` 是一个 `Map<String, Set<Watcher>>` 类型的数据结构，用于将数据节点的路径和 Watcher 对象进行一一映射后管理起来。

在 `Packet.createBB()` 中，ZooKeeper 只会将 `requestHeader` 和 `reqeust` 两个属性进行序列化，也就是说，尽管 `WatchResgistration` 被封装在了 `Packet` 中，但是并没有被序列化到底层字节数组中去，因此也就不会进行网络传输了。

### 2.2 服务端处理 Watcher

#### 2.2.1 服务端注册 Watcher

服务端收到来自客户端的请求后，在 `org.apache.zookeeper.server.FinalRequestProcessor#processRequest` 中会判断当前请求是否需要注册 Watcher：

```java
            case OpCode.getData: {
                lastOp = "GETD";
                GetDataRequest getDataRequest = new GetDataRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request, getDataRequest);
                DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
                if (n == null) {
                    throw new KeeperException.NoNodeException();
                }
                PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n), ZooDefs.Perms.READ, request.authInfo);
                Stat stat = new Stat();
                byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat, getDataRequest.getWatch() ? cnxn : null);
                rsp = new GetDataResponse(b, stat);
                break;
            }
```

从 `getData` 请求的处理逻辑中，我们可以看到，当 `getDataRequest.getWatch()` 为 true 的时候，ZooKeeper 就认为当前客户端请求需要进行 Watcher 注册，于是就会将当前的 `ServerCnxn` 对象作为一个 Watcher 连同数据节点路径传入 `getData` 方法中去。注意到，抽象类 `ServerCnxn` 实现了 `Watcher` 接口。

数据节点的节点路径和 `ServerCnxn` 最终会被存储在 `WatcherManager` 的 `watchTable` 和 `watch2Paths` 中。`WatchManager` 是 ZooKeeper 服务端 Watcher 的管理者，其内部管理的 `watchTable` 和 `watch2Pashs` 两个存储结构，分别从两个维度对 Watcher 进行存储。

- `watchTable` 是从数据节点路径的粒度来托管 Watcher。
- `watch2Paths` 是从 Watcher 的粒度来控制事件触发需要触发的数据节点。

`org.apache.zookeeper.server.WatchManager#addWatch`

```java
    public synchronized void addWatch(String path, Watcher watcher) {
        HashSet<Watcher> list = watchTable.get(path);
        if (list == null) {
            // don't waste memory if there are few watches on a node
            // rehash when the 4th entry is added, doubling size thereafter
            // seems like a good compromise
            list = new HashSet<Watcher>(4);
            watchTable.put(path, list);
        }
        list.add(watcher);

        HashSet<String> paths = watch2Paths.get(watcher);
        if (paths == null) {
            // cnxns typically have many watches, so use default cap here
            paths = new HashSet<String>();
            watch2Paths.put(watcher, paths);
        }
        paths.add(path);
    }
```

#### 2.2.2 Watcher 触发

`org.apache.zookeeper.server.DataTree#setData`

```java
    public Stat setData(String path, byte data[], int version, long zxid,
            long time) throws KeeperException.NoNodeException {
        Stat s = new Stat();
        DataNode n = nodes.get(path);
        if (n == null) {
            throw new KeeperException.NoNodeException();
        }
        byte lastdata[] = null;
        synchronized (n) {
            lastdata = n.data;
            n.data = data;
            n.stat.setMtime(time);
            n.stat.setMzxid(zxid);
            n.stat.setVersion(version);
            n.copyStat(s);
        }
        // now update if the path is in a quota subtree.
        String lastPrefix;
        if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
          this.updateBytes(lastPrefix, (data == null ? 0 : data.length)
              - (lastdata == null ? 0 : lastdata.length));
        }
        dataWatches.triggerWatch(path, EventType.NodeDataChanged);
        return s;
    }
```

在对指定节点进行数据更新后，通过调用 `org.apache.zookeeper.server.WatchManager#triggerWatch`方法来触发相关的事件：

```java
    public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
        WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
        HashSet<Watcher> watchers;
        synchronized (this) {
            watchers = watchTable.remove(path);
            if (watchers == null || watchers.isEmpty()) {
                return null;
            }
            for (Watcher w : watchers) {
                HashSet<String> paths = watch2Paths.get(w);
                if (paths != null) {
                    paths.remove(path);
                }
            }
        }
        for (Watcher w : watchers) {
            if (supress != null && supress.contains(w)) {
                continue;
            }
            w.process(e);
        }
        return watchers;
    }
```

无论是 `dataWatches` 还是 `childWatches` 管理器，Watcher 的触发逻辑都是一致的，基本步骤如下。

1. 封装 `WatchedEvent`。

   首先将通知状态（`KeeperState`）、事件类型（`EventType`）以及节点路径（`Path`）封装成一个 `WatchedEvent` 对象。

2. 查询 Watcher。

   根据数据节点的节点路径从 `watchTable` 中取出对应的 Watcher。如果没有找到 Watcher，说明没有任何客户端在该数据节点上注册过 Watcher，直接退出。而如果找到了这个 Watcher，会将其提取出来，同时会直接从 `watchTable` 和 `watch2Paths` 中将其删除——从这里我们也可以看出，Watcher 在服务端是一次性的，即触发一次就失效了。

3. 调用 `process` 方法来触发 Watcher。

   在这一步中，会逐个依次地调用从步骤2中找出的所有 Watcher 的 `process` 方法。这里的 `process` 方法，事实上就是 `ServerCnxn` 的对应方法：

   `org.apache.zookeeper.server.NIOServerCnxn#process`

   ```java
       @Override
       synchronized public void process(WatchedEvent event) {
           ReplyHeader h = new ReplyHeader(-1, -1L, 0);
           // Convert WatchedEvent to a type that can be sent over the wire
           WatcherEvent e = event.getWrapper();

           sendResponse(h, e, "notification");
       }
   ```

   在 `process` 方法中，主要逻辑如下。

   - 在请求头中标记 “-1”，表明当前是一个通知。
   - 将 `WawtchedEvent` 包装成 `WatcherEvent`，以便进行网络传输序列化。
   - 向客户端发送该通知。

## 3. 客户端回调 Watcher

### 3.1 SendThread 接收事件通知

对于一个来自服务端的响应，客户端都是由 `org.apache.zookeeper.ClientCnxn.SendThread#readResponse` 方法来统一进行处理的，如果响应头 `replyHdr` 中标识了 XID 为 -1，表明这是一个通知类型的响应。

```java
            if (replyHdr.getXid() == -1) {
                // -1 means notification
                WatcherEvent event = new WatcherEvent();
                event.deserialize(bbia, "response");

                // convert from a server path to a client path
                if (chrootPath != null) {
                    String serverPath = event.getPath();
                    if(serverPath.compareTo(chrootPath)==0)
                        event.setPath("/");
                    else if (serverPath.length() > chrootPath.length())
                        event.setPath(serverPath.substring(chrootPath.length()));
                    else {
                    	LOG.warn("Got server path " + event.getPath()
                    			+ " which is too short for chroot path "
                    			+ chrootPath);
                    }
                }
                WatchedEvent we = new WatchedEvent(event);
                eventThread.queueEvent( we );
                return;
            }
```

处理过程大体上分为以下 4 个主要步骤：

1. 反序列化。

   将字节流转换成 `WatcherEvent` 对象。

2. 处理 chrootPath。

   如果客户端设置了 chrootPath 属性，那么需要对服务端传过来的完整的节点路径进行 `chrootPath` 处理，生成客户端的一个相对节点路径。

3. 还原 `WatchedEvent`。

   将 `WatcherEvent` 对象转换成 `WatchedEvent`。

4. 回调 Watcher。

   将 `WatchedEvent` 对象交给 `EventThread` 线程，在下一个轮询周期中进行 Watcher 回调。

### 3.2 EventThread 处理事件通知

`SendThread` 接收到服务端的通知事件后，会通过调用 `EventThread.queueEvent` 方法将事件传给 `EventThread` 线程，其逻辑如下：

`org.apache.zookeeper.ClientCnxn.EventThread#queueEvent`

```java
public void queueEvent(WatchedEvent event) {
    if (event.getType() == EventType.None
            && sessionState == event.getState()) {
        return;
    }
    sessionState = event.getState();

    // materialize the watchers based on the event
    WatcherSetEventPair pair = new WatcherSetEventPair(
            watcher.materialize(event.getState(), event.getType(),event.getPath()), event);
    // queue the pair (watch set & event) for later processing
    waitingEvents.add(pair);
}
```

`queueEvent` 方法首先会根据该通知事件，从 `ZKWatchManager` 中取出所有相关的 Watcher：

```java
        @Override
        public Set<Watcher> materialize(Watcher.Event.KeeperState state, Watcher.Event.EventType type, String clientPath) {
            Set<Watcher> result = new HashSet<Watcher>();

            switch (type) {
            // ...            
            case NodeDataChanged:
            case NodeCreated:
                synchronized (dataWatches) {
                    addTo(dataWatches.remove(clientPath), result);
                }
                synchronized (existWatches) {
                    addTo(existWatches.remove(clientPath), result);
                }
                break;
            // ...
            }

            return result;
        }
    }
```

客户端在识别出事件类型 `EventType` 后，会从相应的 Watcher 存储（即 `dataWatches`，`existWatches` 或 `childWatches` 中的一个或多个）中去除对应的 Watcher。注意，此处使用的是 `remove` 接口，因此也表明了客户端的 Watcher 机制同样也是一次性的，即一旦被触发后，该 Watcher 就失效了。

获取到相关的所有 Watcher 后，会将其放入 `waitingEvents` 这个队列中去。`WaitingEvents` 是一个待处理 Watcher 队列，`EventThread` 的 `run` 方法会不断对该队列进行处理。`EventThread` 线程每次都会从 `waitingEvents` 队列中取出一个 Watcher，并进行串行同步处理。注意，此处 `processEvent` 方法中的 `Watcher` 才是之前客户端真正注册的 Watcher，调用其 `process` 方法就可以实现 Watcher 的回调了。

## 4. 自问自答

### 4.1 连接中断时，客户端如何处理？

1. 客户端抛出 `EndOfStreamException` 异常，此时客户端状态还是 `CONNECTED`。

2. `SendThread` 处理异常，清理连接，将当前所有请求置为失败，错误码是 `CONNECTIONLOSS`。

   `org.apache.zookeeper.ClientCnxn.SendThread#cleanup`

   ```java
           private void cleanup() {
               clientCnxnSocket.cleanup();
               synchronized (pendingQueue) {
                   for (Packet p : pendingQueue) {
                       conLossPacket(p);
                   }
                   pendingQueue.clear();
               }
               synchronized (outgoingQueue) {
                   for (Packet p : outgoingQueue) {
                       conLossPacket(p);
                   }
                   outgoingQueue.clear();
               }
           }
   ```

   `org.apache.zookeeper.ClientCnxn#conLossPacket`

   ```java
       private void conLossPacket(Packet p) {
           if (p.replyHeader == null) {
               return;
           }
           switch (state) {
           case AUTH_FAILED:
               p.replyHeader.setErr(KeeperException.Code.AUTHFAILED.intValue());
               break;
           case CLOSED:
               p.replyHeader.setErr(KeeperException.Code.SESSIONEXPIRED.intValue());
               break;
           default:
               p.replyHeader.setErr(KeeperException.Code.CONNECTIONLOSS.intValue());
           }
           finishPacket(p);
       }
   ```

3. 创建 `None-Disconnected` 事件，发送给 `EventThread`。此时 ZooKeeper 客户端仍持有之前注册的所有 Watcher。

### 4.2 客户端如何在重连后重新向服务器注册 Watcher？

1. `SendThread` 选下一个服务器地址请求 TCP 连接。

2. 连上之后发送 `ConnectRequest`，其中 `sessionid` 和 `password`是当前会话的数据。

3. 假设客户端重试比较快，session 还没超时，则服务端返回连接成功的 `ConnectResponse` 。（反之若 session 过期，则校验失败，客户端会抛出 `SessionExpired` 异常并退出。

4. 客户端收到相应，发送 `SyncConnected` 事件。

5. 客户端发送 `SetWatches` 请求，重建 Watcher。这个包实际上是在 `SendThread` 调用 `org.apache.zookeeper.ClientCnxn.SendThread#primeConnection` 方法时，和 `ConnectRequest` 请求先后添加到发送队列中的。

   ```java
           void primeConnection() throws IOException {
               isFirstConnect = false;
               long sessId = (seenRwServerBefore) ? sessionId : 0;
               ConnectRequest conReq = new ConnectRequest(0, lastZxid, sessionTimeout, sessId, sessionPasswd);
               synchronized (outgoingQueue) {
                   // We add backwards since we are pushing into the front
                   // Only send if there's a pending watch
                   // TODO: here we have the only remaining use of zooKeeper in
                   // this class. It's to be eliminated!
                   if (!disableAutoWatchReset) {
                       List<String> dataWatches = zooKeeper.getDataWatches();
                       List<String> existWatches = zooKeeper.getExistWatches();
                       List<String> childWatches = zooKeeper.getChildWatches();
                       if (!dataWatches.isEmpty() || !existWatches.isEmpty() || !childWatches.isEmpty()) {

                           Iterator<String> dataWatchesIter = prependChroot(dataWatches).iterator();
                           Iterator<String> existWatchesIter = prependChroot(existWatches).iterator();
                           Iterator<String> childWatchesIter = prependChroot(childWatches).iterator();
                           long setWatchesLastZxid = lastZxid;

                           while (dataWatchesIter.hasNext() || existWatchesIter.hasNext() || childWatchesIter.hasNext()) {
                               List<String> dataWatchesBatch = new ArrayList<String>();
                               List<String> existWatchesBatch = new ArrayList<String>();
                               List<String> childWatchesBatch = new ArrayList<String>();
                               int batchLength = 0;

                               // Note, we may exceed our max length by a bit when we add the last
                               // watch in the batch. This isn't ideal, but it makes the code simpler.
                               while (batchLength < SET_WATCHES_MAX_LENGTH) {
                                   final String watch;
                                   if (dataWatchesIter.hasNext()) {
                                       watch = dataWatchesIter.next();
                                       dataWatchesBatch.add(watch);
                                   } else if (existWatchesIter.hasNext()) {
                                       watch = existWatchesIter.next();
                                       existWatchesBatch.add(watch);
                                   } else if (childWatchesIter.hasNext()) {
                                       watch = childWatchesIter.next();
                                       childWatchesBatch.add(watch);
                                   } else {
                                       break;
                                   }
                                   batchLength += watch.length();
                               }

                               SetWatches sw = new SetWatches(setWatchesLastZxid,
                                       dataWatchesBatch,
                                       existWatchesBatch,
                                       childWatchesBatch);
                               RequestHeader h = new RequestHeader();
                               h.setType(ZooDefs.OpCode.setWatches);
                               h.setXid(-8);
                               Packet packet = new Packet(h, new ReplyHeader(), sw, null, null);
                               outgoingQueue.addFirst(packet);
                           }
                       }
                   }

                   for (AuthData id : authInfo) {
                       outgoingQueue.addFirst(new Packet(new RequestHeader(-4,
                               OpCode.auth), null, new AuthPacket(0, id.scheme,
                               id.data), null, null));
                   }
                   outgoingQueue.addFirst(new Packet(null, null, conReq, null, null, readOnly));
               }
               clientCnxnSocket.enableReadWriteOnly();
           }
   ```

6. 服务端处理 `SetWatches` 请求。