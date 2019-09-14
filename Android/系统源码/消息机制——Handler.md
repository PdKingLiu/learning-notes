参考Android开发艺术探索，基于Android5.0。
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


[参考 https://www.jianshu.com/p/377bb840802f](https://www.jianshu.com/p/377bb840802f)

# 消息队列的工作原理
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