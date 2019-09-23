@[toc]
## 概述
缓存是为了在下次使用资源时，能够快速的相应，避免多次IO操作。
HTTP协议中，天然支持对缓存的支持。

## 缓存
HTTP缓存主要是通过请求和相应报文头中对应的Header信息来控制缓存。

这要涉及两个Header。

- Catch-Control：设定缓存策略，超出时间是多少。
- ETag：当前返回数据的验证令牌，可能是Hash值，也可能是其他值，主要是用于在下次请求的时候带上，让服务端以此判断当前数据是否有更改。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190923162320553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
服务端在返回相应数据的时候，会在报文头中添加用于描述当前相应的内容类型、数据长度、缓存策略（Catch-Control）、验证令牌（ETag）等信息。

上图是一次请求相应的事务，客户端请求一个资源的时候，服务端返回了一个200的状态码，表示相应正常，相应的数据长度为1024字节，建议客户端将此资源缓存最多120秒，并提供一个ETag令牌，用来当做当前资源版本的唯一标识。

## ETag 数据令牌
在没有数据令牌的情况下，大概步骤是这样的：

1. 客户端会首先找到本地的缓存，然后发现它已经失效，我发再次使用
2. 客户端再次发起请求获取资源并缓存。然后再更新缓存超过时间。

max-age，只是一个参考值，是需要平衡的，太短会导致请求频繁，太长又会导致无法及时刷新客户端资源。

这就是ETag要解决的问题。

服务端生成并返回这个ETag，通常他是Hash值或者其他指纹，客户端无需关心他的规则，只需知道它是当前数据的一个唯一标识。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190923171233416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

客户端在请求下载资源时，将通过`If-Nono-Match`这个请求报头，将ETag发送至服务端。

如果ETag和服务端的一致，服务端返回一个304状态码（不包含资源内容），则表明资源没有发生变化，可使用本地缓存数据，并刷新超时时间。

>If-None-Match，他标识比较ETag是否不一致。

## Catch-Control
之前只为Catch-Control设置了一个max-age，但是其实还有一些其他配置。

1. no-cache 和 no-store
	这两个参数都表示每一次请求，都需要发送一个网络请求。
	他们之间的区别是
	**no-cache：** 并不是真的不缓存数据，他只是每次都确认资源是否过期，利用ETag令牌一定程度上减少传输流量
	**no-store：** 要求客户端每次都重新请求资源，不做任何缓存处理。
2. public 和 private
	**public：** 是一种默认的策略，表示当前缓存是开放的，任何请求想用的中间环节都可以对其进行缓存。
	**private：** 则表示当前相应是针对单个用户的，不建议对其进行缓存。


![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092318030331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)