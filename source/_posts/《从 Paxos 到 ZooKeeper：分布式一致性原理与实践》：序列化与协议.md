---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：序列化与协议
date: 2017-12-06 20:05:22
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

## 1. 使用 Jute 进行序列化

使用 Jute 来对对象进行序列化和反序列化，大体可以分为 4 步：

1. 实体类需要实现 `Record` 接口的 `serialize` 和 `deserialize` 方法。
2. 构建一个序列化器 `ByteOutputArchive`。
3. 调用实体类的 `serialize` 方法，将对象序列化到指定 tag 中去。
4. 调用实体类的 `deserialize` 方法，从指定的 tag 中反序列化出数据内容。

## 2. 深入 Jute

### 2.1 Record 接口

Jute 定义了自己独特的序列化格式 `Record`。

`org.apache.jute.Record`

```java
public interface Record {
    public void serialize(OutputArchive archive, String tag)
        throws IOException;
    public void deserialize(InputArchive archive, String tag)
        throws IOException;
}
```

所有实体类通过实现 `Record` 接口的这两个方法，来定义自己将如何被序列化和反序列化。其中 `archive` 是底层真正的序列化器和反序列化器，并且每个 `archive` 中可以包含对多个对象的序列化和反序列化，因此两个接口方法中都标记了参数 `tag`，用于向序列化器和反序列化器标识对象自己的标记。

`OutputArchive` 和 `InputArchive` 分别是 Jute 底层的序列化器和反序列化器接口定义。在最新版本的 Jute 中，分别有 `BinaryOutputArchive`/`BinaryInputArchive`、`CsvoutputArchive`/`CsvInputArchive` 和 `XmlOutputArchive`/`XmlInputArchive` 三种实现。无论哪种实现，都是基于 `OutputStream` 和 `InputStream` 进行操作。

## 3. 通信协议

基于 TCP/IP 协议，ZooKeeper 实现了自己的通信协议来完成客户端与服务端、服务端与服务端之间的网络通信。ZooKeeper 通信协议整体上的设计非常简单，对于请求，主要包含请求头和请求体，而对于响应，则主要包含响应头和响应体。

### 3.1 协议解析：请求部分

#### 3.1.1 请求头：RequestHeader

`org.apache.zookeeper.proto.RequestHeader`

```java
public class RequestHeader implements Record {  
    private int xid;  
    private int type;  
}  
```

xid 用于记录客户端请求发起的先后序号，用来确保单个客户端请求的响应顺序。type 代表请求的操作类型，所有操作类型都被定义在类 `org.apache.zookeeper.ZooDefs.OpCode` 中。根据协议规定，除非是会话创建请求，其他所有的客户端请求中都会带上请求头。

#### 3.1.2 请求体：Request

协议的请求体部分是指请求的主体内容部分，包含了请求的所有操作内容。不同的请求类型，其请求体部分的结构是不同的。

`org.apache.zookeeper.proto.GetDataRequest`

```java
public class GetDataRequest implements Record {  
    private String path;  
    private boolean watch;  
}  
```

![image](https://user-images.githubusercontent.com/12514722/34356711-12dbb0c6-ea7b-11e7-9142-0857caa65127.png)

### 3.2 协议解析：响应部分

#### 3.2.1 响应头：ReplyHeader

`org.apache.zookeeper.proto.ReplyHeader`

```java
class ReplyHeader implements Record  {  
    int xid;  
    long zxid;  
    int err;  
}  
```

xid 与请求头中的xid一致，zxid 表示 ZooKeeper 服务器上当前最新的事务 ID，err 则是一个错误码，当请求处理过程中出现异常情况时，会在这个错误码中标识出来，

#### 3.2.2 响应体：Response

协议的响应体部分包含了响应的所有返回数据，不同的响应类型，其响应体部分的结构是不同的。

`org.apache.zookeeper.proto.GetDataResponse`

```java
class GetDataResponse implements Record {
    private byte[] data;
    private org.apache.zookeeper.data.Stat stat;
}
```

![image](https://user-images.githubusercontent.com/12514722/34356771-d2132276-ea7b-11e7-8793-d1ccde42ec79.png)