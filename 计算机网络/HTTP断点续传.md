@[toc]
## HTTP断点续传
HTTP 1.1 开始支持获取文件的部分内容，这为并行下载以及断点续传提供了技术支持。

他是通过Header里面的两个参数实现的。客户端发送请求是Range，服务端相应时对应的是Content-Range。

### Range
用于请求头中，指定第一个字节的位置和最后一个字节的位置。

一般格式为

	Range:[开始字节]-[结束字节]

例如

	Range:0-10		请求0-10字节的内容
	Range:10-		请求10字节之后的内容
	Range:-100		请求倒数100个字节

- 如果结束字节小于开始字节，则sever会忽略这个Range请求并回应一个200.
- 如果结束字节大于文件长度，那么这个Range请求被认为是不能满足的，并回应一个416

详细示例

	Range: bytes=0-499    		表示第 0-499 字节范围的内容 
	
	Range: bytes=500-999		表示第 500-999 字节范围的内容 
	
	Range: bytes=-500   		表示最后 500 字节的内容 
	
	Range: bytes=500-   		表示从第 500 字节开始到文件结束部分的内容 
	
	Range: bytes=0-0,-1 		表示第一个和最后一个字节 
	
	Range: bytes=500-600,601-999 	同时指定几个范围

### Content-Range
用在响应头中，在发出Range请求后，服务器会在Content-Range头部返回当前接受的范围和文件总大小。

	Content-Range:[开始字节]-[结束字节]

例如请求下载整个文件

```html
GET /test.rar HTTP/1.1 
Connection: close 
Host: 116.1.219.219 
Range: bytes=0-801   //一般请求下载整个文件是bytes=0- 或不用这个头
```

正常相应

```html
HTTP/1.1 200 OK 
Content-Length: 801      
Content-Type: application/octet-stream 
Content-Range: bytes 0-800/801   //801:文件总大小
```
在相应完成后，返回的响应头内容也不同

	HTTP/1.1 200 Ok（不使用断点续传方式） 
	HTTP/1.1 206 Partial Content（使用断点续传方式）

### 资源变化
有时在下载一些资源时，偶尔暂停一会然后再继续下载，可能会出现又重头下载的情况。

这看起来是HTTP的范围请求失效了，但也可能是因为你请求的资源在请求的过程中发生了改变。

在从服务器下载某个资源时，我们要注意资源可能发变动。

在HTTP协议中我们可以通过Etag或者Last-Modified来标识资源是否发生变化。

- ETag：当前文件的一个相关标记，用于标识文件的唯一性。
- Last-Modified：标记当前文件最后被修改的时间。

在HTTP的范围请求内，可以使用这两个字段来区分分段请求的资源，是否有修改过，只需要在请求头中，将它放在IF-Range这个请求头中即可，If-Range使用ETag或者Last-Modified两个参数任意一个，原样填入即可。

#### Last-Modified
If-Modified-Since，和 Last-Modified 一样都是用于记录页面最后修改时间的 HTTP 头信息，只是 Last-Modified 是由服务器往客户端发送的 HTTP 头，而 If-Modified-Since 则是由客户端往服务器发送的头，可以看到，再次请求本地存在的 cache 页面时，客户端会通过 If-Modified-Since 头将先前服务器端发过来的 Last-Modified 最后修改时间戳发送回去，这是为了让服务器端进行验证，通过这个时间戳判断客户端的页面是否是最新的，如果不是最新的，则返回新的内容，如果是最新的，则返回 304 告诉客户端其本地 cache 的页面是最新的，于是客户端就可以直接从本地加载页面了 。 

#### ETag
Etag 仅仅是一个和文件相关的标记，可以是一个版本标记，例如："v1.0.0"；或者说像 "627-4d648041f6b80" 这样的一串编码。



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190922192822109.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

这样一来，如果两次请求的资源文件没有发生改变就会返回206状态码，开始后续的操作。反之返回200状态码，表示文件发生改变，要从头下载。

### 小结
HTTP1.1协议定义了断点续传相关的HTTPRange和Content-Range字段。

一个简单的实现大概如下：

 1. 客户端下载一个1024K的文件，已经下载了其中512K 

  2. 网络中断，客户端请求续传，因此需要在HTTP头中申明本次需要续传的片段： Range:bytes=512000-  这个头通知服务端从文件的512K位置开始传输文件 

  3. 服务端收到断点续传请求，从文件的512K位置开始传输，并且在HTTP头中增加：Content-Range:bytes 512000-/1024000  并且此时服务端返回的HTTP状态码应该是206，而不是200。 