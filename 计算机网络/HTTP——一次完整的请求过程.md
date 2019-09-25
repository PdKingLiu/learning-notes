@[toc]
## 概述
当我们点击一个网址后，它能够呈现在我们的面前，在这个过程中，究竟发生了什么。

整体流程如下：

1. DNS查询  
2. 三次握手   
3. TLS/SSL握手 
4. HTTP请求   
5. 四次挥手断开连接

## 详细过程

### 1. DNS查询
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925145929171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
浏览器会使用DNS协议返回域名所对应的的IP地址。

DNS是一个应用层协议，并且他选择的运输层协议是UDP。

在DNS服务器上，域名和和他所对应的IP地址存储为一条记录。

所有的记录都不可能只存储在一台服务器上，无论多么强大的服务器都扛不住全球上亿次的并发量。

大致来说，有三种DNS服务器，**跟服务器，顶级域DNS服务器和权威DNS服务器**。

顶级域DNS服务器主要负责诸如com、org、net等顶级域名。

跟DNS服务器存储了所有顶级域DNS服务器的IP地址，也就是说可以通过根服务器找到顶级域服务器。

然后可以选择一个顶级服务器，请求该顶级域服务器，顶级服务器拿到域名后应当做出判断并给出负责当前域的权威服务器地址。以百度为例的话，顶级域服务器将返回所有负责 baidu 这个域的权威服务器地址。

选择一个权威服务器地址，向他继续查询相应域名对应的IP地址。最终权威服务器会返回具体的IP地址。


实际上DNS是有缓存功能的，首先会从现有的缓存中找。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925151419995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

上图是一个请求案例（假设请求www.xxxx.com）：

1. 主机向本地DNS发送查询报文，入如果本地服务器缓存中有，将直接返回结果。
2. 本地服务器返现缓存中没有，于是从内置的内部根服务器列表中选择一个发送查询报文
3. 根域名解析一下这个后缀名，告诉本地服务器负责.com的所有顶级服务器列表
4. 本地服务器选择一个顶级域服务器继续查询，.com顶级服务器拿到域名后继续解析，返回负责.xx域的所有权威服务器列表。
5. 本地服务区从返回的权威服务器选择一个再次发送报文，最终会从某一个服务器上得到具体的IP地址
6. 返回主机结果

> 例如查询www.linux178.com这个域名的IP
> 运营商的DNS服务器首先查找自身的缓存，找到对应的条目，且没有过期，则解析成功。如果没有找到对应的条目，则有运营商的DNS代我们的浏览器发起迭代DNS解析请求，它首先是会找根域的DNS的IP地址（这个DNS服务器都内置13台根域的DNS的IP地址），找打根域的DNS地址，就会向其发起请求（请问www.linux178.com这个域名的IP地址是多少啊？），根域发现这是一个顶级域com域的一个域名，于是就告诉运营商的DNS我不知道这个域名的IP地址，但是我知道com域的IP地址，你去找它去，于是运营商的DNS就得到了com域的IP地址，又向com域的IP地址发起了请求（请问www.linux178.com这个域名的IP地址是多少?）,com域这台服务器告诉运营商的DNS我不知道www.linux178.com这个域名的IP地址，但是我知道linux178.com这个域的DNS地址，你去找它去，于是运营商的DNS又向linux178.com这个域名的DNS地址（这个一般就是由域名注册商提供的，像万网，新网等）发起请求（请问www.linux178.com这个域名的IP地址是多少？），这个时候linux178.com域的DNS服务器一查，诶，果真在我这里，于是就把找到的结果发送给运营商的DNS服务器，这个时候运营商的DNS服务器就拿到了www.linux178.com这个域名对应的IP地址，并返回给Windows系统内核，内核又把结果返回给浏览器，终于浏览器拿到了www.linux178.com  对应的IP地址，该进行一步的动作了。

### 2. 三次握手
**回顾TCP、UDP**
TCP、UDP是传输层的两个协议，前者基于连接的可靠传输协议，后者是无连接的不可靠传输协议。前者适合于一些对数据完整性要求高的场合，后者适合允许数据丢失但对传输速率要求特别高的场景。

**报文格式**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925154734865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925154825479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

TCP的三次握手是为了确保通讯双方能够稳定的建立连接并完成数据报文的请求与相应动作。



**第一步**

- 客户端向服务端发送一份特殊的TCP报文，该报文不包含数据，是一份特殊的报文，它的首段`SYN`字段值为1。

- 客户端还会生成一个初始序列号，填在报文的`seq（假设x）`字段，表示当前报文序号是这个，并且后续的分组会基于这个序号递增。

**第二步**

- 如果分组丢失了，那么客户端会经过某个时间间隔再次尝试发送。

- 分组到达服务端后，服务端拆开报文，会看到这是一个特殊的`SYN`握手报文。于是为此连接分配资源。

- 然后服务端开始构建相应报文，回发一个连接应答`SYN`，响应报文中的`SYN`会被置为1，`ACK`=1，并且服务器也将生成一个初始`seq（假设y）`放置在相应报文字段中。

- 最后，服务端给相应报文中的确认字段赋值，这个值就是客户端发来的那个`seq + 1）`。

整体意思就是：**我同意你的请求，我的初始序号为xxx，你的初始序号我已经接收**。

**第三步**

客户端收到服务端的相应报文，于是分配TCP连接需要的资源。

客户端发送最后的确认，序列号为`x + 1`，确认号为`y + 1`，控制位`SYN=0`，`ACK=1`

连接建立，以后的每份报文中，`SYN`均为0。

整个流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925172905701.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

### 3. SSL握手
***这个过程参考上篇文章***

[HTTPS——学习总结](https://blog.csdn.net/CodeFarmer__/article/details/101229749#HTTPS_88)

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

![https://img-blog.csdnimg.cn/20190924221908243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70](https://img-blog.csdnimg.cn/20190924221908243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 4. HTTP请求  
建立连接后就可以发送HTTP请求了

详细内容参考

[HTTP协议——学习总结](https://blog.csdn.net/CodeFarmer__/article/details/101104669)

### 5. 四次挥手
接下来就是拆除一个TCP连接的过程（四次挥手）。

TCP是全双工通信模式，服务端和客户端两方其实是相对的，谁是客户单谁是服务器是相对的。

所以TCP连接的断开两方都可主动请求断开连接。

假设客户端请求断开：

**第一步**

客户端构建特殊TCP报文，FIN置为1。

**第二步**

服务端收到FIN报文，于是相应客户端一个ACK报文，告诉客户端，请求关闭报文已经收到。

**第三步**

服务端发送一个FIN报文，告诉客户端，我将要关闭连接了。

**第四步**

客户端返回一个ACK相应报文，告诉服务端，我收到了你发的报文了，你可关闭连接了。

主要流程如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925174207686.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 首先客户端发出请求释放连接。报文首部FIN=1，序列号为seq=u，发送完报文后，客户端进入FIN-WAIT-1状态。
- 服务端接收到连接释放请求，发出确认报文，ACK=1，ack=u+1，序列号seq=v，服务器进入CLOSE-WAIT状态。
- 客户端收到服务器的确认请求后，进入FIN-WAIT-2状态。等待服务端发送连接释放报文。
- 服务器发送完最后的数据，向客户端发送最后的释放报文。FIN=1，ack=u+1，序列号seq=w，此时服务端进入LAST-ACK状态。等待客户端确认。
- 客户端接收到连接释放报文，发出确认报文，ACK=1，ack=w+1，seq=u+1，此时客户端进入TIME-WAIT等待状态。
- 服务端收到客户端发出的确认进入CLOSE状态。

------

**整体举例回顾一下：**

A没有东西要发给B了，请求释放连接。A发送一个报文，FIN设置为1，进入FIN-WAIT-1状态，等待B回发一个确认，此时A还可以接收数据，不再发送消息。

B收到释放连接后会给回发A一个确认，进入CLOSE-WAIT状态，继续发送还没发送完的数据。

A接收到B的确认信号进入FIN-WAIT-2状态。

B发送完剩余的数据后向A发送一个释放请求，进入LAST-ACK状态。等待A的确认。

A收到B的释放报文后，发出确认报文。进入TIME-WAIT状态。

B收到A的确认后立刻进入CLOSE状态。

A的TCP连接还没没有释放，必须经过2*MSL（报文最长寿命）时间后才进入CLOSE状态。

## 为什么不是二次握手
建立三次握手主要是因为A发送了再一次的确认，那么A为什么会再确认一次呢，主要是为了防止已失效的连接请求报文段又突然传送给B，从而产生了错误。

当客户端发送了连接请求，但是在某个网络结点滞留了较长时间，以致误到请求释放的某个时间到达服务器，其实它实际上只是一个已失效的连接请求报文段。但是服务端收到此失效的请求后，误以为客户端需要进行连接并将确认报文段发送给客户端，同意建立连接，但实际上A并没有想要发送连接请求，不理会服务器的确认，也不会给服务器发送数据。这样**导致服务器一直在等待**。因此服务器的许多资源就会浪费了。但是使用三次握手的方式就不会出现这样的情况。对于上面的情况，客户端不理会服务端，就会知道A不要求建立连接，也就不会发生资源的浪费。

## 为什么不是三次挥手
当服务器收到客户端发来的SYN请求报文后，可以直接发送SYN+ACK报文。其中SYN是用来同步的，ACK是用来应答的。但是关闭TCP连接时，服务器并不会立刻关闭连接，他必须先回复一个ACK报文，告诉客户端：“你发的FIN报文我收到了，等我的所有报文都发送完了，我才能发送FIN报文。”所以需要四次挥手。

## 为什么等待2MSL
2MSL是一份报文在网络中最长的时间。超过该时间报文都会被丢弃。而如果客户端最后的确认报文于网络中丢失的话，服务端必将发起超时请求，重新发送第三次挥手动作，此时等待中的客户端就可随即重新发送一份确认请求。

这是为什么客户端等待一个最长报文传输时间的原因。有人可能好奇为什么前面的各次请求都没有做超时等待而只最后一次数据发送做了超时等待？

原因很简单，就是 TCP 自带计时能力，超过一定时间没有收到某个报文的确认报文，会自动重新发送，而这里如果不做等待而直接关闭连接，那么我如何知道服务端到底收到没我的确认报文呢。

通过等待一个最长周期，如果这个周期内没有收到服务端的报文请求，那么我们的确认报文必然是到达了服务端了的，否则重复发送一次即可。

## 为什么HTTP基于TCP
目前在Internet中所有的传输都是通过TCP/IP进行的，HTTP协议作为TCP/IP模型中应用层的协议也不例外，TCP是一个端到端的可靠的面向连接的协议，所以HTTP基于传输层TCP协议不用担心数据的传输的各种问题。

> 参考
> [https://mp.weixin.qq.com/s?src=11&timestamp=1569377175&ver=1873&signature=hhpRCEtqhrJEB2uaHVNHoXwN5XuvXJqQ6NXn1FwdOZT77h437Rdg6YLbPoWsihr4ekkcWJ97aoQWEEhh5VLMQBORZvZXXjANYLjLM6Q8TXo8kGh3gc2LpUGeV5lYIv91&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1569377175&ver=1873&signature=hhpRCEtqhrJEB2uaHVNHoXwN5XuvXJqQ6NXn1FwdOZT77h437Rdg6YLbPoWsihr4ekkcWJ97aoQWEEhh5VLMQBORZvZXXjANYLjLM6Q8TXo8kGh3gc2LpUGeV5lYIv91&new=1) 
> [https://mp.weixin.qq.com/s?src=11&timestamp=1569374232&ver=1873&signature=RW1*PVsIi5aHUa9RteFFoa6mvKU2Ye55jAUkYEA0hrFZE20g8e5QGE*sSVYFNv5iPLkJOn5IAjo6nNTDg9BZUMkIK86UO6pYjwh4zezshL6Do84eyc3hKyCWsb*OaVJT&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1569374232&ver=1873&signature=RW1*PVsIi5aHUa9RteFFoa6mvKU2Ye55jAUkYEA0hrFZE20g8e5QGE*sSVYFNv5iPLkJOn5IAjo6nNTDg9BZUMkIK86UO6pYjwh4zezshL6Do84eyc3hKyCWsb*OaVJT&new=1)
> [https://mp.weixin.qq.com/s?src=11&timestamp=1569374232&ver=1873&signature=MRNK-pfQ56bIevKdYgM8zBAJfQ-T4yt88lVBgp1rN637hTE4lJl6u74Icpb6ywCcIRbXUSw0S*aICTzUt567y0AT2oyaXlfbNWaO1wtzU6Y8cmJnRsWRcfiXDj8-T6JW&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1569374232&ver=1873&signature=MRNK-pfQ56bIevKdYgM8zBAJfQ-T4yt88lVBgp1rN637hTE4lJl6u74Icpb6ywCcIRbXUSw0S*aICTzUt567y0AT2oyaXlfbNWaO1wtzU6Y8cmJnRsWRcfiXDj8-T6JW&new=1)
> [https://www.jianshu.com/p/4cbb1bf4771c](https://www.jianshu.com/p/4cbb1bf4771c)

