@[toc]
 # 概述
 Handler是Android消息机制的上层接口，这使得在开发过程中只需要和Handler交互即可。
 Android消息机制主要指的是Handler的运行机制，Handler底层运行需要MessageQueue和Looper的支撑。

- MessageQueue ：消息队列，它内部存储了一组消息，对外提供插入和删除。内部存储结构是链表。
- Looper   ：MessageQueue只是一个消息的存储单元，不能处理消息，而Looper就填补了这个功能，Looper以无限循环的方式查询是否有新消息，如果有的话就处理，否则就一直等待。
- ThreadLocal ：ThreadLocal的作用是在每个线程中存储数据，ThreadLocal可以在不同的线程中互不干扰的存储并提供数据，通过ThreadLocal可以轻松的获取每个线程的Looper。

创建Handler

给当前线程创建Looper，或者在一个有Looper的线程创建Handler。否则会抛出异常。

创建完Handler，Looper以及MessageQueue就可以和Handler一起协同工作了。

可以通过Handler的post方法将一个Runnable投递到Handler内部的Looper中去处理，也可以通过Handler的send方法发送一个消息，这个消息同样会在Looper中处理
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190914213709964.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

# ThreadLocal
这里和书上的不一样，ThreadLocal的源码已经发成了很大的改变。

ThreadLocal是一个数据结构，有点像HashMap，可以保存一个值，并且保证各个线程的数据互不干扰。

```
ThreadLocal<String> localName = new ThreadLocal();
localName.set("aaa");
String name = localName.get();
```
线程1初始化为aaa，线程2在此get时，值将会是null。

源码

```
 public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

可以看出每个线程有一个ThreadLocalMap的数据结构，当执行set时，会将值存在指定的ThreadLocalMap中，由于每个线程的ThreadLocalMap是不一样的，所以各个线程互不干扰。

ThreadLoalMap

ThreadLoalMap 是一个类似HashMap的数据结构。
在ThreadLoalMap中，是初始化一个大小为一个Entry的数组，Entry对象用来保存每一个key-value键值对，只不过可以永远是ThreadLocal对象。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190914215723113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

看一看ThreadLoalMap插入一个key-values的实现。

```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
每个ThreadLocal都有一个Hash值threadLocalHashCode
插入过程中根据ThreadLocal对象的hash值，定位到table中的位置i
插入过程：

1.  如果这个位置是空的，那么就初始化一个Entry对象放在位置i上
2. 如果i已经有对象了并且key值是一样的，那么直接覆盖之前的值。
3. 如果i的entry对象和即将设置的key没关系，那么只能找下一个空位置。



# MessageQueue工作原理
消息队列指的是MessageQueue，MessageQueue主要包含两个操作，插入和读取，对应方法enqueueMessage和next，虽然MessageQueue叫做消息队列，但是他的内部实现是通过单链表维护消息列表。


enqueueMessage源码
```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

其就是链表的插入操作。

next源码

```

    Message next() {
		·····
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
		······
        }
    }

```

next是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息时，next方法会返回这条消息，并从单链表中删除。


# Looper工作原理

Looper在Android消息机制中扮演着消息循环的角色，会不断的从MessageQueue中查看是否有消息，如果有新消息就会立即处理，否则就会阻塞在那里。在构造方法中会创建一个MessageQueue消息队列然后保存当前线程对象。

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

Handler工作需要Looper，没有Looper的线程会报错，通过Looper.prepare()即可为当前线程创建一个Looper，通过Looper.loop()开启消息循环

```
        new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler();
                Looper.loop();
            }
        }).start();
```

除了prepare，还提供了了prepareMainLooper方法，这个方法主要是给主线程也就是ActivityThread创建Looper使用的。
Looper提供quit和quitSafely来退出一个Looper，区别是前者会直接退出，后者会处理完所有的消息后才退出。退出后Handler的send方法会返回false。
在子线程中如果手动创建了一个Looper，那么在事情完成之后需要调用quit方法终止消息循环，否则子线程会一直处于等待的状态。

Looper最重要的一个方法是loop方法，只有调用loop方法之后，消息循环才会真正的开始起作用。

```
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```

源码比较好理解，loop方法是一个死循环，唯一跳出循环的方式是MessageQueue的next方法返回了null，当Looper的quit方法被调用后，Looper就会调用MessageQueue的quit或quitSaftly方法通知消息队列退出，此时他的next方法会返回null，当没有消息时，next方法会一直堵塞在那里，这也导致loop方法一直阻塞在那里。

如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息。

	 msg.target.dispatchMessage(msg);

msg.target是发送这条消息的对象。

最终发送的消息最终又交给他的dispatchMessage();方法处理了。

Handler的dispatchMessage方法是在创建Handler时所使用的Looper中执行的，这样就成功的将代码切换到指定的线程中去了。


# Handler工作原理

Handler的主要工作是发送消息和接收消息。消息的发送可以通过post的一系列方法以及send的一系列方法实现，post最终是通过send的一系列方法来实现的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917173749899.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917173815171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

```
    public final boolean sendMessage(Message msg)   {
        return sendMessageDelayed(msg, 0);
    }
```

```
    public final boolean sendMessageDelayed(Message msg, long delayMillis)    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

发送消息的方法最终会调用`sendMessageAtTime`并且最后调用` queue.enqueueMessage(msg, uptimeMillis);`请消息插入到消息队列。然后MessageQueue的next方法就会返回这个消息给Looper，Looper接收到消息后交给Handler处理，这样就会调用dispatchMessage()方法。

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
可以清楚的看到，先检查Message的callback是否为null，不为null就通过handleCallBack来处理。
Message的callBack的一个Runnable对象，实际上是post方法传递的Runnable

```
	private static void handleCallback(Message message) {
	    message.callback.run();
	}
```
handleCallback逻辑很简单，实际上就是运行run方法。

然后就是mCallback，检查mCallBack是否为null，不为null就调用他的handleMessage方法。

```
  public interface Callback {
      /**
       * @param msg A {@link android.os.Message Message} object
       * @return True if no further handling is desired
       */
      public boolean handleMessage(Message msg);
  }
```

Callback是一个接口，就是在创建Handler Handler handler = new Handler(callback)时构造方法里面的参数。

```
 Handler handler = new Handler(new Handler.Callback() {
     @Override
     public boolean handleMessage(Message msg) {
         return false;
     }
 });
```

最后调用Handler本身的方法handleMessage处理消息。它是一个空实现。

```
    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
```


Handler的流程处理消息的流程图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917171456625.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

# 主线程消息循环

```
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
        // It will be in the format "seq=114"
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
主线程通过Looper.prepareMainLooper(); 来创建Looper和MessageQueue。

```
  class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        public static final int EXIT_APPLICATION        = 111;
        public static final int RECEIVER                = 113;
        public static final int CREATE_SERVICE          = 114;
        public static final int SERVICE_ARGS            = 115;
        public static final int STOP_SERVICE            = 116;

        public static final int CONFIGURATION_CHANGED   = 118;
        public static final int CLEAN_UP_CONTEXT        = 119;
        public static final int GC_WHEN_IDLE            = 120;
        public static final int BIND_SERVICE            = 121;
        public static final int UNBIND_SERVICE          = 122;
        public static final int DUMP_SERVICE            = 123;
        public static final int LOW_MEMORY              = 124;
        public static final int PROFILER_CONTROL        = 127;
        public static final int CREATE_BACKUP_AGENT     = 128;
        public static final int DESTROY_BACKUP_AGENT    = 129;
        public static final int SUICIDE                 = 130;
        ······
  }
```
主线程的消息循环开始了以后，ActivityThread还需要一个Handler来和消息队列进行交互，这个Handler就是Activity.H，它内部定义了一组消息类型，主要包括四大组件的启动和停止。

ActivityThread通过ApplicationThread和AMS进行进程间通信，AMS以进程间通信的方式完成ActivityThread的请求后会回调Application中的Binder方法，然后ApplicationThred会向H发送消息，H收到消息后会将Application中的逻辑切换到ActivityThread中去执行，即切换到主线程中去执行。这个过程就是主线程的消息循环模型。

这段话暂时没理解，先留着。

收藏网上的几张图

![在这里插入图片描述](https://img-blog.csdn.net/20161129184124711)

![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NDQzNjUtMTg0ZWE5NGVjMWI1Y2UwNS5wbmc?x-oss-process=image/format,png)

![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NDQzNjUtYzg2Yzg1MmZhMGE2NGQ1Yi5wbmc?x-oss-process=image/format,png)

>参考
>《Android 开发艺术探索》
>[https://www.jianshu.com/p/377bb840802f](https://www.jianshu.com/p/377bb840802f)
>[https://blog.csdn.net/ly502541243/article/details/52062179](https://blog.csdn.net/ly502541243/article/details/52062179)
>[https://blog.csdn.net/qq_30379689/article/details/53394061](https://blog.csdn.net/qq_30379689/article/details/53394061)
>[https://www.jianshu.com/p/f0b23ee5a922](https://www.jianshu.com/p/f0b23ee5a922)
>
>