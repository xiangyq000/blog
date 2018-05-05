---
title: 《Redis 设计与实现：Sentinel》
date: 2017-10-20 23:52:46
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

Sentinel 是 Redis 的高可用性解决方案：由一个或多个 Sentinel 实例组成的 Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。

<!--more-->

## 1. 启动并初始化 Sentinel

启动一个 Sentinel 可以使用命令：

```
$ redis-sentinel /path/to/your/sentinel.conf
```

或者命令：

```
$ redis-server /path/to/your/sentinel.conf --sentinel
```

这两个命令的效果完全相同。

当一个 Sentinel 启动时，它需要执行以下步骤：

1. 初始化服务器。
2. 将普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。
3. 初始化 Sentinel 状态。
4. 根据给定的配置文件，初始化 Sentinel 的监视主服务器列表。
5. 创建连向主服务器的网络连接。

### 1.1 初始化服务器

Sentinel 模式下 Redis 服务器主要功能的使用情况

| 功能                                     | 使用情况                                     |
| -------------------------------------- | ---------------------------------------- |
| 数据库和键值对方面的命令，比如 `SET`、`DEL`、`FLUSHDB`。 | 不使用。                                     |
| 事务命令，比如 `MULTI` 和 `WATCH`。             | 不使用。                                     |
| 脚本命令，比如 `EVAL`。                        | 不使用。                                     |
| RDB 持久化命令，比如 `SAVE` 和 `BGSAVE`。        | 不使用。                                     |
| AOF 持久化命令，比如 `BGREWRITEAOF`。           | 不使用。                                     |
| 复制命令，比如 `SLAVEOF`。                     | Sentinel 内部可以使用，但客户端不可以使用。               |
| 发布与订阅命令，比如 `PUBLISH` 和 `SUBSCRIBE`。    | `SUBSCRIBE`、`PSUBSCRIBE`、`UNSUBSCRIBE`、`PUNSUBSCRIBE` 四个命令在 Sentinel 内部和客户端都可以使用，但 `PUBLISH` 命令只能在 Sentinel 内部使用。 |
| 文件事件处理器（负责发送命令请求、处理命令回复）。              | Sentinel 内部使用，但关联的文件事件处理器和普通 Redis 服务器不同。 |
| 时间事件处理器（负责执行 `serverCron` 函数）。         | Sentinel 内部使用，时间事件的处理器仍然是 `serverCron` 函数，`serverCron` 函数会调用 `sentinel.c/sentinelTimer` 函数，后者包含了 Sentinel 要执行的所有操作。 |

### 1.2 使用 Sentinel 专用代码

启动 Sentinel 的第二个步骤就是将一部分普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。

比如说， 普通 Redis 服务器使用 `redis.h/REDIS_SERVERPORT` 常量的值作为服务器端口：

```
#define REDIS_SERVERPORT 6379

```

而 Sentinel 则使用 `sentinel.c/REDIS_SENTINEL_PORT` 常量的值作为服务器端口：

```
#define REDIS_SENTINEL_PORT 26379

```

除此之外， 普通 Redis 服务器使用 `redis.c/redisCommandTable` 作为服务器的命令表，而 Sentinel 则使用 `sentinel.c/sentinelcmds` 作为服务器的命令表，并且其中的 `INFO` 命令会使用 Sentinel 模式下的专用实现 `sentinel.c/sentinelInfoCommand` 函数， 而不是普通 Redis 服务器使用的实现 `redis.c/infoCommand` 函数。

`sentinelcmds` 命令表也解释了为什么在 Sentinel 模式下， Redis 服务器不能执行诸如 `SET`、 `DBSIZE`、 `EVAL` 等等这些命 ——因为服务器根本没有在命令表中载入这些命令：`PING`、 SENTINEL 、`INFO`、`SUBSCRIBE`、`UNSUBSCRIBE`、`PSUBSCRIBE` 和 `PUNSUBSCRIBE` 这七个命令就是客户端可以对 Sentinel 执行的全部命令了。

### 1.3 初始化 Sentinel 状态

在应用了 Sentinel 的专用代码之后，接下来，服务器会初始化一个 `sentinel.c/sentinelState` 结构（后面简称 “Sentinel 状态”），这个结构保存了服务器中所有和 Sentinel 功能有关的状态 （服务器的一般状态仍然由 `redis.h/redisServer` 结构保存）：

```
struct sentinelState {

    // 当前纪元，用于实现故障转移
    uint64_t current_epoch;

    // 保存了所有被这个 sentinel 监视的主服务器
    // 字典的键是主服务器的名字
    // 字典的值则是一个指向 sentinelRedisInstance 结构的指针
    dict *masters;

    // 是否进入了 TILT 模式？
    int tilt;

    // 目前正在执行的脚本的数量
    int running_scripts;

    // 进入 TILT 模式的时间
    mstime_t tilt_start_time;

    // 最后一次执行时间处理器的时间
    mstime_t previous_time;

    // 一个 FIFO 队列，包含了所有需要执行的用户脚本
    list *scripts_queue;

} sentinel;

```

### 1.4 初始化 Sentinel 状态的 `masters` 属性

Sentinel 状态中的 `masters` 字典记录了所有被 Sentinel 监视的主服务器的相关信息，其中：

- 字典的键是被监视主服务器的名字。
- 而字典的值则是被监视主服务器对应的 `sentinel.c/sentinelRedisInstance` 结构。

每个 `sentinelRedisInstance` 结构（后面简称 “实例结构”）代表一个被 Sentinel 监视的 Redis 服务器实例（instance），这个实例可以是主服务器、从服务器、或者另外一个 Sentinel 。

实例结构包含的属性非常多，以下代码展示了实例结构在表示主服务器时使用的其中一部分属性：

```
typedef struct sentinelRedisInstance {

    // 标识值，记录了实例的类型，以及该实例的当前状态
    int flags;

    // 实例的名字
    // 主服务器的名字由用户在配置文件中设置
    // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
    // 格式为 ip:port ，例如 "127.0.0.1:26379"
    char *name;

    // 实例的运行 ID
    char *runid;

    // 配置纪元，用于实现故障转移
    uint64_t config_epoch;

    // 实例的地址
    sentinelAddr *addr;

    // SENTINEL down-after-milliseconds 选项设定的值
    // 实例无响应多少毫秒之后才会被判断为主观下线（subjectively down）
    mstime_t down_after_period;
    
    /* Master specific. */
    dict *sentinels;    /* Other sentinels monitoring the same master. */
    dict *slaves;       /* Slaves for this master instance. */
    
    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数
    // 判断这个实例为客观下线（objectively down）所需的支持投票数量
    int quorum;

    // SENTINEL parallel-syncs <master-name> <number> 选项的值
    // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
    int parallel_syncs;

    // SENTINEL failover-timeout <master-name> <ms> 选项的值
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;

    // ...

} sentinelRedisInstance;
```

`sentinelRedisInstance.addr` 属性是一个指向 `sentinel.c/sentinelAddr` 结构的指针，这个结构保存着实例的 IP 地址和端口号：

```
typedef struct sentinelAddr {

    char *ip;

    int port;

} sentinelAddr;

```

对 Sentinel 状态的初始化将引发对 `masters` 字典的初始化，而 `masters` 字典的初始化是根据被载入的 Sentinel 配置文件来进行的。

配置文件的一个例子：

```
#####################
# master1 configure #
#####################

sentinel monitor master1 127.0.0.1 6379 2

sentinel down-after-milliseconds master1 30000

sentinel parallel-syncs master1 1

sentinel failover-timeout master1 900000

#####################
# master2 configure #
#####################

sentinel monitor master2 127.0.0.1 12345 5

sentinel down-after-milliseconds master2 50000

sentinel parallel-syncs master2 5

sentinel failover-timeout master2 450000
```

### 1.5 创建连向主服务器的网络连接

初始化 Sentinel 的最后一步是创建连向被监视主服务器的网络连接：Sentinel 将成为主服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关的信息。

对于每个被 Sentinel 监视的主服务器来说，Sentinel 会创建两个连向主服务器的异步网络连接：

- 一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复。
- 另一个是订阅连接，这个连接专门用于订阅主服务器的 `__sentinel__:hello` 频道。

在 Redis 目前的发布与订阅功能中， 被发送的信息都不会保存在 Redis 服务器里面，如果在信息发送时，想要接收信息的客户端不在线或者断线，那么这个客户端就会丢失这条信息。因此，为了不丢失 `__sentinel__:hello` 频道的任何信息，Sentinel 必须专门用一个订阅连接来接收该频道的信息。

而另一方面，除了订阅频道之外，Sentinel 还又必须向主服务器发送命令，以此来与主服务器进行通讯，所以 Sentinel 还必须向主服务器创建命令连接。

并且因为 Sentinel 需要与多个实例创建多个网络连接，所以 Sentinel 使用的是异步连接。

## 2. 获取主服务器信息

Sentinel 默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送 `INFO` 命令，并通过分析 `INFO` 命令的回复来获取主服务器的当前信息。

举个例子，假设主服务器 mater 有三个从服务器 slave0、slave1 和 slave2，并且一个 Sentinel 正在连接主服务器，那么 Sentinel 将持续地向主服务器发送 `INFO` 命令，并获得类似于以下内容的回复：

```
# Server
...
run_id: // 略
...

# Replication
role:master
connected_slaves:3
slave0:ip=127.0.0.1,port=11111,state=online,offset=43,lag=0
slave1:ip=127.0.0.1,port=22222,state=online,offset=43,lag=0
slave2:ip=127.0.0.1,port=33333,state=online,offset=43,lag=0
...

# Other sections
...
```

通过分析主服务器返回的 `INFO` 命令回复，Sentinel 可以获取以下两方面的信息：

- 主服务器本身的信息，包括服务器运行 ID `run_id`，服务器角色 `role` 等；
- 主服务器属下的所有从服务器信息。包括从服务器的 IP 地址、端口号等，根据这些信息，Sentinel 无须用户提供从服务器的地址信息，就可以自动发现从服务器。

根据 `run_id` 域和 `role` 域记录的信息，Sentinel 将对主服务器的实例结构进行更新。至于主服务返回的从服务器信息，则会被用于更新主服务器实例结构的 `slaves` 字典，这个字典记录了主服务器属下的从服务器名单：

- 字典的键是由 Sentinel 自动设置的从服务器名字，格式为 `ip:port`。
- 字典的值是从服务器对应的实例结构。

Sentinel 在分析 `INFO` 命令中包含的从服务器信息时，会检查从服务器对应的实例结构是否已经存在于 `slaves` 字典：

- 如果已存在，进行更新；
- 否则创建一个新的实例结构。

主服务器实例结构和从服务器实例结构之间的区别：

- 主服务器实例结构的 `flags` 属性的值为 `SRI_MASTER`，而从服务器实例结构的 `flags` 属性的值为  `SRI SLAVE`。
- 主服务器实例结构的 `name` 属性的值是用户使用 Sentinel 配置文件设置的，而从服务器实例结构的 `name` 属性的值则是 Sentinel 根据从服务器的 IP 地址和端口号自动设置的。

## 3. 获取从服务器信息

当 Sentinel 发现主服务器有新的从服务器时，Sentinel 除了会创建相应的从服务器实例结构外，还会创建从服务器的命令连接和订阅连接。

同样，Sentinel也会以以每 10 秒一次的频率，通过命令连接向从服务器发送 `INFO` 命令，并获得类似于以下内容的回复：

```
# Server
...
run_id: // 略
...
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
slave_repl_offset:11887
slave_priority:100

# Other sections
...
```

根据 `INFO` 命令的回复，Sentinel 会提取出以下信息：

- 从服务器的运行 ID `run_id`。
- 从服务器的角色 `role`。
- 主服务器的 IP 地址 `master_host`，以及主服务器的端口号 `master_port。`
- 主从服务器的连接状态 `master_link_status。`
- 从服务器的优先级 `slave_priority。`
- 从服务器的复制偏移量 `slave_repl_offset`。

根据这些信息，Sentinel 会对从服务器的实例结构进行更新。

## 4. 向主服务器和从服务器发送消息

在默认情况下，Sentinel 会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送以下格式的命令：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

这条命令向服务器的 `__sentinel__:hello` 频道发送了一条消息，信息的内容由多个参数组成：

- 其中以 `s_` 开头的参数记录的是 Sentinel 本身的信息。
- 而以 `m_` 开头的参数记录的则是主服务器的信息。如果 Sentinel 正在监视的是主服务器，那么这些参数记录的就是主服务器的信息；如果 Sentinel 正在监视的是从服务器，那么这些参数记录的就是从服务器正在复制的主服务器的信息。

## 5. 接收来自主服务器和从服务器的频道信息

当 Sentinel 与一个主服务器或者从服务器建立起订阅连接之后，Sentinel 就会通过订阅连接，向服务器发送以下命令：

```
SUBSCRIBE __sentinel__:hello
```

Sentinel 对 `__sentinel__:hello` 频道的订阅会一直持续到 Sentinel 与服务器的连接断开为止。这也就是说，对于每个与 Sentinel 连接的服务器，Sentinel 既通过命令连接向服务器的 `__sentinel__:hello` 频道发送消息，又通过订阅连接从服务器的 `__sentinel__:hello` 频道接收消息。

对于监视同一个服务器的多个 Sentinel 来说，一个 Sentinel 发送的消息会被其他 Sentinel 接收到，这些消息会被用于更新其他 Sentinel 对发送消息 Sentinel 的认知，也会被用于更新其他 Sentinel 对被监视服务器的认知。

当一个 Sentinel 从 `__sentinel__:hello` 频道收到一条消息时，Sentinel 会对这条消息进行分析，提取出消息中的 Sentinel IP 地址、Sentinel 端口号、Sentinel 运行 ID等八个参数，并进行以下检查：

- 如果信息中记录的 Sentinel 运行 ID 和接收信息的 Sentinel 运行 ID 相同，那么说明这条消息是 Sentinel 自己发送的，Sentinel 将丢弃这条消息，不做进一步处理。
- 相反地，如果信息中记录的 Sentinel 运行 ID 和接收信息的 Sentinel 运行 ID 不相同，那么说明这条消息是监视同一个服务器的其他 Sentinel 发来的，接收消息的 Sentinel 将根据信息中的各个参数，对相应主服务器的实例结构进行更新。

（如果某个 Sentinel 发现自己的配置纪元低于接收到的配置纪元，则会用新的配置更新自己的配置？）

### 5.1 更新 sentinels 字典

Sentinel 为主服务器创建的实例结构中的 `sentinels` 字典保存了除 Sentinel 本身之外，所有同样监视这个主服务器的其他 Sentinel 的资料：

- `sentinels` 字典的键是其中一个 Sentinel 的名字，格式为 `ip:port`。
- `sentinels` 字典的值则是键所对应 Sentinel 的实例结构。

当一个 Sentinel 接收到其他 Sentinel 发来的消息时（称发送消息的 Sentinel 为源 Sentinel，接收消息的 Sentinel 为目标 Sentinel），目标 Sentinel 会从信息中分析并提取出以下两方面的参数：

- 与 Sentinel 有关的参数：源 Sentinel 的 IP 地址、端口号、运行 ID 和配置纪元。
- 与主服务器有关的参数：源 Sentinel 正在监视的主服务器的名字、IP 地址、端口号和配置纪元。

根据信息中提取出的主服务器参数，目标 Sentinel 会在自己的 Sentinel 状态的 `masters` 字典中查找相应的主服务器实例结构，然后根据提取出的 Sentinel 参数，检查主服务器实例结构的 `sentinels` 字典，源 Sentinel 的实例结构是否存在：

- 如果源 Sentinel 的实例结构已经存在，那么对源 Sentinel 的实力结构进行更新。
- 如果源 Sentinel 的实例结构不存在，那么说明源 Sentinel 是刚刚开始监视主服务器的新 Sentinel，目标 Sentinel 会为源 Sentinel 创建一个新的实例结构，并将这个结构添加到 `sentinels` 字典里面。

### 5.3 创建连向其他 Sentinel 的命令连接

当 Sentinel 通过频道信息发现一个新的 Sentinel 时，它不仅会为新 Sentinel 在 `sentinels` 字典中创建相应的实例结构，还会创建一个连向新 Sentinel 的命令连接，而新 Sentinel 也同样会创建连向这个 Sentinel 的命令连接，最终监视同一主服务器的多个 Sentinel 将形成相互连接的网络：Sentinel A 有连向 Sentinel B 的命令连接，而 Sentinel B 也有连向 Sentinel A 的命令连接。

使用命令连接相连的各个 Sentinel 可以通过向其他 Sentinel 发送命令请求来进行信息交换。

Sentinel 之间只会创建命令连接，不会创建订阅连接。

##  6. 检测主观下线状态

在默认情况下，Sentinel 会以每秒一次的频率向所有与它创建了命令连接的实例（包括主服务器、从服务器、其他 Sentinel 在内）发送 `PING` 命令，并通过实例返回的 `PING` 命令回复来判断实例是否在线。

实例对 `PING` 命令的回复可以分为以下两种情况：

- 有效回复：实例返回 `+PONG`、`-LOADING`、`-MASTERDOWN` 三种回复的其中一种。
- 无效回复：实例返回除 `+PONG`、`-LOADING`、`-MASTERDOWN` 三种回复之外的其他回复，或者在指定时限内没有返回任何回复。

Sentinel 配置文件中的 `down-after-milliseconds` 选项指定了 Sentinel 判断实例进入主观下线所需的时间长度：如果一个实例在 `down-after-milliseconds` 毫秒内，连续向 Sentinel 返回无效回复，那么 Sentinel 会修改这个实例所对应的实例结构，在结构的 `flags` 属性中打开 `SRI_S_DOWN` 标识，以此来表示这个实例已经进入主观下线状态。

用户设置的 `down-after-milliseconds` 选项的值，不仅会被 Sentinel 用来判断主服务器的主观下线状态，还会被用于判断主服务器属下的所有从服务器，以及所有同样监视这个主服务器的其他 Sentinel 的主观下线状态。需要注意的是，对于监视同一个主服务器的多个 Sentinel 来说，这些 Sentinel 所设置的 `down-after-milliseconds` 选项的值也可能不同，因此，当一个 Sentinel 将主服务器判断为主观下线时，其他 Sentinel 可能仍然会认为主服务器处于在线状态。

## 7. 检查客观下线状态

当 Sentinel 将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他 Sentinel 进行询问，看它们是否也认为主服务器已经进入了下线状态（可以是主观下线或者客观下线）。当 Sentinel 从其他 Sentinel 那里接收到足够数量的已下线判断之后，Sentinel 就会将主服务器判定为客观下线，并对从服务器执行故障转移操作。

### 7.1 发送 SENTINEL is-master-down-by-addr 命令

Sentinel 使用

```
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

命令询问其他 Sentinel 是否同意主服务器已下线。

| **参数**          |                  **意义**                  |
| :-------------- | :--------------------------------------: |
| `ip`            |      被 Sentinel 判断为主观下线的主服务器的 IP 地址      |
| `port`          |       被 Sentinel 判断为主观下线的主服务器的端口号        |
| `current_epoch` |     Sentinel 当前的配置纪元，用于选举领头 Sentinel     |
| `runid`         | 可以是 \* 符号或者 Sentinel 的运行 ID：\* 符号代表命令仅仅用于检测主服务器的客观下线状态，而 Sentinel 的运行 ID 则用于选举领头 Sentinel |

### 7.2 接收 SENTINEL is-master-down-by-addr 命令

当一个 Sentinl（目标 Sentinel）接收到另一个 Sentinel（源 Sentinel）发来的 `SENTINEL is-master-down-by-addr` 命令时，目标 Sentinel 会分析并取出命令请求中包含的各个参数，并根据其中的主服务器 IP 和端口号，检查主服务器是否已下线，然后向源 Sentinel 返回一条包含三个参数的 Multi Bulk 回复作为回复：

1. `<down_state>`
2. `leader_runid`
3. `leader_epoch`

| **参数**         |                  **意义**                  |
| :------------- | :--------------------------------------: |
| `donw_state`   | 目标 Sentinel 对主服务器的检查结果，1 代表主服务器已下线，0 代表主服务器未下线 |
| `leader_runid` | 可以是 \* 符号或者目标 Sentinel 的局部领头 Sentinel 的运行 ID：\* 符号代表命令仅仅用于检测主服务器的客观下线状态，而局部领头 Sentinel 的运行 ID 则用于选举领头 Sentinel |
| `leader_epoch` | 目标 Sentinel 的局部领头 Sentinel 的配置纪元，用于选举领头 Sentinel。仅在 `leader_runid` 的值不为 \* 时有效，如果 `leader_epoch` 的值为 \*，那么 `leader_epoch` 总为 0 |

### 7.2 接收 SENTINEL is-master-down-by-addr 命令的回复

根据其他 Sentinel 发回的 `SENTINEL is-master-down-by-addr` 命令回复，Sentinel 将统计其他 Sentinel 同意主服务器已下线的数量，当这一数量达到配置指定的判断客观下线所需的数量时，Sentinel 会将主服务器实例结构 `flags` 属性的 `SRI_O_DOWN` 标识打开，表示主服务器已经进入客观下线状态。

当认为主服务器已经进入下线状态的 Sentinel 的数量，超过 Sentinel 配置中设置的 `quorum` 参数的值，那么该 Sentinel 就会认为主服务器已经进入了客观下线状态。根据配置，对于监视同一个主服务器的多个 Sentinel 来说，它们将主服务器判断为客观下线的条件可能也不同。

## 8. 选举领头 Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个 Sentinel 会进行协商，选举出一个领头 Sentinel，并由领头 Sentinel 对下线主服务器执行故障转移操作。

以下是 Redis 选举领头 Sentinel 的规则和方法：

1. 所有在线的监视同一个主服务器的多个 Sentinel 都有被选为领头 Sentinel 的资格。
2. 每次进行领头 Sentinel 选举之后，不论选举是否成功，所有 Sentinel 的配置纪元（configration epoch）的值都会自增一次。
3. 在一个配置纪元里面，所有 Sentinel 都有一次将某个 Sentinel 设置为局部领头 Sentinel 的机会，并且局部领头一旦设置，在这配置纪元里面就不能再更改了。
4. 每个发现主服务器进入客观下线的 Sentinel 都会要求其他 Sentinel 将自己设置为局部领头 Sentinel。
5. 当一个 Sentinel（源 Sentinel）向另一个 Sentinel（目标 Sentinel）发送 `SENTINEL is-master-down-by-addr` 命令，并且命令中的 `runid` 参数不是 \* 符号而是源 Sentinel 的运行 ID 时，这表示源 Sentinel 要求目标 Sentinel 将前者设置为后者的局部领头 Sentinel。
6. Sentinel 设置局部领头 Sentinel 的规则是先到先得：最先向目标 Sentinel 发送设置要求的源 Sentinel 将成为目标 Sentinel 的局部领头 Sentinel，而之后接收到的所有设置要求都会被目标 Sentinel拒绝。
7. 目标 Sentinel 在接收到 `SENTINEL is-master-down-by-addr` 命令之后，将向源 Sentinel 返回一条命令回复，回复中的 `leader_runid` 参数和 `leader_epoch` 参数分别记录了目标 Sentinel 的局部领头 Sentinel 的运行 ID 和配置纪元。
8. 源 Sentinel 在接收到目标 Sentinel 返回的命令回复之后，会检查回复中的 `leader_epoch` 参数的值和自己的配置纪元是否相同，如果相同的话，那么源 Sentinel 继续取出回复中的 `leader_runid` 参数，如果 `leader_runid` 参数的值和源 Sentinel 的运行 ID 一致，那么表示目标 Sentinel 将源 Sentinel 设置成了局部领头 Sentinel。
9. 如果有某个 Sentinel 被半数以上的 Sentinel 设置成了局部领头 Sentinel，那么这个 Sentinel 成为领头 Sentinel。
10. 因为领头 Sentinel 的产生需要半数以上的 Sentinel 支持，并且每个 Sentinel 在每个配置纪元里面只能设置一次局部领头 Sentinel，所以在一个配置纪元里面，只会出现一个领头 Sentinel。
11. 如果在给定时限内，没有一个 Sentinel 被选举为领头 Sentinel，那么各个 Sentinel 将在一段时间之后再次进行选举，直到选举出领头 Sentinel 为止。

## 9. 故障转移

在选举产生出领头 Sentinel 之后，领头 Sentinel 将对已下线的主服务器执行故障转移操作，该操作包含以下三个步骤：

1. 在已下线主服务器属下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器。
2. 让已下线主服务器属下的所有从服务器改为复制新的主服务器。
3. 将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，他就会成为新的主服务器的从服务器。

### 9.1 选出新的主服务器

故障转移操作的第一步要做的就是在已下线主服务器属下的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送 `SLAVEOF no one` 命令，将这个从服务器转换为主服务器。

在发送 `SLAVEOF no one` 命令之后，领头 Sentinel 会以每秒一次的频率（平时是每十秒一次），向被升级的从服务器发送 `INFO` 命令，并观察命令回复中的角色（role）信息，当被升级服务器的 role 从原来的 slave 变为 master 时，领头 Sentinel 就知道被选中的从服务器已经顺利升级为主服务器了。

领头 Sentinel 将已下线的主服务器的所有从服务器保存到一个列表中，然后按下面进行过滤：

- 删除列表中所有处于下线或断线转发太的从服务器；
- 删除列表中所有最近 5s 内没有回复过领头 Sentinel 的 `INFO` 命令的从服务器；
- 删除所有与已下线的主服务器连接断开超过 `down-after-millisecondes * 10` 毫秒的从服务器；

之后，领头 Sentinel将根据从服务器的优先级排序（选出优先级最高的）；如果具有多个相同最高优先级的从服务器，那么再按照从服务器的复制偏移量排序（选出偏移量最大的）；如果还有相同的从服务器，那么按照 `runid` 进行排序，选出其中 `runid` 最小的。

### 9.2 修改从服务器的复制目标

当新的主服务器出现之后，领头 Sentinel 下一步要做的就是，让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可以通过向从服务器发送 `SLAVEOF` 命令来实现。

### 9.3 将旧的主服务器变为从服务器

故障转移操作最后要做的是，将已下线的主服务器设置为新的主服务器的从服务器。