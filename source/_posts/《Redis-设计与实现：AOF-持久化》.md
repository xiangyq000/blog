---
title: 《Redis 设计与实现：AOF 持久化》
date: 2017-10-14 16:57:41
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

AOF 持久化保存数据库状态的方法是将服务器执行的命令保存到文件中。被写入 AOF 文件的所有命令都是以 Redis 的命令请求协议格式保存的。

服务器在启动时，可以通过载入和执行 AOF 文件中保存的命令来还原服务器关闭之前的数据库状态，以下就是服务器载入 AOF 文件并还原数据库状态时打印的日志：

```
[8321] 05 Sep 11:58:50.448 # Server started, Redisversion 2.9.11
[8321] 05 Sep 11:58:50.449 * DB loaded from append only file: 0.000 seconds
[8321] 05 Sep 11:58:50.449 * The server is now ready to accept connections on port 6379 
```

<!--more-->

## 1. AOF 持久化的实现

### 1.1 命令追加
当 AOF 持久化功能处于打开状态时， 服务器在执行完一个写命令之后， 会以协议格式将被执行的写命令追加到服务器状态的 `aof_buf` 缓冲区的末尾：

```c
struct redisServer {

    // ...

    // AOF 缓冲区
    sds aof_buf;

    // ...
};
```

### 1.2 AOF 文件的写入与同步
Redis 的服务器进程就是一个事件循环（loop）， 这个循环中的文件事件负责接收客户端的命令请求， 以及向客户端发送命令回复， 而时间事件则负责执行像 `serverCron` 函数这样需要定时运行的函数。

因为服务器在处理文件事件时可能会执行写命令， 使得一些内容被追加到 `aof_buf` 缓冲区里面， 所以在服务器每次结束一个事件循环之前， 它都会调用 `flushAppendOnlyFile` 函数， 考虑是否需要将 `aof_buf` 缓冲区中的内容写入和保存到 AOF 文件里面， 这个过程可以用以下伪代码表示：

```
def eventLoop():

    while True:

        # 处理文件事件，接收命令请求以及发送命令回复
        # 处理命令请求时可能会有新内容被追加到 aof_buf 缓冲区中
        processFileEvents()

        # 处理时间事件
        processTimeEvents()

        # 考虑是否要将 aof_buf 中的内容写入和保存到 AOF 文件里面
        flushAppendOnlyFile()
```

`flushAppendOnlyFile` 函数的行为由服务器配置的 `appendfsync` 选项的值来决定， 各个不同值产生的行为如下表所示：

| `appendfsync` 选项的值 | `flushAppendOnlyFile` 函数的行为              |
| ------------------ | ---------------------------------------- |
| `always`           | 将 `aof_buf` 缓冲区中的所有内容写入并同步到 AOF 文件。      |
| `everysec`         | 将 `aof_buf` 缓冲区中的所有内容写入到 AOF 文件， 如果上次同步 AOF 文件的时间距离现在超过一秒钟， 那么再次对 AOF 文件进行同步， 并且这个同步操作是由一个线程专门负责执行的。 |
| `no`               | 将 `aof_buf` 缓冲区中的所有内容写入到 AOF 文件， 但并不对 AOF 文件进行同步， 何时同步由操作系统来决定。 |

如果用户没有主动为 `appendfsync` 选项设置值， 那么 `appendfsync` 选项的默认值为 `everysec`。

### 1.3 AOF 持久化的效率和安全性
服务器配置 `appendfsync` 选项的值直接决定 AOF 持久化功能的效率和安全性。

- 当 `appendfsync` 的值为 `always` 时， 服务器在每个事件循环都要将 `aof_buf` 缓冲区中的所有内容写入到 AOF 文件， 并且同步 AOF 文件， 所以 `always` 的效率是 `appendfsync` 选项三个值当中最慢的一个， 但从安全性来说， `always` 也是最安全的， 因为即使出现故障停机， AOF 持久化也只会丢失一个事件循环中所产生的命令数据。

- 当 `appendfsync` 的值为 `everysec` 时， 服务器在每个事件循环都要将 `aof_buf` 缓冲区中的所有内容写入到 AOF 文件， 并且每隔超过一秒就要在子线程中对 AOF 文件进行一次同步： 从效率上来讲， `everysec` 模式足够快， 并且就算出现故障停机， 数据库也只丢失一秒钟的命令数据。

- 当 `appendfsync` 的值为 `no` 时， 服务器在每个事件循环都要将 `aof_buf` 缓冲区中的所有内容写入到 AOF 文件， 至于何时对 AOF 文件进行同步， 则由操作系统控制。因为处于 `no` 模式下的 `flushAppendOnlyFile` 调用无须执行同步操作， 所以该模式下的 AOF 文件写入速度总是最快的， 不过因为这种模式会在系统缓存中积累一段时间的写入数据， 所以该模式的单次同步时长通常是三种模式中时间最长的： 从平摊操作的角度来看， `no` 模式和 `everysec` 模式的效率类似， 当出现故障停机时， 使用 `no` 模式的服务器将丢失上次同步 AOF 文件之后的所有写命令数据。

## 2. AOF 文件的载入与数据还原

因为 AOF 文件里面包含了重建数据库状态所需的所有写命令，所以服务器只要读入并重新执行一遍 AOF 文件里面保存的写命令，就可以还原服务器关闭之前的数据库状态。

Redis 读取 AOF 文件并还原数据库状态的详细步骤如下：

1. 创建一个不带网络连接的伪客户端（fake client）：因为 Redis 的命令只能在客户端上下文中执行，而载入 AOF 文件时所使用的命令直接来源于 AOF 文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行 AOF 文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样。
2. 从 AOF 文件中分析并读取出一条写命令。
3. 使用伪客户端执行被读出的写命令。
4. 一直执行步骤 2 和 步骤 3，直到 AOF 文件中的所有写命令都被处理完毕为止。



## 3. AOF重写

为了解决 AOF 文件体积膨胀的问题，Redis 提供了 AOF 重写功能：Redis 服务器可以创建一个新的 AOF 文件来替代现有的 AOF 文件，新旧两个文件所保存的数据库状态是相同的，但是新的 AOF 文件不会包含任何浪费空间的冗余命令，通常体积会较旧 AOF 文件小很多。

### 3.1 AOF 文件重写的实现

AOF 重写并不需要对原有 AOF 文件进行任何的读取、写入、分析等操作，这个功能是通过读取服务器当前的数据库状态来实现的。首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录该键值对的多个命令。整个重写过程可以用以下的伪代码表示：

```
def AOF_REWRITE(tmp_tile_name):

  f = create(tmp_tile_name)

  # 遍历所有数据库
  for db in redisServer.db:

    # 如果数据库为空，那么跳过这个数据库
    if db.is_empty(): continue

    # 写入 SELECT 命令，用于切换数据库
    f.write_command("SELECT " + db.number)

    # 遍历所有键
    for key in db:

      # 如果键带有过期时间，并且已经过期，那么跳过这个键
      if key.have_expire_time() and key.is_expired(): continue

      if key.type == String:

        # 用 SET key value 命令来保存字符串键

        value = get_value_from_string(key)

        f.write_command("SET " + key + value)

      elif key.type == List:

        # 用 RPUSH key item1 item2 ... itemN 命令来保存列表键

        item1, item2, ..., itemN = get_item_from_list(key)

        f.write_command("RPUSH " + key + item1 + item2 + ... + itemN)

      elif key.type == Set:

        # 用 SADD key member1 member2 ... memberN 命令来保存集合键

        member1, member2, ..., memberN = get_member_from_set(key)

        f.write_command("SADD " + key + member1 + member2 + ... + memberN)

      elif key.type == Hash:

        # 用 HMSET key field1 value1 field2 value2 ... fieldN valueN 命令来保存哈希键

        field1, value1, field2, value2, ..., fieldN, valueN =\
        get_field_and_value_from_hash(key)

        f.write_command("HMSET " + key + field1 + value1 + field2 + value2 +\
                        ... + fieldN + valueN)

      elif key.type == SortedSet:

        # 用 ZADD key score1 member1 score2 member2 ... scoreN memberN
        # 命令来保存有序集键

        score1, member1, score2, member2, ..., scoreN, memberN = \
        get_score_and_member_from_sorted_set(key)

        f.write_command("ZADD " + key + score1 + member1 + score2 + member2 +\
                        ... + scoreN + memberN)

      else:

        raise_type_error()

      # 如果键带有过期时间，那么用 EXPIREAT key time 命令来保存键的过期时间
      if key.have_expire_time():
        f.write_command("EXPIREAT " + key + key.expire_time_in_unix_timestamp())

    # 关闭文件
    f.close()
```

在实际中，为了避免在执行命令时造成客户端输入缓冲区溢出，重写程序在处理列表、哈希表、集合、有序集合这四种可能会带有多个元素的键时，会先检查键所包含的元素数量，如果元素的数量超过了`redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD` 常量的值，那么重写程序将使用多条命令来记录键的值，而不单单使用一条命令。

在目前版本中，`REDIS_AOF_REWRITE_ITEMS_PER_CMD` 常量的值为 64，这也就是说，如果一个集合键包含了超过 64 个元素，那么重写程序会用多条SADD命令来记录这个集合，并且每条命令设置的元素数量也为 64 个：

```
SADD <set-key> <elem1> <elem2> ... <elem64>
SADD <set-key> <elem65> <elem66> ... <elem128>
SADD <set-key> <elem129> <elem130> ... <elem192>
```

另一方面如果一个列表键包含了超过 64 个项，那么重写程序会用多条 `RPUSH` 命令来保存这个列表，并且每条命令设置的项数量也为 64 个：

```
SADD <list-key> <item1> <item2> ... <item64>
SADD <list-key> <item65> <item66> ... <item128>
SADD <list-key> <item129> <item130> ... <item192>
```

### 3.2 AOF 后台重写

Redis 决定将 AOF 重写程序放到（后台）子进程里执行， 这样做可以同时达到两个目的：

1. 子进程进行 AOF 重写期间，服务器进程（父进程）可以继续处理命令请求。
2. 子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。

子进程在进行 AOF 重写期间，服务器进程还需要继续处理命令请求，而新的命令可能会对现有的数据库状态进行修改，从而使得服务器当前的数据库状态和重写后的 AOF 文件所保存的数据库状态不一致。

为了解决这种数据不一致问题，Redis 服务器设置了一个 AOF 重写缓冲区，这个缓冲区在服务器创建子进程之后开始使用，当 Redis 服务器执行完一个写命令之后，它会同时将这个写命令发送给 AOF 缓冲区和 AOF 重写缓冲区。这也就是说，在子进程执行 AOF 重写期间，服务器进程需要执行以下三个工作：

1. 执行客户端发来的命令。
2. 将执行后的写命令追加到 AOF 缓冲区。
3. 将执行后的写命令追加到 AOF 重写缓冲区。

这样一来，可以保证:

- AOF 缓冲区的内容会定期被写入和同步到 AOF 文件，对现有 AOF 文件的处理工作会如常进行。
- 从创建子进程开始，服务器执行的所有写命令都会被记录到 AOF 重写缓冲区里面。

当子进程完成 AOF 重写工作之后，它会向父进程发送一个信号，父进程在接到该信号之后，会调用一个信号处理函数，并执行以下工作：

- 将 AOF 重写缓冲区中的所有内容写入到新 AOF 文件中，这时新 AOF 文件所保存的数据库状态将和服务器当前的数据库状态一致。
- 对新的 AOF 文件进行改名，原子地（atomic）覆盖现有的 AOF 文件，完成新旧两个 AOF 文件的替换。

信号处理函数执行完毕之后，父进程就可以继续像往常一样接受命令请求了。

在整个 AOF 后台重写过程中，只有信号处理函数执行时会对服务器进程（父进程）造成阻塞，其他时候，AOF 后台重写都不会阻塞父进程，这将 AOF 重写对服务器性能造成的影响降到了最低。