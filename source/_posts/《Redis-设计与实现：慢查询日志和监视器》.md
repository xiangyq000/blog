---
title: 《Redis 设计与实现：慢查询日志和监视器》
date: 2017-10-22 11:58:24
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

## 1. 慢查询日志

Redis 的慢查询日志功能用于记录执行时间超过给定时长的命令请求，用户可以通过这个功能产生的日志来监视和优化查询速度。

服务器配置有两个和慢查询日志相关的选项：

- `slowlog-log-slower-than` 选项指定执行时间超过多少微秒（`1` 秒等于 `1,000,000` 微秒）的命令请求会被记录到日志上。

- `slowlog-max-len` 选项指定服务器最多保存多少条慢查询日志。

  服务器使用先进先出的方式保存多条慢查询日志：当服务器储存的慢查询日志数量等于 `slowlog-max-len` 选项的值时，服务器在添加一条新的慢查询日志之前，会先将最旧的一条慢查询日志删除。

<!--more-->

### 1.1 慢查询记录的保存

服务器状态中包含了几个和慢查询日志功能有关的属性：

```c
struct redisServer {

    // ...

    // 下一条慢查询日志的 ID
    long long slowlog_entry_id;

    // 保存了所有慢查询日志的链表
    list *slowlog;

    // 服务器配置 slowlog-log-slower-than 选项的值
    long long slowlog_log_slower_than;

    // 服务器配置 slowlog-max-len 选项的值
    unsigned long slowlog_max_len;

    // ...

};
```

`slowlog_entry_id` 属性的初始值为 `0` ，每当创建一条新的慢查询日志时，这个属性的值就会用作新日志的 `id` 值，之后程序会对这个属性的值增一。

`slowlog` 链表保存了服务器中的所有慢查询日志， 链表中的每个节点都保存了一个 `slowlogEntry` 结构， 每个 `slowlogEntry` 结构代表一条慢查询日志：

```c
typedef struct slowlogEntry {

    // 唯一标识符
    long long id;

    // 命令执行时的时间，格式为 UNIX 时间戳
    time_t time;

    // 执行命令消耗的时间，以微秒为单位
    long long duration;

    // 命令与命令参数
    robj **argv;

    // 命令与命令参数的数量
    int argc;

} slowlogEntry;
```

### 1.2 慢查询日志的阅览和删除

可以使用 `SLOWLOG GET` 命令查看服务器所保存的慢查询日志，伪代码如下：

```
def SLOWLOG_GET(number=None):

    # 用户没有给定 number 参数
    # 那么打印服务器包含的全部慢查询日志
    if number is None:
        number = SLOWLOG_LEN()

    # 遍历服务器中的慢查询日志
    for log in redisServer.slowlog:

        if number <= 0:
            # 打印的日志数量已经足够，跳出循环
            break
        else:
            # 继续打印，将计数器的值减一
            number -= 1

        # 打印日志
        printLog(log)

```

查看日志数量的 `SLOWLOG LEN` 命令可以用以下伪代码来定义：

```
def SLOWLOG_LEN():

    # slowlog 链表的长度就是慢查询日志的条目数量
    return len(redisServer.slowlog)

```

另外，用于清除所有慢查询日志的 `SLOWLOG RESET` 命令可以用以下伪代码来定义：

```
def SLOWLOG_RESET():

    # 遍历服务器中的所有慢查询日志
    for log in redisServer.slowlog:

        # 删除日志
        deleteLog(log)
```

### 1.3 添加新日志

在每次执行命令的之前和之后，程序都会记录微秒格式的当前 UNIX 时间戳，这两个时间戳之间的差就是服务器执行命令所耗费的时长，服务器会将这个时长作为参数之一传给 `slowlogPushEntryIfNeeded` 函数，而 `slowlogPushEntryIfNeeded` 函数则负责检查是否需要为这次执行的命令创建慢查询日志，以下伪代码展示了这一过程：

```
# 记录执行命令前的时间
before = unixtime_now_in_us()

# 执行命令
execute_command(argv, argc, client)

# 记录执行命令后的时间
after = unixtime_now_in_us()

# 检查是否需要创建新的慢查询日志
slowlogPushEntryIfNeeded(argv, argc, before-after)

```

`slowlogPushEntryIfNeeded` 函数的作用有两个：

1. 检查命令的执行时长是否超过 `slowlog-log-slower-than` 选项所设置的时间，如果是的话，就为命令创建一个新的日志，并将新日志添加到 `slowlog` 链表的表头。
2. 检查慢查询日志的长度是否超过 `slowlog-max-len` 选项所设置的长度，如果是的话，那么将多出来的日志从 `slowlog` 链表中删除掉。

以下是 `slowlogPushEntryIfNeeded` 函数的实现代码：

```
void slowlogPushEntryIfNeeded(robj **argv, int argc, long long duration) {

    // 慢查询功能未开启，直接返回
    if (server.slowlog_log_slower_than < 0) return;

    // 如果执行时间超过服务器设置的上限，那么将命令添加到慢查询日志
    if (duration >= server.slowlog_log_slower_than)
        // 新日志添加到链表表头
        listAddNodeHead(server.slowlog,slowlogCreateEntry(argv,argc,duration));

    // 如果日志数量过多，那么进行删除
    while (listLength(server.slowlog) > server.slowlog_max_len)
        listDelNode(server.slowlog,listLast(server.slowlog));
}

```

`slowlogCreateEntry` 函数根据传入的参数，创建一个新的慢查询日志，并将 `redisServer.slowlog_entry_id` 的值增一。

## 2. 监视器

通过执行 `MONITOR` 命令，客户端可以将自己变为一个监视器，实时地接收并打印出服务器当前处理的命令请求的相关信息：

```
redis> MONITOR
OK
1378822099.421623 [0 127.0.0.1:56604] "PING"
1378822105.089572 [0 127.0.0.1:56604] "SET" "msg" "hello world"
1378822109.036925 [0 127.0.0.1:56604] "SET" "number" "123"
1378822140.649496 [0 127.0.0.1:56604] "SADD" "fruits" "Apple" "Banana" "Cherry"
1378822154.117160 [0 127.0.0.1:56604] "EXPIRE" "msg" "10086"
1378822257.329412 [0 127.0.0.1:56604] "KEYS" "*"
1378822258.690131 [0 127.0.0.1:56604] "DBSIZE"

```

每当一个客户端向服务器发送一条命令请求时，服务器除了会处理这条命令请求之外，还会将关于这条命令请求的信息发送给所有监视器。

### 2.1 成为监视器

发送 `MONITOR` 命令可以让一个普通客户端变为一个监视器，该命令的实现原理可以用以下伪代码来实现：

```
def MONITOR():

    # 打开客户端的监视器标志
    client.flags |= REDIS_MONITOR

    # 将客户端添加到服务器状态的 monitors 链表的末尾
    server.monitors.append(client)

    # 向客户端返回 OK
    send_reply("OK")
```

### 2.2 向监视器发送命令信息

服务器在每次处理命令请求之前，都会调用 `replicationFeedMonitors` 函数，由这个函数将被处理命令请求的相关信息发送给各个监视器。

以下是 `replicationFeedMonitors` 函数的伪代码定义，函数首先根据传入的参数创建信息，然后将信息发送给所有监视器：

```
def replicationFeedMonitors(client, monitors, dbid, argv, argc):

    # 根据执行命令的客户端、当前数据库的号码、命令参数、命令参数个数等参数
    # 创建要发送给各个监视器的信息
    msg = create_message(client, dbid, argv, argc)

    # 遍历所有监视器
    for monitor in monitors:

        # 将信息发送给监视器
        send_message(monitor, msg)
```