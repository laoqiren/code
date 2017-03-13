---
title: TLS/SSL原理浅析
date: 2017-03-09 16:40:19
tags: [HTTPS,TLS,Web安全]
categories: 
- Web安全
---
做人做事，安全很重要，这纷繁复杂的Web世界亦如此。你肯定不希望你在网络上的通讯被别人窥探，篡改。

SSL/TLS正是担负保护我们传输数据的角色。在这里分享我关于HTTPS有关学习总结，系列包括原理分析，实战部署，安全问题等。
![](https://blog.cloudflare.com/content/images/2014/Sep/ssl_handshake_diffie_hellman.jpg)(图片来源网络)
<!--more-->

# 从宏观来看TLS协议

### OSI模型
TLS/SSL传输层安全协议位于OSI七层模型中的表示层，用于数据表示，转换和加密。下层为TCP/IP层，上层通过ALPN(应用层协议协商)扩展建立不同的应用层协议。

### 实现方式

TLS由记录协议实现（record protocol)实现，其上又有四个子协议： 握手协议（handshake protocol),密钥规格变更协议(change cipher spec protocol),应用数据协议(application data protocol)和警报协议(alert protocol)

1. 记录协议保证数据的完整性和安全性（MAC和加密)
2. 握手协议保证身份验证


# 数字签名原理

数字签名可以保证数据的完整性，这在我们后面进行证书验证要用到，这在我们进行数字签名需要用到RSA加密算法（非对称加密）

假设两个同学A,B，A要发送文件给B:

A用散列函数算法计算file的hash值如SHA256算法，可以保证不同的文档不一样的hash值，加入一些元素据（如哈希算法）进行编码，然后用自己的私钥进行加密编码数据，形成数字签名，加入到file中，发送给B。

B收到file后，用相同hash算法计算文件的hash值，用A的公钥解密，解码得到文档中的hash值，将两个hash值进行对比，如果两者相同，就能'确认'file来自A且没有经过更改。

可见这种签名方法结合了hash和RSA用私钥加密数据的方式。

但是这里还有一个问题，B如何确保得到的公钥是来自A的？这就是我们后面要讲的公钥证书(certificate)了。

# 身份验证

身份验证用来保证数据发送给指定的目标了，这个和TSL握手协议中的密钥交换部分紧密联系。

身份验证主要用RSA算法，还是上面的A,B，A发送数据给B,A为了保证将数据发送给B,A用B的公钥加密数据，然后B拿到后用自己的私钥解密。这样就相当于身份验证了。但是RSA有个问题就是前向保密性问题，如果B的私钥被别人获取，那么A和B之前所有的交流都透露出去了。所以我们在进行密钥交换时，一般用DH方式，采用ECDH Params和RSA结合的方式。


# 握手协议过程详解

这里以我对github抓包数据为例进行分析。

### ClientHello
客户端将功能和首选项发送给服务器，如协议版本，随机数（客户端Random,这个会用于后面生成premaster key 即预主钥)，Session ID可以用于恢复会话，Cipher Suites表示能够接受的密码套件，Compression压缩，Extensions扩展

抓包：
![clientHello](http://7xsi10.com1.z0.glb.clouddn.com/client.jpg)
第一次Session ID为空表明客户端不想恢复之前的会话

### ServerHello

服务器选择一些参数，返回给客户端，如选择的密码套件，扩展，另外还要发送Random(服务端random,与上面的client random都要参与生成预密钥)
![serverHello](http://7xsi10.com1.z0.glb.clouddn.com/serverSayHello.jpg)

我们可以看到SessionTickets扩展（可以用于恢复会话）和ALPN扩展（之前提过），图中还可以看到这次握手选择的套件：
```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```
表示使用ECDHE密钥交换方式，用RSA算法进行身份验证，AES加密算法用于对称加密，128表示加密强度，GCM表示加密模式，SHA256用于生成MAC或PRF
### Certificate

服务器发送自己的证书。证书可以用于服务端和客户端的相互验证，然后提供服务端的publick key给客户端，用于后面密钥交换的身份验证工具。

**我们来看看客户端拿到证书后都干了什么：**

1. 客户端存有证书颁发机构（CA）的公钥，然后这个证书会经过CA签名（就是在上面我讲的数字签名），客户端用CA的公钥解密证书，确保证书来自该CA

2. 从解密的证书中拿到服务端的公钥，客户端用这个公钥去解密证书，如果解密成功，说明这个证书是指定服务端发过来的。

### ServerkeyExchange

服务端发送用于密钥交换的额外数据，如选择DH交换算法时，这个阶段会带上DH Server Params,如文章首页图所示一样。

### ServerHelloDone
服务端表明握手信息发送结束。

**上面的三个Certificate,ServerkeyExchange,ServerHelloDone实际上是一次性发送的**

### ClientKeyExchange
客户端发送密钥交换所需要的客户端信息，如ECDH交换算法中，会携带DH Client Params,如文章首页图所示。
![clientExchange](http://7xsi10.com1.z0.glb.clouddn.com/clientExchange.jpg)

**在这里我们来分析密钥交换的具体过程（这里以ECDH交换方式为例）：**
1. 服务端在SeverKeyExchange中包含了后面会用到的DH Server Params
2. 客户端在验证服务端的证书后，发送DH Client Params

3. 这个时候客户端和服务端都有了两个DH Params,就可以生成Premaster secret了

4. 两端使用DH Params加上两个随机数（两个Hello发送的Random)，就可以生成master secret了

5. 两端再根据master secret加上两个随机数就可以生成Key了

主密钥生成：
```
master_secret = PRF(premaster,"master secret",ClientHello.random,ServerHello.random)
```
最终Key生成：
```
key_blok = PRF(master_secret, "key expansion",server_random+client_random)
```
nice!
### ChangeCipherSpec
表明已经生成Key，客户端和服务端在生成好密钥后，就可以开始使用对称加密技术用生成的Key进行加密传输数据了。

# 其他细节

### 服务端要求客户端身份验证

再ServerKeyExchange后再发送CertificateRequest请求，客户端就得和server一样发送自己的证书

### 会话恢复

**Session ID**
之前在ClientHello提到的Session ID派上用场了，首次握手它是空的，表明不想恢复之前的会话，之后的握手，可以将Session ID放如Session ID参数，如果服务端愿意恢复，就带上相同的Session ID,然后使用之前协商的主密钥生成新的Key,再进行加密传输数据。

**session ticket**
作为一种扩展引入的一种新的会话恢复机制。
服务器取出所有会话数据进行加密，以票证方式返回客户端，接下来客户端将票证提交回server验证解密来恢复会话。但是规格也允许客户端像上面一样发送Session ID,服务端还是的回复同样的值。

我们在第一次访问github后不久再去访问看看，抓包：
![sessionID](http://7xsi10.com1.z0.glb.clouddn.com/sessionID.jpg)

会发现后面握手就简单的几步：ClientHello, ServerHello,Server标识完成，Client标识完成

会话恢复可以减少时间，减少资源消耗。

# 结束语
文章是我结合一些文章和书籍和自己的理解后总结的，欢迎讨论。接下来还会从实战部署，安全问题等方面进一步总结HTTPS。

### 参考资料

数字签名:[http://www.youdzone.com/signature.html](http://www.youdzone.com/signature.html)

TLS握手: [https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/)




