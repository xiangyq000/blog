---
title: ZooKeeper Java Api
date: 2017-01-01 14:41:17
tags: ZooKeeper
categories: ZooKeeper
---

## Api 测试

<!--more-->

```java
package ZooKeeperApi;

import com.alibaba.fastjson.JSON;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import java.util.List;

/**
 * Created by xyq on 2017/12/28.
 */
public class ZooKeeperApi {

    private static final String addr = "localhost:2181";
    private static final int timeout = 5000;

    public static void main(String[] args) {

        try {
            // 创建 ZooKeeper 实例
            ZooKeeper zooKeeper = new ZooKeeper(addr, timeout, new MyWatcher());
            while(zooKeeper.getState() != ZooKeeper.States.CONNECTED) {}

            long sessionId = zooKeeper.getSessionId();
            byte[] passwd = zooKeeper.getSessionPasswd();
            System.out.println("ZooKeeper connected, sessionId=" + sessionId);

            // 同步创建临时顺序节点
            String syncCreateRes = zooKeeper.create("/sync-ephemeral-sequential-znode-", "sync_create".getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            System.out.println("sync create znode, path=" + syncCreateRes);

            // 异步检测临时顺序节点是否存在, 并注册 Watcher
            zooKeeper.exists(syncCreateRes, true, new MyStatCallback(), "ctx");

            // 异步删除临时顺序节点
            zooKeeper.delete(syncCreateRes, -1, new MyVoidCallback(), "ctx");

            // 同步创建持久顺序节点
            String persistentNode = zooKeeper.create("/async-persistent-sequential-znode-", "async_create".getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);

            // 异步获取持久顺序节点数据，并注册 Watcher
            zooKeeper.getData(persistentNode, true, new MyDataCallback(), "ctx");

            // 异步更新持久顺序节点数据
            zooKeeper.setData(persistentNode, "new_data".getBytes(), -1, new MyStatCallback(), "ctx");

            // 异步获取持久顺序节点子节点, 并注册 Watcher
            zooKeeper.getChildren(persistentNode, true, new MyChildren2Callback(), "ctx");
            // 异步在持久顺序节点下创建临时子节点
            zooKeeper.create(persistentNode + "/child", "child".getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL, new MyStringCallback(), "ctx");

            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static class MyWatcher implements Watcher {
        @Override
        public void process(WatchedEvent watchedEvent) {
            System.out.println("receive an event, keeperState=" + watchedEvent.getState() +
                    ", type=" + watchedEvent.getType());
        }
    }

    static class MyStringCallback implements AsyncCallback.StringCallback {
        @Override
        public void processResult(int i, String s, Object o, String s1) {
            System.out.println("async create znode result, resultCode=" + i +
                    ", path=" + s + ", ctx=" + o + ", realPath=" + s1);
        }
    }

    static class MyVoidCallback implements AsyncCallback.VoidCallback {
        @Override
        public void processResult(int i, String s, Object o) {
            System.out.println("async delete znode result, resultCode=" + i + ", path=" + s + ", ctx=" + o);
        }
    }

    static class MyChildren2Callback implements AsyncCallback.Children2Callback {
        @Override
        public void processResult(int i, String s, Object o, List<String> list, Stat stat) {
            System.out.println("async get znode children result, resultCode=" + i + ", path=" + s +
                    ", ctx=" + o + ", childrenList=" + JSON.toJSONString(list) + ", stat=" + JSON.toJSONString(stat));
        }
    }

    static class MyStatCallback implements AsyncCallback.StatCallback {
        @Override
        public void processResult(int i, String s, Object o, Stat stat) {
            System.out.println("async judge znode existence result, resultCode=" + i + ", path=" + s +
                    ", ctx=" + o + ", stat=" + JSON.toJSONString(stat));
        }
    }

    static class MyDataCallback implements AsyncCallback.DataCallback {
        @Override
        public void processResult(int i, String s, Object o, byte[] bytes, Stat stat) {
            System.out.println("async get znode data result, resultCode=" + i + ", path=" + s +
                    ", ctx=" + o + ", stat=" + JSON.toJSONString(stat));
        }
    }
}
```

运行结果：

```
ZooKeeper connected, sessionId=99251216681467907
receive an event, keeperState=SyncConnected, type=None
sync create znode, path=/sync-ephemeral-sequential-znode-0000000064
async judge znode existence result, resultCode=0, path=/sync-ephemeral-sequential-znode-0000000064, ctx=ctx, stat={"aversion":0,"ctime":1514453932664,"cversion":0,"czxid":1320,"dataLength":11,"ephemeralOwner":99251216681467907,"mtime":1514453932664,"mzxid":1320,"numChildren":0,"pzxid":1320,"version":0}
receive an event, keeperState=SyncConnected, type=NodeDeleted
async delete znode result, resultCode=0, path=/sync-ephemeral-sequential-znode-0000000064, ctx=ctx
async get znode data result, resultCode=0, path=/async-persistent-sequential-znode-0000000065, ctx=ctx, stat={"aversion":0,"ctime":1514453932674,"cversion":0,"czxid":1322,"dataLength":12,"ephemeralOwner":0,"mtime":1514453932674,"mzxid":1322,"numChildren":0,"pzxid":1322,"version":0}
receive an event, keeperState=SyncConnected, type=NodeDataChanged
async judge znode existence result, resultCode=0, path=/async-persistent-sequential-znode-0000000065, ctx=ctx, stat={"aversion":0,"ctime":1514453932674,"cversion":0,"czxid":1322,"dataLength":8,"ephemeralOwner":0,"mtime":1514453932680,"mzxid":1323,"numChildren":0,"pzxid":1322,"version":1}
async get znode children result, resultCode=0, path=/async-persistent-sequential-znode-0000000065, ctx=ctx, childrenList=[], stat={"aversion":0,"ctime":1514453932674,"cversion":0,"czxid":1322,"dataLength":8,"ephemeralOwner":0,"mtime":1514453932680,"mzxid":1323,"numChildren":0,"pzxid":1322,"version":1}
receive an event, keeperState=SyncConnected, type=NodeChildrenChanged
async create znode result, resultCode=0, path=/async-persistent-sequential-znode-0000000065/child, ctx=ctx, realPath=/async-persistent-sequential-znode-0000000065/child
```

## 小结

1. ZooKeeper 不支持递归创建，即无法在父节点不存在的情况下创建一个子节点。另外，如果一个节点已经存在了，那么创建同名节点的时候会抛出 `NodeExistsException` 异常。

2. 目前，ZooKeeper 的节点内容只支持字节数组（`byte[]`）类型。

3. 在同步接口调用中，我们需要关注接口抛出异常的可能，但是在异步接口中，所有的异常再回调函数中通过 Result Code 来体现。

   `org.apache.zookeeper.KeeperException.Code`

   ```java
       public static enum Code implements CodeDeprecated {
           /** Everything is OK */
           OK (Ok),

           /** System and server-side errors.
            * This is never thrown by the server, it shouldn't be used other than
            * to indicate a range. Specifically error codes greater than this
            * value, but lesser than {@link #APIERROR}, are system errors.
            */
           SYSTEMERROR (SystemError),

           /** A runtime inconsistency was found */
           RUNTIMEINCONSISTENCY (RuntimeInconsistency),
           /** A data inconsistency was found */
           DATAINCONSISTENCY (DataInconsistency),
           /** Connection to the server has been lost */
           CONNECTIONLOSS (ConnectionLoss),
           /** Error while marshalling or unmarshalling data */
           MARSHALLINGERROR (MarshallingError),
           /** Operation is unimplemented */
           UNIMPLEMENTED (Unimplemented),
           /** Operation timeout */
           OPERATIONTIMEOUT (OperationTimeout),
           /** Invalid arguments */
           BADARGUMENTS (BadArguments),
           
           /** API errors.
            * This is never thrown by the server, it shouldn't be used other than
            * to indicate a range. Specifically error codes greater than this
            * value are API errors (while values less than this indicate a
            * {@link #SYSTEMERROR}).
            */
           APIERROR (APIError),

           /** Node does not exist */
           NONODE (NoNode),
           /** Not authenticated */
           NOAUTH (NoAuth),
           /** Version conflict */
           BADVERSION (BadVersion),
           /** Ephemeral nodes may not have children */
           NOCHILDRENFOREPHEMERALS (NoChildrenForEphemerals),
           /** The node already exists */
           NODEEXISTS (NodeExists),
           /** The node has children */
           NOTEMPTY (NotEmpty),
           /** The session has been expired by the server */
           SESSIONEXPIRED (SessionExpired),
           /** Invalid callback specified */
           INVALIDCALLBACK (InvalidCallback),
           /** Invalid ACL specified */
           INVALIDACL (InvalidACL),
           /** Client authentication failed */
           AUTHFAILED (AuthFailed),
           /** Session moved to another server, so operation is ignored */
           SESSIONMOVED (-118),
           /** State-changing request is passed to read-only server */
           NOTREADONLY (-119);
       }
   ```

4. 在更新数据时，`version` 参数使用 -1 表示针对最新版本进行更新。

5. 无论节点是否存在，通过调用 `exists` 接口都可以注册 Watcher，能够对节点创建、节点删除和节点数据更新事件进行监听，但不监听子节点的各种变化。