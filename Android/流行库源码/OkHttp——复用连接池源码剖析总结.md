@[toc]
## OKHttp的复用连接池
为了解决TCP握手和挥手的效率问题，HTTP有一种叫做keepalive connections的机制，而OKHttp支持5个并发socket连接，默认keepAlive时间为5分钟，接下来看看OKHttp是怎么复用连接的
### 1. 主要变量与构造方法
连接池的类位于ConnectionPool，他的主要变量及作用如下：

```java
	//线程池
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  /** The maximum number of idle connections for each address. */
  //最大空闲连接数
  private final int maxIdleConnections;
  //连接时间
  private final long keepAliveDurationNs;
  //回收空闲连接的runnable
  private final Runnable cleanupRunnable = new Runnable() {···};
  //双向队列，里面维护了RealConnection也就是socket物理连接的包装
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  //记录连接失败的路线名单，当连接失败就会把失败的线路加进去
  final RouteDatabase routeDatabase = new RouteDatabase();
  boolean cleanupRunning;
```

构造方法

```java
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }
```
默认socket最大连接数为5个，连接时间为5分钟。ConnectionPool实例是在OKHttpClient实例化是创建的。

### 2. 缓存操作
`ConnectionPool对Deque <RealConnection>`进行操作的方法分别是put、get、connectionBecameIdle和evictAll这几个操作，分别对应放入连接、获取连接、移除连接和移除所有连接。

```java
  /** Returns a recycled connection to {@code address}, or null if no such connection exists. */
  RealConnection get(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.allocations.size() < connection.allocationLimit
          && address.equals(connection.route().address)
          && !connection.noNewStreams) {
        streamAllocation.acquire(connection);
        return connection;
      }
    }
    return null;
  }

  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }

```

再添加Deque之前要先清理空闲的线程，在get的时候，遍历connections缓存列表，当某个缓存列表中次连接的地址完全匹配时，则直接复用缓存列表中的connection作为request的连接。

### 3. 自动回收连接
OKHttp是根据StreamAllocation引用计数是否为0来实现自动回收连接的。我们在put操作前首先需要调用 executor.execute(cleanupRunnable); 来清理闲置线程。

下面是cleanupRunnable是实现：

```java
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```
线程不断调用cleanup方法进行清理，并返回下次需要清理的间隔时间，然后调用wait方法进行等待以释放锁与时间片。当等待时间到了后，再次进行清理并返回下次要清理的间隔。

```java
  /**
   * Performs maintenance on this pool, evicting the connection that has been idle the longest if
   * either it has exceeded the keep alive limit or the idle connections limit.
   *
   * <p>Returns the duration in nanos to sleep until the next scheduled call to this method. Returns
   * -1 if no further cleanups are required.
   */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        
      connections.remove(longestIdleConnection);	// 2
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;  	// 3
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```

cleanup的方法所做的事情简单总结就是根据连接中的引用计数来计算空闲连接数和活跃连接数，然后标记处空闲的连接。

在代码2处，如果空闲连接keepAlive时间超过5分钟，或者空闲连接数超过5个，则从Deque中移除此连接。

接下来是根据空闲连接或者活跃连接返回下次需要清理的时间：

如果空闲连接大于0，则返回到期时间

如果活跃连接大于0，则返回默认的keepAlive时间5分钟

代码3处，如果没有任何连接则返回-1

可以看到判断连接是否闲置是通过`pruneAndGetAllocationCount(connection, now) > 0`得到的。

```java
  /**
   * Prunes any leaked allocations and then returns the number of remaining live allocations on
   * {@code connection}. Allocations are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);

      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked allocation. This is an application bug.
      Internal.logger.warning("A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?");
      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }
```

pruneAndGetAllocationCount 方法通过遍历connection.allocations来判断其中的StreamAllocation是否使用，如果使用，则接着遍历下一个StreamAllocation，如果未被使用，则从列表中移除。

最后判断列表是否为空，如果为空则说明此连接没有引用了，这时返回0，表示此连接是空闲连接，否则返回非0的数，表示此连接是活跃连接。那么StreamAllocation到底是什么呢？

### 4. 引用计数
在OKHttp代码调用中，使用了类似引用计数的方式跟踪socket流的调用。这里的计数兑现是StreamAllocation，他反复执行acquire和release操作，这两个方法其实是在改变RealConnection中的`List<Reference<StreamAllocation>>`的大小。

```java
  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    connection.allocations.add(new WeakReference<>(this));
  }

  /** Remove this allocation from the connection's list of allocations. */
  private void release(RealConnection connection) {
    for (int i = 0, size = connection.allocations.size(); i < size; i++) {
      Reference<StreamAllocation> reference = connection.allocations.get(i);
      if (reference.get() == this) {
        connection.allocations.remove(i);
        return;
      }
    }
    throw new IllegalStateException();
  }
```

RealConnection是socket物理连接的包装，它里面维护了`List<Reference<StreamAllocation>>`的引用。List中的StreamAllocation的数量也就是socket被引用的计数。如果计数为0，则说明此连接没有被使用，也就是空闲的，需要通过下文的算法实现回收；如果计数不为0，则表示上层代码仍然在引用，无需关闭连接。

### 5. 小结
连接复用的核心就是用`Deque<RealConnection>`来存储连接，通过put、get、connectionBecameIdle和evictAll几个操作来对Deque进行操作，另外通过判断连接中的计数对象StreamAllocation来进行自动回收连接。

