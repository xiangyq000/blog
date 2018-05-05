---
title: 《Redis 设计与实现：服务器》
date: 2017-10-06 17:30:26
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

## 0. redisServer 数据结构

<!--more-->

```c
struct redisServer {
    /* General */
    char *configfile;           /* Absolute config file path, or NULL */
    int hz;                     /* serverCron() calls frequency in hertz */
    redisDb *db;
    dict *commands;             /* Command table */
    dict *orig_commands;        /* Command table before command renaming. */
    aeEventLoop *el;
    unsigned lruclock:REDIS_LRU_BITS; /* Clock for LRU eviction */
    int shutdown_asap;          /* SHUTDOWN needed ASAP */
    int activerehashing;        /* Incremental rehash in serverCron() */
    char *requirepass;          /* Pass for AUTH command, or NULL */
    char *pidfile;              /* PID file path */
    int arch_bits;              /* 32 or 64 depending on sizeof(long) */
    int cronloops;              /* Number of times the cron function run */
    char runid[REDIS_RUN_ID_SIZE+1];  /* ID always different at every exec. */
    int sentinel_mode;          /* True if this instance is a Sentinel. */
    /* Networking */
    int port;                   /* TCP listening port */
    int tcp_backlog;            /* TCP listen() backlog */
    char *bindaddr[REDIS_BINDADDR_MAX]; /* Addresses we should bind to */
    int bindaddr_count;         /* Number of addresses in server.bindaddr[] */
    char *unixsocket;           /* UNIX socket path */
    mode_t unixsocketperm;      /* UNIX socket permission */
    int ipfd[REDIS_BINDADDR_MAX]; /* TCP socket file descriptors */
    int ipfd_count;             /* Used slots in ipfd[] */
    int sofd;                   /* Unix socket file descriptor */
    list *clients;              /* List of active clients */
    list *clients_to_close;     /* Clients to close asynchronously */
    list *slaves, *monitors;    /* List of slaves and MONITORs */
    redisClient *current_client; /* Current client, only used on crash report */
    char neterr[ANET_ERR_LEN];   /* Error buffer for anet.c */
    uint64_t next_client_id;    /* Next client unique ID. Incremental. */
    /* RDB / AOF loading information */
    int loading;                /* We are loading data from disk if true */
    off_t loading_total_bytes;
    off_t loading_loaded_bytes;
    time_t loading_start_time;
    off_t loading_process_events_interval_bytes;
    /* Fast pointers to often looked up command */
    struct redisCommand *delCommand, *multiCommand, *lpushCommand, *lpopCommand,
                        *rpopCommand;
    /* Fields used only for stats */
    time_t stat_starttime;          /* Server start time */
    long long stat_numcommands;     /* Number of processed commands */
    long long stat_numconnections;  /* Number of connections received */
    long long stat_expiredkeys;     /* Number of expired keys */
    long long stat_evictedkeys;     /* Number of evicted keys (maxmemory) */
    long long stat_keyspace_hits;   /* Number of successful lookups of keys */
    long long stat_keyspace_misses; /* Number of failed lookups of keys */
    size_t stat_peak_memory;        /* Max used memory record */
    long long stat_fork_time;       /* Time needed to perform latest fork() */
    double stat_fork_rate;          /* Fork rate in GB/sec. */
    long long stat_rejected_conn;   /* Clients rejected because of maxclients */
    long long stat_sync_full;       /* Number of full resyncs with slaves. */
    long long stat_sync_partial_ok; /* Number of accepted PSYNC requests. */
    long long stat_sync_partial_err;/* Number of unaccepted PSYNC requests. */
    list *slowlog;                  /* SLOWLOG list of commands */
    long long slowlog_entry_id;     /* SLOWLOG current entry ID */
    long long slowlog_log_slower_than; /* SLOWLOG time limit (to get logged) */
    unsigned long slowlog_max_len;     /* SLOWLOG max number of items logged */
    size_t resident_set_size;       /* RSS sampled in serverCron(). */
    long long stat_net_input_bytes; /* Bytes read from network. */
    long long stat_net_output_bytes; /* Bytes written to network. */
    /* The following two are used to track instantaneous metrics, like
     * number of operations per second, network traffic. */
    struct {
        long long last_sample_time; /* Timestamp of last sample in ms */
        long long last_sample_count;/* Count in last sample */
        long long samples[REDIS_METRIC_SAMPLES];
        int idx;
    } inst_metric[REDIS_METRIC_COUNT];
    /* Configuration */
    int verbosity;                  /* Loglevel in redis.conf */
    int maxidletime;                /* Client timeout in seconds */
    int tcpkeepalive;               /* Set SO_KEEPALIVE if non-zero. */
    int active_expire_enabled;      /* Can be disabled for testing purposes. */
    size_t client_max_querybuf_len; /* Limit for client query buffer length */
    int dbnum;                      /* Total number of configured DBs */
    int daemonize;                  /* True if running as a daemon */
    clientBufferLimitsConfig client_obuf_limits[REDIS_CLIENT_TYPE_COUNT];
    /* AOF persistence */
    int aof_state;                  /* REDIS_AOF_(ON|OFF|WAIT_REWRITE) */
    int aof_fsync;                  /* Kind of fsync() policy */
    char *aof_filename;             /* Name of the AOF file */
    int aof_no_fsync_on_rewrite;    /* Don't fsync if a rewrite is in prog. */
    int aof_rewrite_perc;           /* Rewrite AOF if % growth is > M and... */
    off_t aof_rewrite_min_size;     /* the AOF file is at least N bytes. */
    off_t aof_rewrite_base_size;    /* AOF size on latest startup or rewrite. */
    off_t aof_current_size;         /* AOF current size. */
    int aof_rewrite_scheduled;      /* Rewrite once BGSAVE terminates. */
    pid_t aof_child_pid;            /* PID if rewriting process */
    list *aof_rewrite_buf_blocks;   /* Hold changes during an AOF rewrite. */
    sds aof_buf;      /* AOF buffer, written before entering the event loop */
    int aof_fd;       /* File descriptor of currently selected AOF file */
    int aof_selected_db; /* Currently selected DB in AOF */
    time_t aof_flush_postponed_start; /* UNIX time of postponed AOF flush */
    time_t aof_last_fsync;            /* UNIX time of last fsync() */
    time_t aof_rewrite_time_last;   /* Time used by last AOF rewrite run. */
    time_t aof_rewrite_time_start;  /* Current AOF rewrite start time. */
    int aof_lastbgrewrite_status;   /* REDIS_OK or REDIS_ERR */
    unsigned long aof_delayed_fsync;  /* delayed AOF fsync() counter */
    int aof_rewrite_incremental_fsync;/* fsync incrementally while rewriting? */
    int aof_last_write_status;      /* REDIS_OK or REDIS_ERR */
    int aof_last_write_errno;       /* Valid if aof_last_write_status is ERR */
    int aof_load_truncated;         /* Don't stop on unexpected AOF EOF. */
    /* RDB persistence */
    long long dirty;                /* Changes to DB from the last save */
    long long dirty_before_bgsave;  /* Used to restore dirty on failed BGSAVE */
    pid_t rdb_child_pid;            /* PID of RDB saving child */
    struct saveparam *saveparams;   /* Save points array for RDB */
    int saveparamslen;              /* Number of saving points */
    char *rdb_filename;             /* Name of RDB file */
    int rdb_compression;            /* Use compression in RDB? */
    int rdb_checksum;               /* Use RDB checksum? */
    time_t lastsave;                /* Unix time of last successful save */
    time_t lastbgsave_try;          /* Unix time of last attempted bgsave */
    time_t rdb_save_time_last;      /* Time used by last RDB save run. */
    time_t rdb_save_time_start;     /* Current RDB save start time. */
    int rdb_child_type;             /* Type of save by active child. */
    int lastbgsave_status;          /* REDIS_OK or REDIS_ERR */
    int stop_writes_on_bgsave_err;  /* Don't allow writes if can't BGSAVE */
    int rdb_pipe_write_result_to_parent; /* RDB pipes used to return the state */
    int rdb_pipe_read_result_from_child; /* of each slave in diskless SYNC. */
    /* Propagation of commands in AOF / replication */
    redisOpArray also_propagate;    /* Additional command to propagate. */
    /* Logging */
    char *logfile;                  /* Path of log file */
    int syslog_enabled;             /* Is syslog enabled? */
    char *syslog_ident;             /* Syslog ident */
    int syslog_facility;            /* Syslog facility */
    /* Replication (master) */
    int slaveseldb;                 /* Last SELECTed DB in replication output */
    long long master_repl_offset;   /* Global replication offset */
    int repl_ping_slave_period;     /* Master pings the slave every N seconds */
    char *repl_backlog;             /* Replication backlog for partial syncs */
    long long repl_backlog_size;    /* Backlog circular buffer size */
    long long repl_backlog_histlen; /* Backlog actual data length */
    long long repl_backlog_idx;     /* Backlog circular buffer current offset */
    long long repl_backlog_off;     /* Replication offset of first byte in the
                                       backlog buffer. */
    time_t repl_backlog_time_limit; /* Time without slaves after the backlog
                                       gets released. */
    time_t repl_no_slaves_since;    /* We have no slaves since that time.
                                       Only valid if server.slaves len is 0. */
    int repl_min_slaves_to_write;   /* Min number of slaves to write. */
    int repl_min_slaves_max_lag;    /* Max lag of <count> slaves to write. */
    int repl_good_slaves_count;     /* Number of slaves with lag <= max_lag. */
    int repl_diskless_sync;         /* Send RDB to slaves sockets directly. */
    int repl_diskless_sync_delay;   /* Delay to start a diskless repl BGSAVE. */
    /* Replication (slave) */
    char *masterauth;               /* AUTH with this password with master */
    char *masterhost;               /* Hostname of master */
    int masterport;                 /* Port of master */
    int repl_timeout;               /* Timeout after N seconds of master idle */
    redisClient *master;     /* Client that is master for this slave */
    redisClient *cached_master; /* Cached master to be reused for PSYNC. */
    int repl_syncio_timeout; /* Timeout for synchronous I/O calls */
    int repl_state;          /* Replication status if the instance is a slave */
    off_t repl_transfer_size; /* Size of RDB to read from master during sync. */
    off_t repl_transfer_read; /* Amount of RDB read from master during sync. */
    off_t repl_transfer_last_fsync_off; /* Offset when we fsync-ed last time. */
    int repl_transfer_s;     /* Slave -> Master SYNC socket */
    int repl_transfer_fd;    /* Slave -> Master SYNC temp file descriptor */
    char *repl_transfer_tmpfile; /* Slave-> master SYNC temp file name */
    time_t repl_transfer_lastio; /* Unix time of the latest read, for timeout */
    int repl_serve_stale_data; /* Serve stale data when link is down? */
    int repl_slave_ro;          /* Slave is read only? */
    time_t repl_down_since; /* Unix time at which link with master went down */
    int repl_disable_tcp_nodelay;   /* Disable TCP_NODELAY after SYNC? */
    int slave_priority;             /* Reported in INFO and used by Sentinel. */
    char repl_master_runid[REDIS_RUN_ID_SIZE+1];  /* Master run id for PSYNC. */
    long long repl_master_initial_offset;         /* Master PSYNC offset. */
    /* Replication script cache. */
    dict *repl_scriptcache_dict;        /* SHA1 all slaves are aware of. */
    list *repl_scriptcache_fifo;        /* First in, first out LRU eviction. */
    unsigned int repl_scriptcache_size; /* Max number of elements. */
    /* Limits */
    unsigned int maxclients;            /* Max number of simultaneous clients */
    unsigned long long maxmemory;   /* Max number of memory bytes to use */
    int maxmemory_policy;           /* Policy for key eviction */
    int maxmemory_samples;          /* Pricision of random sampling */
    /* Blocked clients */
    unsigned int bpop_blocked_clients; /* Number of clients blocked by lists */
    list *unblocked_clients; /* list of clients to unblock before next loop */
    list *ready_keys;        /* List of readyList structures for BLPOP & co */
    /* Sort parameters - qsort_r() is only available under BSD so we
     * have to take this state global, in order to pass it to sortCompare() */
    int sort_desc;
    int sort_alpha;
    int sort_bypattern;
    int sort_store;
    /* Zip structure config, see redis.conf for more information  */
    size_t hash_max_ziplist_entries;
    size_t hash_max_ziplist_value;
    size_t list_max_ziplist_entries;
    size_t list_max_ziplist_value;
    size_t set_max_intset_entries;
    size_t zset_max_ziplist_entries;
    size_t zset_max_ziplist_value;
    size_t hll_sparse_max_bytes;
    time_t unixtime;        /* Unix time sampled every cron cycle. */
    long long mstime;       /* Like 'unixtime' but with milliseconds resolution. */
    /* Pubsub */
    dict *pubsub_channels;  /* Map channels to list of subscribed clients */
    list *pubsub_patterns;  /* A list of pubsub_patterns */
    int notify_keyspace_events; /* Events to propagate via Pub/Sub. This is an
                                   xor of REDIS_NOTIFY... flags. */
    /* Scripting */
    lua_State *lua; /* The Lua interpreter. We use just one for all clients */
    redisClient *lua_client;   /* The "fake client" to query Redis from Lua */
    redisClient *lua_caller;   /* The client running EVAL right now, or NULL */
    dict *lua_scripts;         /* A dictionary of SHA1 -> Lua scripts */
    mstime_t lua_time_limit;  /* Script timeout in milliseconds */
    mstime_t lua_time_start;  /* Start time of script, milliseconds time */
    int lua_write_dirty;  /* True if a write command was called during the
                             execution of the current script. */
    int lua_random_dirty; /* True if a random command was called during the
                             execution of the current script. */
    int lua_timedout;     /* True if we reached the time limit for script
                             execution. */
    int lua_kill;         /* Kill the script if true. */
    /* Latency monitor */
    long long latency_monitor_threshold;
    dict *latency_events;
    /* Assert & bug reporting */
    char *assert_failed;
    char *assert_file;
    int assert_line;
    int bug_report_start; /* True if bug report header was already logged. */
    int watchdog_period;  /* Software watchdog period in ms. 0 = off */
};
```

<!--more-->

## 1. 命令请求的执行过程

### 1.1 发送命令请求

Redis 服务器的命令请求来自 Redis 客户端，当用户在客户端中键入一个命令请求时，客户端会将这个命令请求转换成协议格式，然后通过连接到服务器的套接字，将协议格式的命令请求发送给服务器。

<!--more-->

### 1.2 读取命令请求

当客户端与服务器之间的连接套接字因为客户端的写入而变得可读时，服务器将调用命令请求处理器来执行以下操作：

1. 读取套接字中协议格式的命令请求，并将其保存到客户端状态的输入缓冲区里面。
2. 对输入缓冲区中的命令请求进行分析，提取出命令请求中包含的命令参数，以及命令参数的个数，然后分别将参数和参数个数保存到客户端状态的 `argv` 属性和 `argc` 属性里面。
3. 调用命令执行器，执行客户端指定的命令。

### 1.3 命令执行器（1）：查找命令实现

命令执行器要做的第一件事就是根据客户端状态的 `argv[0]` 参数，在命令表（command table）中查找参数所指定的命令，并将找到的命令保存到客户端状态的 `cmd` 属性里面。

命令表是一个字典，字典的键是一个个命令名字，比如 `"set"`、`"get"`、`"del"`，等等；而字典的值则是一个个 `redisCommand` 结构，每个 `redisCommand` 结构记录了一个 Redis 命令的实现信息：

```c
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags; /* Flags as string representation, one char per flag. */
    int flags;    /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
};
```

| 属性名            | 类型                   | 作用                                       |
| -------------- | -------------------- | ---------------------------------------- |
| `name`         | `char *`             | 命令的名字，比如 `"set"` 。                       |
| `proc`         | `redisCommandProc *` | 函数指针，指向命令的实现函数，比如 `setCommand`。 `redisCommandProc` 类型的定义为 `typedef void redisCommandProc(redisClient *c);`。 |
| `arity`        | `int`                | 命令参数的个数，用于检查命令请求的格式是否正确。如果这个值为负数 `-N`，那么表示参数的数量大于等于 `N`。 注意命令的名字本身也是一个参数，比如说 `SET msg"hello world"` 命令的参数是 `"SET"`、`"msg"`、`"hello world"`，而不仅仅是 `"msg"` 和 `"hello world"`。 |
| `sflags`       | `char *`             | 字符串形式的标识值，这个值记录了命令的属性，比如这个命令是写命令还是读命令，这个命令是否允许在载入数据时使用，这个命令是否允许在 Lua 脚本中使用，等等。 |
| `flags`        | `int`                | 对 `sflags` 标识进行分析得出的二进制标识，由程序自动生成。服务器对命令标识进行检查时使用的都是 `flags` 属性而不是 `sflags` 属性，因为对二进制标识的检查可以方便地通过 `&`、`^`、`~` 等操作来完成。 |
| `calls`        | `long long`          | 服务器总共执行了多少次这个命令。                         |
| `milliseconds` | `long long`          | 服务器执行这个命令所耗费的总时长。                        |

`flags` 属性可以使用的标识符及意义：

| 标识   | 意义                                       | 带有这个标识的命令                                |
| ---- | ---------------------------------------- | ---------------------------------------- |
| `w`  | 这是一个写入命令，可能会修改数据库。                       | SET、RPUSH、DEL，等等。                        |
| `r`  | 这是一个只读命令，不会修改数据库。                        | GET、STRLEN、EXISTS，等等。                    |
| `m`  | 这个命令可能会占用大量内存，执行之前需要先检查服务器的内存使用情况，如果内存紧缺的话就禁止执行这个命令。 | SET、APPEND、RPUSH、 LPUSH、SADD、SINTERSTORE，等等。 |
| `a`  | 这是一个管理命令。                                | SAVE、BGSAVE、SHUTDOWN，等等。                 |
| `p`  | 这是一个发布与订阅功能方面的命令。                        | PUBLISH、SUBSCRIBE、PUBSUB，等等。             |
| `s`  | 这个命令不可以在 Lua 脚本中使用。                      | BRPOP、BLPOP、BRPOPLPUSH、SPOP，等等。          |
| `R`  | 这是一个随机命令，对于相同的数据集和相同的参数，命令返回的结果可能不同。     | SPOP、SRANDMEMBER、SSCAN、RANDOMKEY，等等。     |
| `S`  | 当在 Lua 脚本中使用这个命令时，对这个命令的输出结果进行一次排序，使得命令的结果有序。 | SINTER、SUNION、SDIFF、SMEMBERS、KEYS，等等。    |
| `l`  | 这个命令可以在服务器载入数据的过程中使用。                    | INFO、SHUTDOWN、PUBLISH，等等。                |
| `t`  | 这是一个允许从服务器在带有过期数据时使用的命令。                 | SLAVEOF、PING、INFO，等等。                    |
| `M`  | 这个命令在监视器（monitor）模式下不会自动被传播（propagate）。  | EXEC                                     |

### 1.4 命令执行器（2）：执行预备操作

到目前为止，服务器已经将执行命令所需的命令实现函数（保存在客户端状态的 `cmd` 属性）、参数（保存在客户端状态的 `argv` 属性）、参数个数（保存在客户端状态的 `argc` 属性）都收集齐了，但是在真正执行命令之前，程序还需要进行一些预备操作，从而确保命令可以正确、顺利地被执行，这些操作包括：

- 检查客户端状态的 `cmd` 指针是否指向 `NULL` ，如果是的话，那么说明用户输入的命令名字找不到相应的命令实现，服务器不再执行后续步骤，并向客户端返回一个错误。
- 根据客户端 `cmd` 属性指向的 `redisCommand` 结构的 `arity` 属性，检查命令请求所给定的参数个数是否正确，当参数个数不正确时，不再执行后续步骤，直接向客户端返回一个错误。比如说，如果 `redisCommand` 结构的 `arity` 属性的值为 -3，那么用户输入的命令参数个数必须大于等于 3 个才行。
- 检查客户端是否已经通过了身份验证，未通过身份验证的客户端只能执行 `AUTH` 命令，如果未通过身份验证的客户端试图执行除 `AUTH` 命令之外的其他命令，那么服务器将向客户端返回一个错误。
- 如果服务器打开了 `maxmemory` 功能，那么在执行命令之前，先检查服务器的内存占用情况，并在有需要时进行内存回收，从而使得接下来的命令可以顺利执行。如果内存回收失败，那么不再执行后续步骤，向客户端返回一个错误。
- 如果服务器上一次执行 `BGSAVE` 命令时出错，并且服务器打开了 `stop-writes-on-bgsave-error` 功能，而且服务器即将要执行的命令是一个写命令，那么服务器将拒绝执行这个命令，并向客户端返回一个错误。
- 如果客户端当前正在用 `SUBSCRIBE` 命令订阅频道， 或者正在用 `PSUBSCRIBE` 命令订阅模式， 那么服务器只会执行客户端发来的 `SUBSCRIBE`、`PSUBSCRIBE`、`UNSUBSCRIBE`、`PUNSUBSCRIBE` 四个命令， 其他别的命令都会被服务器拒绝。
- 如果服务器正在进行数据载入，那么客户端发送的命令必须带有 `l` 标识（比如 `INFO`、`SHUTDOWN`、`PUBLISH`，等等）才会被服务器执行，其他别的命令都会被服务器拒绝。
- 如果服务器因为执行 Lua 脚本而超时并进入阻塞状态，那么服务器只会执行客户端发来的 `SHUTDOWN nosave` 命令和 `SCRIPT KILL` 命令， 其他别的命令都会被服务器拒绝。
- 如果客户端正在执行事务， 那么服务器只会执行客户端发来的 `EXEC`、`DISCARD`、`MULTI`、`WATCH` 四个命令， 其他命令都会被放进事务队列中。
- 如果服务器打开了监视器功能， 那么服务器会将要执行的命令和参数等信息发送给监视器。

当完成了以上预备操作之后， 服务器就可以开始真正执行命令了。

### 1.5 命令执行器（3）：调用命令的实现函数

在前面的操作中，服务器已经将要执行命令的实现保存到了客户端状态的 `cmd` 属性里面，并将命令的参数和参数个数分别保存到了客户端状态的 `argv` 属性和 `argc` 属性里面，当服务器决定要执行命令时，它只要执行以下语句就可以了：

```
// client 是指向客户端状态的指针

client->cmd->proc(client);

```

因为执行命令所需的实际参数都已经保存到客户端状态的 `argv` 属性里面了，所以命令的实现函数只需要一个指向客户端状态的指针作为参数即可。

被调用的命令实现函数会执行指定的操作，并产生相应的命令回复，这些回复会被保存在客户端状态的输出缓冲区里面（`buf` 属性和 `reply` 属性），之后实现函数还会为客户端的套接字关联命令回复处理器，这个处理器负责将命令回复返回给客户端。

### 1.6 命令执行器（4）：执行后续工作

在执行完实现函数之后，服务器还需要执行一些后续工作：

- 如果服务器开启了慢查询日志功能，那么慢查询日志模块会检查是否需要为刚刚执行完的命令请求添加一条新的慢查询日志。
- 根据刚刚执行命令所耗费的时长，更新被执行命令的 `redisCommand` 结构的 `milliseconds` 属性，并将命令的 `redisCommand` 结构的 `calls` 计数器的值增一。
- 如果服务器开启了 AOF 持久化功能，那么 AOF 持久化模块会将刚刚执行的命令请求写入到 AOF 缓冲区里面。
- 如果有其他从服务器正在复制当前这个服务器，那么服务器会将刚刚执行的命令传播给所有从服务器。

当以上操作都执行完了之后，服务器对于当前命令的执行到此就告一段落了，之后服务器就可以继续从文件事件处理器中取出并处理下一个命令请求了。

### 1.7 将命令回复发送给客户端

命令实现函数会将命令回复保存到客户端的输出缓冲区里面，并为客户端的套接字关联命令回复处理器，当客户端套接字变为可写状态时，服务器就会执行命令回复处理器，将保存在客户端输出缓冲区中的命令回复发送给客户端。

当命令回复发送完毕之后，回复处理器会清空客户端状态的输出缓冲区，为处理下一个命令请求做好准备。

### 1.8 客户端接收并打印命令回复

当客户端接收到协议格式的命令回复之后，它会将这些回复转换成人类可读的格式，并打印给用户观看。

## 2. serverCron 函数

### 2.1 更新服务器时间缓存

Redis 服务器中有不少功能需要获取系统的当前时间，而每次获取系统的当前时间都需要执行一次系统调用，为了减少系统调用的执行次数，服务器状态中的 `unixtime` 属性和 `mstime` 属性被用作当前时间的缓存。

由于 `serverCron` 函数默认会以每 100 毫秒一次的频率更新 `unixtime` 属性和 `mstime` 属性，所以这两个属性记录的时间精确度并不高：

- 服务器只会在打印日志、更新服务器的 LRU 时钟、决定是否执行持久化任务、计算服务器上线时间（uptime）这类对事件精确度要求不高的功能上 “使用 unixtime 属性和 mstime 属性”。
- 对于为键设置过期时间、添加慢查询日志这种需要高精度时间的功能来说，服务器还是会再次执行系统调用，从而获得最准确的系统当前时间。

### 2.2 更新 LRU 时钟

服务器状态中的 `lruclock` 属性保存了服务器的 LRU 时钟，也是服务器时间缓存的一种：

```c
struct redisServer{

	//...

	// 默认每 10 秒更新一次时钟缓存
	// 用于计算键的空转(idle)时长
	unsigned lruclock:22;

	//...

}
```

每个 Redis 对象都会有一个 `lru` 属性，保存了对象最后一次被命令访问的时间：

```c
typedef struct redisObject {
  
	//...

	unsigned lrulock:22;

	//...
} robj;
```

当服务器要计算一个数据库键的空转时间（也即是数据库键对应的值对象的空转时间），程序会用服务器的 `lruclock` 属性记录的时间减去对象的 `lru` 属性记录的时间，得出的计算结果就是这个对象的空转时间。

`serverCron` 函数默认以每 10 秒一次的频率更新 `lruclock` 属性的值，因为这个时钟不是实时的，所以根据这个属性计算出来的 LRU 时间实际上只是一个模糊的估算值。

`lruclock` 时钟的当前值可以通过 `INFO server` 命令的 `lru_clock` 域查看。

### 2.3 更新服务器每秒执行命令次数

``serverCron` 函数中的 `trackOperationsPerSecond` 函数会以每 100ms 一次的频率执行，这个函数的功能是以抽样计算的方式，估算并记录服务器在最近一秒钟处理的命令请求数量。可以通过 `INFO stats` 命令的 `instantaneous_ops_per_sec ` 域查看。

### 2.4 更新服务器内存峰值记录

服务器状态中的 `stat_peak_memory` 属性记录了服务器的内存峰值大小：

```c
struct redisServer{

	//...

	// 已使用内存峰值
	size_t stat_peak_memory;

	//...

}
```

每次 `serverCron` 函数执行时，程序都会查看服务器当前使用的内存数量，并与 `stat_peak_memory` 保存的数值进行比较，如果当前使用的内存数量比 `stat_peak_memory` 属性记录的值要大，那么程序就将当前使用的内存数量记录到 `stat_peak_memory` 属性里面。

`INFO memory` 命令的 `used_memory_peak` 和 `used_memory_peak_human` 两个域分别以两种格式记录了服务器的内存峰值：

```
redis>INFO memory
# Memory
...
used_momory_peak:501824
used_memory_peak_human:490.06K
...
```

### 2.5 处理SIGTERM信号

在启动服务器时，Redis 会为服务器进程的 `SIGTERM` 信号关联处理器 `sigtermHandler` 函数，这个信号处理器负责在服务器接到 `SIGTERM` 信号时，打开服务器状态的 `shutdown_asap` 标识：

```c
// SIGTERM信号的处理器
static void sigtermHandler(int sig){

	// 打印日至
	redisLogFromHandler(REDIS_WARNIING,"Received SIGTERM,scheduling shutdown...");

	// 打开关闭标识
	server.shutdown_asap = 1;

}
```

每次 `serverCron` 函数运行时，程序都会对服务器状态的 `shutdown_asap` 属性进行检查，并根据属性的值决定是否关闭服务器：

```c
struct redisServer{

	//...

	// 关闭服务器的标识：
	// 值为1时，关闭服务器，
	// 值为0时，不做动作。
	int shutdown_asap;

	//...

};
```

服务器在关闭自身之前会进行 RDB 持久化操作，这也是服务器拦截 `SIGTERM` 信号的原因，如果服务器一接到 `SIGTERM` 信号就立即关闭，那么它就没办法执行持久化操作了。

### 2.6 管理客户端资源

`serverCron` 函数每次执行都会调用 `clientsCron` 函数，`clientsCron` 函数会对一定数量的客户端进行以下两种检查：

- 如果客户端和服务器之间的连接已经超时，那么程序释放这个客户端。
- 如果客户端在上一次执行命令请求后，输入缓冲区的大小超过了一定的长度，那么程序会释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区，从而防止客户端的输入缓冲区耗费了过多的内存。

### 2.7 管理数据库资源

`serverCron` 函数每次执行都会调用 `databaseCron` 函数，这个函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。

### 2.8 执行被延迟的 BGREWRITEAOF

服务器在执行 `BGSAVE` 期间，如果客户端向服务器请求了 `BGREWRITEAOF` 命令，那么服务器会将 `BGREWRITEAOF` 命令延迟到 `BGSAVE` 执行完毕之后执行。

服务器的 `aof_rewrite_scheduled` 标识记录了服务器是否延迟了 `BGREWRITEAOF` 命令：

```c
struct redisServer{

	//...

	// 如果为 1，那么表示有 BGREWRITEAOF 命令被延迟了
	int aof_rewrite_scheduled;

	//...

};
```

每次 `serverCron` 函数执行时，函数都会检查 `BGSAVE` 命令或者 `BGREWRITEAOF` 命令是否正在执行，如果这两个命令都没在执行，并且 `aof_rewrite_scheduled` 属性的值为 1，那么服务器就会执行之前被推延的 `BGREWRITEAOF` 命令。

### 2.9 检查持久化操作的运行状态

服务器状态用 `rdb_child_pid` 和 `aof_child_pid` 属性记录了执行 `BGSAVE` 命令和 `BGREWRITEAOF` 命令的子进程的 ID，这两个属性可以用于检查这两个命令是否正在执行：

```c
struct redisServer{

	//...

	// 记录执行 BGSAVE 命令的子进程 ID：
	// 如果服务器没有在执行 BGSAVE，
	// 那么这个属性的值为 -1。
	pid_t rdb_child_pid;
  
	// 记录执行 BGREWRITEAOF 命令的子进程 ID：
	// 如果服务器没有在执行 BGREWRITEAOF，
	// 那么这个属性的值为 -1。
	pid_t aof_child_pid;
  
	//...

};
```

`serverCron` 函数每次执行都会检查 `rdb_child_pid` 和 `aof_child_pid` 两个属性的值，只要其中一个属性的值不为 -1，程序会执行一次 `wait3` 函数，检查子进程是否有信号发来服务器进程：

- 如果有信号到达，那么表示新的 RDB 文件已经生成完毕（对于 `BGSAVE` 命令来说）或者 AOF 文件已经重写完毕（对于 `BGREWRITEAOF` 命令来说），服务器需要进行相应命令的后续操作，比如用新的 RDB 文件替换现有的 RDB 文件，或者用重写后的 AOF 文件替换现有的 AOF 文件。
- 如果没有信号到达，那么表示持久化操作未完成，程序不做动作。

另一方面，如果 `rdb_child_pid` 和 `aof_child_pid` 两个属性的值都为 -1，那么表示服务器没有在进行持久化操作，在这种情况下，程序执行以下三个检查：

1. 查看是否有 `BGREWRITEAOF` 被延迟了，如果有的话，那么开始一次新的 `BGREWRITEAOF` 操作。
2. 检查服务器自动保存条件是否已经被满足，如果条件满足，并且服务器没有在执行其他持久化操作，那么服务器开始一次新的 `BGSAVE` 操作（因为条件 1 可能会引发一次 `BGREWRITEAOF`，所以在这个检查中，程序会再次确认服务器是否已经在执行持久化操作了）。
3. 检查服务器设置的 AOF 重写条件是否满足，如果条件满足，并且服务器没有在执行其他持久化操作，那么服务器将开始一次新的 `BGREWRITEAOF` 操作（因为条件 1 和条件 2 都可能会引起新的持久化操作，所以在这个检查中，我们要再次确认服务器是否已经在执行持久化操作了）。

### 2.10 将 AOF 缓冲区中的内容写入 AOF 文件

如果服务器开启了 AOF 持久化功能，并且 AOF 缓冲区里面还有待写入的数据，那么 `serverCron` 函数会调用相应的程序，将 AOF 缓冲区中的内容写入到 AOF 文件里面。

### 2.11 关闭异步客户端

服务器会关闭那些输出缓冲区大小超出限制的客户端。

### 2.12 增加 cronloops 计数器的值

服务器状态的 `cronloops` 属性记录了 `serverCron` 函数执行的次数：

```c
struct redisServer {

	//...

	// serverCron 函数的运行次数计数器
	// serverCron 函数每执行一次，这个属性的值就增一
	int cronloops;

	//...
}
```

`cronloops` 属性目前在服务器中的唯一作用，就是在复制模块中实现 “每执行 serverCron 函数 N 次就执行一次指定代码” 的功能。

## 3. 初始化服务器

### 3.1 初始化服务器状态结构

初始化服务器的第一步就是创建一个 `struct redisServer` 类型的实例变量 server 作为服务器的状态，并为结构中的各个属性设置默认值。初始化 server 变量的工作由 `redis.c/initServerConfig` 函数完成。

`initServerConfig` 函数中，大部分是对 server 的属性设置默认值，还有一部分是调用 `populateCommandTable` 函数对 Redis 的命令表初始化。全局变量 `redisCommandTable` 是 `redisCommand` 类型的数组，保存 Redis 支持的所有命令。`server.commands` 是一个 `dict`，保存命令名到 `redisCommand` 的映射。`populateCommandTable` 函数会遍历全局 `redisCommandTable` 表，把每条命令插入到 `server.commands` 中，根据每个命令的属性设置其 `flags`。以下是这个函数的部分代码： 

```c
void initServerConfig(void) {

    // 设置服务器的运行 id 
    getRandomHexChars(server.runid,REDIS_RUN_ID_SIZE);
    
    // 为运行 id 加上结尾字符
    server.runid[REDIS_RUN_ID_SIZE] = '\0';
    
    // 设置默认配置文件路径
    server.configfile = NULL;
    
    // 设置默认服务器频率
    server.hz = REDIS_DEFAULT_HZ;
    
    // 设置服务器的运行架构
    server.arch_bits = (sizeof(long) == 8) ? 64 : 32;
    
    // 设置默认服务器端口号
    server.port = REDIS_SERVERPORT;
    
    // ...

　　/* Command table -- we initiialize it here as it is part of the
　　* initial configuration, since command names may be changed via
　　* redis.conf using the rename-command directive. */
　　// 初始化命令表
　　// 在这里初始化是因为接下来读取 .conf 文件时可能会用到这些命令
　　server.commands = dictCreate(&commandTableDictType,NULL);
　　server.orig_commands = dictCreate(&commandTableDictType,NULL);
　　populateCommandTable();
　　server.delCommand = lookupCommandByCString("del");
　　server.multiCommand = lookupCommandByCString("multi");
　　server.lpushCommand = lookupCommandByCString("lpush");
　　server.lpopCommand = lookupCommandByCString("lpop");
　　server.rpopCommand = lookupCommandByCString("rpop");

　　// ...
　　
}
```

以下是 `initServerConfig` 函数完成的主要工作：

- 设置服务器的运行 ID。
- 设置服务器的默认运行频率。
- 设置服务器的默认配置文件路径。
- 设置服务器的运行架构。
- 设置服务器的默认端口号。
- 设置服务器的默认 RDB 持久化条件和 AOF 持久化条件。
- 初始化服务器的 LRU 时钟。
- 创建命令表。

`initServerConfig` 函数设置的服务器状态属性基本都是一些整数、浮点数、或者字符串属性，除了命令表之外，`initServerConfig` 函数没有创建服务器状态的其他数据结构，数据库、慢查询日志、Lua 环境、共享对象这些数据结构在之后的步骤才会被创建出来。

### 3.2 载入配置选项

在启动服务器时，用户可以通过给定配置参数或者指定配置文件来修改服务器的默认配置。服务器在用 `initServerConfig` 函数初始化完 server 变量之后，就会开始载入用户给定的配置参数和配置文件，并根据用户设定的配置，对 server 变量相关属性的值进行修改。

- 如果用户为这些属性的相应选项指定了新的值，那么服务器就使用用户指定的值来更新相应的属性。
- 如果用户没有为属性的相应选项设置新的值，那么服务器就沿用之前 `initServerConfig` 函数为属性设置的默认值。

### 3.3 初始化服务器数据结构

在之前执行 `initServerConfig` 函数初始化 server 状态时，程序只创建了命令表一个数据结构，不过除了命令表之外，服务器状态还包含其他数据结构，比如：

- `server.clients` 链表，这个链表记录了所有与服务器相连的客户端的状态结构，链表的每一个节点都包含一个 `redisClient` 结构实例。
- `server.db` 数组，数组中包含了服务器的所有数据库。
- 用于保存频道订阅信息的 `server.pubsub_channels` 字典，以及用于保存模式订阅信息的 `server.pubsub_patterns` 链表。
- 用于执行 Lua 脚本的 Lua 环境 `server.lua`。
- 用于保存慢查询日志的 `server.slowlog` 属性。

当初始化服务器进行到这一步，服务器将调用initServer函数，为以上提到的数据结构分配内存，并在有需要时，为这些数据结构设置或关联初始化值。

服务器到现在才初始化数据结构的原因在于，服务器必须先载入用户指定的配置选项，然后才能正确地对数据结构进行初始化。如果在执行 `initServerConfig` 函数时就对数据结构进行初始化，那么一旦用户通过配置选项修改了和数据结构有关的服务器状态属性，服务器就要重新调整和修改已创建的数据结构。为了避免出现这种麻烦的情况，服务器选择了将 server 状态的初始化分为两步进行，`initServerConfig` 函数主要负责初始化一般属性，而 `initServer` 函数主要负责初始化数据结构。

除了初始化数据结构之外，`initServer` 还进行了一些非常重要的设置操作：

- 为服务器设置进程信号处理器。
- 创建共享对象，这些对象包含 Redis 服务器经常用到的一些值，服务器通过重用这些共享对象来避免反复创建相同的对象。
- 打开服务器的监听端口，并为监听套接字关联连接应答事件处理器，等待服务器正式运行时接受客户端的连接。
- 为 `serverCron` 函数创建时间事件，等待服务器正式运行时执行 `serverCron` 函数。
- 如果 AOF 持久化功能已经打开，那么打开现有的 AOF 文件，如果 AOF 文件不存在，那么创建并打开一个新的 AOF 文件，为 AOF 写入做好准备。
- 初始化服务器后台 I/O 模块（bio），为将来的 I/O 操作做准备。

当 `initServer` 函数执行完毕之后，服务器将用 ASCII 字符在日志中打印出 Redis 的图标，以及 Redis 的版本号信息。

### 3.4 还原数据库状态

在完成了对服务器状态 server 变量的初始化之后，服务器需要载入 RDB 文件或者 AOF 文件，并根据文件记录的内容来还原服务器的数据库状态。

根据服务器是否启用了 AOF 持久化功能，服务器载入数据时所使用的目标文件会有所不同：

- 如果服务器启用了 AOF 持久化功能，那么服务器使用 AOF 文件来还原数据库状态；
- 相反地，如果服务器没有启用 AOF 持久化功能，那么服务器使用 RDB 文件来还原数据库状态。

当服务器完成数据库状态还原工作之后，服务器将在日志中打印出载入文件并还原数据库状态所耗费的时长：

```
[5244] 21 Nov 22:43:49.084 * DB loaded from disk: 0.068 seconds
```

### 3.5 执行事件循环

在初始化的最后一步，服务器将打印出以下日志：

```
[5244] 21 Nov 22:43:49.084 * The server is now ready to accept connections on port 6379
```

并开始执行服务器的事件循环。

至此，服务器的初始化工作圆满完成，服务器现在开始可以接受客户端的连接请求，并处理客户端发来的命令请求了。