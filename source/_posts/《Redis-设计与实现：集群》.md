---
title: 《Redis 设计与实现：集群》
date: 2017-10-07 15:42:23
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

## 1. 节点

一个 Redis 集群通常由多个节点（node）组成，在刚开始的时候，每个节点都是相互独立的，它们都处于一个只包含自己的集群当中，要组建一个真正可工作的集群，我们必须将各个独立的节点连接起来，构成一个包含多个节点的集群。

连接各个节点的工作可以使用 `CLUSTER MEET` 命令来完成，该命令的格式如下：`CLUSTER MEET <ip> <port>`。

向一个节点 `node` 发送 `CLUSTER MEET` 命令，可以让 `node` 节点与 `ip` 和 `port` 所指定的节点进行握手（handshake），当握手成功时，`node` 节点就会将 `ip` 和 `port` 所指定的节点添加到 `node` 节点当前所在的集群中。

<!--more-->

### 1.1 启动节点

一个节点就是一个运行在集群模式下的 Redis 服务器， Redis 服务器在启动时会根据 `cluster-enabled` 配置选项的是否为 `yes` 来决定是否开启服务器的集群模式。

节点（运行在集群模式下的 Redis 服务器）会继续使用所有在单机模式中使用的服务器组件。除此之外， 节点会继续使用 `redisServer` 结构来保存服务器的状态， 使用 `redisClient` 结构来保存客户端的状态， 至于那些只有在集群模式下才会用到的数据， 节点将它们保存到了 `cluster.h/clusterNode` 结构， `cluster.h/clusterLink` 结构， 以及 `cluster.h/clusterState` 结构里面。

### 1.2 集群数据结构

`clusterNode` 结构保存了一个节点的当前状态，比如节点的创建时间，节点的名字，节点当前的配置纪元，节点的 IP 和地址，等等。

每个节点都会使用一个 `clusterNode` 结构来记录自己的状态，并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的 `clusterNode` 结构，以此来记录其他节点的状态：

```
typedef struct clusterNode {
    mstime_t ctime; /* Node object creation time. */
    char name[CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */
    int flags;      /* CLUSTER_NODE_... */
    uint64_t configEpoch; /* Last configEpoch observed for this node */
    unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
    int numslots;   /* Number of slots handled by this node */
    int numslaves;  /* Number of slave nodes, if this is a master */
    struct clusterNode **slaves; /* pointers to slave nodes */
    struct clusterNode *slaveof; /* pointer to the master node. Note that it
                                    may be NULL even if the node is a slave
                                    if we don't have the master node in our
                                    tables. */
    mstime_t ping_sent;      /* Unix time we sent latest ping */
    mstime_t pong_received;  /* Unix time we received the pong */
    mstime_t fail_time;      /* Unix time when FAIL flag was set */
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    mstime_t orphaned_time;     /* Starting time of orphaned master condition */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[NET_IP_STR_LEN];  /* Latest known IP address of this node */
    int port;                   /* Latest known port of this node */
    clusterLink *link;          /* TCP/IP link with this node */
    list *fail_reports;         /* List of nodes signaling this as failing */
} clusterNode;
```

`clusterNode` 结构的 `link` 属性是一个 `clusterLink` 结构，该结构保存了连接节点所需的有关信息，比如套接字描述符，输入缓冲区和输出缓冲区：

```
/* clusterLink encapsulates everything needed to talk with a remote node. */
typedef struct clusterLink {
    mstime_t ctime;             /* Link creation time */
    int fd;                     /* TCP socket file descriptor */
    sds sndbuf;                 /* Packet send buffer */
    sds rcvbuf;                 /* Packet reception buffer */
    struct clusterNode *node;   /* Node related to this link if any, or NULL */
} clusterLink;
```

`redisClient` 结构和 `clusterLink` 结构都有自己的套接字描述符和输入、输出缓冲区，这两个结构的区别在于，`redisClient` 结构中的套接字和缓冲区是用于连接客户端的，而 `clusterLink` 结构中的套接字和缓冲区则是用于连接节点的。

最后，每个节点都保存着一个 `clusterState` 结构，这个结构记录了在当前节点的视角下，集群目前所处的状态——比如集群是在线还是下线，集群包含多少个节点，集群当前的配置纪元，诸如此类：

```
typedef struct clusterState {
    clusterNode *myself;  /* This node */
    uint64_t currentEpoch;
    int state;            /* CLUSTER_OK, CLUSTER_FAIL, ... */
    int size;             /* Num of master nodes with at least one slot */
    dict *nodes;          /* Hash table of name -> clusterNode structures */
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    zskiplist *slots_to_keys;
    /* The following fields are used to take the slave state on elections. */
    mstime_t failover_auth_time; /* Time of previous or next election. */
    int failover_auth_count;    /* Number of votes received so far. */
    int failover_auth_sent;     /* True if we already asked for votes. */
    int failover_auth_rank;     /* This slave rank for current auth request. */
    uint64_t failover_auth_epoch; /* Epoch of the current election. */
    int cant_failover_reason;   /* Why a slave is currently not able to
                                   failover. See the CANT_FAILOVER_* macros. */
    /* Manual failover state in common. */
    mstime_t mf_end;            /* Manual failover time limit (ms unixtime).
                                   It is zero if there is no MF in progress. */
    /* Manual failover state of master. */
    clusterNode *mf_slave;      /* Slave performing the manual failover. */
    /* Manual failover state of slave. */
    long long mf_master_offset; /* Master offset the slave needs to start MF
                                   or zero if stil not received. */
    int mf_can_start;           /* If non-zero signal that the manual failover
                                   can start requesting masters vote. */
    /* The followign fields are used by masters to take state on elections. */
    uint64_t lastVoteEpoch;     /* Epoch of the last vote granted. */
    int todo_before_sleep; /* Things to do in clusterBeforeSleep(). */
    long long stats_bus_messages_sent;  /* Num of msg sent via cluster bus. */
    long long stats_bus_messages_received; /* Num of msg rcvd via cluster bus.*/
} clusterState;
```

### 1.3 CLUSTER MEET 命令的实现

通过向节点 A 发送 CLUSTER MEET 命令，客户端可以让接收命令的节点 A 将另一个节点 B 添加到节点 A 当前所在的集群里面：`CLUSTER MEET <ip> <port>`。

收到命令的节点 A 将与节点 B 进行握手（handshake），以此来确认彼此的存在，并为将来的进一步通信打好基础：

1. 节点 A 会为节点 B 创建一个 `clusterNode` 结构，并将该结构添加到自己的 `clusterState.nodes` 字典里面。
2. 之后，节点 A 将根据 `CLUSTER MEET` 命令给定的 IP 地址和端口号， 向节点 B 发送一条 `MEET` 消息（message）。
3. 如果一切顺利，节点 B 将接收到节点 A 发送的 `MEET` 消息，节点 B 会为节点 A 创建一个 `clusterNode` 结构，并将该结构添加到自己的 `clusterState.nodes` 字典里面。
4. 之后，节点 B 将向节点 A 返回一条 `PONG` 消息。
5. 如果一切顺利，节点 A 将接收到节点 B 返回的 `PONG` 消息，通过这条 `PONG` 消息节点 A 可以知道节点 B 已经成功地接收到了自己发送的 `MEET` 消息。
6. 之后，节点 A 将向节点 B 返回一条 `PING` 消息。
7. 如果一切顺利，节点 B 将接收到节点 A 返回的 `PING` 消息，通过这条 `PING` 消息节点 B 可以知道节点 A 已经成功地接收到了自己返回的 `PONG` 消息，握手完成。

之后，节点 A 会将节点 B 的信息通过 Gossip 协议传播给集群中的其他节点，让其他节点也与节点 B 进行握手，最终，经过一段时间之后，节点 B 会被集群中的所有节点认识。

## 2. 槽指派

Redis 集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为 16384 个槽（slot），数据库中的每个键都属于这 16384 个槽的其中一个，集群中的每个节点可以处理 0 个或最多 16384 个槽。 

当数据库中的 16384 个槽都有节点在处理时，集群处于上线状态（ok）；相反地，如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态（fail）。

通过向节点发送 `CLUSTER ADDSLOTS` 命令，我们可以将一个或多个槽指派（assign）给节点负责：

````
CLUSTER ADDSLOTS <slot> [slot …]
````

### 2.1 记录节点的槽指派信息

`clusterNode` 结构的 `slots` 属性和 `numslot` 属性记录了节点负责处理哪些槽：

```c
struct clusterNode {

	// ...
	
	unsigned char slots[16384/8];
	
	int numslots;
	
	// ...
	
};
```

`slots` 属性是一个二进制位数组（bit array），这个数组的长度为 16384/8=2048 个字节，共包含 16384 个二进制位。

Redis 以 0 为起始索引，16383 为终止索引，对 slots 数组中的 16384 个二进制位进行编号，并根据索引 `i` 上的二进制位的值来判断节点是否负责处理槽 `i`：

- 如果 `slots` 数组在索引 `i` 上的二进制位的值为 1，那么表示节点负责处理槽 `i`；
- 如果 `slots` 数组在索引 `i` 上的二进制位的值为 0，那么表示节点不负责处理槽 `i`；

因为取出和设置 `slots` 数组中的任意一个二进制位的值的复杂度仅为 $O(1)$，所以对于一个给定节点的 `slots` 数组来说，程序检查节点是否负责处理某个槽，又或者将某个槽指派给节点负责，这两个动作的复杂度都是 $O(1)$。

至于 `numslots` 属性则记录节点负责处理的槽的数量，也即是 `slots` 数组中值为 1 的二进制位的数量。

### 2.2 传播节点的槽指派信息

一个节点除了会将自己负责处理的槽记录在 `clusterNode` 结构的 `slots` 属性和 `numslots` 属性之外，它还会将自己的 `slots` 数组通过消息发送给集群中的其他节点，以此来告知其他节点自己目前负责处理哪些槽。

当节点 A 通过消息从节点 B 那里接收到节点 B 的 `slots` 数组时，节点 A 会在自己的 `clusterState.nodes` 字典中查找节点 B 对应的 `clusterNode` 结构，并对结构中的 `slots` 数组进行保存或者更新。

因为集群中的每个节点都会将自己的 `slots` 数组通过消息发送给集群中的其他节点，并且每个接收到 `slots` 数组的节点都会将数组保存到相应节点的 `clusterNode` 结构里面，因此，集群中的每个节点都会知道数据库中的 16384 个槽分别被指派给了集群中的哪些节点。

### 2.3 记录集群所有槽的指派信息

`clusterState` 结构中的 `slots` 数组记录了集群中所有 16384 个槽的指派信息：

```
typedef struct clusterState {

	// ...

	clusterNode *slots[16384];

	// ...
	
} clusterState;
```

`slots` 数组包含了 16384 个项，每个数组项都是一个指向 `clusterNode` 结构的指针：

- 如果 `slots[i]` 指针指向 NULL，那么表示槽 `i` 尚未指派给任何节点。
- 如果 `slots[i]` 指针指向一个 `clusterNode` 结构，那么表示槽 `i` 已经指派给了 `clusterNode` 结构所代表的节点。

如果只将槽指派信息保存在各个节点的 `clusterNode.slots` 数组里，会出现一些无法高效地解决的问题，而 `clusterState.slots` 数组的存在解决了这些问题：

- 如果节点只使用 `clusterNode.slots` 数组来记录槽的指派信息，那么为了知道槽 `i` 是否已经被指派，或者槽 `i` 被指派给了哪个节点，程序需要遍历 `clusterState.nodes` 字典中的所有 `clusterNode` 结构，检查这些结构的 `slots` 数组，直到找到负责处理槽 `i` 的节点为止，这个过程的复杂度为 $O(N)$，其中 $N$ 为 `clusterState.nodes` 字典保存的 `clusterNode` 结构的数量。
- 而通过将所有槽的指派信息保存在 `clusterState.slots` 数组里面，程序要检查槽 `i` 是否已经被指派，又或者取得负责处理槽 `i` 的节点，只需要访问 `clusterState.slots[i]` 的值即可，这个操作的复杂度仅为 $O(1)$。

要说明的一点是，虽然 `clusterState.slots` 数组中记录了集群中所有槽的指派信息，但使用 `clusterNode` 结构的 `slots` 数组来记录单个节点的槽指派信息仍然是有必要的：

- 因为当程序需要将某个节点的槽指派信息通过消息发送给其他节点时，程序只需要将相应节点的 `clusterNode.slots` 数组整个发送出去就可以了。
- 另一方面，如果 Redis 不使用 `clusterNode.slots` 数组，而单独使用 `clusterState.slots` 数组的话，那么每次要将节点 A 的槽指派信息传播给其他节点时，程序必须先遍历整个 `clusterState.slots` 数组，记录节点 A 负责处理哪些槽，然后才能发送节点 A 的槽指派信息，这比直接发送 `clusterNode.slots` 数组要麻烦和低效得多。

`clusterState.slots` 数组记录了集群中所有槽的指派信息，而 `clusterNode.slots` 数组只记录了 `clusterNode` 结构所代表的节点的槽指派信息，这是两个 `slots` 数组的关键区别所在。

### 2.4 CLUSTER ADDSLOTS命令的实现

`CLUSTER ADDSLOTS` 命令接受一个或多个槽作为参数，并将所有输入的槽指派给接收该命令的节点负责：

```
CLUSTER ADDSLOTS <slot> [slot ...]
```

`CLUSTER ADDSLOTS` 命令的实现可以用以下伪代码来表示：

```
def CLUSTER_ADDSLOTS(*all_input_slots):
	
	# 遍历所有输入槽，检查它们是否都是未指派槽
	for i in all_input_slots:

	# 如果有哪怕一个槽已经被指派给了某个节点
	# 那么向客户端返回错误，并终止命令执行
	if clusterState.slots[i] != NULL:
		reply_error()
		return

	# 如果所有输入槽都是未指派槽
	# 那么再次遍历所有输入槽，将这些槽指派给当前节点
	for i in all_input_slots:

		# 设置clusterState结构的slots数组
		# 将slots[i]的指针指向代表当前节点的clusterNode结构
		clusterState.slots[i] = clusterState.myself

		# 访问代表当前节点的clusterNode结构的slots数组
		# 将数组在索引i上的二进制位设置为 1
		setSlotBit(clusterState.myself.slots, i)
```

最后，在 `CLUSTER ADDSLOTS` 命令执行完毕之后，节点会通过发送消息告知集群中的其他节点，自己目前正在负责处理哪些槽。

## 3. 在集群中执行命令

在对数据库中的 16384 个槽都进行了指派之后，集群就会进入上线状态，这时客户端就可以向集群中的节点发送数据命令了。

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

- 如果键所在的槽正好指派了当前节点，那么节点直接执行这个命令。
- 如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个 `MOVED` 错误，指引客户端转向（redirect）至正确的节点，并再次发送之前想要执行的命令。

### 3.1 计算键属于哪个槽

节点使用以下算法来计算给定键 `key` 属于哪个槽：

```
def slot_number(key):
	return CRC16(key) & 16383
```

其中 `CRC16(key)` 语句用于计算键 `key` 的 `CRC-16` 校验和，而 `& 16383` 语句则用于计算出一个介于0 至 16383 之间的整数作为键 `key` 的槽号。

使用 `CLUSTER KEYSLOT <key>` 命令可以查看一个给定键属于哪个槽，以下是该命令的伪代码实现：

```
def CLUSTER_KEYSLOT(key)
	
	# 计算槽号
	slot = slot_number(key)
	
	# 将槽号返回给客户端
	reply_client(slot)
```

### 3.2 判断槽是否由当前节点负责处理

当节点计算出键所属的槽 `i` 之后，节点就会检查自己在 `clusterState.slots` 数组中的项 `i`，判断键所在的槽是否由自己负责：

1. 如果 `clusterState.slots[i]` 等于 `clusterState.myself`，那么说明槽 `i` 由当前节点负责，节点可以执行客户端发送的命令。
2. 如果 `clusterState.slots[i]` 不等于 `clusterState.myself`，那么说明槽 `i` 并非由当前节点负责，节点会根据 `clusterState.slots[i]` 指向的 `clusterNode` 结构所记录的节点 IP 和端口，向客户端返回 `MOVED` 错误，指引客户端转向至正在处理槽 `i` 的节点。

### 3.3 MOVED 错误

当节点发现键所在的槽并非由自己负责处理时，节点会向客户端返回一个 `MOVED` 错误，指引客户端转向至正在负责槽的节点，`MOVED` 错误的格式为：

```
MOVED <slot> <ip>:<port>
```

其中 `slot` 为键所在的槽，而 `ip` 和 `port` 则是负责处理槽 `slot` 的节点的 IP 地址和端口号。

当客户端接收到节点返回的 `MOVED` 错误时，客户端会根据 `MOVED` 错误中提供的 IP 地址和端口号，转向至负责处理槽 `slot` 的节点，并向该节点重新发送之前想要执行的命令。

一个集群客户端通常会与集群中的多个节点创建套接字连接，而所谓的节点转向实际上就是换一个套接字来发送命令。

如果客户端尚未与想要转向的节点创建套接字连接，那么客户端会现根据 `MOVED` 错误提供的 IP 地址和端口号来连接节点，然后再进行转向。

集群模式的 `redis-cli` 客户端在接收到 `MOVED` 错误时，并不会打印出 `MOVED` 错误，而是根据 `MOVED` 错误自动进行节点跳转，并打印出转向信息，所以我们是看不见节点返回的 `MOVED` 错误的。

但是，如果我们使用单机（stand alone）模式的 `redis-cli` 客户端，再次向节点发送相同的命令，那么 `MOVED` 错误就会被客户端打印出来。这是因为单机模式的 `redis-cli` 客户端不清楚 `MOVED` 错误的作用，所以它只会直接将 `MOVED` 错误直接打印出来，而不会进行自动转向。

### 3.4 节点数据库的实现

节点和单机服务器在数据库方面的一个区别是，节点只能使用 0 号数据库，而单机 Redis 服务器则没有这一限制。

另外，除了将键值对保存在数据库里面之外，节点还会用 `clusterState` 结构中的 `slots_to_keys` 跳跃表来保存键和槽之间的关系：

```c
typdef struct clusterState {
	
	// ...
  
 	zskiplist *slots_to_keys;
  
	// ...
  
} clusterState;
```

`slots_to_keys` 跳跃表每个节点的分值（score）都是一个槽号，而每个节点的成员（member）都是一个数据库键：

- 每当节点往数据库中添加一个新的键值对时，节点就会将这个键以及键的槽号关联到 `slots_to_keys` 跳跃表。
- 当节点删除数据库中的某个键值对时，节点就会在 `slots_to_keys` 跳跃表解除被删除键与槽号的关联。

通过在 `slots_to_keys` 跳跃表中记录各个数据库键所属的槽，节点可以很方便地对属于某个或某些槽的所有数据库键进行批量操作。

## 4. 重新分片

Redis 集群的重新分片操作可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点被移动到目标节点。 

重新分片操作可以在线（online）进行，在重新分片过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。

Redis 集群的重新分片操作由 Redis 的集群管理软件 redis-trib 负责执行，Redis 提供了进行重新分片所需的所有命令，而 redis-trib 则通过向源节点和目标节点发送命令来进行重新分片操作。

redis-trib 对集群的单个槽 slot 进行重新分片的步骤如下：

1. redis-trib 对目标节点发送 `CLUSTER SETSLOT <slot> IMPORTING <source_id>` 命令，让目标节点准备好从源节点导入（import）属于槽 `slot` 的键值对。
2. `redis-trib` 对源节点发送 `CLUSTER SETSLOT <slot> MIGRATING <target_id>` 命令，让源节点准备好将属于槽 `slot` 的键值对迁移（migrate）至目标节点。
3. `redis-trib` 向源节点发送 `CLUSTER GETKEYSINSLOT <slot> <count>` 命令，获得最多 `count` 个属于槽 `slot` 的键值对的键名。
4. 对于步骤 3 获得的每个键名，redis-trib 都向源节点发送一个 `MIGRATE <target_ip> <target_port> <key_name> 0 <timeout>` 命令，将被选中的键原子地从源节点迁移至目标节点。
5. 重复执行步骤 3 和步骤 4，直到源节点保存的所有属于槽 `slot` 的键值对都被迁移至目标节点为止。
6. redis-trib 向集群中的任意一个节点发送 `CLUSTER SETSLOT <slot> NODE <target_id>` 命令，将槽 `slot` 指派给目标节点，这一指派信息会通过消息发送至整个集群，最终集群中的所有节点都会知道槽 `slot` 已经被指派给了目标节点。

如果重新分片涉及多个槽，那么 redis-trib 将对每个给定的槽分别执行上面给出的步骤。

## 5. ASK 错误

在进行重新分片期间，源节点向目标节点迁移一个槽的过程中，可能会出现这样一种情况：属于被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。

当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

- 源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令
- 相反地，如果源节点没能在自己的数据库里面找到指定的键，那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个 `ASK` 错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

### 5.1 CLUSTER SETSLOT IMPORTING 命令的实现

`clusterState` 结构的 `importing_slots_from` 数组记录了当前节点正在从其他节点导入的槽：

```
typedef struct clusterState {
	
	// ...
	
	clusterNode *importing_slots_from[16384]
	
	// ...
	
} clusterState;
```

如果 `importing_slots_from[i]` 的值不为 NULL，而是指向一个 `clusterNode` 结构，那么表示当前节点正在从 `clusterNode` 所代表的节点导入槽 `i`。

在对集群进行重新分片的时候，向目标节点发送命令：

```

```

 可以将目标节点 `clusterState.importing_slots_from[i]` 的值设置为 `source_id` 所代表节点的 `clusterNode` 结构。

### 5.2 CLUSTER SETSLOT MIGRATING 命令的实现

`clusterState` 结构的 `migrating_slots_to` 数组记录了当前节点正在迁移至其他节点的槽：

```
typedef struct clusterState {
	
	// ...
	
	clusterNode *migrating_slots_to[16384]
	
	// ...
	
} clusterState;
```

如果 `migrating_slots_to[i]` 的值不为 NULL，而是指向一个 `clusterNode` 结构，那么表示当前节点正在将槽 `i` 迁移至 `clusterNode` 所代表的节点。

在对集群进行重新分片的时候，向源节点发送命令：

```
CLUSTER SETSLOT <i> MIGRATING <target_id>
```

 可以将源节点 `clusterState.migrating_slots_to[i]` 的值设置为 `target_id` 所代表节点的 `clusterNode` 结构。

### 5.3 ASK 错误

如果节点收到一个关于键 `key` 的命令请求，并且键 `key` 所属的槽 `i` 正好就指派给了这个节点，那么节点会尝试在自己的数据库里查找键 `key`，如果找到了的话，节点就直接执行客户端发送的命令。

与此相反，如果节点没有在自己的数据库里找到键 `key`，那么节点会检查自己的 `clusterState.migrating_slots_to[i]`，看键 `key` 所属的槽 `i` 是否正在进行迁移，如果槽 `i` 的确在进行迁移的话，那么节点会向客户端发送一个 `ASK` 错误，引导客户端到正在导入槽 `i` 的节点去超找键 `key`。

接到 `ASK` 错误的客户端会根据错误提供的 IP 和端口号，转向至正在导入槽的目标节点，然后首先向目标节点发送一个 `ASKING` 命令，之后再重新发送原本想要执行的命令。

### 5.4 ASKING 命令

`ASKING` 命令唯一要做的就是打开发送该命令的客户端的 `REDIS_ASKING` 标识，以下是该命令的伪代码实现：

```
def ASKING():

	# 打开标识
	client.flags != REDIS_ASKING
	
	# 向客户端发送 OK 回复
	reply("OK)
```

在一般情况下，如果客户端向节点发送一个关于槽 `i` 的命令，而槽 `i` 又没有指派给这个节点的话，那么节点将向客户端返回一个 `MOVED` 错误；但是，如果节点的 `clusterState.importing_slots_from[i]` 显示节点正在导入槽 `i`， 并且发送命令的客户端带有 `REDIS_ASKING` 标识，那么节点将破例执行这个关于槽 `i` 的命令一次。

当客户端接收到 `ASK` 错误并转向至正在导入槽的节点时，客户端会先向节点发送一个 `ASKING` 命令，然后才重新发送想要执行的命令，这是因为如果客户端不发送 `ASKING` 命令，而直接发送想要执行的命令的话，那么客户端发送的命令将被节点拒绝执行，并返回 `MOVED` 错误。

另外要注意的是，客户端的 `REDIS_ASKING` 标识是一个一次性标识，当节点执行了一个带有 `REDIS_ASKING` 标识的客户端发送的命令之后，客户端的 `REDIS_ASKING` 标识就会被移除。

### 5.5 ASK 错误和 MOVED 错误的区别

`ASK` 错误和 `MOVED` 错误都会导致客户端转向，它们的区别在于：

- `MOVED` 错误代表槽的负责权已经从一个节点转移到了另一个节点：客户端收到关于槽 `i` 的 `MOVED` 错误之后，客户端每次收到关于槽 `i` 的命令请求时，都可以直接将命令请求发送至 `MOVED` 错误所指向的节点，因为该节点就是目前负责处理槽 `i` 的节点。
- 与此相反，`ASK` 错误只是两个节点在迁移槽的过程中使用的一种临时措施：在客户端收到关于槽 `i` 的 `ASK` 错误之后，客户端只会在接下来的一次命令请求中将关于槽 `i` 的命令请求发送至 `ASK` 错误所指向的节点，但这种转向不会对客户端今后发送关于槽 `i` 的命令请求产生任何影响，客户端仍然会将关于槽 `i` 的命令请求发送至目前负责处理槽 `i` 的节点，除非 `ASK` 错误再次出现。

## 6. 复制与故障迁移

Redis 集群中的节点分为主节点（master）和从节点（slave），其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。

### 6.1 设置从节点

向一个节点发送命令：

```
CLUSTER REPLICATE <node_id>
```

可以让接收命令的节点成为 `node_id` 所指定节点的从节点，并开始对主节点进行复制：

- 接收到该命令的节点首先会在自己的  `clusterState.nodes` 字典里找到 `node_id` 所对应节点的 `clusterNode` 结构，并将自己的 `clusterState.myself.slaveof` 指针指向这个结构，以此来记录这个节点正在复制的主节点：

  ```c
  struct clusterNode {
  	// ...
  	
  	// 如果这是一个从节点，那么指向主节点
  	struct clusterNode *slaveof;
  	
  	// ...
  	
  }
  ```

- 然后节点会修改自己在 `clusterState.myself.flags` 中的属性，关闭原本的 `REDIS_NODE_MASTER` 标识，打开 `REDIS_NODE_SLAVE` 标识，表示这个节点已经由原来的主节点变成了从节点。

- 最后，节点会调用复制代码，并根据 `clusterState.myself.slaveof` 指向的 `clusterNode` 结构所保存的 IP 地址和端口号，对主节点进行复制。因为节点的复制功能和单机 Redis 服务器的复制功能使用了相同的代码，所以让从节点复制主节点相当于向从节点发送命令 `SLAVEOF <master_ip> <master_port>`。

一个节点成为从节点，并开始复制某个主节点这一信息会通过消息发送给集群中的其他节点，最终集群中的所有节点都会知道某个从节点正在复制某个主节点。

集群中的所有节点都会在代表主节点的 `clusterNode` 结构的 `slaves` 属性和 `numslaves` 属性中记录正在复制这个主节点的从节点名单：

```c
struct clusterNode {
  
	// ...
	
	// 正在复制这个主节点的从节点数量
	int numslaves;

	// 一个数组
	// 每个数组项指向一个正在复制这个主节点的从节点的 clusterNode 结构
	struct clusterNode **slaves;
	
	// ...
	
}
```

### 6.2 故障检测

集群中的每个节点都会定期地向集群中的其他节点发送 `PING` 消息，以此来检测对方是否在线，如果接收 `PING` 消息的节点没有在规定的时间内，向发送 `PING` 消息的节点返回 `PONG` 消息，那么发送 `PING` 消息的节点就会将接收 `PING` 消息的节点标记为疑似下线（probable fail，PFAIL）。

集群中的各个节点会通过互相发送消息的方式来交换集群中各个节点的状态信息。

当一个主节点 A 通过消息得知主节点 B 认为主节点 C 进入了疑似下线状态时，主节点 A 会在自己的 `clusterState.nodes` 字典中找到主节点 C 所对应的 `clusterNode` 结构，并将主节点 B 的下线报告（failure report）添加到 `clusterNode` 结构的 `fail_reports` 链表里面：

```c
struct clusterNode {
	
	// ...
  
	// 一个链表，记录了所有其他节点对该节点的下线报告
	list *fail_reports;

  	// ...
  
}
```

每个下线报告由一个 `clusterNodeFailReport` 结构表示：

```c
struct clusterNodeFailReport {

	// 报告目标节点已经下线的节点
	struct clusterNode *node;

	// 最后一次从 node 节点收到下线报告的时间
	// 程序使用这个时间戳来检查下线报告是否过期
	// (与当前时间差太久的下线报告会被删除)
	mstime_t time;
} typedef clusterNodeFailReport;
```

如果在一个集群里面，半数以上负责处理槽的主节点都将某个主节点 x 报告为疑似下线，那么这个主节点 x 将被标记为已下线（FAIL），将主节点 x 标记为已下线的节点会向集群广播一条关于主节点 x 的 `FAIL` 消息，所有收到这条 `FAIL` 消息的节点都会立即将主节点 x 标记为已下线。

### 6.3 故障转移

当一个从节点发现自己正在复制的主节点进入了已下线状态时，从节点将开始对下线主节点进行故障转移，以下是故障转移的执行步骤：

1. 复制下线主节点的所有从节点里面，会有一个从节点被选中。
2. 被选中的从节点会执行 `SLAVEOF no one` 命令，成为新的主节点。
3. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己。 
4. 新的主节点向集群广播一条 `PONG` 消息，这条 `PONG` 消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽。
5. 新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成。

### 6.4 选举新的主节点

新的主节点是通过选举产生的，以下是集群选举新的主节点的方法：

1. 集群的配置纪元是一个自增计数器，它的初始值为 0。
2. 当集群里的某个节点开始一次故障转移操作时，集群配置纪元的值会被增一。
3. 对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票。
4. 当从节点发现自己正在复制的主节点进入已下线状态时，从节点会想集群广播一条 `CLUSTER_TYPE_FAILOVER_AUTH_REQUEST` 消息，要求所有接收到这条消息、并且具有投票权的主节点向这个从节点投票。
5. 如果一个主节点具有投票权（它正在负责处理槽），并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条 `CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK` 消息，表示这个主节点支持从节点成为新的主节点。
6. 每个参与选举的从节点都会接收 `CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK` 消息，并根据自己收到了多少条这种消息来同济自己获得了多少主节点的支持。
7. 如果集群里有 N 个具有投票权的主节点，那么当一个从节点收集到大于等于 N/2+1 张支持票时，这个从节点就会当选为新的主节点。
8. 因为在每一个配置纪元里面，每个具有投票权的主节点只能投一次票，所以如果有 N 个主节点进行投票，那么具有大于等于 N/2+1 张支持票的从节点只会有一个，这确保了新的主节点只会有一个。
9. 如果在一个配置纪元里面没有从节点能收集到足够多的支持票，那么集群进入一个新的配置纪元，并再次进行选举，知道选出新的主节点为止。

## 7. 消息

集群中的各个节点通过发送和接收消息（message）来进行通信，我们称发送消息的节点为发送者（sender），接收消息的节点为接收者（receiver）。节点发送的消息主要有以下五种：

- `MEET` 消息：当发送者接到客户端发送的 `CLUSTER MEET` 命令时，发送者会向接收者发送 `MEET` 消息， 请求接收者加入到发送者当前所处的集群里面。
- `PING` 消息：集群里的每个节点默认每隔一秒钟就会从已知节点列表中随机选出五个节点，然后对这五个节点中最长时间没有发送过 `PING` 消息的节点发送 `PING` 消息，以此来检测被选中的节点是否在线。除此以外，如果节点 A 最后一次收到节点 B 发送的 `PONG` 消息的时间，距离当前时间已经超过了节点 A 的 `cluster-node-timeout` 选项设置时长的一半，那么节点 A 也会向节点 B 发送 `PING` 消息，这可以防止节点 A 因为长时间没有随机选中节点 B 作为 `PING` 消息的发送对象而导致对节点 B 的信息更新滞后。
- `PONG` 消息：当接收者收到发送者发来的 `MEET` 消息或者 `PING` 消息时，为了向发送者确认这条 `MEET` 消息或者 `PING` 消息已到达，接收者会向发送者返回一条 `PONG` 消息。另外，一个节点也可以通过向集群广播 `PONG` 消息来让集群中的其他节点立即刷新关于这个节点的认识，例如当一次故障转移操作成功之后，新的主节点会向集群广播一条 `PONG` 消息，以此来让集群中的其他节点立即知道这个节点已经变成了主节点，并且接管了已下线节点负责的槽。
- `FAIL` 消息：当一个主节点 A 判断另一个主节点 B 已经进入 `FAIL` 状态时，节点 A 会向集群广播一条关于节点 B 的 `FAIL` 消息，所有收到这条消息的节点都会立即将节点 B 标记为已下线。
- `PUBLISH` 消息：当节点收到一个 `PUBLISH` 命令时，节点会执行这个命令，并向集群广播一条 `PUBLISH` 消息，所有接收到这条 `PUBLISH` 消息的节点都会执行相同的 `PUBLISH` 命令。

一条消息由消息头（header）和消息正文（data）组成。

###  7.1 消息头

节点发送的所有消息都由一个消息头包裹，消息头除了包含消息正文之外，还记录了消息发送者自身的一些信息，因为这些信息也会被消息接受者用到，所以严格来讲，我们可以认为消息头本身也是消息的一部分。

每个消息头都由一个`cluster.h/clusterMsg`结构表示：

```c
typedef struct {
  // 消息的长度，包括消息头和消息正文
  uint32_t totlen;
  
  // 消息的类型
  uint16_t type;
  
  // 消息正文包含的节点信息数量
  // 只在发送MEET、PING、PONG这三种Gossip协议的消息时使用
  uint16_t count;
  
  // 发送者所处的配置纪元
  uint64_t currentEpoch;
  
  // 如果发送者是一个master，那么这里记录的是发送者的配置纪元
  // 如果发送者是一个slave，那么这里记录的是发送者正在复制的master的配置纪元
  uint64_t configEpoch;
  
  // 发送者的名字(ID)
  char sender[REDIS_CLUSTER_NAMELEN];
  
  // 发送者目前的槽指派信息
  unsigned char myslots[REDIS_CLUSTER_SLOTS/8];
  
  // 如果发送者是一个slave，那么这里记录的是它正在复制的master的名字
  // 如果发送者是一个master，那么这里记录的是REDIS_NODE_NULL_NAME
  char slaveof[REDIS_CLUSTER_NAMELEN];
  
  // 发送者的端口号
  uint16_t port;
  
  // 发送者的标识值
  uint16_t flags;
  
  // 发送者所处集群的状态
  unsigned char state;
  
  // 消息的正文
  union clusterMsgData data;
} cllusterMsg;
```

`clusterMsg.data` 属性指向联合体 `cluster.h/clusterMsgData`，这个联合体就是消息的正文：

```c
union clusterMsgData {
  struct {
    // 每条 MEET、PING、PONG 消息都包含两个 clusterMsgDataGossip 结构
    clusterMsgDataGossip[1];
  } ping;
  
  // FAIL 消息的正文
  struct {
    clusterMsgDataFail about;
  } fail;
  
  // PUBLISH 消息的正文
  struct {
    clusterMsgDataPublish msg;
  } publish;
};
```

`clusterMsg` 结构的 `currentEpoch`、`sender`、`myslots` 等属性记录了发送者的节点信息，接收者可以根据这些信息，在自己的 `clusterState.nodes `字典中找到发送者对应的 `clusterNode` 结构进行更新。

### 7.2 MEET、PING、PONG 消息的实现

Redis 集群中的各个节点通过 Gossip 协议来交换节点的状态信息，其中 Gossip 协议由 `MEET`、`PING`、`PONG` 三种消息实现，这三种消息的正文都是由两个 `cluster.h/clusterMsgDataGossip` 结构组成：

```c
typedef struct {
  // 节点的名字
  char nodename[REDIS_CLUSTER_NAMELEN];
  
  // 最后一次向该节点发送 PING 消息的时间戳
  uint32_t ping_sent;
  
  // 最后一次从该节点接收到 PONG 消息的时间戳
  uint32_t pong_received;
  
  // 节点的IP
  char ip[16];
  
  // 节点的端口
  uint16_t port;
  
  // 节点的标识符
  uint16_t flags;
} clusterMsgDataGossip;
```

因为 `MEET`、`PING`、`PONG` 三种消息都是用相同的消息正文，所以节点通过消息头的 `type` 属性来判断一条消息是 `MEET`消息、`PING` 消息还是 `PONG` 消息。

每次发送 `MEET`、`PING`、`PONG ` 消息时，发送者从自己的已知节点中随机选出两个节点（可以是主节点或从节点），并将这两个被选中的节点的信息分别保存到两个 `cluster.h/clusterMsgDataGossip` 结构里面。

`clusterMsgDataGossip` 结构记录了被选中的节点的名字、发送者与被选中节点最后一次发送和接收 `PING` 和 `PONG` 消息的时间戳，被选中节点的 IP 地址和端口号，以及被选中节点的标识值。

当接收者收到 `MEET`、`PING`、`PONG` 消息时，接收者会访问消息正文中的两个 `clusterMsgDataGossip` 结构，并根据自己是否认识 `clusterMsgDataGossip` 记录的被选中节点来选择进行哪种操作：

- 如果被选中节点不存在于接收者的已知节点列表，那么说明接收者是第一次接触到被选中节点，接收者将根据结构中记录的 IP 地址和端口号等信息，与被选中节点进行握手。
- 如果被选中节点存在于接收者的已知节点列表，那么说明接收者之前已经与被选中节点进行过接触，接收者将根据 `clusterMsgDataGossip` 结构记录的信息，对被选中的节点所对应的 `clusterNode` 结构进行更新。

### 7.3 FAIL 消息的实现

当集群里的主节点 A 将主节点 B 标记为已下线时（FAIL）时，主节点 A 将向集群广播一条关于主节点 B 的 `FAIL` 消息，所有接收到这条 `FAIL` 消息的节点都会将主节点 B 标记为已下线。

`FAIL` 消息的正文由 `cluster.h/clusterMsgDataFail` 结构表示，这个结构只包含一个 `nodeName` 属性，该属性记录了已下线节点的名字：

```c
typedef struct {
  char nodename[REDIS_CLUSTER_NAMELEN];
} clusterMsgDataFail;
```



### 7.4 PUBLISH 消息的实现

当客户端向集群中的某个节点发送命令：

```
PUBLISH <channel> <message>
```

的时候，接收到 `PUBLISH` 命令的节点不仅会向 `channel` 频道发送消息 `message`，它还会向集群广播一条 `PUBLISH` 消息，所有接收到这条 `PUBLISH` 消息的节点都会向 `channnel` 频道发送 `message` 消息。

`PUBLISH` 消息的正文由 `cluster.h/clusterMsgDataPublish` 结构表示：

```c
typedef struct {
	
	uint32_t channel_len;

	uint32_t message_len;

	// 定义为 8 字节只是为了对齐其他消息结构
	// 实际的长度由保存的内容决定
	unsigned char bulk_data[8];
  
} clusterMsgDataPublish;
```

`clusterMsgDataPublish` 结构的 `bulk_data` 属性是一个字节数组，这个字节数组保存了客户端通过 `PUBLISH` 命令发送给节点的 `channel` 参数和 `message` 参数，而结构的 `channel_len` 和 `message_len` 则分别保存了 `channel` 参数的长度和 `message` 参数的长度：

- 其中 `bulk_data` 的 0 字节至 channel_len - 1 字节保存的是 `channel` 参数。
- 而 `bulk_data` 的 channel_len 字节至 channel_len + message_len - 1 字节保存的是 `message` 参数。