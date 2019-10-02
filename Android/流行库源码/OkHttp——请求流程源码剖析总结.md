@[toc]
基本的用法不赘述了，在官网就能看到。
[https://github.com/square/okhttp](https://github.com/square/okhttp)

## OkHttp 整体的请求流程
### 1. 请求处理开始
当需要请求网络时，我们会调用。

```java
 client.newCall(request).enqueue();
 client.newCall(request).execute();
```
他们都是从`newCall()`调用

```java
  @Override public Call newCall(Request request) {
    return new RealCall(this, request);
  }
```
newCall()方法new了一个RealCall对象，他的enqueue请求源码
```java
  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
  }
```
execute请求源码：

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain(false);
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

能够看到他们的最终的请求是`dispatcher`来完成的。

### 2. Dispather任务调度
Dispatcher主要用于任务的调度。

他主要有如下变量

```java

  private int maxRequests = 64;//最大请求数
  
  private int maxRequestsPerHost = 5;//最大请求主机数

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;//消费者线程池

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();//将要运行的异步任务队列

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();//正在运行的异步任务队列

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();//正在运行的同步任务队列
```

他的构造方法如下

```java
  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
Dispatcher可以接收用户自己设置的线程池，如果没有设置线程池则创建默认的线程池。

当调用enqueue方法实际上是调用了Dispatcher的enqueue方法

```java
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

如果满足
- 运行的异步线程小于最大请求数量
- 运行主机数小于5

则会把他加入到线程池并执行

否则把他加入到等待队列。

后过头看一下Dispatcher当时传入的参数。

	client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
	Dispaer.enqueue(AsyncCall call) {}

最终给线程池传入的是一个AsyncCall对象，他实现了Runnable，run方法会调用AsyncCall的execute方法。

```java
  boolean signalledCallback = false;
  try {
    Response response = getResponseWithInterceptorChain(forWebSocket);
    if (canceled) {
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    if (signalledCallback) {
      // Do not signal the callback twice!
      logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}
```
暂且先看无论如何都会执行的`client.dispatcher().finished(this);`

```java
  synchronized void finished(AsyncCall call) {
    if (!runningAsyncCalls.remove(call)) throw new AssertionError("AsyncCall wasn't running!");
    promoteCalls();
  }
```
他首先将这个请求从runningAsyncCalls中移除了，然后调用了promoteCalls()方法。

```java
  private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```
可以很清晰的看到这实际上就是取出下一个请求并执行。

让我们再回到AsyncCall的execute方法。

```java
​```java
  boolean signalledCallback = false;
  try {
    Response response = getResponseWithInterceptorChain(forWebSocket);
    if (canceled) {
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    if (signalledCallback) {
      // Do not signal the callback twice!
      logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}
```
注意这句代码

	Response response = getResponseWithInterceptorChain(forWebSocket);

他返回了一个Response 对象，这就是在请求网络。

### 3. Interceptor拦截器
跟进到getResponseWithInterceptorChain()方法。

```java
  private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
    Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
  }
```
`getResponseWithInterceptorChain()`方法创建了ApplicationInterceptorChain对象，他是RealCall的内部类，是一个拦截器。

然后他执行了proceed方法。

```java
@Override public Response proceed(Request request) throws IOException {
   // If there's another interceptor in the chain, call that.
   if (index < client.interceptors().size()) {
     Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
     //从拦截器列表中取出拦截器
     Interceptor interceptor = client.interceptors().get(index);
     Response interceptedResponse = interceptor.intercept(chain);

     if (interceptedResponse == null) {
       throw new NullPointerException("application interceptor " + interceptor
           + " returned null");
     }

     return interceptedResponse;
   }

   // No more interceptors. Do HTTP.
   return getResponse(request, forWebSocket);
 }
```

proceed方法每次从拦截器列表取出拦截器。当存在多个拦截器时会在`Response interceptedResponse = interceptor.intercept(chain);`阻塞，并等待下一个拦截器的调用返回。

拦截器是一种能够监控、重写、重试调用的机制。通常情况下，拦截器用来添加、移除、转换请求和相应头部信息。比如将域名替换为IP地址，在请求头中添加host属性等等。

后头看最后一行

`return getResponse(request, forWebSocket);`

如果没有更多拦截器的话就会执行网络请求。

```java
  /**
   * Performs the request and returns the response. May return null if this call was canceled.
   */
  Response getResponse(Request request, boolean forWebSocket) throws IOException {
    // Copy body metadata to the appropriate request headers.
    RequestBody body = request.body();
    if (body != null) {
      Request.Builder requestBuilder = request.newBuilder();

      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }

      request = requestBuilder.build();
    }

    // Create the initial HTTP engine. Retries and redirects need new engine for each attempt.
    engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null);

    int followUpCount = 0;
    while (true) {
      if (canceled) {
        engine.releaseStreamAllocation();
        throw new IOException("Canceled");
      }

      boolean releaseConnection = true;
      try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
      } catch (RequestException e) {
        // The attempt to interpret the request failed. Give up.
        throw e.getCause();
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        HttpEngine retryEngine = engine.recover(e.getLastConnectException(), null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }
        // Give up; recovery is not possible.
        throw e.getLastConnectException();
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        HttpEngine retryEngine = engine.recover(e, null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }

        // Give up; recovery is not possible.
        throw e;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          StreamAllocation streamAllocation = engine.close();
          streamAllocation.release();
        }
      }

      Response response = engine.getResponse();
      Request followUp = engine.followUpRequest();

      if (followUp == null) {
        if (!forWebSocket) {
          engine.releaseStreamAllocation();
        }
        return response;
      }

      StreamAllocation streamAllocation = engine.close();

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (!engine.sameConnection(followUp.url())) {
        streamAllocation.release();
        streamAllocation = null;
      }

      request = followUp;
      engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response);
    }
  }
```

有点长，其他的先放下。

注意到其中的这样一段代码
```java
 try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
      } 
```
看一看到创建了HTTPEngine类并调用了sendRequest和readResponse方法。

### 4. 缓存策略

```java
  public void sendRequest() throws RequestException, RouteException, IOException {
    if (cacheStrategy != null) return; // Already sent.
    if (httpStream != null) throw new IllegalStateException();

    Request request = networkRequest(userRequest);

	//获取client中的Cache，同时Cache在初始化时会读取缓存目录中曾经请求过的所有信息。
    InternalCache responseCache = Internal.instance.internalCache(client);
    Response cacheCandidate = responseCache != null
        ? responseCache.get(request)
        : null;

    long now = System.currentTimeMillis();
    cacheStrategy = new CacheStrategy.Factory(now, request, cacheCandidate).get();
  	//网络
    networkRequest = cacheStrategy.networkRequest;
    //缓存
    cacheResponse = cacheStrategy.cacheResponse;

    if (responseCache != null) {
      responseCache.trackResponse(cacheStrategy);
    }

	//缓存不可用
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

	//	不进行网络请求，也不存在缓存。返回504
    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      userResponse = new Response.Builder()
          .request(userRequest)
          .priorResponse(stripBody(priorResponse))
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_BODY)
          .build();
      return;
    }

	//不进行网络请求，但是缓存有效，返回缓存。
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      userResponse = cacheResponse.newBuilder()
          .request(userRequest)
          .priorResponse(stripBody(priorResponse))
          .cacheResponse(stripBody(cacheResponse))
          .build();
      userResponse = unzip(userResponse);
      return;
    }

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
	//    访问网络
    boolean success = false;
    try {
      httpStream = connect();
      httpStream.setHttpEngine(this);

      if (writeRequestHeadersEagerly()) {
        long contentLength = OkHeaders.contentLength(request);
        if (bufferRequestBody) {
          if (contentLength > Integer.MAX_VALUE) {
            throw new IllegalStateException("Use setFixedLengthStreamingMode() or "
                + "setChunkedStreamingMode() for requests larger than 2 GiB.");
          }

          if (contentLength != -1) {
            // Buffer a request body of a known length.
            httpStream.writeRequestHeaders(networkRequest);
            requestBodyOut = new RetryableSink((int) contentLength);
          } else {
            // Buffer a request body of an unknown length. Don't write request headers until the
            // entire body is ready; otherwise we can't set the Content-Length header correctly.
            requestBodyOut = new RetryableSink();
          }
        } else {
          httpStream.writeRequestHeaders(networkRequest);
          requestBodyOut = httpStream.createRequestBody(networkRequest, contentLength);
        }
      }
      success = true;
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (!success && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
  }
```

上面的代码显然实在进行网络请求，首先是对缓存做了处理。

cacheCandidate是上次与服务器交互缓存的Response，缓存基于Map，key是请求中url的md5，value是在文件中查询到的缓存，页面置换基于LRU算法，我们只需要知道cacheCandidate是一个可以读取缓存Header的Response即可。

然后根据cacheStrategy得到了networkRequest和cacheResponse这两个值，根据这两个值得数据是否为null，来进一步的处理，在networkRequest和CacheResponse都为null的情况下，也就是不进行网络请求并且缓存不存在或者过期，这是返回504,。当networkRequest为null时，也就是不进行网络请求，如果缓存可以使用则直接返回缓存，其他情况为请求网络。

然后就是readResponse()方法：

```java
  public void readResponse() throws IOException {
    if (userResponse != null) {
      return; // Already ready.
    }
    if (networkRequest == null && cacheResponse == null) {
      throw new IllegalStateException("call sendRequest() first!");
    }
    if (networkRequest == null) {
      return; // No network response to read.
    }

    Response networkResponse;

    if (forWebSocket) {
      httpStream.writeRequestHeaders(networkRequest);
      networkResponse = readNetworkResponse();
    } else if (!callerWritesRequestBody) {
      networkResponse = new NetworkInterceptorChain(0, networkRequest).proceed(networkRequest);
    } else {
      // Emit the request body's buffer so that everything is in requestBodyOut.
      if (bufferedRequestBody != null && bufferedRequestBody.buffer().size() > 0) {
        bufferedRequestBody.emit();
      }

      // Emit the request headers if we haven't yet. We might have just learned the Content-Length.
      if (sentRequestMillis == -1) {
        if (OkHeaders.contentLength(networkRequest) == -1
            && requestBodyOut instanceof RetryableSink) {
          long contentLength = ((RetryableSink) requestBodyOut).contentLength();
          networkRequest = networkRequest.newBuilder()
              .header("Content-Length", Long.toString(contentLength))
              .build();
        }
        httpStream.writeRequestHeaders(networkRequest);
      }

      // Write the request body to the socket.
      if (requestBodyOut != null) {
        if (bufferedRequestBody != null) {
          // This also closes the wrapped requestBodyOut.
          bufferedRequestBody.close();
        } else {
          requestBodyOut.close();
        }
        if (requestBodyOut instanceof RetryableSink) {
          httpStream.writeRequestBody((RetryableSink) requestBodyOut);
        }
      }

      networkResponse = readNetworkResponse();
    }

    receiveHeaders(networkResponse.headers());

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (validate(cacheResponse, networkResponse)) {
        userResponse = cacheResponse.newBuilder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();
        releaseStreamAllocation();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        InternalCache responseCache = Internal.instance.internalCache(client);
        responseCache.trackConditionalCacheHit();
        responseCache.update(cacheResponse, stripBody(userResponse));
        userResponse = unzip(userResponse);
        return;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    userResponse = networkResponse.newBuilder()
        .request(userRequest)
        .priorResponse(stripBody(priorResponse))
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (hasBody(userResponse)) {
      maybeCache();
      userResponse = unzip(cacheWritingResponse(storeRequest, userResponse));
    }
  }
```

这个方法主要是解析响应报头，如果有缓存就用缓存的数据，则用缓存的数据并更新缓存，否则就用网络请求返回的数据。

注意这句

      if (validate(cacheResponse, networkResponse)) 

这是检查缓存有效的方法。

```java
  private static boolean validate(Response cached, Response network) {
    if (network.code() == HTTP_NOT_MODIFIED) {
      return true;
    }

    // The HTTP spec says that if the network's response is older than our
    // cached response, we may return the cache's response. Like Chrome (but
    // unlike Firefox), this client prefers to return the newer response.
    Date lastModified = cached.headers().getDate("Last-Modified");
    if (lastModified != null) {
      Date networkLastModified = network.headers().getDate("Last-Modified");
      if (networkLastModified != null
          && networkLastModified.getTime() < lastModified.getTime()) {
        return true;
      }
    }

    return false;
  }
```
如果服务器返回的状态码是304，那么缓存有效。

如果缓存过期或者强制放弃缓存，则缓存策略全部交给服务器判断，客户端只需求发送GET请求即可。

条件GET请求有两种方式：一种是Last-Modified-Date，一种是ETag。这里使用Last-Modified-Date，通过缓存网络请求相应中的Last-Modified来计算是否是最新数据。

### 5. 失败重连
回到RealCall的getResponse方法

```java
  /**
   * Performs the request and returns the response. May return null if this call was canceled.
   */
  Response getResponse(Request request, boolean forWebSocket) throws IOException {
      ······
      boolean releaseConnection = true;
      try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
      } catch (RequestException e) {
        // The attempt to interpret the request failed. Give up.
        throw e.getCause();
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        //1. 失败重连
        HttpEngine retryEngine = engine.recover(e.getLastConnectException(), null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }
        // Give up; recovery is not possible.
        throw e.getLastConnectException();
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        //2. 失败重连
        HttpEngine retryEngine = engine.recover(e, null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }

        // Give up; recovery is not possible.
        throw e;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          StreamAllocation streamAllocation = engine.close();
          streamAllocation.release();
        }
      }

      Response response = engine.getResponse();
      Request followUp = engine.followUpRequest();

      if (followUp == null) {
        if (!forWebSocket) {
          engine.releaseStreamAllocation();
        }
        return response;
      }

      StreamAllocation streamAllocation = engine.close();

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (!engine.sameConnection(followUp.url())) {
        streamAllocation.release();
        streamAllocation = null;
      }

      request = followUp;
      engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response);
    }
  }
```

在上述代码的1、2处当发生IOException或RouteException时会执行

	HttpEngine retryEngine = engine.recover(e.getLastConnectException(), null);

```java
  public HttpEngine recover(IOException e, Sink requestBodyOut) {
    if (!streamAllocation.recover(e, requestBodyOut)) {
      return null;
    }

    if (!client.retryOnConnectionFailure()) {
      return null;
    }

    StreamAllocation streamAllocation = close();

    // For failure recovery, use the same route selector with a new connection.
    return new HttpEngine(client, userRequest, bufferRequestBody, callerWritesRequestBody,
        forWebSocket, streamAllocation, (RetryableSink) requestBodyOut, priorResponse);
  }
```
实际上就是重新创建了HttpEngine并返回来完成重建。

这就是OKHttp整体的请求流程。流程图如下：


![在这里插入图片描述](https://img-blog.csdnimg.cn/2019100214570589.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)