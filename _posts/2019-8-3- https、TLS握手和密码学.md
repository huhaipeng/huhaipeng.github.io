---
layout:     post
title:     https、TLS握手和密码学
subtitle: 
date:       2019-8-3
author:     HHP
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    -密码学
---

## 前言

http和https是我们上网经常会看到的字眼，但http和https到底是什么，有什么不同，又有什么联系？本人因为工作需要，也因为自己的兴趣，一直有接触和研究密码学的知识。今天就借本文说一下自己的见解。



## 正文

### 预备知识：

#### 密码学

密码学三大原则：安全性、不可否认性、消息完整性。

我的理解密码学就是加密和解密两个过程。加密就是用密钥通过加密算法加密你的明文信息，解密就是用密钥通过加密算法来解密还原你的信息。加密可以分为两大类：**对称加密和非对称加密**。

* 对称加密：双方具有相同的密钥，使用同样的加密算法来加解密。这个很好理解。
* 非对称加密：双方各自具有自己的一对密钥对，包括一条私钥和一条密钥。私钥是自己保管的，绝不能泄露出去，公钥可以公布出去。非对称加密的神奇之处在于用公钥加密的信息，能且只能通过对应的私钥解密出来。由于私钥是自己保管的，所以中间人即使窃听到你的加密报文，会因为没有私钥而不能解密。具体关于非对称加密的知识大家可以到网上查找，这里不展开。著名的非对称加密算法有RSA 、ECDH等。

#### 消息摘要

消息摘要（message digest）的意思简单来理解就是给你的消息分配一个唯一的id(其实不能严格说唯一，因为消息摘要一般用哈希，哈希会有可能冲突)，这个id专属于你这条消息。你这个消息只要改了任意一个地方，这个改了的消息的消息摘要都会和之前没改的那条消息的消息摘要不同。通过消息摘要，可以确保你的消息完整性，可以用来判断你的消息是否被删改。常用的消息摘要算法有MD5，SHA1，SHA256等等。他们的原理都是将你的消息通过**哈希算法**生成一定位数的ID。双方通过比较这个ID来确保消息的完整性。

#### 证书

至于证书，顾名思义，其实就是服务器的一个身份证明，证明我这个服务器是可靠的。证书里面包含了服务器的公钥，这是证书里面最重要的部分，之后协商密钥的时候就是用的这个公钥。证书是由CA颁发的，是CA签过名的。所以和现实生活一样，证书的含金量和可靠性又多大，取决于证书的颁发机构有多权威和可信赖的程度有多大。实际上，CA都是比较可以信赖的，这个可以放心。证书详细的使用过程下文会讲到。这里大家先有个概念。







### http和https

http全称hyper text transfer protocol，中文名是超文本传输协议，是应用层的一种常见协议，有自己的通讯格式。这里不具体介绍http协议，只是强调http是一种**明文**传输的协议。

https全称hyper text transfer protocol secure，中文名超文本传输安全协议。和http相比，只多了一个s，所以，https还是本质上还是基于http的。

那https和http有什么不同呢？实际上，https只是在http层下多了一层安全套接字层（Secure Socket Layer，简称SSL），这个SSL，也是应用层的一个东西。这个SSL有什么意义呢？正如其名，就是为了安全。因为正如刚才所强调的，http是一个明文传输协议，你发给对方的消息，黑客完全可以从中窃听到你的报文，完全没有安全和隐私可言。自从有了ssl，这个问题解决了。

那么具体来说，https传输整个流程和http传输有什么不同呢？其实，就是多了一个SSL/TLS握手的过程。普通的http传输，在经过tcp三次握手之后就开始传输http数据了；而https，则是在三向交握之后，先进行TLS握手，握手过程中协商好对称密钥，握手完之后在每次发送http数据之前，都会先经过SSL层利用握手协商好的密钥进行加密之后再发送出去。对方在接收到加密后的http数据之后经过ssl层解密然后再交给http层。整个过程就是这样，也很清爽。

下面来详细介绍一下TLS握手详细知识以及握手流程。



### SSL/TLS握手

#### SSL/TLS的发展

SSL是TLS的前身，其实二者是一样的，都是安全协议。TLS是SSL的升级版本。发展的路线是从SSL 1.0 --> SSL 2.0 --> SSL 3.0 -->TLS-->1.0 -->TLS 2.0 -->TLS 3.0。TLS 1.3在 [RFC 8446](https://tools.ietf.org/html/rfc8446) 中定义，于2018年8月发表，是最新的安全协议。但是目前用的最多的还是 TLS 2.0 协议。实际上，SSL1.0版本从未公开过，因为存在严重的安全漏洞。而2.0版本在1995年2月发布，但因为存在数个严重的安全漏洞而被3.0版本替代。但有的比较就的客户端还是会使用SSL 2.0，所以服务端也要做支持。

所谓SSL/TLS握手，其实就是客户端与服务端共同协商出对称密钥来进行通讯的过程。下面我以TLS 2.0为例讲一下整个握手流程。

#### TLS握手流程

##### client hello

下图是我向www.baidu.com发起的一个https请求。可以看到，tcp三次握手完成之后，客户端首先发了一个`client hello` 报文，这个`client hello`报文主要是要告诉server端自己支持什么样的**密码学套件**，所以这个报文最重要的字段是`cipher suites`。可以看到，这里客户端传了31个密码学套件给server端。之后server端收到这个client hello报文之后会从中选择一个密码学套件来进行接下来的握手。

密码学套件是什么东西？其实简单理解就是用于双方协商密钥的时候要共同遵循的一套密码学规定。那么一个密码学套件包含了什么东西？大家可以看到，这个密码学套件的格式都是一样的。我以`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`这个密码学套件为例，其他的都是一样的去分析。其中第一个字段`TLS`是使用TLS协议的意思。然后第二个字段`ECDHE`是协商对称密钥时所使用的非对称加密算法，常用的还有`RSA`,` DH`, `ECDH`, `ECDHE`。第三个字段`RSA`是验证证书的算法。WITH 前面除去`TLS`一般都会有两个字段,不过有时我们会看到例如这种`TLS_RSA_WITH_AES_256_GCM_SHA384`,即WITH前面只有一个`RSA`,说明协商对称密钥和验证证书的算法都是`RSA`。WITH后面的`AES_256_GCM`字段说明在加密http数据是所使用的对称加密算法，最后的`SHA384`是消息摘要算法，在每次传输加密数据的时候发送方都要生成一个消息摘要，而对方则需要验证这个消息摘要。

上面提到了非对称加密和证书两个概念。这里插个问题，**为什么要搞那么复杂要用非对称加密来协商对称加密密钥，直接用非对称加密算法不就好了吗？像ssh一样，又安全又不用那么复杂的流程。**

下面来解答一下。

确实，非对称加密很安全，但是非对称加密有个两个缺点：**加解密慢**和**计算过程消耗cpu大**。我们在访问资源的时候，都希望能得到快速的响应。若每次请求和收到信息都要经过一次非对称加密，用户的体验肯定十分的差。而且对于服务器来说，要处理十万百万的并发，这对服务器的cpu负载会造成多大的负载啊！！而对称加密虽然没那么安全，但却运算的比较快，效率高，这就是第一个问题的解答。

差点忘了，client hello报文还有一个很很重要的字段就是`Random`,大家可以看到下图的Random是一大串16进制的数字，这个随机数是用在之后生成密钥中的。

![](https://b2.bmp.ovh/imgs/2019/08/52eab0af0509909f.png)





##### server hello

如下图，client hello报文结束之后，就是服务器回复 server hello报文了。server hello报文中比较重要的`Random`字段和`cipher Suite`字段。Random是服务器产生的一个随机数，要发给客户端，也是用作生成密钥。cipher Suite字段是服务器从client hello报文的一大堆的密码学套件中选择出来的一个套件，之后client和server之间的握手就都用这个密码学套件了，这里服务器选择的密码学套件是`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`。至于服务器怎么选择，在nginx里面是可以很简单的进行配置的。还有一个字段可以看到，是`session ID`。之后client用这个session id去握手时，可以复用上一次的握手信息，减少一次握手的RTT的时间开销。其实除了sesion id之外，还有一种叫session ticket的技术，和cookie原理差不多，有兴趣可以自己去了解，这里不展开。

![](https://b2.bmp.ovh/imgs/2019/08/0327a7e4bee5ecd9.png)



##### Certificate (TLS握手过程的核心)

服务器发完server hello 报文之后，紧接着就会发证书（certificate）给cilent了。上文有提及到，证书是由CA颁发的，证书里面包含了服务器的公钥，之后的对称密钥协商就是用的这个公钥来进行的。整个握手过程其实最终就是为了传个公钥给client，所以这步是核心。

下面第二幅图是我导出的百度的certificate，用openssl命令解码出来的证书。(这里解码不是解密的意思，只是将base64解码而已，**证书其实是明文传输的**)。

上文说到，百度的服务器最终选择的是`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`这个密码学套件，也就是说协商密钥所使用的非对称加密算法是ECDHE，而验证服务器的算法是RSA。

ECDHE和RSA算法这里简单说一下，这是一种基于椭圆曲线的加密算法，相比于RSA算法，它具有前向保密的优点。用ECDHE算法，每次https会话都会客户端和服务端都会生成各自的临时使用的用于协商密钥的公钥私钥对，这样即使中间人足够牛逼，在通讯过程中获取到了某一方的私钥，ok，他能解出这次的会话内容，但是之前的会话他是解不出来的，因为每一次的公钥和私钥都是临时生成的，这就是前向保密的意思。而如果协商密钥的方法是RSA的话，那么每一次协商密钥都会使用证书里面的公钥（对应这里就是下图二的`RSA Public-Key`字段下面的那2048bit）来协商密钥。在私钥保管的很好的情况下安全性当然没问题，但是只要私钥一泄漏，那些之前用这个公钥私钥对的数据全部都能被解出来，所以RSA用来这密钥的协商是没有前向保密性的。如果看了这里，觉得介绍的太简单很看懂的朋友，可以到这个链接看更详细的说明，这个链接说的很详细也很通俗 [扫盲 HTTPS 和 SSL/TLS 协议](https://program-think.blogspot.com/2016/09/https-ssl-tls-3.html) 。

好了，当client收到下面的证书之后，就要开始验证证书了。怎样验证呢？大家可以看到，我导出的证书最下面那里有一个`Signature Algorithm: sha256WithRSAEncryption`的字段，下面带了2048 bits（256字节）的数字。意思是证书的摘要是使用的SHA256哈希算法，签名算法使用的是RSA。

这个字段的2048bits数字是怎么来的呢？是这样的，我们到CA申请证书的时候，我们自己把域名和公钥都给了CA之后，CA帮我们弄好一张统计格式的的预证书,然后用消息摘要算法，像这里就是用了sha256算法，来对这个预证书生成256bits的哈希值。之后再用自己的私钥利用对应的非对称加密算法，像这里就是用了RSA算法，来对这个哈希值进行签名，生成2048bits的密文。预证书加上这个签名字段，就是一张完整的证书了。

当客户端收到这张证书的时候，客户端会同样的使用SHA256哈希算法来对这张证书进行哈希得出一个256bits的哈希值（就是我们说的消息摘要），暂且用h1来表示这个哈希值。然后用客户端本身保存的对应的CA的公钥解密`Signature Algorithm: sha256WithRSAEncryption`字段下面的那串2048bits的数字，还原出正确的那个消息摘要h2。然后比较h1和h2的值，若相等，则说明证书是可靠的，没被人篡改过。若证书在传输过程中被篡改过，那么h1是不会等于h2的。为什么这样的确保证书的可靠性？其实本质是CA的私钥是自己保管的，我们默认CA是不会把私钥泄露出去的。这里的逻辑稍微有一点绕，大家可以慢慢想想这样做的原因和必要性。

之后，若h1==h2，那么client会验证域名是否对应，像这里就是验证我客户端请求的域名是否是`X509v3 Subject Alternative Name`下面那一堆域名中的一个，若是，则整个验证通过，进行下一步。

顺带提一句，其实传证书是可以双向的，即如果服务器有要求，客户端也要传自己的证书给服务器。不过现实中，一般都只是client验证server就好，刚才说的双向认证是在server端对client端也有验证要求，安全性比较高的情况下才会使用。

![](https://b2.bmp.ovh/imgs/2019/08/bf5335a2cc54c565.png)

```shell
haley@ubuntu:~/Desktop$ openssl x509 -in baidu.cert.cer -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            2c:ee:19:3c:18:82:78:ea:3e:43:75:73
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = BE, O = GlobalSign nv-sa, CN = GlobalSign Organization Validation CA - SHA256 - G2
        Validity
            Not Before: May  9 01:22:02 2019 GMT
            Not After : Jun 25 05:31:02 2020 GMT
        Subject: C = CN, ST = beijing, L = beijing, OU = service operation department, O = "Beijing Baidu Netcom Science Technology Co., Ltd", CN = baidu.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:b4:c6:bf:da:53:20:0f:ea:40:f3:b8:52:17:66:
                    3b:36:01:8d:12:b4:99:0d:d3:9b:6c:18:53:b1:19:
                    08:b0:fa:73:47:3e:0d:3a:79:62:78:61:2e:54:3c:
                    49:7c:56:da:c0:be:61:55:d5:42:70:6a:10:be:f5:
                    bd:8d:64:96:21:00:93:63:09:87:b7:19:ba:0e:20:
                    3e:49:c8:53:ed:02:8f:46:01:eb:a1:07:93:73:bb:
                    ed:f1:b3:c9:e2:fb:dd:f0:39:2a:83:ad:f4:41:98:
                    bc:86:ea:ba:74:a8:a6:e3:d0:e5:c5:8e:b3:0b:b2:
                    d2:ac:91:74:0e:ff:80:10:23:36:62:65:08:b4:87:
                    f5:57:0c:25:c7:00:d8:f5:a8:5d:b8:33:41:a7:2a:
                    5f:db:fa:70:9e:21:bb:ae:42:16:66:07:69:fe:1c:
                    26:2a:81:0f:ab:73:e3:d6:52:20:a4:6d:a8:6c:d4:
                    66:48:a4:6f:f2:68:0a:c5:65:a1:4e:bf:04:7a:40:
                    43:1c:d3:75:fb:75:ac:19:d6:4a:35:05:6e:cf:d5:
                    65:d1:44:ca:6b:0c:58:04:c4:85:4f:1f:be:2c:32:
                    d1:f1:c6:28:fb:f9:26:36:b5:6d:fa:cb:96:a2:a0:
                    d0:bc:f8:51:df:07:44:bd:8f:6f:67:c0:d4:af:d9:
                    cd:c3
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            Authority Information Access: 
                CA Issuers - URI:http://secure.globalsign.com/cacert/gsorganizationvalsha2g2r1.crt
                OCSP - URI:http://ocsp2.globalsign.com/gsorganizationvalsha2g2

            X509v3 Certificate Policies: 
                Policy: 1.3.6.1.4.1.4146.1.20
                  CPS: https://www.globalsign.com/repository/
                Policy: 2.23.140.1.2.2

            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 CRL Distribution Points: 

                Full Name:
                  URI:http://crl.globalsign.com/gs/gsorganizationvalsha2g2.crl

            X509v3 Subject Alternative Name: 
                DNS:baidu.com, DNS:click.hm.baidu.com, DNS:cm.pos.baidu.com, DNS:log.hm.baidu.com, DNS:update.pan.baidu.com, DNS:wn.pos.baidu.com, DNS:*.91.com, DNS:*.aipage.cn, DNS:*.aipage.com, DNS:*.apollo.auto, DNS:*.baidu.com, DNS:*.baidubce.com, DNS:*.baiducontent.com, DNS:*.baidupcs.com, DNS:*.baidustatic.com, DNS:*.baifae.com, DNS:*.baifubao.com, DNS:*.bce.baidu.com, DNS:*.bcehost.com, DNS:*.bdimg.com, DNS:*.bdstatic.com, DNS:*.bdtjrcv.com, DNS:*.bj.baidubce.com, DNS:*.chuanke.com, DNS:*.dlnel.com, DNS:*.dlnel.org, DNS:*.dueros.baidu.com, DNS:*.eyun.baidu.com, DNS:*.fanyi.baidu.com, DNS:*.gz.baidubce.com, DNS:*.hao123.baidu.com, DNS:*.hao123.com, DNS:*.hao222.com, DNS:*.im.baidu.com, DNS:*.map.baidu.com, DNS:*.mbd.baidu.com, DNS:*.mipcdn.com, DNS:*.news.baidu.com, DNS:*.nuomi.com, DNS:*.safe.baidu.com, DNS:*.smartapps.cn, DNS:*.ssl2.duapps.com, DNS:*.su.baidu.com, DNS:*.trustgo.com, DNS:*.xueshu.baidu.com, DNS:apollo.auto, DNS:baifae.com, DNS:baifubao.com, DNS:dwz.cn, DNS:mct.y.nuomi.com, DNS:www.baidu.cn, DNS:www.baidu.com.cn
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Key Identifier: 
                76:B5:E6:D6:49:F8:F8:36:EA:75:A9:6D:5E:4D:55:5B:37:5C:FD:C7
            X509v3 Authority Key Identifier: 
                keyid:96:DE:61:F1:BD:1C:16:29:53:1C:C0:CC:7D:3B:83:00:40:E6:1A:7C

            CT Precertificate SCTs: 
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : BB:D9:DF:BC:1F:8A:71:B5:93:94:23:97:AA:92:7B:47:
                                38:57:95:0A:AB:52:E8:1A:90:96:64:36:8E:1E:D1:85
                    Timestamp : May  9 01:22:04.826 2019 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:20:2C:7B:4D:C0:F9:85:47:8A:2D:0A:C0:79:
                                3B:D6:B4:B5:66:F8:AA:FB:82:58:AD:23:36:FE:16:BC:
                                A6:83:99:21:02:21:00:C0:2F:CD:9C:99:20:CB:7D:91:
                                5F:D2:8B:C6:13:10:73:B5:C1:54:03:33:41:9F:A6:6A:
                                C5:14:93:CF:69:2B:6B
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 6F:53:76:AC:31:F0:31:19:D8:99:00:A4:51:15:FF:77:
                                15:1C:11:D9:02:C1:00:29:06:8D:B2:08:9A:37:D9:13
                    Timestamp : May  9 01:22:03.983 2019 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:20:03:32:68:9E:39:D0:EB:5F:19:61:DB:A7:
                                12:69:6F:28:44:81:02:A5:3C:C2:A3:13:D5:7E:98:26:
                                5F:20:1A:A0:02:21:00:A7:8B:62:B3:B0:B4:44:32:E2:
                                11:FF:45:8D:55:11:2C:36:AB:29:93:44:C8:34:5C:CE:
                                7C:35:57:31:AE:AB:12
    Signature Algorithm: sha256WithRSAEncryption
         aa:b9:cd:52:8e:dc:36:5d:47:d4:8b:f3:32:17:06:46:83:60:
         a3:27:05:49:29:b1:1b:46:6e:38:fe:93:fe:09:43:6c:d2:a1:
         58:24:12:42:b7:ab:41:f8:47:0a:7d:64:b5:75:dc:5a:45:14:
         b2:a4:18:6b:9c:b7:3b:8f:b3:7e:d2:bd:c0:72:4b:35:05:ae:
         0d:2d:19:1f:50:73:72:5a:df:97:18:3b:db:2a:f3:de:44:ce:
         64:2d:c1:1e:84:cc:76:24:3e:30:67:23:26:e8:4f:f7:0b:f6:
         ec:69:d7:7f:51:a9:a0:6f:b8:c4:14:e2:c0:4a:4a:c4:00:5d:
         57:6a:c9:41:c4:25:2b:32:18:aa:62:a8:1e:49:81:73:1c:81:
         5f:5e:fa:e4:94:32:c3:50:6d:8e:aa:cc:6c:4c:53:0c:fa:8f:
         4e:34:79:9f:a5:60:c0:f8:50:75:b8:a1:9d:01:e6:ab:25:23:
         0c:3b:24:02:40:58:24:ff:34:02:8b:94:61:10:68:2f:b6:80:
         e3:d0:5f:4a:0a:a7:02:d2:c0:98:3e:1d:e8:02:c8:27:71:26:
         b2:a8:87:b6:db:9d:10:47:4b:c2:13:62:34:c6:d0:3c:39:09:
         39:25:8f:fe:a2:f4:f3:fb:df:9b:27:3d:fc:d0:28:e8:6d:dc:
         dd:17:d3:1f

```



##### server key Exchange、server hello done、client Key Exchange

接下来大家看到的是服务器发来server key Exchange报文和客户端发来的client Key Exchange报文。其实这两个报文不是在所有的握手流程中都会出现。这里出现的原因是密钥协商算法使用的是ECDHE算法，ECDHE就是在server key Exchange和client Key Exchange两个报文中交换双方临时生成的公钥，然后根据椭圆曲线来计算出预主密钥密钥。想稍微深入了解具体过程的可以看我上面给的链接[扫盲 HTTPS 和 SSL/TLS 协议](https://program-think.blogspot.com/2016/09/https-ssl-tls-3.html)。对于RSA算法来说，是不需要server key Exchange和client Key Exchange这两个报文的，服务器发了证书给client之后，直接发sever hello done就好了。之后会利用之前双方的随机数和预主密钥来进行计算出对称密钥。

![](https://b2.bmp.ovh/imgs/2019/08/642d3fc7cac04198.png)

##### Change cipher Spec 

这个Change Cipher是不携带任何数据的，只是一个双方宣告信道开始的流程。

![](https://b2.bmp.ovh/imgs/2019/08/5d56e2b1ae516c65.png)

##### Encrypted Handshake Message

握手最后，双方都会发送Encrypted Handshake Message报文。客户端发送的叫 Client Finished ，也是整个握手过程中，用协商好的对称密钥发送给服务端的第一个加密消息。这个Finish数据包保证了整个握手过程中，数据不能被篡改。任何中间人改动了中间的某一个数据，都逃脱不掉最后的哈希运算的检查。因为这个哈希运算是完整的计算从握手开始到结束的所有内容的。

服务端接收后，服务端用同样的方式计算出已交互的握手消息的摘要，与用密钥解密后的消息进行对比，一致的话，说明两端生成的主密钥一致，完成了密钥交换。

之后服务端也会发送自己的 Client Finished。和上面的客户端的 Encrypted Handshake Message 一样，是服务端发出的第一条加密信息。

客户端按照协商好的主密钥解密并验证正确后，握手阶段完成。

![](https://b2.bmp.ovh/imgs/2019/08/6100b8f1e44917cd.png)

##### Application Data

之后就是使用对称密钥来加解密http数据来进行通讯啦~~



友情连接：https://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247484753&idx=2&sn=ac47c7e8ae6d930140103871426b6af4&chksm=fda52c95cad2a583ba642a65408698ffa4b8a7855960d808a5247d28f660aff2187dc6f817dd&mpshare=1&scene=1&srcid=&sharer_sharetime=1571985334967&sharer_shareid=4d3052ace5c6c8d6e00b94921c37dcb6&key=5635ff6e36e9525ed207cfd83c8d036aa59abe353fb870f9ce9c8e5cd76fa46f9554f4a9d64880c16db93158ea1e9b3b396550a77be863525c67301358f2476fe0b14968ce4b48ec357e34a3bb4f04a9&ascene=1&uin=MjE3MzA5MjIyMA%3D%3D&devicetype=Windows+10&version=62070141&lang=zh_CN&pass_ticket=aP9LA%2B1J9KqbPR%2B4N%2BULLu%2FuG5mWWoBWVL4FQogxbKax9G48nmzRYEV3xvfT9eKH



--END--

![](https://b2.bmp.ovh/imgs/2019/08/7f38f46059f59fd6.jpg)