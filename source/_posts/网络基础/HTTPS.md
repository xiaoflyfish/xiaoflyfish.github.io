---
title: HTTPS
categories: 
- 网络基础
---

**HTTP与HTTPS的区别**

HTTP 是明文传输协议，HTTPS 协议是由 SSL+HTTP 协议构建的可进行加密传输、身份认证的网络协议，比 HTTP 协议安全

- HTTPS比HTTP更加安全，对搜索引擎更友好，利于SEO,谷歌、百度优先索引HTTPS网页;
- HTTPS需要用到SSL证书，而HTTP不用;
- HTTPS标准端口443，HTTP标准端口80;
- HTTPS基于传输层，HTTP基于应用层;
- HTTPS在浏览器显示绿色安全锁，HTTP没有显示;

**工作原理**

HTTPS 协议会对传输的数据进行加密，而加密过程是使用了非对称加密实现。但其实，HTTPS 在内容传输的加密上使用的是对称加密，非对称加密只作用在证书验证阶段

HTTPS的整体过程分为证书验证和数据传输阶段，具体的交互过程如下：

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201205203648.png)

1.Client发起一个HTTPS（比如`https://juejin.im/user/4283353031252967`）的请求

2.Server把事先配置好的公钥证书返回给客户端。

3.Client验证公钥证书：比如是否在有效期内，证书的用途是不是匹配Client请求的站点，是不是在CRL吊销列表里面，它的上一级证书是否有效，这是一个递归的过程，直到验证到根证书（操作系统内置的Root证书或者Client内置的Root证书）。如果验证通过则继续，不通过则显示警告信息。

4.Client使用伪随机数生成器生成加密所使用的对称密钥，然后用证书的公钥加密这个对称密钥，发给Server。

5.Server使用自己的私钥解密这个消息，得到对称密钥。至此，Client和Server双方都持有了相同的对称密钥。

6.Server使用对称密钥加密“明文内容A”，发送给Client。

7.Client使用对称密钥解密响应的密文，得到“明文内容A”。

8.Client再次发起HTTPS的请求，使用对称密钥加密请求的“明文内容B”，然后Server使用对称密钥解密密文，得到“明文内容B”。