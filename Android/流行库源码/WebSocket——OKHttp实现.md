[toc]

# WebSocket协议概述
- Webscoket是Web浏览器和服务器之间的一种全双工通信协议。比如说，服务器可以在任意时刻发送消息给浏览器。

- WebSocket并不是全新的协议，而是利用了HTTP协议来建立连接。

- WebSocket连接由浏览器发起，请求协议是一个标准的HTTP请求，格式如下：
  ```java
  GET ws://localhost:3000/ws/chat HTTP/1.1
  Host: localhost
  Upgrade: websocket
  Connection: Upgrade
  Origin: http://localhost:3000
  Sec-WebSocket-Key: client-random-string
  Sec-WebSocket-Version: 13
  ```
该请求和普通的HTTP请求有几点不同：

- GET请求的地址不是类似/path/，而是以ws://开头的地址；
- 请求头Upgrade: websocket和Connection: Upgrade表示这个连接将要被转换为WebSocket连接；
- Sec-WebSocket-Key是用于标识这个连接，并非用于加密数据；
- Sec-WebSocket-Version指定了WebSocket的协议版本。

服务器如果接受该请求，就会返回如下响应：

```java
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: server-random-string
```
响应代码101表示本次连接的HTTP协议即将被更改，更改后的协议就是Upgrade: websocket指定的WebSocket协议。
# OKHttp实现
## 连接握手
```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  . . . . . .
  @Override public WebSocket newWebSocket(Request request, WebSocketListener listener) {
    RealWebSocket webSocket = new RealWebSocket(request, listener, new SecureRandom());
    webSocket.connect(this);
    return webSocket;
  }
```
这里会创建一个 RealWebSocket 对象，然后执行其 connect() 方法建立连接。

RealWebSocket 构造方法如下

```java
  public RealWebSocket(Request request, WebSocketListener listener, Random random) {
    if (!"GET".equals(request.method())) {
      throw new IllegalArgumentException("Request must be GET: " + request.method());
    }
    this.originalRequest = request;
    this.listener = listener;
    this.random = random;
    byte[] nonce = new byte[16];
    random.nextBytes(nonce);
    this.key = ByteString.of(nonce).base64();
    this.writerRunnable = new Runnable() {
      @Override public void run() {
        try {
          while (writeOneFrame()) {
          }
        } catch (IOException e) {
          failWebSocket(e, null);
        }
      }
    };
  }
```
主要的是初始化了 key，以备后续连接建立及握手之用。Key 是一个16字节长的随机数经过 Base64 编码得到的。此外还初始化了 writerRunnable 等。

**连接建立及握手过程如下：**

```java
  public void connect(OkHttpClient client) {
    client = client.newBuilder()
        .protocols(ONLY_HTTP1)
        .build();
    final int pingIntervalMillis = client.pingIntervalMillis();
    final Request request = originalRequest.newBuilder()
        .header("Upgrade", "websocket")
        .header("Connection", "Upgrade")
        .header("Sec-WebSocket-Key", key)
        .header("Sec-WebSocket-Version", "13")
        .build();
    call = Internal.instance.newWebSocketCall(client, request);
    call.enqueue(new Callback() {
      @Override public void onResponse(Call call, Response response) {
        try {
          checkResponse(response);
        } catch (ProtocolException e) {
          failWebSocket(e, response);
          closeQuietly(response);
          return;
        }
        // Promote the HTTP streams into web socket streams.
        StreamAllocation streamAllocation = Internal.instance.streamAllocation(call);
        streamAllocation.noNewStreams(); // Prevent connection pooling!
        Streams streams = streamAllocation.connection().newWebSocketStreams(streamAllocation);
        // Process all web socket messages.
        try {
          listener.onOpen(RealWebSocket.this, response);
          String name = "OkHttp WebSocket " + request.url().redact();
          initReaderAndWriter(name, pingIntervalMillis, streams);
          streamAllocation.connection().socket().setSoTimeout(0);
          loopReader();
        } catch (Exception e) {
          failWebSocket(e, null);
        }
      }
      @Override public void onFailure(Call call, IOException e) {
        failWebSocket(e, null);
      }
    });
  }
```
其主要完成的动作：

向服务器发送一个HTTP请求，HTTP 请求的特别之处在于，它包含了如下的一些Headers：

```java
Upgrade: WebSocket
Connection: Upgrade
Sec-WebSocket-Key: 7wgaspE0Tl7/66o4Dov2kw==
Sec-WebSocket-Version: 13
```
其中 Upgrade 和 Connection header 向服务器表明，请求的目的就是要将客户端和服务器端的通讯协议从 HTTP 协议升级到 WebSocket 协议，同时在请求处理完成之后，连接不要断开。

Sec-WebSocket-Key header 值正是我们前面看到的key，它是 WebSocket 客户端发送的一个 base64 编码的密文，要求服务端必须返回一个对应加密的 “Sec-WebSocket-Accept” 应答，否则客户端会抛出 “Error during WebSocket handshake” 错误，并关闭连接。

来自于 HTTP 服务器的响应到达的时，则表明连接已建立。

尽管连接已经建立，还要为数据的收发做一些准备。

**这些准备中的第一步就是检查 HTTP 响应：**

```java
  void checkResponse(Response response) throws ProtocolException {
    if (response.code() != 101) {
      throw new ProtocolException("Expected HTTP 101 response but was '"
          + response.code() + " " + response.message() + "'");
    }
    String headerConnection = response.header("Connection");
    if (!"Upgrade".equalsIgnoreCase(headerConnection)) {
      throw new ProtocolException("Expected 'Connection' header value 'Upgrade' but was '"
          + headerConnection + "'");
    }
    String headerUpgrade = response.header("Upgrade");
    if (!"websocket".equalsIgnoreCase(headerUpgrade)) {
      throw new ProtocolException(
          "Expected 'Upgrade' header value 'websocket' but was '" + headerUpgrade + "'");
    }
    String headerAccept = response.header("Sec-WebSocket-Accept");
    String acceptExpected = ByteString.encodeUtf8(key + WebSocketProtocol.ACCEPT_MAGIC)
        .sha1().base64();
    if (!acceptExpected.equals(headerAccept)) {
      throw new ProtocolException("Expected 'Sec-WebSocket-Accept' header value '"
          + acceptExpected + "' but was '" + headerAccept + "'");
    }
  }
  . . . . . .
  public void failWebSocket(Exception e, Response response) {
    Streams streamsToClose;
    synchronized (this) {
      if (failed) return; // Already failed.
      failed = true;
      streamsToClose = this.streams;
      this.streams = null;
      if (cancelFuture != null) cancelFuture.cancel(false);
      if (executor != null) executor.shutdown();
    }
    try {
      listener.onFailure(this, e, response);
    } finally {
      closeQuietly(streamsToClose);
    }
  }
```
其中的检测如下：
- 响应码是 101。
- “Connection” header 的值为 “Upgrade”，以表明服务器并没有在处理完请求之后把连接个断开。
- “Upgrade” header 的值为 “websocket”，以表明服务器接受后面使用 WebSocket 来通信。
- “Sec-WebSocket-Accept” header 的值为，key + WebSocketProtocol.ACCEPT_MAGIC 做 SHA1 hash，然后做 base64 编码，来做服务器接受连接的验证。关于这部分的设计的详细信息，可参考 WebSocket 协议规范。

**第二步：**

为数据收发做准备的第二步是，初始化用于输入输出的 Source 和 Sink。Source 和 Sink 创建于之前发送HTTP请求的时候。这里会阻止在这个连接上再创建新的流。

```java
  public RealWebSocket.Streams newWebSocketStreams(final StreamAllocation streamAllocation) {
    return new RealWebSocket.Streams(true, source, sink) {
      @Override public void close() throws IOException {
        streamAllocation.streamFinished(true, streamAllocation.codec());
      }
    };
  }
```
Streams是一个 BufferedSource 和 BufferedSink 的holder：

```java
  public abstract static class Streams implements Closeable {
    public final boolean client;
    public final BufferedSource source;
    public final BufferedSink sink;
    public Streams(boolean client, BufferedSource source, BufferedSink sink) {
      this.client = client;
      this.source = source;
      this.sink = sink;
    }
  }
```

**第三步是调用回调 onOpen()。**

**第四步是初始化 Reader 和 Writer：**

```java
  public void initReaderAndWriter(
      String name, long pingIntervalMillis, Streams streams) throws IOException {
    synchronized (this) {
      this.streams = streams;
      this.writer = new WebSocketWriter(streams.client, streams.sink, random);
      this.executor = new ScheduledThreadPoolExecutor(1, Util.threadFactory(name, false));
      if (pingIntervalMillis != 0) {
        executor.scheduleAtFixedRate(
            new PingRunnable(), pingIntervalMillis, pingIntervalMillis, MILLISECONDS);
      }
      if (!messageAndCloseQueue.isEmpty()) {
        runWriter(); // Send messages that were enqueued before we were connected.
      }
    }
    reader = new WebSocketReader(streams.client, streams.source, this);
  }
```
OkHttp使用 WebSocketReader 和 WebSocketWriter 来处理数据的收发。在发送数据时将数据组织成帧。

- WebSocket 的所有数据发送动作，都会在单线程线程池的线程中，通过 WebSocketWriter 执行。
- 在这里会创建 ScheduledThreadPoolExecutor 用于跑数据的发送操作。
- WebSocket 协议中主要会传输两种类型的帧，一是控制帧，主要是用于连接保活的 Ping 帧等；
- 二是用户数据载荷帧。在这里会根据用户的配置，调度 Ping 帧周期性地发送。
- 我们在调用 WebSocket 的接口发送数据时，数据并不是同步发送的，而是被放在了一个消息队列中。
- 发送消息的 Runnable 从消息队列中读取数据发送。这里会检查消息队列中是否有数据，如果有的话，会调度发送消息的 Runnable 执行。

**第五步是配置socket的超时时间为0，也就是阻塞IO。**

**第六步执行 loopReader()。这实际上是进入了消息读取循环了，也就是数据接收的逻辑了。**


## 数据发送
可以通过 WebSocket 接口的 send(String text) 和 send(ByteString bytes) 分别发送文本的和二进制格式的消息。

```java
  @Override public boolean send(String text) {
    if (text == null) throw new NullPointerException("text == null");
    return send(ByteString.encodeUtf8(text), OPCODE_TEXT);
  }
  @Override public boolean send(ByteString bytes) {
    if (bytes == null) throw new NullPointerException("bytes == null");
    return send(bytes, OPCODE_BINARY);
  }
  private synchronized boolean send(ByteString data, int formatOpcode) {
    // Don't send new frames after we've failed or enqueued a close frame.
    if (failed || enqueuedClose) return false;
    // If this frame overflows the buffer, reject it and close the web socket.
    if (queueSize + data.size() > MAX_QUEUE_SIZE) {
      close(CLOSE_CLIENT_GOING_AWAY, null);
      return false;
    }
    // Enqueue the message frame.
    queueSize += data.size();
    messageAndCloseQueue.add(new Message(formatOpcode, data));
    runWriter();
    return true;
  }
  . . . . . .
  private void runWriter() {
    assert (Thread.holdsLock(this));
    if (executor != null) {
      executor.execute(writerRunnable);
    }
  }
```
- 调用发送数据的接口时，做的事情主要是将数据格式化，构造消息，放进一个消息队列，然后调度 writerRunnable 执行。

- 当消息队列中的未发送数据超出最大大小限制，WebSocket 连接会被直接关闭。对于发送失败过或被关闭了的 WebSocket，将无法再发送信息。

- 在 writerRunnable 中会循环调用 writeOneFrame() 逐帧发送数据，直到数据发完，或发送失败。

**在 WebSocket 协议中，客户端需要发送 四种类型 的帧：**

- PING 帧——PING帧用于连接保活，它的发送是在 PingRunnable 中执行的，在初始化 Reader 和 Writer 的时候，就会根据设置调度执行或不执行。
- PONG 帧——PONG 帧是对服务器发过来的 PING 帧的响应，同样用于保活连接。
- CLOSE 帧——CLOSE 帧用于关闭连接
- MESSAGE 帧

除PING 帧外的其它 三种 帧，都在 writeOneFrame() 中发送。

我们主要关注用户数据发送的部分。PONG 帧具有最高的发送优先级。在没有PONG 帧需要发送时，writeOneFrame() 从消息队列中取出一条消息，如果消息不是 CLOSE 帧，则主要通过如下的过程进行发送：

```java
  boolean writeOneFrame() throws IOException {
    WebSocketWriter writer;
    ByteString pong;
    Object messageOrClose = null;
    int receivedCloseCode = -1;
    String receivedCloseReason = null;
    Streams streamsToClose = null;
    synchronized (RealWebSocket.this) {
      if (failed) {
        return false; // Failed web socket.
      }
      writer = this.writer;
      pong = pongQueue.poll();
      if (pong == null) {
        messageOrClose = messageAndCloseQueue.poll();
  . . . . . .
      } else if (messageOrClose instanceof Message) {
        ByteString data = ((Message) messageOrClose).data;
        BufferedSink sink = Okio.buffer(writer.newMessageSink(
            ((Message) messageOrClose).formatOpcode, data.size()));
        sink.write(data);
        sink.close();
        synchronized (this) {
          queueSize -= data.size();
        }
      } else if (messageOrClose instanceof Close) {
```
数据发送可以总结如下：

- 创建一个 BufferedSink 用于数据发送。
- 将数据写入前面创建的 BufferedSink 中。
- 关闭 BufferedSink。
- 更新 queueSize 以正确地指示未发送数据的长度。

创建的 Sink 是一个 FrameSink，可以开出是FrameSink 的 write() 方法将数据发送出去。

## 数据接收
在握手的HTTP请求返回之后，会在HTTP请求的回调里，启动消息读取循环 loopReader()：

```java
  public void loopReader() throws IOException {
    while (receivedCloseCode == -1) {
      // This method call results in one or more onRead* methods being called on this thread.
      reader.processNextFrame();
    }
  }
```
在这个循环中，不断通过 WebSocketReader 的 processNextFrame() 读取消息，直到收到了关闭连接的消息。


```java
final class WebSocketReader {
  public interface FrameCallback {
    void onReadMessage(String text) throws IOException;
    void onReadMessage(ByteString bytes) throws IOException;
    void onReadPing(ByteString buffer);
    void onReadPong(ByteString buffer);
    void onReadClose(int code, String reason);
  }
  . . . . . .
  void processNextFrame() throws IOException {
    readHeader();
    if (isControlFrame) {
      readControlFrame();
    } else {
      readMessageFrame();
    }
  }
  private void readHeader() throws IOException {
    if (closed) throw new IOException("closed");
    // Disable the timeout to read the first byte of a new frame.
    int b0;
    long timeoutBefore = source.timeout().timeoutNanos();
    source.timeout().clearTimeout();
    try {
      b0 = source.readByte() & 0xff;
    } finally {
      source.timeout().timeout(timeoutBefore, TimeUnit.NANOSECONDS);
    }
    opcode = b0 & B0_MASK_OPCODE;
    isFinalFrame = (b0 & B0_FLAG_FIN) != 0;
    isControlFrame = (b0 & OPCODE_FLAG_CONTROL) != 0;
    // Control frames must be final frames (cannot contain continuations).
    if (isControlFrame && !isFinalFrame) {
      throw new ProtocolException("Control frames must be final.");
    }
    boolean reservedFlag1 = (b0 & B0_FLAG_RSV1) != 0;
    boolean reservedFlag2 = (b0 & B0_FLAG_RSV2) != 0;
    boolean reservedFlag3 = (b0 & B0_FLAG_RSV3) != 0;
    if (reservedFlag1 || reservedFlag2 || reservedFlag3) {
      // Reserved flags are for extensions which we currently do not support.
      throw new ProtocolException("Reserved flags are unsupported.");
    }
    int b1 = source.readByte() & 0xff;
    isMasked = (b1 & B1_FLAG_MASK) != 0;
    if (isMasked == isClient) {
      // Masked payloads must be read on the server. Unmasked payloads must be read on the client.
      throw new ProtocolException(isClient
          ? "Server-sent frames must not be masked."
          : "Client-sent frames must be masked.");
    }
    // Get frame length, optionally reading from follow-up bytes if indicated by special values.
    frameLength = b1 & B1_MASK_LENGTH;
    if (frameLength == PAYLOAD_SHORT) {
      frameLength = source.readShort() & 0xffffL; // Value is unsigned.
    } else if (frameLength == PAYLOAD_LONG) {
      frameLength = source.readLong();
      if (frameLength < 0) {
        throw new ProtocolException(
            "Frame length 0x" + Long.toHexString(frameLength) + " > 0x7FFFFFFFFFFFFFFF");
      }
    }
    frameBytesRead = 0;
    if (isControlFrame && frameLength > PAYLOAD_BYTE_MAX) {
      throw new ProtocolException("Control frame must be less than " + PAYLOAD_BYTE_MAX + "B.");
    }
    if (isMasked) {
      // Read the masking key as bytes so that they can be used directly for unmasking.
      source.readFully(maskKey);
    }
  }
```
processNextFrame() 先读取 Header 的两个字节，然后根据 Header 的信息，读取数据内容。

通过帧的 Header 确定了是数据帧，则会执行 readMessageFrame() 读取消息帧：

```java
  private void readMessageFrame() throws IOException {
    int opcode = this.opcode;
    if (opcode != OPCODE_TEXT && opcode != OPCODE_BINARY) {
      throw new ProtocolException("Unknown opcode: " + toHexString(opcode));
    }
    Buffer message = new Buffer();
    readMessage(message);
    if (opcode == OPCODE_TEXT) {
      frameCallback.onReadMessage(message.readUtf8());
    } else {
      frameCallback.onReadMessage(message.readByteString());
    }
  }
```
在一个消息读取完成之后，会通过回调 FrameCallback 将读取的内容通知出去。

然后这一事件会通知到 RealWebSocket。

```java
public final class RealWebSocket implements WebSocket, WebSocketReader.FrameCallback {
  . . . . . .
  @Override public void onReadMessage(String text) throws IOException {
    listener.onMessage(this, text);
  }
  @Override public void onReadMessage(ByteString bytes) throws IOException {
    listener.onMessage(this, bytes);
  }
```
在 RealWebSocket 中，这一事件又被通知到我们在应用程序中创建的回调 WebSocketListener。

## 连接保活
保活通过PING帧和PONG来实现。

若用户设置了PING帧的发送周期，在握手的HTTP请求返回时，消息读取循环会调度PingRunnable周期性的向服务器发送PING帧：

```java
  private final class PingRunnable implements Runnable {
    PingRunnable() {
    }
    @Override public void run() {
      writePingFrame();
    }
  }
  void writePingFrame() {
    WebSocketWriter writer;
    synchronized (this) {
      if (failed) return;
      writer = this.writer;
    }
    try {
      writer.writePing(ByteString.EMPTY);
    } catch (IOException e) {
      failWebSocket(e, null);
    }
  }
```
通过 WebSocket 通信的双方，在收到对方发来的 PING 帧时，需要用PONG帧来回复。

在 WebSocketReader 的 readControlFrame() 中可以看到这一点：

```java
  private void readControlFrame() throws IOException {
    Buffer buffer = new Buffer();
    if (frameBytesRead < frameLength) {
      if (isClient) {
        source.readFully(buffer, frameLength);
      } else {
        while (frameBytesRead < frameLength) {
          int toRead = (int) Math.min(frameLength - frameBytesRead, maskBuffer.length);
          int read = source.read(maskBuffer, 0, toRead);
          if (read == -1) throw new EOFException();
          toggleMask(maskBuffer, read, maskKey, frameBytesRead);
          buffer.write(maskBuffer, 0, read);
          frameBytesRead += read;
        }
      }
    }
    switch (opcode) {
      case OPCODE_CONTROL_PING:
        frameCallback.onReadPing(buffer.readByteString());
        break;
      case OPCODE_CONTROL_PONG:
        frameCallback.onReadPong(buffer.readByteString());
        break;
```

```java
  @Override public synchronized void onReadPing(ByteString payload) {
    // Don't respond to pings after we've failed or sent the close frame.
    if (failed || (enqueuedClose && messageAndCloseQueue.isEmpty())) return;
    pongQueue.add(payload);
    runWriter();
    pingCount++;
  }
  @Override public synchronized void onReadPong(ByteString buffer) {
    // This API doesn't expose pings.
    pongCount++;
  }
```
可见在收到 PING 帧的时候，总是会发一个 PONG 帧出去。在收到一个 PONG 帧时，则通常只是记录一下，然后什么也不做。如我们前面所见，PONG 帧在 writerRunnable 中被发送出去：

```java
public final class RealWebSocket implements WebSocket, WebSocketReader.FrameCallback {
  . . . . . .
      if (pong != null) {
        writer.writePong(pong);
      } else if (messageOrClose instanceof Message) {

```
PONG 帧的发送与 PING 帧的非常相似：

```java
final class WebSocketWriter {
  . . . . . .
  /** Send a pong with the supplied {@code payload}. */
  void writePong(ByteString payload) throws IOException {
    synchronized (this) {
      writeControlFrameSynchronized(OPCODE_CONTROL_PONG, payload);
    }
  }
```
## 连接关闭
连接的关闭可以通过WebSocket 接口的 close(int code, String reason)方法关闭。

```java
public final class RealWebSocket implements WebSocket, WebSocketReader.FrameCallback {
  . . . . . .
  @Override public boolean close(int code, String reason) {
    return close(code, reason, CANCEL_AFTER_CLOSE_MILLIS);
  }
  synchronized boolean close(int code, String reason, long cancelAfterCloseMillis) {
    validateCloseCode(code);
    ByteString reasonBytes = null;
    if (reason != null) {
      reasonBytes = ByteString.encodeUtf8(reason);
      if (reasonBytes.size() > CLOSE_MESSAGE_MAX) {
        throw new IllegalArgumentException("reason.size() > " + CLOSE_MESSAGE_MAX + ": " + reason);
      }
    }
    if (failed || enqueuedClose) return false;
    // Immediately prevent further frames from being enqueued.
    enqueuedClose = true;
    // Enqueue the close frame.
    messageAndCloseQueue.add(new Close(code, reasonBytes, cancelAfterCloseMillis));
    runWriter();
    return true;
  }
```
在执行关闭连接动作前，会先检查一下 close code 的有效性在合法范围内。

检查完了之后，会构造一个 Close 消息放入发送消息队列，并调度 writerRunnable 执行。Close 消息可以带有不超出 123 字节的字符串，以作为 Close message，来说明连接关闭的原因。

连接的关闭分为主动关闭和被动关闭。客户端先向服务器发送一个 CLOSE 帧，然后服务器回复一个 CLOSE 帧，对于客户端而言，这个过程为主动关闭；反之则为对客户端而言则为被动关闭。 

在 writerRunnable 执行的 writeOneFrame() 实际发送 CLOSE 帧：

```java
public final class RealWebSocket implements WebSocket, WebSocketReader.FrameCallback {
  . . . . . .
        messageOrClose = messageAndCloseQueue.poll();
        if (messageOrClose instanceof Close) {
          receivedCloseCode = this.receivedCloseCode;
          receivedCloseReason = this.receivedCloseReason;
          if (receivedCloseCode != -1) {
            streamsToClose = this.streams;
            this.streams = null;
            this.executor.shutdown();
          } else {
            // When we request a graceful close also schedule a cancel of the websocket.
            cancelFuture = executor.schedule(new CancelRunnable(),
                ((Close) messageOrClose).cancelAfterCloseMillis, MILLISECONDS);
          }
        } else if (messageOrClose == null) {
          return false; // The queue is exhausted.
        }
      }
  . . . . . .
      } else if (messageOrClose instanceof Close) {
        Close close = (Close) messageOrClose;
        writer.writeClose(close.code, close.reason);
        // We closed the writer: now both reader and writer are closed.
        if (streamsToClose != null) {
          listener.onClosed(this, receivedCloseCode, receivedCloseReason);
        }
      } else {
```

发送 CLOSE 帧也分为主动关闭的发送还是被动关闭的发送。

- 对于被动关闭，在发送完 CLOSE 帧之后，连接被最终关闭，因而，发送 CLOSE 帧之前，这里会停掉发送消息用的 executor。而在发送之后，则会通过 onClosed() 通知用户。
- 而对于主动关闭，则在发送前会调度 CancelRunnable 的执行，发送后不会通过 onClosed() 通知用户。

而对于主动关闭，则在发送前会调度 CancelRunnable 的执行，发送后不会通过 onClosed() 通知用户。

```java
final class WebSocketWriter {
  . . . . . .
  void writeClose(int code, ByteString reason) throws IOException {
    ByteString payload = ByteString.EMPTY;
    if (code != 0 || reason != null) {
      if (code != 0) {
        validateCloseCode(code);
      }
      Buffer buffer = new Buffer();
      buffer.writeShort(code);
      if (reason != null) {
        buffer.write(reason);
      }
      payload = buffer.readByteString();
    }
    synchronized (this) {
      try {
        writeControlFrameSynchronized(OPCODE_CONTROL_CLOSE, payload);
      } finally {
        writerClosed = true;
      }
    }
  }
```

将 CLOSE 帧发送到网络的过程与 PING 和 PONG 帧的颇为相似，仅有的差别就是 CLOSE 帧有载荷。关于掩码位和掩码自己的规则，同样适用于 CLOSE 帧的发送。

----

CLOSE 的读取在 WebSocketReader 的 readControlFrame()中，读到 CLOSE 帧时，WebSocketReader 会将这一事件通知出去：

```java
final class WebSocketReader {
  . . . . . .
  private void readControlFrame() throws IOException {
    Buffer buffer = new Buffer();
    if (frameBytesRead < frameLength) {
      if (isClient) {
        source.readFully(buffer, frameLength);
      } else {
        while (frameBytesRead < frameLength) {
          int toRead = (int) Math.min(frameLength - frameBytesRead, maskBuffer.length);
          int read = source.read(maskBuffer, 0, toRead);
          if (read == -1) throw new EOFException();
          toggleMask(maskBuffer, read, maskKey, frameBytesRead);
          buffer.write(maskBuffer, 0, read);
          frameBytesRead += read;
        }
      }
    }
    switch (opcode) {
  . . . . . .
      case OPCODE_CONTROL_CLOSE:
        int code = CLOSE_NO_STATUS_CODE;
        String reason = "";
        long bufferSize = buffer.size();
        if (bufferSize == 1) {
          throw new ProtocolException("Malformed close payload length of 1.");
        } else if (bufferSize != 0) {
          code = buffer.readShort();
          reason = buffer.readUtf8();
          String codeExceptionMessage = WebSocketProtocol.closeCodeExceptionMessage(code);
          if (codeExceptionMessage != null) throw new ProtocolException(codeExceptionMessage);
        }
        frameCallback.onReadClose(code, reason);
        closed = true;
        break;
      default:
        throw new ProtocolException("Unknown control opcode: " + toHexString(opcode));
    }
  }
```

读到 CLOSE 帧时，WebSocketReader 会将这一事件通知出去。

```java
public final class RealWebSocket implements WebSocket, WebSocketReader.FrameCallback {
  . . . . . .
  @Override public void onReadClose(int code, String reason) {
    if (code == -1) throw new IllegalArgumentException();
    Streams toClose = null;
    synchronized (this) {
      if (receivedCloseCode != -1) throw new IllegalStateException("already closed");
      receivedCloseCode = code;
      receivedCloseReason = reason;
      if (enqueuedClose && messageAndCloseQueue.isEmpty()) {
        toClose = this.streams;
        this.streams = null;
        if (cancelFuture != null) cancelFuture.cancel(false);
        this.executor.shutdown();
      }
    }
    try {
      listener.onClosing(this, code, reason);
      if (toClose != null) {
        listener.onClosed(this, code, reason);
      }
    } finally {
      closeQuietly(toClose);
    }
  }
```

对于收到的 CLOSE 帧处理同样分为主动关闭的情况和被动关闭的情况。与 CLOSE 发送时的情形正好相反，若是主动关闭，则在收到 CLOSE 帧之后，WebSocket 连接最终断开，因而需要停掉executor，被动关闭则暂时不需要。

收到 CLOSE 帧，总是会通过 onClosing() 将事件通知出去。

对于主动关闭的情形，最后还会通过 onClosed() 通知用户，连接已经最终关闭。

## 生命周期概括

- 连接通过一个HTTP请求握手并建立连接。WebSocket 连接可以理解为是通过HTTP请求建立的普通TCP连接。
- WebSocket 做了二进制分帧。WebSocket 连接中收发的数据以帧为单位。主要有用于连接保活的控制帧 PING 和 PONG，用于用户数据发送的 MESSAGE 帧，和用于关闭连接的控制帧 CLOSE。
- 连接建立之后，通过 PING 帧和 PONG 帧做连接保活。
- 一次 send 数据，被封为一个消息，通过一个或多个 MESSAGE帧进行发送。一个消息的帧和控制帧可以交叉发送，不同消息的帧之间不可以。
- WebSocket 连接的两端相互发送一个 CLOSE 帧以最终关闭连接。