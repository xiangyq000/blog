---
title: 《Redis 设计与实现：数据库》
date: 2017-10-03 18:44:02
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

## 1. 服务器中的数据库

Redis 服务器将所有数据库都保存在服务器状态 `redis.h/redisServer` 结构的 `db` 数组中，`db` 数组的每个项都是一个 `redis.h/redisDb` 结构，每个 `redisDb` 结构代表一个数据库：

```c
struct redisServer{
  
	//...
  
	//一个数组，保存着服务器中的所有数据库
	redisDb *db;
  
	//服务器的数据库数量
	int dbnum;
};
```
在初始化服务器时，程序会根据服务器状态的 `dbnum` 属性来决定应该创建多少个数据库。

`dbnum` 属性的值由服务器配置的 `database` 选项决定，默认情况下，该选项的值为 16，所以 Redis 服务器默认会创建 16 个数据库。

<!--more-->

## 2. 切换数据库
每个 Redis 客户端都有自己的目标数据库，每当客户端执行数据库写命令或者数据库读命令的时候，目标数据库就会成为这些命令的操作对象。

默认情况下，Redis 客户端的目标数据库为 0 号数据库，但客户端可以通过执行 `SELECT` 命令来切换目标数据库。

在服务器内部，客户端状态 `redisClient` 结构的 `db` 属性记录了客户端当前的目标数据库，这个属性是一个指向 `redisDb` 结构的指针：

```c
typedef struct redisClient {

	// ...

	//记录客户端当前正在使用的数据库
	redisDb *db;
	
	// ...
  
} redisClient;
```
`redisClient.db` 指针指向 `redisServer.db` 数组的其中一个元素，而被指向的元素就是客户端的目标数据库。

通过修改 `redisClient.db` 指针，让它指向服务器中的不同数据库，从而实现切换目标数据库的功能——这就是 `SELECT` 命令的实现原理。

## 3. 数据库空间
Redis 是一个键值对（key-value pair）数据库服务器，服务器中的每个数据库都由一个 `redis.h/redisDb` 结构表示，其中，`redisDb` 结构的 `dict` 字典保存了数据库中的所有键值对，我们将这个字典称为键空间（key space）：

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```
键空间和用户所见的数据库是直接对应的：

- 键空间的键也就是数据库的键，每个键都是一个字符串对象。
- 键空间的值也就是数据库的值，每个值可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的任意一种 Redis 对象。

当使用 Redis 命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作，其中包括：

- 在读取一个键之后（读操作和写操作都要对键进行读取），服务器会根据键是否存在来更新服务器的键空间命中（hit）次数或键空间不命中（miss）次数，这两个值可以在 `INFO stats` 命令的 `keyspace_hits` 属性和 `keyspace_misses` 属性中查看。
- 在读取一个键之后，服务器会更新键的 LRU（最后一次使用）时间，这个值可以用于计算键的闲置时间，使用 `OBJECT idletime` 命令可以查看键 key 的闲置时间。
- 如果服务器在读取一个键时发现该键已经过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作。
- 如果有客户端使用 `WATCH` 命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏（dirty），从而让事务程序注意到这个键已经被修改过。
- 服务器每次修改一个键之后，都会对脏（dirty）键计数器的值增 1，这个计数器会触发服务器的持久化以及复制操作。
- 如果服务器开启了数据库通知功能，那么在对键进行修改之后，服务器将按配置发送相应的数据库通知。

## 4. 设置键的生存时间或过期时间
通过 `EXPIRE` 命令或者 `PEXPIRE` 命令，客户端可以以秒或者毫秒精度为数据库中的某个键设置生存时间（Time To Live，TTL），在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为 0 的键。

与 `EXPIRE` 命令和 `PEXPIRE` 命令类似，客户端可以通过 `EXPIREAT` 命令或 `PEXPIREAT` 命令，以秒或者毫秒精度给数据库中的某个键设置过期时间（expire time）。

`TTL` 命令和 `PTTL` 命令接受一个带有生存时间或者过期时间的键，返回这个键的剩余生存时间，也就是，返回距离这个键被服务器自动删除还有多长时间。

### 4.1 设置过期时间
Redis 有四个不同的命令可以用于设置键的生存时间（键可以存在多久）或过期时间（键什么时候会被删除）：

- `EXPIRE＜key＞＜ttl＞` 命令用于将键 key 的生存时间设置为 ttl 秒。
- `PEXPIRE＜key＞＜ttl＞` 命令用于将键 key 的生存时间设置为 ttl 毫秒。
- `EXPIREAT＜key＞＜timestamp＞` 命令用于将键 key 的过期时间设置为 timestamp 所指定的秒数时间戳。
- `PEXPIREAT＜key＞＜timestamp＞` 命令用于将键 key 的过期时间设置为 timestamp 所指定的毫秒数时间戳。

虽然有多种不同单位和不同形式的设置命令，但实际上 `EXPIRE`、`PEXPIRE`、`EXPIREAT` 三个命令都是使用 `PEXPIREAT` 命令来实现的：无论客户端执行的是以上四个命令中的哪一个，经过转换之后，最终的执行效果都和执行 `PEXPIREAT` 命令一样。

首先，`EXPIRE` 命令可以转换成 `PEXPIRE` 命令：

```
def EXPIRE(key,ttl_in_sec):

    #将TTL从秒转换成毫秒
    ttl_in_ms = sec_to_ms(ttl_in_sec)
    
    PEXPIRE(key, ttl_in_ms)
```

接着，`PEXPIRE` 命令又可以转换成 `PEXPIREAT` 命令：

```
def PEXPIRE(key,ttl_in_ms):

    #获取以毫秒计算的当前 UNIX 时间戳
    now_ms = get_current_unix_timestamp_in_ms()
    
    #当前时间加上 TTL，得出毫秒格式的键过期时间
    PEXPIREAT(key,now_ms+ttl_in_ms)
```

并且，`EXPIREAT` 命令也可以转换成 `PEXPIREAT` 命令：

```
def EXPIREAT(key,expire_time_in_sec):

    #将过期时间从秒转换为毫秒
    expire_time_in_ms = sec_to_ms(expire_time_in_sec)
    
    PEXPIREAT(key, expire_time_in_ms)
```

最终，`EXPIRE`、`PEXPIRE` 和 `EXPIREAT` 三个命令都会转换成 `PEXPIREAT` 命令来执行。

### 4.2 保存过期时间
redisDb 结构的 `expires` 字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典：

- 过期字典的键是一个指针，这个指针指向键空间中的某个键对象（也即是某个数据库键）。在实际中，键空间的键和过期字典的键都指向同一个键对象。
- 过期字典的值是一个 long long 类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的 UNIX 时间戳。

当客户端执行 `PEXPIREAT` 命令（或者其他三个会转换成 `PEXPIREAT` 命令的命令）为一个数据库键设置过期时间时，服务器会在数据库的过期字典中关联给定的数据库键和过期时间。

以下是 `PEXPIREAT` 命令的伪代码定义：

```
def PEXPIREAT(key, expire_time_in_ms):

    #如果给定的键不存在于键空间，那么不能设置过期时间
    if key not in redisDb.dict:
        return0
        
    #在过期字典中关联键和过期时间
    redisDb.expires[key] = expire_time_in_ms
    
    #过期时间设置成功
    return 1
```

### 4.3 移除过期时间
`PERSIST` 命令可以移除一个键的过期时间，它是 `PEXPIREAT` 命令的反操作： `PERSIST` 命令在过期字典中查找给定的键，并解除键和值（过期时间）在过期字典中的关联。

以下是 `PERSIST` 命令的伪代码定义：

```
def PERSIST(key):
	#如果键不存在，或者键没有设置过期时间，那么直接返回
	if key not in redisDb.expires:
	    return0
	#移除过期字典中给定键的键值对关联
	redisDb.expires.remove(key)
	#键的过期时间移除成功
	return 1
```

### 4.4 计算并返回剩余生存时间
`TTL` 命令以秒为单位返回键的剩余生存时间，而 `PTTL` 命令则以毫秒为单位返回键的剩余生存时间。`TTL` 和 `PTTL` 两个命令都是通过计算键的过期时间和当前时间之间的差来实现的，以下是这两个命令的伪代码实现：

```
def PTTL(key):

    #键不存在于数据库
    if key not in redisDb.dict:
        return-2
        
    #尝试取得键的过期时间
    #如果键没有设置过期时间，那么 expire_time_in_ms将为 None
    expire_time_in_ms = redisDb.expires.get(key)
    
    #键没有设置过期时间
    if expire_time_in_ms is None:
		return -1
		
    #获得当前时间
    now_ms = get_current_unix_timestamp_in_ms()
    
    #过期时间减去当前时间，得出的差就是键的剩余生存时间
    return(expire_time_in_ms - now_ms)
    
def TTL(key):

    #获取以毫秒为单位的剩余生存时间
    ttl_in_ms = PTTL(key)
    
    if ttl_in_ms ＜ 0:
        #处理返回值为-2和-1的情况
        return ttl_in_ms
    else:
		#将毫秒转换为秒
		return ms_to_sec(ttl_in_ms)
```

### 4.5 过期键的判定
通过过期字典，程序可以用以下步骤检查一个给定键是否过期：

1. 检查给定键是否存在于过期字典：如果存在，那么取得键的过期时间。
2. 检查当前 UNIX 时间戳是否大于键的过期时间：如果是的话，那么键已经过期；否则的话，键未过期。

可以用伪代码来描述这一过程：

```
def is_expired(key):
    
    #取得键的过期时间
    expire_time_in_ms = redisDb.expires.get(key)
    
    #键没有设置过期时间
    if expire_time_in_ms is None:
        return False
    
    #取得当前时间的UNIX时间戳
    now_ms = get_current_unix_timestamp_in_ms()
    
    #检查当前时间是否大于键的过期时间
    if now_ms ＞ expire_time_in_ms:
        #是，键已经过期
        return True
    else:
    	#否，键未过期
    	return False
```

## 5. 过期键删除策略
有三种不同的过期键删除策略：

- 定时删除：在设置键的过期时间的同时，创建一个定时器（timer），让定时器在键的过期时间来临时，立即执行对键的删除操作。
- 惰性删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
- 定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

在这三种策略中，第一种和第三种为主动删除策略，而第二种则为被动删除策略。

### 5.1 定时删除
定时删除策略对内存是最友好的：通过使用定时器，定时删除策略可以保证过期键会尽可能快地被删除，并释放过期键所占用的内存。

另一方面，定时删除策略的缺点是，它对 CPU 时间是最不友好的：在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分 CPU 时间，在内存不紧张但是 CPU 时间非常紧张的情况下，将 CPU 时间用在删除和当前任务无关的过期键上，无疑会对服务器的响应时间和吞吐量造成影响。

除此之外，创建一个定时器需要用到 Redis 服务器中的时间事件，而当前时间事件的实现方式——无序链表，查找一个事件的时间复杂度为 $O(N)$ ——并不能高效地处理大量时间事件。

### 5.2 惰性删除
惰性删除策略对 CPU 时间来说是最友好的：程序只会在取出键时才对键进行过期检查，这可以保证删除过期键的操作只会在非做不可的情况下进行，并且删除的目标仅限于当前处理的键，这个策略不会在删除其他无关的过期键上花费任何 CPU 时间。

惰性删除策略的缺点是，它对内存是最不友好的：如果一个键已经过期，而这个键又仍然保留在数据库中，那么只要这个过期键不被删除，它所占用的内存就不会释放。

在使用惰性删除策略时，如果数据库中有非常多的过期键，而这些过期键又恰好没有被访问到的话，那么它们也许永远也不会被删除（除非用户手动执行 `FLUSHDB`），我们甚至可以将这种情况看作是一种内存泄漏——无用的垃圾数据占用了大量的内存，而服务器却不会自己去释放它们，这对于运行状态非常依赖于内存的 Redis 服务器来说，肯定不是一个好消息。

### 5.3 定期删除
从上面对定时删除和惰性删除的讨论来看，这两种删除方式在单一使用时都有明显的缺陷：

- 定时删除占用太多 CPU 时间，影响服务器的响应时间和吞吐量。
- 惰性删除浪费太多内存，有内存泄漏的危险。

定期删除策略是前两种策略的一种整合和折中：

- 定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。
- 除此之外，通过定期删除过期键，定期删除策略有效地减少了因为过期键而带来的内存浪费。

定期删除策略的难点是确定删除操作执行的时长和频率：

- 如果删除操作执行得太频繁，或者执行的时间太长，定期删除策略就会退化成定时删除策略，以至于将 CPU 时间过多地消耗在删除过期键上面。
- 如果删除操作执行得太少，或者执行的时间太短，定期删除策略又会和惰性删除策略一样，出现浪费内存的情况。

因此，如果采用定期删除策略的话，服务器必须根据情况，合理地设置删除操作的执行时长和执行频率。

## 6. Redis的过期键删除策略
Redis 服务器实际使用的是惰性删除和定期删除两种策略：通过配合使用这两种删除策略，服务器可以很好地在合理使用CPU时间和避免浪费内存空间之间取得平衡。

### 6.1 惰性删除策略的实现
过期键的惰性删除策略由 `db.c/expireIfNeeded` 函数实现，所有读写数据库的 Redis 命令在执行之前都会调用 `expireIfNeeded` 函数对输入键进行检查：

- 如果输入键已经过期，那么 `expireIfNeeded` 函数将输入键从数据库中删除。
- 如果输入键未过期，那么 `expireIfNeeded` 函数不做动作。

`expireIfNeeded` 函数就像一个过滤器，它可以在命令真正执行之前，过滤掉过期的输入键，从而避免命令接触到过期键。

另外，因为每个被访问的键都可能因为过期而被 `expireIfNeeded` 函数删除，所以每个命令的实现函数都必须能同时处理键存在以及键不存在这两种情况：

- 当键存在时，命令按照键存在的情况执行。
- 当键不存在或者键因为过期而被 `expireIfNeeded` 函数删除时，命令按照键不存在的情况执行。

### 6.2 定期删除策略的实现
过期键的定期删除策略由 `redis.c/activeExpireCycle` 函数实现，每当 Redis 的服务器周期性操作 `redis.c/serverCron` 函数执行时，`activeExpireCycle` 函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的 `expires` 字典中随机检查一部分键的过期时间，并删除其中的过期键。

整个过程可以用伪代码描述如下：

```
#默认每次检查的数据库数量
DEFAULT_DB_NUMBERS = 16

#默认每个数据库检查的键数量
DEFAULT_KEY_NUMBERS = 20

#全局变量，记录检查进度
current_db = 0

def activeExpireCycle():

    #初始化要检查的数据库数量
    #如果服务器的数据库数量比 DEFAULT_DB_NUMBERS要小
    #那么以服务器的数据库数量为准
    if server.dbnum ＜ DEFAULT_DB_NUMBERS:
        db_numbers = server.dbnum
    else:
        db_numbers = DEFAULT_DB_NUMBERS
        
    #遍历各个数据库
    for i in range(db_numbers):
    
        #如果current_db的值等于服务器的数据库数量
        #这表示检查程序已经遍历了服务器的所有数据库一次
        #将current_db重置为0，开始新的一轮遍历
        if current_db == server.dbnum:
            current_db = 0
            
        #获取当前要处理的数据库
        redisDb = server.db[current_db]
        
        #将数据库索引增1，指向下一个要处理的数据库
        current_db += 1
        
        #检查数据库键
        for j in range(DEFAULT_KEY_NUMBERS):
        
            #如果数据库中没有一个键带有过期时间，那么跳过这个数据库
            if redisDb.expires.size() == 0: break
            
            #随机获取一个带有过期时间的键
            key_with_ttl = redisDb.expires.get_random_key()
            
            #检查键是否过期，如果过期就删除它
            if is_expired(key_with_ttl):
                delete_key(key_with_ttl)
                
            #已达到时间上限，停止处理
            if reach_time_limit(): return
```
`activeExpireCycle` 函数的工作模式可以总结如下： 

- 函数每次运行时，都从一定数量的数据库中取出一定数量的随机键进行检查，并删除其中的过期键。
- 全局变量 `current_db` 会记录当前 `activeExpireCycle` 函数检查的进度，并在下一次 `activeExpireCycle` 函数调用时，接着上一次的进度进行处理。
- 随着 `activeExpireCycle` 函数的不断执行，服务器中的所有数据库都会被检查一遍，这时函数将 `current_db` 变量重置为 0，然后再次开始新一轮的检查工作。

## 7. AOF、RDB 和复制功能对过期键的处理

### 7.1 生成 RDB 文件
在执行 `SAVE` 命令或者 `BGSAVE` 命令创建一个新的 RDB 文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的 RDB 文件中。

因此，数据库中包含过期键不会对生成新的RDB文件造成影响。

### 7.2 载入 RDB 文件
在启动 Redis 服务器时，如果服务器开启了 RDB 功能，那么服务器将对 RDB 文件进行载入：

- 如果服务器以主服务器模式运行，那么在载入 RDB 文件时，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库中，而过期键则会被忽略，所以过期键对载入 RDB 文件的主服务器不会造成影响。
- 如果服务器以从服务器模式运行，那么在载入 RDB 文件时，文件中保存的所有键，不论是否过期，都会被载入到数据库中。不过，因为主从服务器在进行数据同步的时候，从服务器的数据库就会被清空，所以一般来讲，过期键对载入 RDB 文件的从服务器也不会造成影响。

### 7.3 AOF 文件写入
当服务器以 AOF 持久化模式运行时，如果数据库中的某个键已经过期，但它还没有被惰性删除或者定期删除，那么 AOF 文件不会因为这个过期键而产生任何影响。

当过期键被惰性删除或者定期删除之后，程序会向 AOF 文件追加（append）一条 `DEL` 命令，来显式地记录该键已被删除。

### 7.4 AOF 重写
和生成 RDB 文件时类似，在执行 AOF 重写的过程中，程序会对数据库中的键进行检查，已过期的键不会被保存到重写后的 AOF 文件中。

### 7.5 复制
当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制：

- 主服务器在删除一个过期键之后，会显式地向所有从服务器发送一个 `DEL` 命令，告知从服务器删除这个过期键。
- 从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键。
- 从服务器只有在接到主服务器发来的 `DEL` 命令之后，才会删除过期键。

通过由主服务器来控制从服务器统一地删除过期键，可以保证主从服务器数据的一致性，也正是因为这个原因，当一个过期键仍然存在于主服务器的数据库时，这个过期键在从服务器里的复制品也会继续存在。

举个例子，有一对主从服务器，它们的数据库中都保存着同样的三个键 message、xxx 和 yyy，其中 message 为过期键。如果这时有客户端向从服务器发送命令 `GET message`，那么从服务器将发现 message 键已经过期，但从服务器并不会删除 message 键，而是继续将 message 键的值返回给客户端，就好像 message 键并没有过期一样。假设在此之后，有客户端向主服务器发送命令 `GET message`，那么主服务器将发现键 message 已经过期：主服务器会删除 message 键，向客户端返回空回复，并向从服务器发送 `DEL message` 命令。从服务器在接收到主服务器发来的 `DEL message` 命令之后，也会从数据库中删除 message 键，在这之后，主从服务器都不再保存过期键 message 了。

## 8. 数据库通知
数据库通知是 Redis 2.8 版本新增加的功能，这个功能可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。

关注 “某个键执行了什么命令” 的通知称为键空间通知（key-space notification），除此之外，还有另一类称为键事件通知（key-event notification）的通知，它们关注的是 “某个命令被什么键执行了”。

服务器配置的 `notify-keyspace-events` 选项决定了服务器所发送通知的类型：

- 想让服务器发送所有类型的键空间通知和键事件通知，可以将选项的值设置为 AKE。
- 想让服务器发送所有类型的键空间通知，可以将选项的值设置为 AK。
- 想让服务器发送所有类型的键事件通知，可以将选项的值设置为 AE。
- 想让服务器只发送和字符串键有关的键空间通知，可以将选项的值设置为 K$。
- 想让服务器只发送和列表键有关的键事件通知，可以将选项的值设置为 El。

### 8.1 发送通知
发送数据库通知的功能是由 `notify.c/notifyKeyspaceEvent` 函数实现的：

```c
void notifyKeyspaceEvent(int type,char *event,robj *key,int dbid);
```
函数的 `type` 参数是当前想要发送的通知的类型，程序会根据这个值来判断通知是否就是服务器配置 `notify-keyspace-events` 选项所选定的通知类型，从而决定是否发送通知。

`event`、`keys` 和 `dbid` 分别是事件的名称、产生事件的键，以及产生事件的数据库号码，函数会根据 `type` 参数以及这三个参数来构建事件通知的内容，以及接收通知的频道名。

每当一个 Redis 命令需要发送数据库通知的时候，该命令的实现函数就会调用 `notify-KeyspaceEvent` 函数，并向函数传递传递该命令所引发的事件的相关信息。

### 8.2 发送通知的实现
以下是 `notifyKeyspaceEvent` 函数的伪代码实现：

```
def notifyKeyspaceEvent(type, event, key, dbid):
    #如果给定的通知不是服务器允许发送的通知，那么直接返回
    if not(server.notify_keyspace_events & type):
        return
        
    #发送键空间通知
    if server.notify_keyspace_events & REDIS_NOTIFY_KEYSPACE:
    
        #将通知发送给频道__keyspace@＜dbid＞__:＜key＞
        #内容为键所发生的事件 ＜event＞
        
        #构建频道名字
        chan = "__keyspace@{dbid}__:{key}".format(dbid=dbid, key=key)
        
        #发送通知
        pubsubPublishMessage(chan, event)
        
    #发送键事件通知
    if server.notify_keyspace_events & REDIS_NOTIFY_KEYEVENT:
    
        #将通知发送给频道__keyevent@＜dbid＞__:＜event＞
        #内容为发生事件的键 ＜key＞
        
        #构建频道名字
        chan = "__keyevent@{dbid}__:{event}".format(dbid=dbid,event=event)
        
        #发送通知
        pubsubPublishMessage(chan, key)
```
`notifyKeyspaceEvent` 函数执行以下操作：

1. `server.notify_keyspace_events` 属性就是服务器配置 `notify-keyspace-events` 选项所设置的值，如果给定的通知类型 `type` 不是服务器允许发送的通知类型，那么函数会直接返回，不做任何动作。
2. 如果给定的通知是服务器允许发送的通知，那么下一步函数会检测服务器是否允许发送键空间通知，如果允许的话，程序就会构建并发送事件通知。
3. 最后，函数检测服务器是否允许发送键事件通知，如果允许的话，程序就会构建并发送事件通知。

另外，`pubsubPublishMessage` 函数是 `PUBLISH` 命令的实现函数，执行这个函数等同于执行 `PUBLISH` 命令，订阅数据库通知的客户端收到的信息就是由这个函数发出的。