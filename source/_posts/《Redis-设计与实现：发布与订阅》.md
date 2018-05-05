---
title: 《Redis 设计与实现：发布与订阅》
date: 2017-10-16 19:13:50
tags: [Redis, Database]
categories: [读书笔记, Redis]
---

Redis 的发布与订阅功能由 `PUBLISH`、`SUBSCRIBE`、`PSUBSCRIBE` 等命令组成。

通过执行 `SUBSCRIBE` 命令，客户端可以订阅一个或多个频道从而成为这些频道的订阅者（subscriber）：每当有其它客户端向被订阅的频道发送消息（message）时，频道的所有订阅者都会收到这条消息。

除了订阅频道之外，客户端还可以通过执行 `PSUBSCRIBE` 命令订阅一个或多个模式，从而成为这些模式的订阅者：每当有其他客户端向某个频道发送消息时，消息不仅会被发送给这个频道的所有订阅者，它还会被发送给所有与这个频道相匹配的模式的订阅者。

<!--more-->

## 1. 频道的订阅与退订

当一个客户端执行 `SUBSCRIBE` 命令订阅某个或某些频道的时候，这个客户端与被订阅频道之间就建立起了一种订阅关系。

Redis 将所有频道的订阅关系都保存在服务器状态的 `pubsub_channels` 字典里面，这个字典的键是某个被订阅的频道，而键的值是一个链表，链表里面记录了所有订阅这个频道的客户端。

```
struct redisServer {
 
	// ...

	// 保存所有频道的订阅关系
	dict *pubsub_channels;  /* Map channels to list of subscribed clients */
 
	// ...
 
};
```

### 1.1 订阅频道

每当客户端执行 `SUBSCRIBE` 命令订阅某个或某些频道的时候，服务器都会将客户端与被订阅的频道在 `pubsub_channels` 字典中进行关联。

根据频道是否已经有其他订阅者，关联操作分为两种情况执行：

1. 如果频道已经有其他订阅者，那么它在 `pubsub_channels` 字典中必然有相应的订阅者链表，程序唯一要做的就是将客户端添加到订阅者链表的末尾。
2. 如果频道还未有任何订阅者，那么它必然不存在于 `pubsub_channels` 字典，程序首先要在 `pubsub_channels` 字典中为频道创建一个键，并将这个键的值设置为空链表，然后再将客户端添加到链表，成为链表的第一个元素。

`SUBSCRIBE` 命令的实现可以⽤以下伪代码来描述：

```
def subscribe(*all_input_channels):

	# 遍历输⼊的所有频道
	for channel in all_input_channels:
		
		# 如果 channel 不存在于 pubsub_channels 字典（没有任何订阅者）
		# 那么在字典中添加 channel 键，并设置它的值为空链表
		if channel not in server.pubsub_channels:
			server.pubsub_channels[channel] = []
		
		# 将订阅者添加到频道所对应的链表的末尾
		server.pubsub_channels[channel].append(client)
```

### 1.2 退订频道

`UNSUBSCRIBE` 命令的⾏为和 `SUBSCRIBE` 命令的⾏为正好相反 —— 当⼀个客户端退订某个或某些频道的时候，服务器将从 `pubsub_channels` 中解除客户端与被退订频道之间的关联：

- 程序会根据被退订频道的名字，在 `pubsub_channels` 字典中找到频道对应的订阅者链表，然后从订阅者链表中删除退订客户端的信息。
- 如果删除退订客户端之后，频道的订阅者链表变成了空链表，那么说明这个频道已经没有任何订阅者了，程序将从 `pubsub_channels` 字典中删除频道对应的键。

`UNSUBSCRIBE` 命令的实现可以⽤以下伪代码来描述：

```
def unsubscribe(*all_input_channels):

	# 遍历要退订的所有频道
	for channel in all_input_channels:
	
		# 在订阅者链表中删除退订的客户端
		server.pubsub_channels[channel].remove(client)
		
		# 如果频道已经没有任何订阅者了（订阅者链表为空）
		# 那么将频道从字典中删除
		
		if len(server.pubsub_channels[channel]) == 0:
			server.pubsub_channels.remove(channel)
```

## 2. 模式的订阅与退订

服务器也将所有模式的订阅关系都保存在服务器状态的 `pubsub_patterns` 属性里面：

```
struct redisServer {
    
	// ...
    
	// 保存所有模式订阅关系
	list *pubsub_patterns;
    
	// ...
	
}
```

`pubsub_patterns` 属性是一个链表，链表中的每个节点都包含着一个 `pubsubPattern` 结构，这个结构的 `pattern` 属性记录了被订阅的模式，而 `client `属性则记录了订阅模式的客户端：

```
typedef struct pubsubPattern {

	// 订阅模式的客户端
	redisClient *client;
    
	// 被订阅的模式
	robj * pattern;
    
} pubsubPattern;
```

### 2.1 订阅模式

每当客户端执行 `PSUBSCRIBE` 命令订阅某个或某些模式的时候，服务器会对每个被订阅的模式执行以下两个操作：

1. 新建一个 `pubsubPattern` 结构，将结构的 `pattern` 属性设置为被订阅的模式，`client` 属性设置为订阅模式的客户端。
2. 将 `pubsubPattern` 结构添加到 `pubsub_patterns` 链表的表尾。

`PSBUSCRIBE` 命令的实现原理可以用以下的伪代码来描述：

```
def psubscribe(*all_input_patterns):
	
	# 遍历输入的所有模式
	for pattern in all_input_patterns:
		
		# 创建新的 pubsubPattern 结构
		# 记录被订阅的模式，以及订阅模式的客户端
		pubsubPattern = create_new_pubsubPattern()
		pubsubPattern.client = client
		pubsubPattern.pattern = pattern
		
		# 将新的 pubsubPattern 追加到 pubsub_patterns 链表末尾
		server.pubsub_patterns.append(pubsubPattern)
```

### 2.2 退订模式

模式的退订命令 `PUNSUBSCRIBE` 是 `PSUBSCRIBE` 命令的反操作。当一个客户端退订某个或某些模式的时候，服务器将在 `pubsub_patterns` 链表中查找并删除那些 `pattern` 属性为退订模式并且 `client` 属性为执行退订命令的客户端的 `pubsubPattern` 结构。

`PUNSUBSCRIBE` 命令的实现原理可以用以下伪代码来描述：

```
def punsubscribe(*all_input_patterns):
	
	# 遍历所有要退订的模式
	for pattern in all_input_patterns:
		
		# 遍历 pubsub_patterns 链表中的所有 pubsubPattern 结构
		for pubsubPattern in server.pubsub_patterns:
		
			# 如果当前客户端和 pubsubPattern 记录的客户端相同
			# 并且要退订的模式也和 pubsubPattern 记录的模式相同
			if client == pubsubPattern.client and pattern == pubsubPattern.pattern
			
			# 那么将这个 pubsubPattern 从链表中删除
			server.pubsub_patterns.remove(pubsubPattern)
```

## 3. 发送消息

当一个 Redis 客户端执行 `PUBLISH <channel> <message>` 命令将消息 `message` 发送给频道 `channel `的时候，服务器需要执行以下两个操作：

1. 将消息 `message` 发送给频道 `channel` 的所有订阅者。
2. 如果有一个或多个模式 `pattern` 与频道 `channel` 相匹配。那么将消息 `message` 发送给 `pattern` 模式的订阅者。

### 3.1 将消息发送给频道订阅者

因为服务器状态中的 `pubsub_channels` 字典记录了所有频道的订阅关系，所以为了将消息发送给 `channel` 频道的所有订阅者，`PUBLISH` 命令要做的就是在 `pubsub_channels` 字典里找到频道 `channel` 的订阅者名单（一个链表），然后将消息发送给名单上的所有客户端。

`PUBLISH` 命令将消息发送给频道订阅者的方法可以用以下伪代码来描述：

```
def channel_publish(channel,message):

	# 如果 channel 键不存在于 pubsub_channels 字典中
	# 那么说明 channel 频道没有任何订阅者
	# 程序不做发送动作，直接返回
	if channel not in server.pubsub_channels:
		return
    
	# 运行到这里，说明 channel 频道至少有一个订阅者
	# 程序遍历 channel 频道的订阅者列表
	# 将消息发送给所有订阅者
	for subscriber in server.pubsub_channels[channel]:
		send_message(subscriber, message)
```

### 3.2 将消息发送给模式订阅者

因为服务器状态中的 `pubsub_patterns `链表记录了所有模式的订阅关系，所以为了将消息发送给所有与 `channel` 频道相匹配的模式的订阅者，`PUBLISH` 命令要做的就是遍历整个 `pubsub_patterns` 链表，查找那些与 `channel` 频道相匹配的模式，并将消息发送给订阅了这些模式的客户端。

`PUBLISH` 命令将消息发送给模式订阅者的方法可以用以下伪代码来描述：

```
def pattern_publish(channel, message):
    
	# 遍历所有模式订阅消息
	for pubsubPattern in server.pubsub_patterns:
    
		# 如果频道和模式相匹配
		if(match(channel, pubsubPatter.pattern)):
        
		# 那么将消息发送给订阅该模式的客户端
			send_message(pubsubPattern.client, message)
```

最后，`PUBLISH` 命令的实现可以用以下伪代码来描述：

```
def publish(channel, message):

    # 将消息发送给 channel 频道的所有订阅者
    channel_publish(channel, message)
    
    # 将消息发送给所有和 channel 频道相匹配的模式的订阅者
    pattern_publish(channel, message)
```

## 4. 查看订阅信息

### 4.1 PUBSUB CHANNELS

`PUBSUB CHANNELS [pattern]` 子命令用于返回服务器当前被订阅的频道。其中 `pattern` 参数是可选的：

- 如果不给定 `pattern` 参数，那么命令返回服务器当前被订阅的所有频道。
- 如果给定 `pattern` 参数，那么命令返回服务器当前被订阅的频道中那些与 `pattern` 模式相匹配的频道。

这个子命令是通过遍历服务器 `pubsub_channels` 字典中的所有键（每个键都是一个被订阅的频道），然后记录并返回所有符合条件的频道来实现的，这个过程可以用以下伪代码来描述：

```
def pubsub_channels(pattern=None):

	# 一个列表，用于记录所有符合条件的频道
	channel_list = []
	
	# 遍历服务器中的所有频道
	# (也即是 pubsub_channels 字典的所有键)
	for channel in server.pubsub_channels:
	
		# 当以下两个条件的任意一个满足时，将频道添加到链表里面：
		# 1）用户没有指定 pattern 参数
		# 2）用户指定了 pattern 参数，并且 channel 和 pattern 匹配
		if (pattern is None) or match(channel, pattern):
			channel_list.append(channel)
	
	# 向客户端返回频道列表
	return channel_list
```

### 4.2 PUBSUB NUMSUB

`PUBSUB NUMSUB [channel1-1 channel-2 ... channel-n]` 子命令接受任意多个频道作为输入参数，并返回这些频道的订阅者数量。这个命令是通过在 `pubsub_channels` 字典中找到频道对应的订阅者链表，这个过程可以用以下伪代码来描述：

```
def pubsub_numsub(*all_input_channels):

	# 遍历输入的所有频道
	for channel in all_input_channels:
	
		# 如果 pubsub_channels 字典中没有 channel 这个键
		# 那么说明 channel 频道没有任何订阅者
		if channel not in server.pubsub_channels:
			# 返回频道名
			reply_channel_name(channel)
			# 订阅者数量为 0
			reply_subscribe_count(0)
			
		# 如果 pubsub_channels 字典中存在 channel 键
		# 那么说明 channel 频道至少有一个订阅者
		else:
			# 返回频道名
			reply_channel_name(channel)
			# 订阅者数量为0
			reply_subscribe_count(len(server.pubsub_channels[channel]))
```

### 4.3 PUBSUB NUMPAT

`PUBSUB NUMPAT` 命令用于返回服务器当前被订阅模式的数量。

这个命令是通过返回 `pubsub_patterns` 链表的长度来实现的，因为这个链表的长度就是服务器被订阅模式的数量，这个过程可以用以下伪代码来描述：

```
def pubsub_numpat():
	
	# pubsub_patterns 链表的长度就是被订阅模式的数量
	reply_pattern_count(len(server.pubsub_patterns))
```

