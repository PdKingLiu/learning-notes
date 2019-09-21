@[toc]
## 概述
HTTP协议，也叫超文本传输协议，是一种详细规定了浏览器和万维网服务器之间相互通信的规则。

HTTP通常承载于TCP协议之上，有时候也承载于TLS或SSL协议层之上，这个时候，就成了我们常说的HTTPS。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190921131937924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
HTTP是一个应用层协议，由请求和相应构成，是一个标准的客户端服务器类型。

HTTP默认端口号是80，HTTPS的端口号是443。


## 特点

HTTP是一个无状态的协议。无状态是指协议对于事物处理没有记忆能力，缺少状态意味着如果后续处理需要前面的消息，则他必须重传。

HTTP是无连接的，限制每次连接只处理一个发送请求，服务端处理完客户端的请求，并收到客户端的应答后，就立即断开，两种之间的传输不是连续性的。


## 工作流程
1. 首先客户机和服务器（默认端口80）建立一个TCP连接，HTTP的工作开始。
2. 建立连接后，客户机通过TCP套接字发送一个请求给服务器，请求的格式：一个请求行、若干请求头、一个空行以及实体内容。
3. 服务器接收请求，返回相应信息。其格式为一个状态行、若干响应头、一个空行以及实体内容。
4. 客户机接收数据，若为HTTP0.9/1.0，则直接释放连接，若为HTTP1.1 ，默认情况下connection模式为keepalive，所以该连接会保持一段时间，在该时间内可以继续接收请求，若connection模式为close，则服务器主动关闭连接。
5. 客户端处理数据。

## HTTP请求
**GET**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190921164719442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**POST**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190921170545948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

**请求行**
请求方式有七种，POST、GET、HEAD、OPTIONS、DELETE、TRACE、PUT。
常用的两种：GET、POST。
若没有设置，默认情况下向服务器发送的都是GET请求，如果需要更改请求方式，可通过更改表单的提交方式实现。
GET方式可以在URL地址后以?的形式带上交给服务器的数据，多个数据之间用&分隔。
GET的特点：在URL地址后附带的参数值有限的，其数据容量通常不超过1k。
POST方式，则可以在请求的实体内容中向服务器发送数据。

**请求头**
紧接着请求行（即第一行）之后的部分，用来说明服务器要使用的附加信息。如Host是请求的目的地址。

**空行**
空行，请求头部后面的空行是必须的

**实体内容**
可以添加任意的其他数据。

**HTTP的不同请求方法**

| 方法    | 作用                                                         |
| ------- | ------------------------------------------------------------ |
| GET     | 请求指定的页面信息，并返回实体主体                           |
| POST    | 向指定资源提交数据进行处理请求                               |
| HEAD    | 类似于GET请求，只不过返回的响应中没有具体的内容，用于获取报头 |
| PUT     | 从客户端向服务器传送的数据取代指定指定的文档内容             |
| DELETE  | 请求服务器删除指定的页面                                     |
| CONNECT | HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器       |
| OPTIONS | 允许客户端查看服务器性能                                     |
| TRACE   | 回显服务器收到的请求，主要用于测试和诊断                     |


## HTTP响应
HTTP响应也有四个部分组成：**状态行、消息报头、空行、相应正文**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092117071697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**状态行**
由HTTP协议版本号、状态码、状态消息三部分组成。

**消息报头**
用来说明客户端要使用的一些附加信息
例如上图：第二行和第三行和第四行为消息报头，Date:生成响应的日期和时间；Content-Type:指定了MIME类型的HTML(text/html),编码类型是ISO-8859-1

**空行**
消息报头后面的空行是必须的

**相应正文**
服务端返回给客户端的文本信息。

## 状态码
状态码有三位数字组成，第一个数组定义了相应的类别，分为五种类别。
1xx：**指示信息——表示请求已接收，继续处理。**
2xx：**成功——表示请求已被成功接收、理解、接受。**
3xx：**重定向——要完成请求必须进行更进一步的操作。**
4xx：**客户端错误——请求有语法错误或无法实现**
5xx：**服务端错误——服务器未能实现合法的请求**

**常见状态码**

| 状态码 |                       | 说明                                                         |
| ------ | --------------------- | ------------------------------------------------------------ |
| 200    | OK                    | 客户端请求成功                                               |
| 400    | Bad Request           | 客户端请求有语法错误，不能被服务器理解                       |
| 401    | Unauthorized          | 请求未经授权，这个状态码必须和www-Authenticate报头域一起使用 |
| 403    | Forbidden             | 服务器收到请求，但是拒绝提供服务                             |
| 404    | Not Found             | 请求资源不存在，例如：输入了错误的URL                        |
| 500    | Internal Server Error | 服务器发生了不可预期的错误                                   |
| 503    | Server Unavailable    | 服务器当前不能处理客户端的请求，一段时间后可能恢复正常       |

## 各版本HTTP
- HTTP 0.9是第一个版本的HTTP协议，已过时。它的组成极其简单，只允许客户端发送GET这一种请求，且不支持请求头。HTTP 0.9具有典型的无状态性，每个事务独立进行处理，事务结束时就释放这个连接。一次HTTP 0.9的传输首先要建立一个由客户端到Web服务器的TCP连接，由客户端发起一个请求，然后由Web服务器返回页面内容，然后连接会关闭。
- HTTP 1.0相对于0.9增加了如下特性。
	1. 请求与相应支持头域
	2. 相应对象以一个相应状态行开始
	3. 相应对象不只限于超文本
	4. 开始支持支持客户端通过POST方法向WEB服务器提交数据，支持GET、HEAD、POST方法。
	5. 短连接。每一个请求建立一个TCP连接，请求完成后立马断开连接。连接无法复用会导致每次请求都经历三次握手和慢启动。

- HTTP 1.1是目前使用最广泛的协议版本。引入了许多关键性能优化：keepalive连接，chunked编码传输，字节范围请求，请求流水线等。
	1. Persistent Connection（keepalive连接）：允许HTTP设备在事务处理结束之后将TCP连接保持在打开的状态，以便未来的HTTP请求重用现在的连接，直到客户端或服务器端决定将其关闭为止。在HTTP1.0中使用长连接需要添加请求头 Connection: Keep-Alive，而在HTTP 1.1 所有的连接默认都是长连接，除非特殊声明不支持（ HTTP请求报文首部加上Connection: close ）。服务器端按照FIFO原则来处理不同的Request。
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092120434435.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
	2.  chunked编码传输：该编码将实体分块传送并逐块标明长度，直到长度为0块表示传输结束，这在实体长度未知时特别有用(比如由数据库动态产生的数据)
	3.  字节范围请求：HTTP1.1支持传送内容的一部分。比方说，当客户端已经有内容的一部分，为了节省带宽，可以只向服务器请求一部分。该功能通过在请求消息中引入了range头域来实现，它允许只请求资源的某个部分。在响应消息中Content-Range头域声明了返回的这部分对象的偏移值和长度。如果服务器相应地返回了对象所请求范围的内容，则响应码206（Partial Content）
	4. 新增了一批Request method：HTTP1.1增加了OPTIONS,PUT, DELETE, TRACE, CONNECT方法。
	5. 缓存处理：HTTP/1.1在1.0的基础上加入了一些cache的新特性，引入了实体标签，一般被称为e-tags，新增更为强大的Cache-Control头。
	6. 请求消息和响应消息都支持Host头域：在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。因此，Host头的引入就很有必要了。

- HTTP 2.0 HTTP 2.0是下一代HTTP协议，目前应用还非常少。主要特点有：
	1. 多路复用（二进制分帧）。HTTP 2.0最大的特点：不会改动HTTP 的语义，HTTP 方法、状态码、URI 及首部字段，等等这些核心概念上一如往常，却能致力于突破上一代标准的性能限制，改进传输性能，实现低延迟和高吞吐量。而之所以叫2.0，是在于新增的二进制分帧层。在二进制分帧层上， HTTP 2.0 会将所有传输的信息分割为更小的消息和帧，并对它们采用二进制格式的编码 ，其中HTTP1.x的首部信息会被封装到Headers帧，而我们的request body则封装到Data帧里面。
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20190921204326146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
	2. HTTP 2.0 通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。相应地，每个数据流以消息的形式发送，而消息由一或多个帧组成，这些帧可以乱序发送，然后再根据每个帧首部的流标识符重新组装。
	3. 头部压缩：当一个客户端向相同服务器请求许多资源时，像来自同一个网页的图像，将会有大量的请求看上去几乎同样的，这就需要压缩技术对付这种几乎相同的信息。
	4. 随时复位：HTTP1.1一个缺点是当HTTP信息有一定长度大小数据传输时，你不能方便地随时停止它，中断TCP连接的代价是昂贵的。使用HTTP2的RST_STREAM将能方便停止一个信息传输，启动新的信息，在不中断连接的情况下提高带宽利用效率。
	5. 服务器端推流：Server Push。客户端请求一个资源X，服务器端判断也许客户端还需要资源Z，在无需事先询问客户端情况下将资源Z推送到客户端，客户端接受到后，可以缓存起来以备后用。
	6. 优先权和依赖：每个流都有自己的优先级别，会表明哪个流是最重要的，客户端会指定哪个流是最重要的，有一些依赖参数，这样一个流可以依赖另外一个流。优先级别可以在运行时动态改变，当用户滚动页面时，可以告诉浏览器哪个图像是最重要的，你也可以在一组流中进行优先筛选，能够突然抓住重点流。

## GET和POST的区别
1. get重点在从服务器上获取资源，post重点在向服务器发送数据。
2. get传输数据是通过URL请求，以field（字段）= value的形式，置于URL后，并用"?"连接，多个请求数据间用"&"连接，如http://127.0.0.1/Test/login.action?name=admin&password=admin，这个过程用户是可见的；post传输数据通过Http的post机制，将字段与对应值封存在请求实体中发送给服务器，这个过程对用户是不可见的；
3. Get传输的数据量小，因为受URL长度限制，但效率较高；
Post可以传输大量数据，所以上传文件时只能用Post方式；
4. get是不安全的，因为URL是可见的，可能会泄露私密信息，如密码等；
post较get安全性较高；
5. get方式只能支持ASCII字符，向服务器传的中文字符可能会乱码。
post支持标准字符集，可以正确传递中文字符。
## 常见HTTP首部字段
- 通用首部字段（请求报文与响应报文都会使用的首部字段）
	Date：创建报文时间
	Connection：连接的管理
	Cache-Control：缓存的控制
	Transfer-Encoding：报文主体的传输编码方式

- 请求首部字段请求报文会使用的首部字段）
	Host：请求资源所在服务器
	Accept：可处理的媒体类型
	Accept-Charset：可接收的字符集
	Accept-Encoding：可接受的内容编码
	Accept-Language：可接受的自然语言
	
- 相应首部字段（响应报文会使用的首部字段）
	Accept-Ranges：可接受的字节范围
	Location：令客户端重新定向到的URI
	Server：HTTP服务器的安装信息
	
- 实体首部字段（请求报文与响应报文的的实体部分使用的首部字段）
	Allow：资源可支持的HTTP方法
	Content-Type：实体主类的类型
	Content-Encoding：实体主体适用的编码方式
	Content-Language：实体主体的自然语言
	Content-Length：实体主体的的字节数
	Content-Range：实体主体的位置范围，一般用于发出部分请求时使用

## HTTP缺点和HTTPS
- 通信使用明文不加密，内容可能被窃听
- 不验证通信方身份，可能遭到伪装
- 无法验证报文完整性，可能被篡改

HTTPS就是HTTP加上加密处理（一般是SSL安全通信线路）+ 认证 + 完整性保护

## Cookie & Session
- Cookie
	Cookie 是Web 服务器发送给客户端的一小段信息，客户端请求时可以读取该信息发送到服务器端，进而进行用户的识别。对于客户端的每次请求，服务器都会将 Cookie 发送到客户端，在客户端可以进行保存，以便下次使用。Cookie 是可以被客户端禁用的。
- Session
	每一个用户都有一个不同的 session，各个用户之间是不能共享的，是每个用户所独享的，在 session 中可以存放信息。在服务器端会创建一个 session 对象，产生一个 sessionID 来标识这个 session 对象，然后将这个 sessionID 放入到 Cookie 中发送到客户端，下一次访问时，sessionID 会发送到服务器，在服务器端进行识别不同的用户。Session 的实现依赖于 Cookie，如果 Cookie 被禁用，那么 session 也将失效。

## Token
**CSRF**（Cross-site request forgery，跨站请求伪造），CSRF(XSRF) 顾名思义，是伪造请求，冒充用户在站内的正常操作。

为了防范CSRF攻击，目前主流的做法就是使用Token。

CSRF 攻击要成功的条件在于攻击者能够预测所有的参数从而构造出合法的请求。所以根据不可预测性原则，我们可以对参数进行加密从而防止 CSRF 攻击。

另一个更通用的做法是保持原有参数不变，另外添加一个参数 Token，其值是随机的。这样攻击者因为不知道 Token 而无法构造出合法的请求进行攻击。

Token使用原则：

- Token 要足够随机————只有这样才算不可预测。
- Token 是一次性的，即每次请求成功后要更新Token————这样可以增加攻击难度，增加预测难度。
- Token 要注意保密性————敏感操作使用 post，防止 Token 出现在 URL 中。