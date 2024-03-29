@[toc]
## 概述
HTTPS协议是以安全为目标的HTTP通道，是安全版的HTTP。

主要是在HTTP下加入SSL、TLS层。

SSL是HTTPS安全的基础。

HTTPS默认端口号为443。

HTTP存在的风险：

1. 窃听：HTTP采用明文传输
2. 篡改：第三方可以修改通信内容
3. 冒充：第三方冒充他人身份进行通信

SSL/TLS是为了解决这些风险设计的，他的作用有下：

1. 所有信息加密传输
2. 具有校验机制
3. 配备身份证书


## SSL
HTTPS并非是应用层的一种新协议。只是HTTP通信接口部分使用SSL和TLS协议代替。

通常HTTP直接和TCP通信。当使用SSL时，则演变成先和SSL通信，再由SSL和TCP通信。简言之，HTTPS其实就是身披SSL协议外壳的HTTP。

SSL采用一种叫做公开密钥加密的加密方式。

它具备以下特征：

- 使用公开的加密算法
- 密钥是保密的
- 使用密钥进行加密解密

这也意味着只有持有密钥就能够解密。如果密钥被攻击者获得，那么加密也就失去了意义。

公开密钥加密分为对称加密和非对称加密。

## 对称加密
加密和解密同用一个密钥的方式称为共享密钥加密，也叫对称加密。

## 非对称加密
非对称加密使用两把密钥，一把叫做私有密钥，一把叫做公开密钥。

使用非对称加密的方式，发送密文的一方使用对方的公开密钥进行加密处理，对方收到被加密的信息后，再是使用自己的私有密钥进行解密。

这种方式不需要发送用来解密的私有密钥，也不必担心密钥被攻击者盗走。

## 数字签名
数字签名是用来确保数据完整性的技术，他可以证明数据是来自谁，证明是否未被篡改过。

数字签名是附加在数据上的一段特殊的加密过的校验码，使用数字签名有以下几个好处。

1. 数字签名可以证明数据的作者是谁，因为数字签名是由数据作者用只有作者本人才知道的私钥生成的校验和，因此，这些校验和就像作者的个人签名一样。
2. 数据签名可以防止数据被篡改，攻击者在没有私钥的情况下，以目前的技术还无法篡改伪造出正确的校验和。

以下是数字签名在传输的一个应用流程：

假设结点A要向结点B发送一段数据：

1. 结点A对要发送的报文生成报文摘要。
2. 结点A用自己的私钥对摘要执行签名函数生成数字签名。
3. 结点A把数字签名附在要发送的报文之后，然后把报文和签名一并发送给结点B。
4. 结点B收到报文后，取出数字签名，用公钥对签名执行反函数得到摘要。
5. 结点B对报文生成报文摘要
6. 对4、5生成的报文摘要进行比较。如果一致则未被修改过。

## 数字证书
有了数字签名就可以验证数据的完整性了，但是这样还存在一个漏洞。使用数字签名时，接收方需要有一个公钥获得报文摘要，若攻击者篡改了公钥，那么就可以用自己的公钥发送篡改后的数据了。

为了解决这一问题，引出了数字证书。

数字证书由数字认证结构颁发，申请人提出申请后，认证机构会对申请人的公开密钥做数字签名，然后分配已签名的密钥。

有了数字签名后，客户端和服务端通信时会有以下流程：

1. 建立SSL时，服务端下发自己的证书给客户端。
2. 客户端拿到数字证书后会向数字认证机构验证公开密钥证书上的数字签名。
3. 验证通过后，客户端从证书取出公开密钥，用这个密钥进行和服务端的通信。

## 加密机制
SSL的加密方式是把两种加密方式混合起来使用的。

非对称加密性能要比对称加密性能慢，在建立SSL连接时，会使用非对称密钥加密，在SSL安全连接建立成功后，会使用对称加密密钥加密。


## 完整的HTTPS通信

**步骤1：**
	
`客户端`发送Client Hello报文开始SSL通信。报文中包含客户端支持的SSL指定版本、加密组件列表

**步骤2：**

`服务端`可进行SSL通信时，会以Server Hello报文作为应答。报文包括SSL版本、加密组件。加密组件是从客户端的加密组件内筛选出来的。

**步骤3：**

之后`服务器`发送Certificate报文。报文中包含公开密钥证书。

**步骤4：**

最后`服务端`发送Server Hello Done报文通知客户端，最初阶段的SSL握手协商部分结束。

**步骤5：**

SSL第一次握手结束后，`客户端`以Client Key Exchange报文作为回应，报文中包含一种称为Pre-master secret的随机密码串。该报文已用步骤3中的公钥加密。

**步骤6：**

接着`客户端`继续发送Change Cipher Spec报文。该报文会提示服务器，在此报文之后的通信会采用Pre-master secret密钥加密。

**步骤7：**

`客户端`发送FInish报文，该报文包含连接至今全部报文的整体校验值，这次握手协商是否成功，要以服务器是否能够解密该报文为判断标准。

**步骤8：**

`服务器`同样发送Change Cipher Spec报文。

**步骤9：**

`服务器`同样发送Finish报文。

**步骤10：**

`服务端和客户端`FInish报文交换完毕后，SSL连接就算建立完成，通信会受到SSL的保护。从此处开始进行应用层协议的通信，即发送HTTP请求。

**步骤11：**

应用层协议通信，即HTTP相应。

**步骤12：**

客户端发送close_notify断开连接。之后发送TCP FIN关闭TCP通信。

------

简单总结一下HTTPS通信，如下图。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190924221908243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

> 参考
> 《图解HTTP》