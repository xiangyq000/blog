---
title: 'Redis: 复制'
date: 2017-03-24 17:43:45
tags: [Redis, Database]
categories: Redis
---

### Redis复制的启动过程

| 步骤   | 主服务器操作                                   | 从服务器操作                                   |
| ---- | ---------------------------------------- | ---------------------------------------- |
| 1    | 等待命令进入                                   | 连接（或者重连接）主服务器，发送 SYNC 命令                 |
| 2    | 开始执行 BGSAVE，并使用缓冲区记录 BGSAVE 之后执行的所有写命令   | 根据配置选项来决定是继续使用现有的数据（如果有的话）来处理客户端的命令请求，还是向发送请求的客户端返回错误 |
| 3    | BGSAVE 执行完毕，向从服务器发送快照文件并在发送期间继续使用缓冲区记录被执行的写命令 | 丢弃所有的旧数据（如果有的话），开始载入主服务器发来的快照文件          |
| 4    | 快照文件发送完毕，开始向服务器发送存储在缓冲区里的写命令             | 完成对快照文件的解释操作，像往常一样开始接受命令请求               |
| 5    | 缓冲区存储的写命令发送完毕；现在开始每执行一个写命令，就向从服务器发送相同的写命令 | 执行主服务发来的所有存储在缓冲区里面的写命令；并从现在开始，接收并执行主服务器传来的每一个写命令 |

<!--more-->

如果用户使用的是SLAVEOF配置选项，那么Redis在启动时首先会载入当前可用的任何快照文件或者AOF文件，然后执行上面表中的复制过程，如果如果用户使用的是SLAVEOF命令，那么Redis会立即尝试连接主服务器，并在连接成功之后，开始上表所示的复制过程。

当多个从服务器连接主服务器的时候，就会出现下表两种情况中的其中一种。

| 当新的从服务器连接主服务器时        | 主服务器的操作                                  |
| --------------------- | ---------------------------------------- |
| 上表中的步骤 3 尚未执行         | 所有的从服务器都会接收到相同的快照文件或者相同的缓冲区写命令           |
| 上表中的步骤 3 正在执行或者已经执行完毕 | 当主服务器与较早的进行的连接的从服务器执行完复制所需的 5 个步骤之后，主服务器会与新连接的从服务器执行一次新的步骤 1 至步骤 5 |

### 主从链
创建多个从服务器可能造成网络不可用——当复制需要通过互联网进行或者需要在不同数据中心之间进行时，尤为如此。因为Redis的主服务器和从服务器并没有什么特别不同的地方，所有从服务器可以拥有自己的从服务器，从而形成主从链(master/slave chaining)。

从服务器对从服务器进行复制的操作上和从服务器对主服务器进行复制的唯一区别在于，如果从服务器X拥有从服务器Y,那么当从服务器X在执行复制过程中的步骤4的时候,他将断开与从服务器Y的连接，导致从服务器Y需要重新连接并重新同步。

当读请求的重要性明显高于写请求的重要性，并且当读请求的数量远远超出一台Redis服务器可以处理的范围时，用户就可以添加新的从服务器来处理读请求。随着负载的不断上升，主服务器可能会无法快速的更新所有从服务器，或者因为重新连接或者重新同步从服务器而导致系统超载。为了解决这个问题，用户可以创建一个由Redis主从节点(master/slave node)组成的中间层来分担主服务器的复制工作。

AOF持久化的同步选项可以控制数据丢失的时间长度：通过将每个写命令同步到硬盘里面，用户几乎可以不损失任何数据(除非系统崩溃或者硬盘驱动器损坏)，但这种做法会对服务器的性能造成影响；另一方面，如果用户将同步的频率设置为每秒一次，那么服务器的性能将会回到正常水平，但故障可能会造成1秒的数据丢失，通过同时使用复制和AOF持久化，我们可以将数据持久化到多台服务器上面。

为了将数据保存到多台机器上面，用户需要为主服务器设置多个从服务器，然后对每个从服务器设置appendonly yes选项和appendfsync everysecz选项(如果有需要的话，也可以对主服务器进行相同的设置)这样的话，用户就可以让多台服务器以每秒一次的频率将数据同步到硬盘上，但只是第一步：因为用户还必须等待主服务器发送的写命令到达从服务器，并且在执行后续操作之前，检查 数据是否被同步到了硬盘里面。

### 更换故障主服务器
假设A、B两台服务器都运行着Redis,其中A为主服务器，B为从服务器，如果此时机器A因为某个暂时无法修复的故障断开了网络连接，决定使用安装了Redis的机器C作为新的主服务器。

更换主服务器的计划很简单：

- 在机器B上执行SAVE命令，在机器B上生成快照文件；
- 将快照文件发送到机器C上，并启动机器C；
- 将机器B设置为机器C的从服务器。