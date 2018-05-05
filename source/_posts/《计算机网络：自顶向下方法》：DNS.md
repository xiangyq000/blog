---
title: 《计算机网络：自顶向下方法》：DNS
date: 2016-09-24 12:09:57
tags: DNS
categories: [Network，读书笔记]
---
## 1. DNS 是什么

DNS（Domain Name System）是

- 一个由分层的 DNS 服务器实现的分布式数据库
- 一个使得主机能够查询分布式数据库的应用层协议
- 运行在 UDP 上，使用 53 号端口

<!--more-->



## 2. DNS 提供的服务

- 域名解析：将主机名（Host）解析为 IP 地址。
- 主机别名：有着复杂主机名的主机能够拥有一个或者多个别名，应用程序可以调用 DNS 来获得主机别名对应的规范主机名以及主机的 IP 地址。
- 邮件服务器别名：电子邮件应用程序可以调用 DNS，对提供的邮件服务器别名进行解析，以获得该主机的规范主机名机器 IP 地址。
- 负载分配：DNS 也用于在冗余的服务器之间进行负载分配。一些繁忙的站点被冗余分布在多台服务器上，每台服务器均运行在不同的端系统上，有着不同的 IP 地址。所以在这种情况下，一个 IP 地址集合与一个规范主机名关联。DNS 数据库存储着这些 IP 地址集合。当客户对该主机名进行 DNS 请求时，服务器用 IP 地址的整个集合进行响应，但在每个回答中循环这些地址次序。因为客户通常总是与 IP 地址排在最前面的服务器建立连接，所以 DNS 就在所有这些冗余服务器之间循环分配了负载。

## 3. 简要工作流程

用浏览器描述 DNS 工作过程：
1. 同一台主机运行着 DNS 应用的客户端。
2. 浏览器从 URL 中抽取出主机名，传给 DNS 应用的客户端。
3. DNS 客户机端向 DNS 服务器发送一个包含主机名的 DNS 请求。
4. DNS 服务器最终将返回一个（或一个集合）该主机名对应的主机 IP 地址。
5. 浏览器向位于该 IP 地址 80 端口的 HTTP 服务器进程发起一个 TCP 连接。

## 4. DNS 服务器的层次结构
DNS 使用了大量的 DNS 服务器，它们以层次方式组织，并分布在全世界范围内。大致来说，有 3 种类型的 DNS 服务器：根 DNS 服务器、顶级域（Top-Level Domain，TLD）DNS 服务器和权威 DNS 服务器。它们以下图的层次结构组织：
![image](https://user-images.githubusercontent.com/12514722/30777329-bbcb4b04-a0ea-11e7-92e0-6b586f378118.png)

- 根 DNS 服务器。在因特网上有 13 个根 DNS 服务器（编号为 A 到 M），它们中的大部分位于北美洲。每台“服务器”实际上是一个冗余服务器的网络，以提供安全性和可靠性。根 DNS 服务器用来返回 TLD 服务器的 IP 地址。
- 顶级域名服务器（top level domain，简称 TLD)。这些服务器负责顶级域名如 com、org、net、edu 和 gov，以及所有国家的顶级域名如 uk、fr 等。TLD 服务器返回权威服务器的 IP 地址。
- 权威 DNS 服务器。每一个在 internet 中的有公共可访问主机的组织或机构，必须提供公共可访问的 DNS 记录，并将之存放在权威 DNS 服务器中。权威 DNS 服务器可以是本组织的服务器也可以租用其他组织的服务器。它用来返回主机的 IP 地址。权威 DNS 存在的理由：每一个机构都有很多的主机，这些主机可能位于不同的地理位置，有不同的 IP 地址，但他们可能有相同的域名，如果这些信息全部由顶级域名服务器进行管理，工作量太大。
- 本地 DNS 服务器：根、TLD和权威 DNS 服务器都处在 DNS 服务器的层次结构中，还有一类重要的 DNS，称为本地 DNS 服务器。一个本地 DNS 服务器严格来说不属于该层次结构，但它却是很重要的。每个 ISP 都有一台本地 DNS 服务器。本地 DNS 服务器起着代理的作用，本地主机将 DNS 请求发向本地 DNS 服务器，本地 DNS 服务器将该请求转发到 DNS 服务器层次结构中。

## 5. DNS 解析流程

### 5.1 浏览器访问域名时的前置步骤
若用户通过浏览器访问某站点，此时存在两个步骤：

1. 浏览器检查缓存中是否有该域名对应的解析过的 IP 地址，若命中则解析过程结束；否则进行步骤 2。
2. 浏览器检查操作系统缓存中是否有该域名对应的 DNS 解析结果，若命中则解析过程结束；否则进行后续步骤。

### 5.2 DNS 在服务器之间的解析步骤
下图例子假设主机 cs.ustc.edu 想知道主机 cs.csu.edu 的 IP 地址，假设 USTC 大学的本地 DNS 服务器为 dns.ustc.edu，同时假设 CSU 大学的权威 DNS 服务器为 dns.csu.edu。

1. 主机 cs.ustc.edu 首先向它的本地 DNS 服务器 dns.ustc.edu 发送一个 DNS 查询报文。该查询报文含有被转换的主机名 cs.csu.edu。 
2. 本地 DNS 服务器将该查询报文发送到根 DNS 服务器，根 DNS 服务器注意到 edu 的前缀，所以将负责 edu 的 TLD 的 IP 地址列表返回给本地 DNS 服务器。
3. 本地 DNS 服务器再次向这些 TLD 服务器之一发送 DNS 查询报文，该 TLD 服务器注意到 csu.edu 的前缀，所以将权威服务器 dns.cs.edu 的 IP 地址返回给本地 DNS 服务器。 
4. 本地 DNS 服务器向权威服务器 dns.cs.edu 发送查询报文，权威服务器用 cs.csu.edu 的 IP 地址进行响应。
5. 最后，本地 DNS 服务器将查询得到的 IP 地址返回给主机 cs.ustc.edu。

![image](https://user-images.githubusercontent.com/12514722/30777348-098fa682-a0eb-11e7-8e16-eb403812d943.png)

DNS 查询也可以一种递归的方式进行：
![image](https://user-images.githubusercontent.com/12514722/30777363-55c71e5e-a0eb-11e7-891d-e9c07d55123b.png)

### 5.3 DNS 缓存
如果每次 DNS 解析都要走完上面介绍的整个流程，就会带来网络带宽的消耗和时延，这对于用户和 DNS 解析系统都是不友好的。所以当本地 DNS 服务器在完成一次查询后就会将得到的主机名到 IP 地址的映射缓存到本地，从而加快 DNS 的解析速度。实际上，解析大多数都是在本地服务器上完成的。由于主机名和 IP 地址之间的映射不是永久的，DNS 服务器在一段时间后，通常是两天，就会丢弃缓存信息。

大部分的 DNS 洪泛攻击可以由本地 DNS 缓存缓解。

在 Linux 下可以通过`/etc/init.d/nscd restart`来清除缓存。

### 5.4 JVM 中的 DNS 缓存
Java 应用中的 JVM 也会缓存 DNS 的解析结果，这个缓存是在 `InetAddress` 类中，缓存时间较特殊，两种缓存策略：

- 缓存正确结果
- 缓存失败的结果

两个缓存时间由两个配置项来控制，配置项是在 `%JAVA_HOME%\lib\security\java.security` 文件中配置的。

两个配置项分别是 `networkaddress.cache.ttl` 和 `networkaddress.cache.negtive.ttl` ，其默认值分别是 -1（永不失效）和 10（保留 10 秒钟）。

修改方式：

- 直接修改 `java.secury` 文件的默认值
- 在 JAVA 启动时加启动参数 `-Dsun.NET.inetaddress.ttl=XXX` 来修改默认值。

如果要用 `InetAddress` 类来解析域名时，一定要采用单例模式，否则会有严重的性能问题，每次都要创建新的类，都要进行完整的域名解析过程。

## 6. DNS 记录
共同实现 DNS 分布式数据库的所有 DNS 服务器存储了资源记录（Resource Record，RR），资源记录提供了主机名到 IP 地址的映射。每个 DNS 回答报文包含了一条或多条资源记录。资源记录是一个包含了下列字段的四元组：

```
（Name，Value，Type，TTL）
```
TTL 是该记录的生存时间，它决定了资源记录应当从缓存中删除的时间。如果域名解析改动较频繁，比如使用动态 IP 等，就应该把 TTL 尽量设小； 如果域名解析不是经常改动，一般可将 TTL 适当设置得大一点，以加快主机的访问速度。Name 和 Value 的值取决于 Type。

- 如果 Type=A，则 Name 是主机名，Value 是该主机对应的 IP 地址。即，一条 A 记录提供了标准的主机名到 IP 地址的映射。
- 如果 TYpe=NS，则 Name 是个域，比如 csu.edu，而 Value 是知道如何获得该域中主机 IP 地址的权威 DNS 服务器的主机名，比如 dns.csu.edu。这个记录用于沿着查询链来路由 DNS 查询。
- 如果 Type=CNAME，则 Value 是别名为 Name 的主机对应的规范主机名。该记录能够向查询的主机提供一个主机名对应的规范主机名。
- 如果 Type=MX，则 Value 是别名为 Name 的邮件服务器的规范主机名。MX 记录允许邮件服务器主机名具有简单的别名。事实上，MX 记录允许一个公司的邮件服务器和 Web 服务器使用相同（别名化）的主机名。为了获得邮件服务器的规范主机名，DNS 客户端应当请求一条 MX 记录；而为了获得其他服务器的规范主机名，DNS 客户应当请求 CNAME 记录。

如果一台 DNS 服务器是用于某特定主机名的权威 DNS 服务器，那么该 DNS 服务器会有一条包含该主机名的类型 A 记录（即使该 DNS 服务器不是其权威 DNS 服务器，它也可能在缓存中包含有一条类型 A 记录）。如果服务器不是用于某主机名的权威服务器，那么该服务器将包含一条类型 NS 记录，该记录对应于包含主机名的域；它还将包括一条类型 A 记录，该记录提供了在 NS 记录的 Value 字段中的 DNS 服务器的 IP 地址。举例来说，假设一台 edu TLD 服务器不是主机 gaia.cs.umass.edu 的权威 DNS 服务器，则该服务器将包含一条包括主机 cs.umass.edu 的域记录，如（umass.edu，dns.umass.edu，NS）；该 edu TLD 服务器还将包含一条类型 A 记录，如（dns.umass.edu，128.119.40.111，A），该记录将名字 dns.umass.edu 映射为一个 IP 地址。

注册一个全新的域名最少要向对应的 TLD 注入 A 型与 NS 型两种记录。

## 7. DNS 报文
![image](https://user-images.githubusercontent.com/12514722/30777355-2974e30e-a0eb-11e7-8437-2f4f5d46a5bf.png)

- DNS 报文格式前 12 个字节是首部区域，其中有几个字段。第一个字段（标识符）是一个 16 比特的数，用于标识该查询。这个标识符会被复制到对查询的回答报文中，以便让客户用它来匹配发送的请求和接收到的回答。标志字段中含有若干标志。1 比特的“查询/回答”标志位指出报文是查询报文（0）还是回答报文（1）。当某 DNS 服务器是所请求名字的权威 DNS 服务器时，1 比特的“权威的”标志位被置在回答报文中。如果客户（主机或者DNS 服务器）在该 DNS 服务器没有某记录时希望它执行递归查询，将设置 1 比特的“希望递归”标志位。如果该 DNS 服务器支持递归查询，在它的回答报文中会对 1 比特的“递归可用”标志位置位。在该首部中，还有 4 个有关数量的字段，这些字段指出了在首部后的 4 类数据区域出现的数量。
- 问题区域包含着正在进行的查询信息。该区域包括：①名字字段，指出正在被查询的主机名字；②类型字段，它指出有关该名字的正被询问的问题类型，例如主机地址是与一个名字相关联（类型 A）还是与某个名字的邮件服务器相关联（类型 MX）。
- 在来自 DNS 服务器的回答中，回答区域包含了对最初请求的名字的资源记录。前面讲过每个资源记录中有 Type（如 A、NS、CNAME 和 MX）字段、Value 字段和 TTL 字段。在回答报文的回答区域中可以包含多条 RR，因此一个主机名能够有多个 IP 地址（例如，就像本节前面讨论的冗余 Web 服务器）。
- 权威区域包含了其他权威服务器的记录。
- 附加区域包含了其他有帮助的记录。例如，对于一个 MX 请求的回答报文的回答区域包含了一条资源记录，该记录提供了邮件服务器的规范主机名。该附加区域包含一个类型 A 记录，该记录提供了用于该邮件服务器的规范主机名的 IP 地址。

## 8. DNS 轮询
### 8.1 原理

大多数域名注册商都支持在 DNS 服务器中为同一个域名配置多个 IP 地址（即为一个主机名设置多条A资源记录）。在应答 DNS 查询时，DNS 服务器对每个查询返回所有轮询主机服务器 IP，其顺序取决于 DNS 服务器配置，以此将客户端的访问引导到不同的 IP 上去，从而达到负载均衡的目的。

DNS 服务器一般基于 BIND（Berkeley Internet Name Domain）实现。BIND 将根据 `rrset-order` 语句定义的次序把配置中设定的所有A记录都发送给客户端，客户端可以使用自己规定的算法从记录中挑选一条。`rrset-order` 语句是主配置文件中 `options` 主语句的一条子语句，可以定义固定、随机和轮询的次序。`order_spec` 定义：

```
[ class class_name ][ type type_name ][ name "domain_name"] order ordering
```

如果没有设定类，默认值为 `ANY`。如果没有设定类型，默认值为 `ANY`。如果没有设定名称，默认值为 "*"。合法的排序参数包括：

- `fixed`：记录以它们在域文件中的顺序
- `random`：记录以随机顺序被返回
- `cyclic`：记录以环顺序被返回

### 8.2 实验
`dig` 三次 note.youdao.com：

```shell
$ dig note.youdao.com

; <<>> DiG 9.8.3-P1 <<>> note.youdao.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38905
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;note.youdao.com.		IN	A

;; ANSWER SECTION:
note.youdao.com.	1069	IN	A	59.111.179.138
note.youdao.com.	1069	IN	A	59.111.179.135
note.youdao.com.	1069	IN	A	59.111.179.136
note.youdao.com.	1069	IN	A	59.111.179.137

;; Query time: 23 msec
;; SERVER: 10.238.14.2#53(10.238.14.2)
;; WHEN: Fri Aug 18 16:17:26 2017
;; MSG SIZE  rcvd: 97
```
```
$ dig note.youdao.com

; <<>> DiG 9.8.3-P1 <<>> note.youdao.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33819
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;note.youdao.com.		IN	A

;; ANSWER SECTION:
note.youdao.com.	1023	IN	A	59.111.179.135
note.youdao.com.	1023	IN	A	59.111.179.136
note.youdao.com.	1023	IN	A	59.111.179.137
note.youdao.com.	1023	IN	A	59.111.179.138

;; Query time: 2 msec
;; SERVER: 10.238.14.2#53(10.238.14.2)
;; WHEN: Fri Aug 18 16:18:11 2017
;; MSG SIZE  rcvd: 97
```

```shell
$ dig note.youdao.com

; <<>> DiG 9.8.3-P1 <<>> note.youdao.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3771
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;note.youdao.com.		IN	A

;; ANSWER SECTION:
note.youdao.com.	1112	IN	A	59.111.179.136
note.youdao.com.	1112	IN	A	59.111.179.137
note.youdao.com.	1112	IN	A	59.111.179.138
note.youdao.com.	1112	IN	A	59.111.179.135

;; Query time: 2 msec
;; SERVER: 10.238.14.2#53(10.238.14.2)
;; WHEN: Fri Aug 18 16:36:44 2017
;; MSG SIZE  rcvd: 97
```

可以看到 `ANSWER SECTION` 中 IP 的顺序是不同的，并且可以猜测这里使用了 cyclic 的轮询方式。假定客户端（如浏览器）总是选择第一条记录，则每次对 note.youdao.com 的 HTTP 请求会被映射到不同IP上。

### 8.3 DNS 轮询的用处
- 负载均衡。如果采用 DNS 轮询技术，将一个域名解析到多台服务器的各自的 IP 地址上，利用浏览器的随机访问对流量进行分摊，则会降低每台服务器的压力，实现均衡负载。故障时保证访问是它的另一个重要应用。从之前的原理的简述我们可以看出，轮询时 DNS 服务器会将所有的 IP 返回给浏览器，浏览器自身的机制会使其在连接出错时继续连接下一个 IP，直到所有 IP 都无法连接或者连接成功为止。这样，只要两台服务器不同时宕机，那么我们几乎可以让网站在线率达到将近百分之百。类似地，在更换服务器地址时，使用另一台服务器做 DNS 轮询进行过渡是一个很好的方法。

- CDN 加速。DNS 轮询是 CDN 的基础，通过对于不同访问线路的不同解析，达到最快的访问速度。
- 内外网的不同访问内容与内容的加速访问。

### 8.4 DNS 负载均衡的优点
1. 将负载均衡的工作交给 DNS，省去了网站管理维护负载均衡服务器的麻烦。
2. 技术实现比较灵活、方便，简单易行，成本低，使用于大多数 TCP/IP 应用。
3. 对于部署在服务器上的应用来说不需要进行任何的代码修改即可实现不同机器上的应用访问。
4. 服务器可以位于互联网的任意位置。
5. 同时许多 DNS 还支持基于地理位置的域名解析，即会将域名解析成距离用户地理最近的一个服务器地址，这样就可以加速用户访问，改善性能。

### 8.5 DNS 负载均衡的缺点
1. 目前的 DNS 是多级解析的，每一级 DNS 都可能缓存 A 记录，当某台服务器下线之后，即使修改了 A 记录，要使其生效也需要较长的时间，这段时间，DNS 仍然会将域名解析到已下线的服务器上，最终导致用户访问失败。
2. 不能够按服务器的处理能力来分配负载。DNS 负载均衡采用的是简单的轮询算法，不能区分服务器之间的差异，不能反映服务器当前运行状态，所以其的负载均衡效果并不是太好。
3. 可能会造成额外的网络问题。为了使本 DNS 服务器和其他 DNS 服务器及时交互，保证 DNS 数据及时更新，使地址能随机分配，一般都要将 DNS 的刷新时间设置的较小，但太小将会使 DNS 流量大增造成额外的网络问题。

## 9. 为什么只有 13 台根服务器？
准确说是 “There are 12 organisations maintaining root servers and 13 root server IPs being used”。

[Why There Are Only 13 DNS Root Name Servers
](https://www.lifewire.com/dns-root-name-servers-3971336)
> Because DNS operation relies on potentially millions of other internet servers finding the root servers at any time, the addresses for root servers must be distributable over IP as efficiently as possible. Ideally, all of these IP addresses should fit into a single packet (datagram) to avoid the overhead of sending multiple messages between servers. In IPv4 in widespread use today, the DNS data that can fit inside a single packet is as small as 512 bytes after subtracting all the other protocol supporting information contained in packets. Each IPv4 address requires 32 bytes. Accordingly, the designers of DNS chose 13 as the number of root servers for IPv4, taking 416 bytes of a packet and leaving up to 96 bytes for other supporting data and the flexibility to add a few more DNS root servers in the future if needed.​

## 10. 跟踪 DNS 解析过程
### 10.1 nslookup
```shell
$ nslookup note.youdao.com
Server:		10.238.14.2
Address:	10.238.14.2#53

Non-authoritative answer:
Name:	note.youdao.com
Address: 59.111.179.135
Name:	note.youdao.com
Address: 59.111.179.136
Name:	note.youdao.com
Address: 59.111.179.137
Name:	note.youdao.com
Address: 59.111.179.138
```

### 10.2 dig
略

### 10.3 dig +trace
```shell
$ dig note.youdao.com +trace

; <<>> DiG 9.8.3-P1 <<>> note.youdao.com +trace
;; global options: +cmd
.			79850	IN	NS	m.root-servers.net.
.			79850	IN	NS	a.root-servers.net.
.			79850	IN	NS	g.root-servers.net.
.			79850	IN	NS	l.root-servers.net.
.			79850	IN	NS	e.root-servers.net.
.			79850	IN	NS	c.root-servers.net.
.			79850	IN	NS	b.root-servers.net.
.			79850	IN	NS	d.root-servers.net.
.			79850	IN	NS	h.root-servers.net.
.			79850	IN	NS	j.root-servers.net.
.			79850	IN	NS	i.root-servers.net.
.			79850	IN	NS	k.root-servers.net.
.			79850	IN	NS	f.root-servers.net.
;; Received 505 bytes from 10.238.14.2#53(10.238.14.2) in 26 ms

com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
;; Received 493 bytes from 192.203.230.10#53(192.203.230.10) in 184 ms

youdao.com.		172800	IN	NS	ns1.yodao.com.
youdao.com.		172800	IN	NS	ns2.yodao.com.
;; Received 107 bytes from 192.43.172.30#53(192.43.172.30) in 367 ms

note.youdao.com.	1200	IN	A	59.111.179.137
note.youdao.com.	1200	IN	A	59.111.179.138
note.youdao.com.	1200	IN	A	59.111.179.135
note.youdao.com.	1200	IN	A	59.111.179.136
;; Received 108 bytes from 61.135.216.245#53(61.135.216.245) in 9 ms
```
