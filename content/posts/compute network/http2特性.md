---
layout: post
title: "http2特性"
date: 2021-9-12
categories: 
    - "计算机网络"
tags: [HTTP2.0]
author: "pan"
---

# HTTP协议发展历史

1. HTTP0.9

   1991年发布。该版本极其简单，只有一个命令GET，不支持请求头
2. HTTP/1.0

   1996年5月发布。引入请求头和响应头；新增请求方法，如head/post

3. HTTP1.1

    1997年1月发布。支持长连接；添加Content-Length字段；分块传输编码等

4. SPDY

    2012年Google发布。HTTP2.0就是基于SPDY设计的，现在已经无人使用。添加多路复用（Multiplexing）；header压缩（DEFLATE算法）；服务端推送等

5. HTTP2.0

   2015年发布。本文主要讲解内容，后文详细讨论。

6. HTTP3.0

    2018年发布。尚未研究，不在本文讨论范围。

[HTTP/2 的wiki介绍](https://zh.wikipedia.org/wiki/HTTP/2)，可以看下定义和发展历史。[RFC 7540](https://httpwg.org/specs/rfc7540.html) 定义了 HTTP/2 的协议规范和细节, [RFC 7541](https://httpwg.org/specs/rfc7541.html)定义了头部压缩。如果有时间，最后就直接看RFC的文档。没有什么资料可以比官方文档写的更清楚。本文只是自已的归纳和整理。难免有些粗陋和错误，望评判指正。

# 一、HTTP2 解决什么问题

HTTP2的提出肯定是为了解决HTTP1.1已经存在的问题。所以HTTP1.1存在那些问题呢？

## 1.1 TCP连接数限制

因为并发的原因一个TCP连接在同一时刻可能发送一个http请求。所以为了更快的响应前端请求，浏览器会建立多个tcp连接，但是第一tcp连接数量是有限制的。现在的浏览器针对同一域名一般最多只能创建6~8个请求；第二创建tcp连接需要三次握手，增加耗时、cpu资源、增加网络拥堵的可能性。所以，缺点明显。

## 1.2 线头阻塞 (Head Of Line Blocking) 问题

每个 TCP 连接同时只能处理一个请求 - 响应，浏览器按 FIFO 原则处理请求，如果上一个响应没返回，后续请求 - 响应都会受阻。为了解决此问题，出现了 管线化 - pipelining 技术，但是管线化存在诸多问题，比如第一个响应慢还是会阻塞后续响应、服务器为了按序返回相应需要缓存多个响应占用更多资源、浏览器中途断连重试服务器可能得重新处理多个请求、还有必须客户端 - 代理 - 服务器都支持管线化。

## 1.3 Header 内容多

每次请求 Header不会变化太多，没有相应的压缩传输优化方案。特别是想cookie这种比较长的字段

对于HTTP1.1存在的这些问题，是有一定的优化方案的，比如用对个域名，文件合并等。但是这些毕竟比较麻烦，甚至无聊。

# 二、基本概念

+ 数据流: 已建立的连接内的双向字节流，可以承载一条或多条消息。
+ 消息: 与逻辑请求或响应消息对应的完整的一系列帧。
+ 帧: HTTP/2 通信的最小单位，每个帧都包含帧头，至少也会标识出当前帧所属的数据流。

这些概念的关系总结如下:

+ 所有通信都在一个 TCP 连接上完成，此连接可以承载任意数量的双向数据流。
+ 每个数据流都有一个唯一的标识符和可选的优先级信息，用于承载双向消息。
+ 每条消息都是一条逻辑 HTTP 消息（例如请求或响应），包含一个或多个帧。
+ 帧是最小的通信单位，承载着特定类型的数据，例如 HTTP 标头、消息负载等等。 来自不同数据流的帧可以交错发送，然后再根据每个帧头的数据流标识符重新组装。

![image.png](/http2特性/8596800-0b880a194532403a.png)

# 三、HTTP2特性有那些
需要强调的是HTTP/2 是对之前 HTTP 标准的扩展，而非替代。 HTTP 的应用语义不变，提供的功能不变，HTTP 方法、状态代码、URI和标头字段等这些核心概念也不变。
我们已经知道http1.x的报文格式由`开始行`，`首部行`,`实体主体`三部分组成。HTTP2将`开始行`，`首部行`封装成`帧`。`实体主体`封装成`帧`。这里的`帧`是HTTP/2所有性能增强的核心。它定义了如何封装 HTTP 消息并在客户端与服务器之间传输。下图可以很好帮助大家理解http1.x和http2的关系。

![image.png](/http2特性/8596800-b8a4ffaac5cd13a0.png)

![image.png](/http2特性/8596800-dfe553251135f12c.png)

HTTP2特性包含一下几个方面

+ 二进制分帧
+ 多路复用
+ 头部压缩
+ 服务端推送（server push）
+ 流量控制
+ 资源优先级和依赖设置

## 3.1 二进制分帧

帧是数据传输的最小单位，以二进制传输代替原本的明文传输，原本的报文消息被划分为更小的数据帧:
![image.png](/http2特性/8596800-e1357788aef5eb72.png)
简言之，HTTP/2 将 HTTP 协议通信分解为二进制编码帧的交换，这些帧对应着特定数据流中的消息。所有这些都在一个 TCP 连接内复用。 这是 HTTP/2 协议所有其他功能和性能优化的基础。

### 3.1.1 HTTP2报文格式

所有帧都是一个固定的 9 字节头部 (payload 之前) 跟一个指定长度的负载 (payload),格式如下。
![image.png](/http2特性/8596800-510476f4527ca128.png)
+ Length：无符号的自然数，24个比特表示，仅表示帧负载（Frame Payload）所占用字节数，不包括帧头所占用的9个字节。 默认大小区间为为0~16,384(2^14)，一旦超过默认最大值2^14(16384)，发送方将不再允许发送，除非接收到接收方定义的SETTINGS_MAX_FRAME_SIZE（一般此值区间为2^14 ~ 2^24）值的通知。
+ Type：定义 frame 的类型，用 8 bits 表示。帧类型决定了帧主体的格式和语义，如果 type 为 unknown 应该忽略或抛弃。
+ Flags：是为帧类型相关而预留的布尔标识。标识对于相同同的帧类型赋予了不同的语义
+ R：是一个保留的比特位。这个比特的语义没有定义，发送时它必须被设置为 (0x0), 接收时需要忽略。
+ Stream Identifier ：用作流控制，用 31 位无符号整数表示。客户端建立的 sid 必须为奇数，服务端建立的 sid 必须为偶数，值 (0x0) 保留给与整个连接相关联的帧 (连接控制消息)，而不是单个流
+ Frame Payload：是主体内容，由帧类型决定

HTTP2共分为十种类型的帧:
![image.png](/http2特性/8596800-1a41c94b2e6fec53.png)

+ HEADERS: 报头帧 (type=0x1)，用来打开一个流或者携带一个首部块片段
+ DATA: 数据帧 (type=0x0)，装填主体信息，可以用一个或多个 DATA 帧来返回一个请求的响应主体
+ PRIORITY: 优先级帧 (type=0x2)，指定发送者建议的流优先级，可以在任何流状态下发送 PRIORITY 帧，包括空闲 (idle) 和关闭 (closed) 的流
+ RST_STREAM: 流终止帧 (type=0x3)，用来请求取消一个流，或者表示发生了一个错误，payload 带有一个 32 位无符号整数的错误码 (Error Codes)，不能在处于空闲 (idle) 状态的流上发送 RST_STREAM 帧
+ SETTINGS: 设置帧 (type=0x4)，设置此 连接 的参数，作用于整个连接
+ PUSH_PROMISE: 推送帧 (type=0x5)，服务端推送，客户端可以返回一个 RST_STREAM 帧来选择拒绝推送的流
+ PING: PING 帧 (type=0x6)，判断一个空闲的连接是否仍然可用，也可以测量最小往返时间 (RTT)
+ GOAWAY: GOWAY 帧 (type=0x7)，用于发起关闭连接的请求，或者警示严重错误。GOAWAY 会停止接收新流，并且关闭连接前会处理完先前建立的流
+ WINDOW_UPDATE: 窗口更新帧 (type=0x8)，用于执行流量控制功能，可以作用在单独某个流上 (指定具体 Stream Identifier) 也可以作用整个连接 (Stream Identifier 为 0x0)，只有 DATA 帧受流量控制影响。初始化流量窗口后，发送多少负载，流量窗口就减少多少，如果流量窗口不足就无法发送，WINDOW_UPDATE 帧可以增加流量窗口大小
+ CONTINUATION: 延续帧 (type=0x9)，用于继续传送首部块片段序列，见 首部的压缩与解压缩

HTTP2 帧和flags的可能组合示意图:

![不同类型的帧可能对应的flags](/http2特性/8596800-e2e8743ba0b57d42.png)
**表中x符号表示该类型的帧的flags可以取的值**
下面看一些几种常见的帧完整结构

### 3.1.1.1 DATA 帧格式

DATA 帧的type为0x0。

![DATA Frame Payload](/http2特性/8596800-edf33793d97804a9.png)

`Pad Length`:? 表示此字段的出现时有条件的，需要设置相应标识 (set flag)，指定 Padding 长度，存在则代表 PADDING flag 被设置
`Data`:传递的数据，其长度上限等于帧的 payload 长度减去其他出现的字段长度
`Padding`:填充字节，没有具体语义，发送时必须设为 0，作用是混淆报文长度，与 TLS 中 CBC 块加密类似

DATA 帧有如下标识 (flags):
+ END_STREAM: bit 0 设为 1 代表当前流的最后一帧
+ PADDED: bit 3 设为 1 代表存在 Padding
  
### 3.1.1.2 HEADERS 帧格式
![HEADERS Frame Payload](/http2特性/8596800-38a8cd5402d4250e.png)
+ `Pad Length`: 指定 Padding 长度，存在则代表 PADDING flag 被设置
+ `E`: 一个比特位声明流的依赖性是否是排他的，存在则代表 PRIORITY flag 被设置
+ `Stream Dependency`: 指定一个 stream identifier，代表当前流所依赖的流的 id，存在则代表 PRIORITY flag 被设置
+ `Weight`: 一个无符号 8 为整数，代表当前流的优先级权重值 (1~256)，存在则代表 PRIORITY flag 被设置
+ `Header Block Fragment`: header 块片段
+ `Padding`: 填充字节，没有具体语义，作用与 DATA 的 Padding 一样，存在则代表 PADDING flag 被设置

HEADERS 帧有以下标识 (flags):
+ END_STREAM: bit 0 设为 1 代表当前 header 块是发送的最后一块，但是带有 END_STREAM 标识的 HEADERS 帧后面还可以跟 CONTINUATION 帧 (这里可以把 CONTINUATION 看作 HEADERS 的一部分)
+ END_HEADERS: bit 2 设为 1 代表 header 块结束
+ PADDED: bit 3 设为 1 代表 Pad 被设置，存在 Pad Length 和 Padding
+ PRIORITY: bit 5 设为 1 表示存在 Exclusive Flag (E), Stream Dependency, 和 Weight

### 3.1.1.3 SETTINGS 帧格式
一个 SETTINGS 帧的 payload 由零个或多个参数组成，每个参数的形式如下:
![SETTINGS Format](/http2特性/8596800-0f5b3af6033565aa.png)
在建立连接开始时双方都要发送 SETTINGS 帧以表明自己期许对方应做的配置，对方接收后同意配置参数便返回带有 ACK 标识的空 SETTINGS 帧表示确认，而且连接后任意时刻任意一方也都可能再发送 SETTINGS 帧调整，SETTINGS 帧中的参数会被最新接收到的参数覆盖
**SETTINGS 帧作用于整个连接，而不是某个流**，而且 SETTINGS 帧的 stream identifier 必须是 0x0，否则接收方会认为错误 (PROTOCOL_ERROR)。
SETTINGS 帧包含以下参数:
+ SETTINGS_HEADER_TABLE_SIZE (0x1): 用于解析 Header block 的 Header 压缩表的大小，初始值是 4096 字节
+ SETTINGS_ENABLE_PUSH (0x2): 可以关闭 Server Push，该值初始为 1，表示允许服务端推送功能
+ SETTINGS_MAX_CONCURRENT_STREAMS (0x3): 代表发送端允许接收端创建的最大流数目
+ SETTINGS_INITIAL_WINDOW_SIZE (0x4): 指明发送端所有流的流量控制窗口的初始大小，会影响所有流，该初始值是 2^16 - 1(65535) 字节，最大值是 2^31 - 1，如果超出最大值则会返回 FLOW_CONTROL_ERROR
+ SETTINGS_MAX_FRAME_SIZE (0x5): 指明发送端允许接收的最大帧负载的字节数，初始值是 2^14(16384) 字节，如果该值不在初始值 (2^14) 和最大值 (2^24 - 1) 之间，返回 PROTOCOL_ERROR
+ SETTINGS_MAX_HEADER_LIST_SIZE (0x6): 通知对端，发送端准备接收的首部列表大小的最大字节数。该值是基于未压缩的首部域大小，包括名称和值的字节长度，外加每个首部域的 32 字节的开销

SETTINGS 帧有以下标识 (flags):

+ ACK: bit 0 设为 1 代表已接收到对方的 SETTINGS 请求并同意设置，设置此标志的 SETTINGS 帧 payload 必须为空

### 3.1.1.4 PRIORITY 帧格式
![image.png](/http2特性/8596800-4a2f85f388088ec7.png)
PRIORITY 帧可以在流的任何状态使用，Header帧中优先级是在打开的时候，注意区别。字段含义和header帧中的一样。**PRIORITY只可作用于特定的流，不可作用于整个连接**
### 3.1.1.5 RST_STREAM 帧格式
![image.png](/http2特性/8596800-88a68f0785ba7087.png)
RST_STREAM帧用于立刻终止一个流
### 3.1.1.6 PUSH_PROMISE 帧格式
![image.png](/http2特性/8596800-5773475b1fb71aee.png)

+ `Pad Length`: 指定 Padding 长度，存在则代表 PADDING flag 被设置
+ `R`: 保留的1bit位
+ `Promised Stream ID`: 31 位的无符号整数，代表PUSH_PROMISE 帧保留的流，对于发送者来说该流标识符必须是可用于下一个流的有效值(该标识是偶数)
+ `Header Block Fragment`: 包含请求首部域的首部块片段
+ `Padding`: 填充字节，没有具体语义，作用与 DATA 的 Padding 一样，存在则代表 PADDING flag 被设置

PUSH_PROMISE 帧有以下标识 (flags):

+ END_HEADERS: bit 2 置 1 代表 header 块结束
+ PADDED: bit 3 置 1 代表 Pad 被设置，存在 Pad Length 和 Padding

### 3.1.1.7 PING 帧格式

![image.png](/http2特性/8596800-2fa83e2ff11654f7.png)
用于判断空闲连接是否可用。
PING 帧有以下标识 (flags):

+ ACK (0x1):设置为0表示对ping帧的回复

### 3.1.1.8 GOAWAY 帧格式

![image.png](/http2特性/8596800-b9d94d3a7af9d90b.png)

### 3.1.1.9 WINDOW_UPDATE 帧格式

WINDOW_UPDATE用于流量控制，可作用于整个连接或者流
![image.png](/http2特性/8596800-1c344f7b3dd7346b.png)
Window Size Increment 表示除了现有的流量控制窗口之外，发送端还可以传送的字节数。取值范围是 1 到 2^31 - 1 字节

### 3.1.1.10 CONTINUATION 帧格式

![image.png](/http2特性/8596800-2e26d1d142c21f65.png)

## 3.2 多路复用

**简而言之：多个http请求可以共用同一个TCP连接**。

![image.png](/http2特性/8596800-d99d62bd12770b49.png)

### 3.2.1 为什么http1.1不能实现多路复用

http1.1 是基于文本分割协议的。我们不知道一个请求什么时候结束，只能一直读取，直到出现空行（http请求结果标志）。所以就不能使用多路复用。要不然就不知道哪个消息是属于哪个请求了。但是HTTP2引入二进制分帧，用 stream id标识帧和请求的对应关系。

## 3.3 头部压缩

HTTP2使用的HPACK作为头部压缩算法。

![image.png](/http2特性/8596800-f3b294c54b626f65.png)

可以清楚地看到 HTTP2 头部使用的也是键值对形式的值，而且 HTTP1 当中的请求行以及状态行也被分割成键值对，还有所有键都是小写，不同于 HTTP1。除此之外，还有一个包含静态索引表和动态索引表的索引空间，实际传输时会把头部键值表压缩，使用的算法即 HPACK，其原理就是匹配当前连接存在的索引空间，若某个键值已存在，则用相应的索引代替首部条目，比如 “:method: GET” 可以匹配到静态索引中的 index 2，传输时只需要传输一个包含 2 的字节即可；若索引空间中不存在，则用字符编码传输，字符编码可以选择哈夫曼编码，然后分情况判断是否需要存入动态索引表中。关于详细的压缩过程见参考文献10。

## 3.4 server push

服务端主动推送,如下图，page.html包含script.js和style.css资源文件。客户端只需要请求page.html，服务端发现page.html中包含资源文件会主动推送给客户端。减少客户端请求的次数。
![image.png](/http2特性/8596800-47ff1e4fad929580.png)
所有服务器推送数据流都由 PUSH_PROMISE 帧发起，表明了服务器向客户端推送所述资源的意图，**并且需要先于请求推送资源的响应数据传输**。 这种传输顺序非常重要: 客户端需要了解服务器打算推送哪些资源，以免为这些资源创建重复请求。 满足此要求的最简单策略是先于父响应（即，DATA 帧）发送所有 PUSH_PROMISE 帧，其中包含所承诺资源的 HTTP 标头。

## 3.5 流量控制

多路复用的流会竞争 TCP 资源，进而导致流被阻塞。流控制机制确保同一连接上的流不会相互干扰。流量控制作用于单个流或整个连接。HTTP/2 通过使用 WINDOW_UPDATE 帧来提供流量控制。例如，客户端可能请求了一个具有较高优先级的大型视频流，但是用户已经暂停视频，客户端现在希望暂停或限制从服务器的传输，以免提取和缓冲不必要的数据。

+ 流量控制是特定于连接的。两种级别的流量控制都位于单跳的端点之间，而不是整个端到端的路径。比如 server 前面有一个 front-end proxy 如 Nginx，这时就会有两个 connection，browser-Nginx, Nginx—server，flow control 分别作用于两个 connection。
+ 流量控制是基于 WINDOW_UPDATE 帧的。接收方公布自己打算在每个流以及整个连接上分别接收多少字节。这是一个以信用为基础的方案。
+ 流量控制是有方向的，由接收者全面控制。接收方可以为每个流和整个连接设置任意的窗口大小。发送方必须尊重接收方设置的流量控制限制。客户方、服务端和中间代理作为接收方时都独立地公布各自的流量控制窗口，作为发送方时都遵守对端的流量控制设置。
+ 无论是新流还是整个连接，流量控制窗口的初始值是 65535 字节。
+ 帧的类型决定了流量控制是否适用于帧。目前，只有 DATA 帧会受流量控制影响，所有其它类型的帧并不消耗流量控制窗口的空间。这保证了重要的控制帧不会被流量控制阻塞。
+ 流量控制不能被禁用。
+ HTTP/2 只定义了 WINDOW_UPDATE 帧的格式和语义，并没有规定接收方如何决定何时发送帧、发送什么样的值，也没有规定发送方如何选择发送包。具体实现可以选择任何满足需求的算法。

## 3.6 资源优先级和依赖设置

客户端可以通过 HEADERS 帧的 PRIORITY 信息指定一个新建立流的优先级，其他期间也可以发送 PRIORITY 帧调整流优先级

每个流都可以显示地依赖另一个流，包含依赖关系表示优先将资源分配给指定的流(上层节点)而不是依赖流

# 参考文献

其中2，7是讲解的比较好的，可以重点参考

1. [HTTP/2协议“多路复用”实现原理](https://segmentfault.com/a/1190000016975064)
2. [HTTP2 详解](https://juejin.cn/post/6844903667569541133)
3. [Okhttp如何开启的Http2.0](https://juejin.cn/post/6844904200241938445)
4. [stakoverflow上关于HTTP2.0多路复用的一个比较好的解释](https://stackoverflow.com/questions/36517829/what-does-multiplexing-mean-in-http-2)
5. [HttpClient doesn't reuse TCP connection for both h2 and h2c connections](https://bugs.openjdk.java.net/browse/JDK-8195137#)
6. [HTTP/2 资料汇总](https://imququ.com/post/http2-resource.html)
7. [HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2?hl=zh-cn)
8. [RFC 7540 Hypertext Transfer Protocol Version 2](https://httpwg.org/specs/rfc7540.html)
9. [RFC 7541 HPACK: Header Compression for HTTP/2](https://httpwg.org/specs/rfc7541.html)
10. [HTTP/2 头部压缩技术介绍](https://imququ.com/post/header-compression-in-http2.html)
11. [谈谈 HTTP/2 的协议协商机制](https://imququ.com/post/protocol-negotiation-in-http2.html)