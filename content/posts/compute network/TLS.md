---
layout: post
title: "TLS"
date: 2021-8-29
categories: 
    - "计算机网络"
tags: [TLS]
author: "pan"
---

本文借助wireshark抓包详细的讲解SSL/TLS协议。HTTPS是为了解决http报文明文传输过程中的安全问题。HTTPS是“HTTP over SSL”的缩写。所以要了解HTTPS就必须先了解SSL/TLS协议。

# 一、HTTP协议的风险
HTTP协议中所有消息都是明文传播，存在如下三大风险
1. 窃听风险（eavesdropping）：第三方可以获知通信内容。
2. 篡改风险（tampering）：第三方可以修改通信内容。
3. 冒充风险（pretending）：第三方可以冒充他人身份参与通信。

为了解决这个三个风险，分别对应如下三个解决方案。
1. 加密：所有信息都是加密传播，第三方无法窃听。
2. 校验：具有校验机制，一旦被篡改，通信双方会立刻发现。
3. 身份验证：配备身份证书，防止身份被冒充。

# 二、SSL/TLS 发展历史
1. 1994年，NetScape公司设计了SSL协议（Secure Sockets Layer）的1.0版，但是未发布。
2. 1995年，NetScape公司发布SSL 2.0版，很快发现有严重漏洞。
3. 1996年，SSL 3.0版问世，得到大规模应用。
4. 1999年，互联网标准化组织ISOC接替NetScape公司，发布了SSL的升级版[TLS](https://en.wikipedia.org/wiki/Secure_Sockets_Layer) 1.0版。
5. 2006: TLS 1.1. 作为 RFC 4346 发布。主要fix了CBC模式相关的如BEAST攻击等漏洞。
6. 2008: TLS 1.2. 作为RFC 5246 发布 。增进安全性。目前(2015年)应该主要部署的版本。
7. 2015之后: TLS 1.3，还在制订中，支持0-rtt，大幅增进安全性，砍掉了aead之外的加密方式。

由于SSL的2个版本都已经退出历史舞台了，所以本文后面只用TLS这个名字。 一般所说的SSL就是TLS。

# 三、报文解析([rfc5246](https://www.rfc-editor.org/rfc/rfc5246.html))
TLS建立连接的过程如下图，先有个大概的印象，后面我们再详细分析。整个需要四次握手。
![TLS认证过程](/TLS/8596800-f7d560ac6fa901d6.png)

**SSL/TLS工作在应用层和传输层之间**，在建立连接的之前需要先建立TCP连接（三次握手），如下图。
![TLS协议抓包](/TLS/8596800-d9013fbee6e6577f.png)
## 3.1 详细过程

### （1）Client Hello

![Client Hello](/TLS/8596800-3cc2687f183c768b.png)
从截图中可以看出TLS协议分为两个部分`记录协议（Record Layer）`和`握手协议（Handshake Protocal）`。
#### 3.1.1 记录协议（Record Layer）
记录协议根据rfc描述`记录协议（Record Layer）`有如下4种类型，即上图中Content Type可以取的值。

![record layer](/TLS/8596800-2c4eae95a03b544a.png)

`记录协议（Record Layer）` 数据结构
![Record layer数据结构](/TLS/8596800-ddcfde0e3d3e67ef.png)
对照着wireshark抓包为：**Content Type：Handshake(22), Version: TLS 1.0(0x0301), Length: 512**

#### 3.1.2 握手协议（Handshake Protocal）
`握手协议（Handshake Protocal）`有如下10种类型。
![Handshake Protocal](/TLS/8596800-9c8cdd20d3cab84f.png)

`握手协议（Handshake Protocal）`数据结构
![image.png](/TLS/8596800-97772640970b40ba.png)
对照着wireshark抓包为：**Handshake Type: Client Hello, Length: 508, Version : TLS 1.2(0x0303)**。 注意记录协议和握手协议的版本可以不一样

#### 3.1.3 客户端支持的加密套件（Cipher suite）
wireshark抓包显示客户端支持的加密套件有31种。**cipher的格式为：认证算法_密钥交换算法_加密算法_ 摘要算法**。server会从这些算法组合中选取一组，作为本次SSL连接中使用。
![cipher suite](/TLS/8596800-120aed05adc144bb.png)


###（2）Server Hello
![Server Hello](/TLS/8596800-31a87530a7f34bf7.png)
消息格式和Client Hello差不多，不再赘述。其中**Server选择的加密算法是Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)**

###（3）Certificate、ServerKeyExchange、ServerHelloDone
![image.png](/TLS/8596800-381cdbda6fba5ea3.png)
1. Certificate
 我们知道https在建立连接的时候是用的非对称加密（RSA），在实际数据传输阶段是用的对称加密（AES等）。非对称加密的是需要获取server段的公钥的。但是这个公钥不是谁生成的都可以用的（存在中间人攻击）。所以需要专门的机构发行的证书才行。那么这个证书就是这里的Certificate，公钥就藏在里面。

2. ServerKeyExchange
![image.png](/TLS/8596800-945321e661053a73.png)

这个包告诉我们服务器和客户端是通过Diffie-Hellman算法来生成最终的密钥（也就是Sessionkey会话密钥），pubkey是Diffie-Hellman算法中的一个参数，这个参数需要通过网络传给客户端，即使它被截取也不会影响安全性。具体见参考文献三。
**在client hello和server hello包中都有Random字段，上面我们是没有说明的，client和server这两个随机数加Diffie-Hellman算法生成出来的这个key。一共三个数字生成最终对称加密的key**

3. ServerHelloDone
根据rfc描述，ServerHelloDone发送完后，等待client回复，client需要验证证书
![image.png](/TLS/8596800-81ea6c9d59c1909d.png)

###（4）Client Key Exchange、Change Cipher Spec、Encrypted Handshake Message(Finishd)
![image.png](/TLS/8596800-9f532cb4982df87f.png)

1. Client Key Exchange
客户端收到服务器发来的ServerKeyExchange包来之后，运行Diffie-Hellman算法生成一个pubkey，然后发送给服务器。通过这一步和上面ServerKeyExchange两个步骤，服务器和客户端分别交换了pubkey，这样他们就可以分别生成了一个一样的sessionkey

2. Change Cipher Spec
指示Server从现在Client开始发送的消息都是加密过的
3. Encrypted Handshake Message
client发送encrypted handshake message, server如果解密没有问题，那么说明之前发送到数据没有被篡改。

###（5）Change Cipher Spec、Encrypted Handshake Message(Finishd)
1. Change Cipher Spec
指示client从现在Server开始发送的消息都是加密过的
2. Encrypted Handshake Message
与client发送的Encrypted Handshake Message目的一样
### （6）Application Data
![Application Data](/TLS/8596800-8aad89f6ff67fba9.png)
从现在开始发送的数据就是采用上述步骤协商出的对称加密密钥加密过的数据了

#### 3.1.4 总结
再回过头来看一下这张图，加密过程总结如下：
![TLS认证过程](/TLS/8596800-51037345e4727e0f.png)

1. client 发送ClientHello，携带一个随机数
2. server 发送ServerHello,携带一个随机数
3. server发送Certificate，携带证书，证书中含有server段的公钥
4. server和client都发送了KeyExchange,通过Diffie-Hellman密钥协商出一个key（也相对于一个随机数）
5. 通过上面三个随机数生成一个key，后面的加密过程都是用的这个key
6. server和client都会用生成的key来发送一条加密后的消息。来验证整个流程的安全性。










# 参考文献
1. [SSL/TLS协议运行机制的概述](https://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
2. [TLS协议报文解析](https://www.jianshu.com/p/607b38c15ccd)
3. [Diffie-Hellman密钥协商算法](https://www.cnblogs.com/qcblog/p/9016704.html)
4. [彻底搞懂HTTPS的加密原理](https://zhuanlan.zhihu.com/p/43789231)
