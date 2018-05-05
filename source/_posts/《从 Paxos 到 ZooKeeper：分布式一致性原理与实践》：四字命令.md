---
title: 《从 Paxos 到 ZooKeeper：分布式一致性原理与实践》：四字命令
date: 2017-12-09 11:27:25
tags: [ZooKeeper, 源码阅读]
categories: [ZooKeeper, 读书笔记]
---

使用方式：

- telnet

  ```shell
  $ telnet localhost 2181
  Trying ::1...
  Connected to localhost.
  Escape character is '^]'.
  conf
  clientPort=2181
  dataDir=/usr/local/var/lib/zookeeper/version-2
  dataLogDir=/usr/local/var/lib/zookeeper/version-2
  tickTime=3000
  maxClientCnxns=0
  minSessionTimeout=6000
  maxSessionTimeout=60000
  serverId=0
  Connection closed by foreign host.
  ```

- nc

  ```shell
  $ echo conf | nc localhost 2181
  clientPort=2181
  dataDir=/usr/local/var/lib/zookeeper/version-2
  dataLogDir=/usr/local/var/lib/zookeeper/version-2
  tickTime=3000
  maxClientCnxns=0
  minSessionTimeout=6000
  maxSessionTimeout=60000
  serverId=0
  ```

<!--more-->

以下示例基于在本地以默认配置起单实例ZooKeeper和Kafka，Kafka连接到ZooKeeper。Mac系统下ZooKeeper配置文件目录为`/usr/local/etc/zookeeper`。

#### 1. conf
conf命令用于输出ZooKeeper服务器运行时使用的基本配置信息，包括clientPort，dataDir和tickTime等。

conf会根据当前的运行模式来决定打印输出的服务器配置信息，如果是单机模式(standalone)，不会输出 initLimit、syncLimit、electionAlg以及electionPort等集群配置信息。

```shell
$ echo conf | nc localhost 2181
clientPort=2181
dataDir=/usr/local/var/lib/zookeeper/version-2
dataLogDir=/usr/local/var/lib/zookeeper/version-2
tickTime=3000
maxClientCnxns=0
minSessionTimeout=6000
maxSessionTimeout=60000
serverId=0
```

#### 2. cons
cons命令用于输出当前这台服务器上所有客户端连接的详细信息，包括每个客户端的客户端IP、会话ID和最后一次与服务器交互的操作类型等。

```shell
$ echo cons | nc localhost 2181
 /127.0.0.1:51436[1](queued=0,recved=493,sent=495,sid=0x15e983124a00000,lop=PING,est=1505792940222,to=6000,lcxid=0x17c,lzxid=0x478,lresp=1505793166428,llat=0,minlat=0,avglat=0,maxlat=129)
 /0:0:0:0:0:0:0:1:51550[0](queued=0,recved=1,sent=0)
```

#### 3. crst
crst命令用于重置所有客户端的连接统计信息。

```shell
$ echo crst | nc localhost 2181
Connection stats reset.
```

#### 4. dump
dump命令用于输出当前集群的所有会话信息，包括这些会话的会话ID，以及每个会话创建的临时节点等信息。另外，如果在Leader服务器上执行该命令的话，还能够看到每个会话的超时时间。

```shell
$ echo dump | nc localhost 2181
SessionTracker dump:
Session Sets (3):
0 expire at Tue Sep 19 11:55:51 CST 2017:
0 expire at Tue Sep 19 11:55:54 CST 2017:
1 expire at Tue Sep 19 11:55:57 CST 2017:
	0x15e983124a00000
ephemeral nodes dump:
Sessions with Ephemerals (1):
0x15e983124a00000:
	/controller
	/brokers/ids/0
```

#### 5. envi
envi命令用于输出ZooKeeper所在服务器运行时的环境信息，包括os.version，java.version和user.home等。

```shell
$ echo envi | nc localhost 2181
Environment:
zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
host.name=bogon
java.version=1.8.0_91
java.vendor=Oracle Corporation
java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre
java.class.path=
java.library.path=/Users/xyq/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
java.io.tmpdir=/var/folders/yq/d0v9dm456zd_xzc1lhllw19m0000gn/T/
java.compiler=<NA>
os.name=Mac OS X
os.arch=x86_64
os.version=10.12.6
user.name=xyq
user.home=/Users/xyq
user.dir=/Users/xyq
```

#### 6. ruok
ruok命令即“are you ok”，用于输出当前ZooKeeper服务器是否正在运行。执行该命令后，如果当前ZooKeeper服务器正在运行，那么返回“imok”，否则没有任何响应输出。

ruok命令的输出仅仅只能表明当前服务器是否正在运行，准确地讲，只能说明2181端口打开着，同时四字命令执行流程正常，但是不能代表ZooKeeper服务器是否运行正常。在很多时候，如果当前服务器无法正常处理客户端的读写请求，甚至已经无法和集群中的其他机器通信，ruok命令依然返回“imok”。

```shell
$ echo ruok | nc localhost 2181
imok
```

#### 7. stat
stat命令用于获取ZooKeeper服务器的运行时状态信息，包括基本的ZooKeeper版本，打包信息，运行时角色，集群数据节点个数等信息，另外还会将当前服务器的客户端连接信息打印出来。

```shell
$ echo stat | nc localhost 2181
Zookeeper version: 3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
Clients:
 /0:0:0:0:0:0:0:1:51369[0](queued=0,recved=1,sent=0)
	
Latency min/avg/max: 0/0/0
Received: 4
Sent: 3
Connections: 1
Outstanding: 0
Zxid: 0x43c
Mode: standalone
Node count: 140
```

#### 8. srvr
srvr命令和stat命令的功能一致，唯一的区别是srvr不会将客户端的连接情况输出，仅仅输出服务器的自身信息。

```shell
$ echo srvr | nc localhost 2181
Zookeeper version: 3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
Latency min/avg/max: 0/0/129
Received: 1979
Sent: 1980
Connections: 2
Outstanding: 0
Zxid: 0x478
Mode: standalone
Node count: 142
```

#### 9. wchs
wchs命令用于输出当前服务器上管理的Watchers的概要信息。

```shell
$ echo wchs | nc localhost 2181
1 connections watching 11 paths
Total watches:11
```

#### 10. wchc
wchc命令用于输出当前服务器上管理的Watchers的详细信息，以会话为单位进行归组，同时列出被该会话注册了Watcher的节点路径。

```shell
$ echo wchc | nc localhost 2181
0x15e9aac69320000
	/controller
	/isr_change_notification
	/admin/preferred_replica_election
	/admin/reassign_partitions
	/brokers/ids
	/admin/delete_topics
	/brokers/topics/springCloudBus
	/config/changes
	/brokers/topics/sleuth
	/brokers/topics/__consumer_offsets
	/brokers/topics
```

#### 10. wchp
wchp命令和wchc命令非常类似，也是用于输出当前服务器上管理的Watchers的详细信息，不同点在于wchp命令的输出信息以节点路径为单位进行归组。

```shell
$ echo wchp | nc localhost 2181
/controller
	0x15e9aac69320000
/isr_change_notification
	0x15e9aac69320000
/admin/preferred_replica_election
	0x15e9aac69320000
/admin/reassign_partitions
	0x15e9aac69320000
/brokers/ids
	0x15e9aac69320000
/admin/delete_topics
	0x15e9aac69320000
/brokers/topics/springCloudBus
	0x15e9aac69320000
/config/changes
	0x15e9aac69320000
/brokers/topics/sleuth
	0x15e9aac69320000
/brokers/topics/__consumer_offsets
	0x15e9aac69320000
/brokers/topics
	0x15e9aac69320000
```

#### 11. mntr
mntr命令用于输出比stat命令更为详尽的服务器统计信息，包括请求处理的延迟情况、服务器内存数据库大小和集群的数据同步情况。

```shell
$ echo mntr | nc localhost 2181
zk_version	3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
zk_avg_latency	0
zk_max_latency	129
zk_min_latency	0
zk_packets_received	2060
zk_packets_sent	2061
zk_num_alive_connections	2
zk_outstanding_requests	0
zk_server_state	standalone
zk_znode_count	142
zk_watch_count	16
zk_ephemerals_count	2
zk_approximate_data_size	10973
zk_open_file_descriptor_count	97
zk_max_file_descriptor_count	10240
```

#### 12. isro
New in 3.4.0: Tests if server is running in read-only mode. The server will respond with "ro" if in read-only mode or "rw" if not in read-only mode.

```shell
$ echo isro | nc localhost 2181
rw
```

#### 13. gtmk
Gets the current trace mask as a 64-bit signed long value in decimal format.

```shell
$ echo gtmk | nc localhost 2181
306
```