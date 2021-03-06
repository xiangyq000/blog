---
title: '《HTTP权威指南》: 缓存'
date: 2016-07-12 15:05:48
tags: HTTP
categories: [读书笔记, Network]
---
### 1. 缓存的处理步骤
现代的商业化代理缓存相当地复杂。这些缓存构建得非常高效，可以支持HTTP和其他一些技术的各种高级特性。但除了一些微妙的细节之外，Web缓存的基本工作原理大多很简单。对一条HTTP GET报文的基本缓存处理过程包括7个步骤：

- 接收---缓存从网络中读取抵达的请求报文；
- 解析---缓存对报文进行解析，提取出URL和各种首部；
- 查询---缓存查看是否有本地副本可用，如果没有，就获取一份副本(并将其保存在本地)；
- 新鲜度检测---缓存查看已缓存副本是否足够新鲜，如果不是，就询问服务器是否有任何更新；
- 创建响应---缓存会用新的首部和已缓存的主体来构建一条响应报文；
- 发送---缓存通过网络将响应发回给客户端；
- 日志---缓存可选地创建一个日志文件条目来描述这个事务。
  <!--more-->

#### 1.1 接收
在第一步中，缓存检测到一条网络连接上的活动，读取输入数据。高性能的缓存会同时从多条输入连接上读取数据，在整条报文抵达之前开始对事务进行处理。

#### 1.2 解析
接下来，缓存将请求报文解析为片断，将首部的各个部分放入易于操作的数据结构中。这样，缓存软件就更容易处理首部字段并修改它们了。

#### 1.3 查询
在第三步中，缓存获取了URL，查找本地副本。本地副本可能存储在内存、本地磁盘，甚至附近的另一台计算机中。专业级的缓存会使用快速算法来确定本地缓存中是否有某个对象。如果本地没有这个文档，它可以根据情形和配置，到原始服务器或父代理中去取，或者返回一条错误信息。已缓存对象中包含了服务器响应主体和原始服务器响应首部，这样就会在缓存命中时返回正确的服务器首部。已缓存对象中还包含了一些元数据（metadata），用来记录对象在缓存中停留了多长时间，以及它被用过多少次等。

#### 1.4 新鲜度检测
HTTP通过缓存将服务器文档的副本保留一段时间。在这段时间里，都认为文档是"新鲜的"，缓存可以在不联系服务器的情况下，直接提供该文档。但一旦已缓存副本停留的时间太长，超过了文档的新鲜度限值（freshness limit），就认为对象“过时”了，在提供该文档之前，缓存要再次与服务器进行确认，以查看文档是否发生了变化。客户端发送给缓存的所有请求首部自身都可以强制缓存进行再验证，或者完全避免验证，这使得事情变得更加复杂了。HTTP有一组非常复杂的新鲜度检测规则，缓存产品支持的大量配置选项，以及与非HTTP新鲜度标准进行互通的需要则使问题变得更加严重了。本章其余的大部分篇幅都用于解释新鲜度的计算问题。

#### 1.5 创建响应
我们希望缓存的响应看起来就像来自原始服务器的一样，缓存将已缓存的服务器响应首部作为响应首部的起点。然后缓存对这些基础首部进行了修改和扩充。缓存负责对这些首部进行改造，以便与客户端的要求相匹配。比如，服务器返回的可能是一条HTTP/1.0响应（甚至是HTTP/0.9响应），而客户端期待的是一条HTTP/1.1响应，在这种情况下，缓存必须对首部进行相应的转换。缓存还会向其中插入新鲜度信息（Cache-Control、Age以及Expires首部），而且通常会包含一个Via首部来说明请求是由一个代理缓存提供的。注意，缓存不应该调整Date首部。Date首部表示的是原始服务器最初产生这个对象的日期。

#### 1.6 发送
一旦响应首部准备好了，缓存就将响应回送给客户端。和所有代理服务器一样，代理缓存要管理与客户端之间的连接。高性能的缓存会尽力高效地发送数据，通常可以避免在本地缓存和网络I/O缓冲区之间进行文档内容的复制。

#### 1.7 日志
大多数缓存都会保存日志文件以及与缓存的使用有关的一些统计数据。每个缓存事务结束之后，缓存都会更新缓存命中和未命中数目的统计数据（以及其他相关的度量值），并将条目插入一个用来显示请求类型、URL和所发生事件的日志文件。

### 2. 保持副本的新鲜
可能不是所有的已缓存副本都与服务器上的文档一致。毕竟，这些文档会随着时间发生变化。报告可能每个月都会变化。在线报纸每天都会发生变化。财经数据可能每过几秒钟就会发生变化。如果缓存提供的总是老的数据，就会变得毫无用处。已缓存数据要与服务器数据保持一致。HTTP有一些简单的机制可以在不要求服务器记住有哪些缓存拥有其文档副本的情况下，保持已缓存数据与服务器数据之间充分一致。HTTP将这些简单的机制称为文档过期（document expiration）和服务器再验证（server revalidation）。

#### 2.1 文档过期
通过特殊的HTTP Cache-Control首部和Expires首部，HTTP让原始服务器向每个文档附加了一个“过期日期”。这些首部说明了在多长时间内可以将这些内容视为新鲜的。在缓存文档过期之前，缓存可以以任意频率使用这些副本，而无需与服务器联系——当然，除非客户端请求中包含有阻止提供已缓存或未验证资源的首部。但一旦已缓存文档过期，缓存就必须与服务器进行核对，询问文档是否被修改过，如果被修改过，就要获取一份新鲜（带有新的过期日期）的副本。

#### 2.2 过期日期和使用期
服务器用HTTP/1.0+的Expires首部或HTTP/1.1的Cache-Control: max-age响应首部来指定过期日期，同时还会带有响应主体。Expires首部和Cache-Control: max-age首部所做的事情本质上是一样的，但由于Cache-Control首部使用的是相对时间而不是绝对日期，所以我们更倾向于使用比较新的Cache-Control首部。绝对日期依赖于计算机时钟的正确设置。

|首部	| 描述|
| ------|------| -----|
|Cache-Control: max-age|	max-age值定义了文档的最大使用期——从第一次生成文档到文档不再新鲜、无法使用为止，最大的合法生存时间(以秒为单位)
|Expires|	指定一个绝对的过期日期。如果过期日期已经过了，就说明文档不再新鲜了|

#### 2.3 服务器再验证
仅仅是已缓存文档过期了并不意味着它和原始服务器上目前处于活跃状态的文档有实际的区别；这只是意味着到了要进行核对的时间了。这种情况被称为“服务器再验证”，说明缓存需要询问原始服务器文档是否发生了变化。缓存并不一定要为每条请求验证文档的有效性——只有在文档过期时它才需要与服务器进行再验证。这样不会提供陈旧的内容，还可以节省服务器的流量，并拥有更好的用户响应时间。

- 如果再验证显示内容发生了变化，缓存会获取一份新的文档副本，并将其存储在旧文档的位置上，然后将文档发送给客户端。
- 如果再验证显示内容没有发生变化，缓存只需要获取新的首部，包括一个新的过期日期，并对缓存中的首部进行更新就行了。

HTTP协议要求行为正确的缓存返回下列内容之一：

- “足够新鲜”的已缓存副本；
- 与服务器进行过再验证，确认其仍然新鲜的已缓存副本；
- 如果需要与之进行再验证的原始服务器出故障了，就返回一条错误报文 ；
- 附有警告信息说明内容可能不正确的已缓存副本。

#### 2.4 用条件方法进行再验证
HTTP的条件方法可以高效地实现再验证。HTTP允许缓存向原始服务器发送一个“条件GET”，请求服务器只有在文档与缓存中现有的副本不同时，才回送对象主体。通过这种方式，将新鲜度检测和对象获取结合成了单个条件GET。向GET请求报文中添加一些特殊的条件首部，就可以发起条件GET。只有条件为真时，Web服务器才会返回对象。HTTP定义了5个条件请求首部。对缓存再验证来说最有用的2个首部是If-Modified-Since和If-None-Match。所有的条件首部都以前缀“If-”开头。

| 首部                       | 描述                                       |
| ------------------------ | ---------------------------------------- |
| If-Modified-Since:<date> | 如果从指定日期之后文档被修改过了，就执行请求的方法。可以与Last-Modified服务器响应首部配合使用，只有在内容被修改后与已缓存版本有所不同的时候才去获取内容 |
| If-None-Match:<tags>     | 服务器可以为文档提供特殊的标签，而不是将其与最近修改日期相匹配，这些标签就像序列号一样。如果已缓存标签与服务器文档中的标签有所不同，If-None-Match首部就会执行所请求的方法 |

#### 2.5 If-Modified-Since:Date再验证
最常见的缓存再验证首部是If-Modified-Since。If-Modified-Since再验证请求通常被称为IMS请求。只有自某个日期之后资源发生了变化的时候，IMS请求才会指示服务器执行请求：

- 如果自指定日期后，文档被修改了，If-Modified-Since条件就为真，通常GET就会成功执行。携带新首部的新文档会被返回给缓存，新首部除了其他信息之外，还包含了一个新的过期日期。
- 如果自指定日期后，文档没被修改过，条件就为假，会向客户端返回一个小的304 Not Modified响应报文，为了提高有效性，不会返回文档的主体。这 些首部是放在响应中返回的，但只会返回那些需要在源端更新的首部。比如，Content-Type首部通常不会被修改，所以通常不需要发送。一般会发送一个新的过期日期。

If-Modified-Since首部可以与Last-Modified服务器响应首部配合工作。原始服务器会将最后的修改日期附加到所提供的文档上去。当缓存要对已缓存文档进行再验证时，就会包含一个If-Modified-Since首部，其中携带有最后修改已缓存副本的日期：

```
If-Modified-Since: <cached last-modified date>
```

如果在此期间内容被修改了，最后的修改日期就会有所不同，原始服务器就会回送新的文档。否则，服务器会注意到缓存的最后修改日期与服务器文档当前的最后修改日期相符，会返回一个304 Not Modified响应。

注意，有些Web服务器并没有将If-Modified-Since作为真正的日期来进行比对。相反，它们在IMS日期和最后修改日期之间进行了字符串匹配。这样得到的语义就是“如果最后的修改不是在这个确定的日期进行的”，而不是“如果在这个日期之后没有被修改过”。将最后修改日期作为某种序列号使用时，这种替代语义能够很好地识别出缓存是否过期，但这会妨碍客户端将If-Modified-Since首部用于真正基于时间的一些目的。

#### 2.6 If-None-Match：实体标签再验证
有些情况下仅使用最后修改日期进行再验证是不够的。

- 有些文档可能会被周期性地重写（比如，从一个后台进程中写入），但实际包含的数据常常是一样的。尽管内容没有变化，但修改日期会发生变化。
- 有些文档可能被修改了，但所做修改并不重要，不需要让世界范围内的缓存都重装数据(比如对拼写或注释的修改)。
- 有些服务器无法准确地判定其页面的最后修改日期。
- 有些服务器提供的文档会在亚秒间隙发生变化(比如，实时监视器)，对这些服务器来说，以一秒为粒度的修改日期可能就不够用了。

为了解决这些问题，HTTP允许用户对被称为实体标签（ETag）的“版本标识符”进行比较。实体标签是附加到文档上的任意标签（引用字符串）。它们可能包含了文档的序列号或版本名，或者是文档内容的校验和及其他指纹信息。

当发布者对文档进行修改时，可以修改文档的实体标签来说明这个新的版本。这样，如果实体标签被修改了，缓存就可以用If-None-Match条件首部来GET文档的新副本了。

#### 2.7 强弱验证器
缓存可以用实体标签来判断，与服务器相比，已缓存版本是不是最新的(与使用最近修改日期的方式很像)。从这个角度来看，实体标签和最近修改日期都是缓存验证器（cache validator）。

HTTP把验证码分为两类：弱验证码（weak validators）和强验证码（strong validators）。弱验证码不一定能唯一标识资源的一个实例，而强验证码必须如此。弱验证码的一个例子是对象的大小字节数。有可能资源的内容改变了，而大小还保持不变，因此假想的字节计数验证码与改变是弱相关的。而资源内容的加密校验和（比如MD5）就是强验证码，当文档改变时它总是会改变

最后修改时间被当作弱验证码，因为尽管它说明了资源最后被修改的时间，但它的描述精度最大就是1秒。因为资源在1秒内可以改变很多次，而且服务器每秒可以处理数千个请求，最后修改日期时间并不总能反应变化情况。ETag首部被当作强验证码，因为每当资源内容改变时，服务器都可以在ETag首部放置不同的值。版本号和摘要校验和也是很好的ETag首部候选，但它们不能带有任意的文本。ETag首部很灵活，它可以带上任意的文本值（以标记的形式），这样就可以用来设计出各种各样的客户端和服务器验证策略。

有时候，客户端和服务器可能需要采用不那么精确的实体标记验证方法。例如，某服务器可能想对一个很大、被广泛缓存的文档进行一些美化修饰，但不想在缓存服务器再验证时产生很大的传输流量。在这种情况下，该服务器可以在标记前面加上“W/”前缀来广播一个“弱”实体标记。对于弱实体标记来说，只有当关联的实体在语义上发生了重大改变时，标记才会变化。而强实体标记则不管关联的实体发生了什么性质的变化，标记都一定会改变。

```
ETag: W/"v2.6"
If-None-Match: W/"v2.6"
```

不管相关的实体值以何种方式发生了变化，强实体标签都要发生变化。而相关实体在语义上发生了比较重要的变化时，弱实体标签也应该发生变化。

注意，原始服务器一定不能为两个不同的实体重用一个特定的强实体标签值，或者为两个语义不同的实体重用一个特定的弱实体标签值。缓存条目可能会留存任意长的时间，与其过期时间无关，有人可能希望当缓存验证条目时，绝对不会再次使用在过去某一时刻获得的验证器，这种愿望可能不太现实。

#### 2.8 实体标签和最近修改日期
如果服务器回送了一个实体标签，HTTP/1.1客户端就必须使用实体标签验证器。如果服务器只回送了一个Last-Modified值，客户端就可以使用If-Modified-Since验证。如果实体标签和最后修改日期都提供了，客户端就应该使用这两种再验证方案，这样HTTP/1.0和HTTP/1.1缓存就都可以正确响应了。

除非HTTP/1.1原始服务器无法生成实体标签验证器，否则就应该发送一个出去，如果使用弱实体标签有优势的话，发送的可能就是个弱实体标签，而不是强实体标签。而且，最好同时发送一个最近修改值。如果HTTP/1.1缓存或服务器收到的请求既带有If-Modified-Since，又带有实体标签条件首部，那么只有这两个条件都满足时，才能返回304 Not Modified响应。

### 3. 控制缓存的能力
服务器可以通过HTTP定义的几种方式来指定在文档过期之前可以将其缓存多长时间。按照优先级递减的顺序，服务器可以：

- 附加一个"Cache-Control: no-store"首部到响应中去；
- 附加一个"Cache-Control: no-cache"首部到响应中去；
- 附加一个"Cache-Control: must-revalidate"首部到响应中去；
- 附加一个"Cache-Control: max-age"首部到响应中去；
- 附加一个"Expires"日期首部到响应中去；
- 不附加过期信息，让缓存确定自己的过期日期。

#### 3.1 no-Store 与 no-Cache 响应首部
HTTP/1.1提供了几种限制对象缓存，或限制提供已缓存对象的方式，以维持对象的新鲜度。no-store首部和no-cache首部可以防止缓存提供未经证实的已缓存对象：

```
Pragma: no-cache
Cache-Control: no-store
Cache-Control: no-cache
```

标识为no-store的响应会禁止缓存对响应进行复制。缓存通常会像非缓存代理服务器一样，向客户端转发一条no-store响应，然后删除对象。

标识为no-cache的响应实际上是可以存储在本地缓存区中的。只是在与原始服务器进行新鲜度再验证之前，缓存不能将其提供给客户端使用。这个首部使用do-not-serve-from-cache-without-revalidation这个名字会更恰当一些。

HTTP/1.1中提供Pragma: no-cache首部是为了兼容于HTTP/1.0+。除了与只理解Pragma: no-cache"的HTTP/1.0应用程序进行交互时，HTTP 1.1应用程序都应该使用Cache-Control: no-cache。

#### 3.2 max-age响应首部
Cache-Control: max-age首部表示的是从服务器将文档传来之时起，可以认为此文档处于新鲜状态的秒数（Cache-Control: max-age=3600）。还有一个s-maxage首部（注意maxage的中间没有连字符），其行为与max-age类似，但仅适用于共享（公有）缓存（Cache-Control: s-maxage=3600）。服务器可以请求缓存不要缓存文档，或者将最大使用期设置为零，从而在每次访问的时候都进行刷新（Cache-Control: max-age=0）。

#### 3.3 Expires响应首部
不推荐使用Expires首部，它指定的是实际的过期日期而不是秒数。HTTP设计者后来认为，由于很多服务器的时钟都不同步，或者不正确，所以最好还是用剩余秒数，而不是绝对时间来表示过期时间。可以通过计算过期值和日期值之间的秒数差来计算类似的新鲜生存期

```
Expires: Fri, 05 Jul 2002, 05:00:00 GMT
```

有些服务器还会回送一个Expires:0响应首部，试图将文档置于永远过期的状态，但这种语法是非法的，可能给某些软件带来问题。应该试着支持这种结构的输入，但不应该产生这种结构的输出。

#### 3.4 must-revalidate 响应首部
可以配置缓存，使其提供一些陈旧(过期)的对象，以提高性能。如果原始服务器希望缓存严格遵守过期信息，可以在原始响应中附加一个Cache-Control: must-revalidate首部。

Cache-Control: must-revalidate响应首部告诉缓存，在事先没有跟原始服务器进行再验证的情况下，不能提供这个对象的陈旧副本。缓存仍然可以随意提供新鲜的副本。如果在缓存进行must-revalidate新鲜度检查时，原始服务器不可用，缓存就必须返回一条504 Gateway Timeout错误。

#### 3.5试探性过期
如果响应中没有Cache-Control: max-age首部，也没有Expires首部，缓存可以计算出一个试探性最大使用期。可以使用任意算法，但如果得到的最大使用期大于24小时，就应该向响应首部添加一个Heuristic Expiration Warning（试探性过期警告，警告13）首部。

#### 3.6 客户端的新鲜度限制
Web 浏览器都有刷新（Refresh）或 重载（Reload）按钮，可以强制对浏览器或代理缓存中可能过期的内容进行刷新。刷新按钮会发布一个附加了Cache-Control请求首部的GET请求，这个请求会强制进行再验证，或者无条件地从服务器获取文档。刷新的确切行为取决于特定的浏览器、文档以及拦截缓存的配置。

客户端可以用Cache-Control请求首部来强化或放松对过期时间的限制。有些应用程序对文档的新鲜度要求很高（比如人工刷新按钮），对这些应用程序来说，客户端可以用Cache-Control首部使过期时间更严格。另一方面，作为提高性能、可靠性或开支的一种折衷方式，客户端可能会放松新鲜度要求。

Cache-Control请求指令：

| 指令                            | 目的                                       |
| ----------------------------- | ---------------------------------------- |
| Cache-Control: max-stale      | 缓存可以随意提供过期的文件。如果指定了参数(s)，在这段时间内，文档就不能过期。这条指令放松了缓存的规则 |
| Cache-Control: min-fresh=(s)  | 至少在未来(s)秒内文档要保持新鲜。这就使缓存规则更加严格了           |
| Cache-Control: max-age = (s)  | 缓存无法返回缓存时间长于(s)秒的文档。这条指令会使缓存规则更加严格，除非同时还发送max-stale指令，在这种情况下，使用期可能会超过其过期时间 |
| Cache-Control: no-cache       | 除非资源进行了再验证，否则这个客户端不会接受已缓存的资源             |
| Cache-Control: no-store       | 缓存应该尽快从存储器中删除文档的所有痕迹，因为其中可能会包含敏感信息       |
| Cache-Control: only-if-cached | 只有当缓存中有副本存在时，客户端才会获取一份副本                 |

