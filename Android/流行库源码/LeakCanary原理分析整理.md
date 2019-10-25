[toc]

# Reference概述
## Reference
Reference主要负责内存的一个状态，Reference类把内存分为四种状态：

- Active：内存一开始分配的都是Active
- Pending：快要放进队列的对象，马上回收的对象。
- Enqueued：对象已经被回收了，已经对象放入队列中了。
- Inactive：最终状态，不能再变为其他状态。

## ReferenceQueue
引用队列，在对象的可达性更改后，垃圾回收器将已注册的引用对象添加到队列中。

**举个例子**

检测一个对象是否被回收，可以使用Reference + ReferenceQueue

- 创建一个引用队列 queue
- 创建 Refrence 对象，并关联引用队列 queue
- 在 reference 被回收的时候，refrence 会被添加到 queue 中

```java
创建一个引用队列  
ReferenceQueue queue = new ReferenceQueue();  

// 创建弱引用，此时状态为Active，并且Reference.pending为空，当前Reference.queue = 上面创建的queue，并且next=null  
WeakReference reference = new WeakReference(new Object(), queue);  
System.out.println(reference);  
// 当GC执行后，由于是虚引用，所以回收该object对象，并且置于pending上，此时reference的状态为PENDING  
System.gc();  

/* ReferenceHandler从pending中取下该元素，并且将该元素放入到queue中，此时Reference状态为ENQUEUED，Reference.queue = ReferenceENQUEUED */  

/* 当从queue里面取出该元素，则变为INACTIVE，Reference.queue = Reference.NULL */  
Reference reference1 = queue.remove();  
System.out.println(reference1);
```
# LeakCanary原理
## 检测泄漏的步骤
- 第一步，获得activity被销毁的信息，也就是要监听到activity的生命周期方法。
- 第二步，判断该对象（activity）是否被回收。若回收了那就是没泄漏，若判断该对象没被回收，说明该对象泄漏了。
- 进行第三步，展示泄漏信息。

## LeakCanary源码
### 第一步——监听
使用上只需在Application.onCreate()中调用一句LeakCanary.install(this)即可。

**进入install方法**

```java
public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
            .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
            .buildAndInstall();
}
```

标准的建造者模式

- listenerServiceClass、excludedRefs 都是给Builder成员变量赋值

**最后的buildAndInstall()方法**

```java
public RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
        throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    //调用build方法构造
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
        LeakCanary.enableDisplayLeakActivity(context);
        if (watchActivities) {
                    //传入build好的RefWatch，调用静态方法进行install
                    ActivityRefWatcher.install((Application) context, refWatcher);
        }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
}
```
完全是build设计模式，构造出一个RefWatcher

**接下来进入install((Application) context, refWatcher)**

```java
public static void install(Application application, RefWatcher refWatcher) {
     new ActivityRefWatcher(application, refWatcher).watchActivities();
}
```
构造**ActivityRefWatcher**并调用watchActivities()监控activity

那么，我们想想作为第三方应用来说，如何监控应用内的每一个Activity？

可以通过系统提供给我们的接口ActivityLifecycleCallbacks，该接口的回调方法与activity生命周期回调方法对应，每当有activity创建或销毁时，可以通过回调方法通知外部

**watchActivities方法**

```java
public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
}
```

**lifecycleCallbacks：**

```java
private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
        new Application.ActivityLifecycleCallbacks() {
            @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState){}
            @Override public void onActivityStarted(Activity activity) {}
            @Override public void onActivityResumed(Activity activity) {}
            @Override public void onActivityPaused(Activity activity) {}
            @Override public void onActivityStopped(Activity activity) {}
            @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {}
            @Override public void onActivityDestroyed(Activity activity) {
                ActivityRefWatcher.this.onActivityDestroyed(activity);
            }
        };
```

- 这里只监控了activityDestory，当有activity被销毁时，交给onActivityDestoryed()处理

**onActivityDestory方法：**

```java
void onActivityDestroyed(Activity activity) {
     refWatcher.watch(activity);
}
```
交给watcher处理。到此，完成了第一步：监听activity销毁信息.

接下来研究watch()方法是如何判断activity是否泄漏的

```java
public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
        return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
            new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
}
```
这里将activity、唯一生成的key、还有queue(是一个ReferenceQueue) 构造一个KeyedWeakReference。并将key值存入retainKeys中保管。该类是WeakReference的子类，如下：

```java
final class KeyedWeakReference extends WeakReference
    public final String key;
    public final String name;
    KeyedWeakReference(Object referent, String key, String name,
        ReferenceQueue
        super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
        this.key = checkNotNull(key, "key");
        this.name = checkNotNull(name, "name");
    }
}
```
这里，我们将activity包装弱引用，并添加了referenceQueue，那么当该activity被GC回收时，我们就可以从referenceQueue中获取该activity的reference。

### 第二步——检测泄漏（核心）

```java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    //将已被回收的activity对象的keyedWeakReference的key值从retainedKeys中删除，以达到
    //过滤目的
    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
        // The debugger can create false leaks.
        return RETRY;
    }
    //如果retainedKeys中不存在reference，说明它已经被回收，返回
    if (gone(reference)) {
        return DONE;
    }
    //手动调用GC
    gcTrigger.runGc();
    //再次过滤
    removeWeaklyReachableReferences();
    //若retainedKeys中还存在该reference（还没有被滤掉），则判断为该reference泄漏，进行下一步dump内存快照
    //展示泄漏信息
    if (!gone(reference)) {
        long startDumpHeap = System.nanoTime();
        long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

        File heapDumpFile = heapDumper.dumpHeap();
        if (heapDumpFile == RETRY_LATER) {
            // Could not dump the heap.
            return RETRY;
        }
        long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
        heapdumpListener.analyze(
                new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
                        gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
}
```
总结一下
- 1、首先将已回收的activity对象的keyedWeakReference的key值从retainedKeys中删除
- 2、如果retainedKeys中不存在reference的key，说明已经回收，返回
- 3、手动执行GC，然后执行 1 再次过滤已经回收的reference
- 4、再次进行 2 ，若retainedKeys中还存在该reference，进行下一步dump内存快照

**removeWeakReachableReferences()方法**

```java
private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    //若queue中存在该keyedWeakReference，则说明该keyedWeakReference对应的activity已被回收
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
        //从retainedKeys中移除，则retainedKeys剩下的就是泄漏的
        retainedKeys.remove(ref.key);
    }
}
```
**其逻辑就是在queue里面的到key，然后从retainedKeys中移除。**

**gone()方法：**

```java
private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
}
```
**就是一个contains判断**

### 第三步——泄漏分析
第三步：展示泄漏信息，也就是获取dumpheap以及对这个dumpheap进行analyze。

调用比较简单，而逻辑实现又是依托另外一个项目：HAHA。

```java
 heapdumpListener.analyze(
            new HeapDump(heapDumpFile, reference.key,
            	reference.name, excludedRefs, watchDurationMs,
                gcDurationMs, heapDumpDurationMs));
```

这里的heapdumpListener是第一步构建watcher时listenerServiceClass(DisplayLeakService.class)里面创建的

**最后是ServiceHeapDumpListener这个类封装了analyze逻辑：**

```java
@Override public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
```
**runAnalysis()方法中就是启动一个服务处理heapDump**

```java
public static void runAnalysis(Context context, HeapDump heapDump,
                               Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
}
```
**HeapAnalyzerService.onHandIntent()中收到启动服务请求，调用checkForLeak**

```java
@Override
protected void onHandleIntent(Intent intent) {
    if (intent == null) {
        CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
        return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
}
```
**checkForLeak()方法：**

```java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
        Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
        return failure(exception, since(analysisStartNanoTime));
    }

    try {
        HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
        HprofParser parser = new HprofParser(buffer);
        //将heapDumpFile转化为Snapshot
        Snapshot snapshot = parser.parse();
        deduplicateGcRoots(snapshot);

        //找到泄漏对象引用
        Instance leakingRef = findLeakingReference(referenceKey, snapshot);

        // False alarm, weak reference was cleared in between key check and heap dump.
        if (leakingRef == null) {
            return noLeak(since(analysisStartNanoTime));
        }

        //返回泄漏对象的最短路径
        return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);
    } catch (Throwable e) {
        return failure(e, since(analysisStartNanoTime));
    }
}
```
主要有以下几个步骤：
- 把.hprof转为Snapshot，这个Snapshot对象就包含了对象引用的所有路径
- 精简gcroots,把重复的路径删除，重新封装成不重复的路径的容器
- 找出泄漏的对象
- 找出泄漏对象的最短路径

其中第三步的**findLeakingReference方法**

```java
private Instance findLeakingReference(String key, Snapshot snapshot) {
    //找到快照中的KeyedWeakReference类对象
    ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
    List<String> keysFound = new ArrayList<>();
    //遍历这个类的所有实例
    for (Instance instance : refClass.getInstancesList()) {
        List<ClassInstance.FieldValue> values = classInstanceValues(instance);
        //key值和最开始定义封装的key值相同，说明该实例是泄漏对象
        String keyCandidate = asString(fieldValue(values, "key"));
        if (keyCandidate.equals(key)) {
            return fieldValue(values, "referent");
        }
        keysFound.add(keyCandidate);
    }
    throw new IllegalStateException(
            "Could not find weak reference with key " + key + " in " + keysFound);
}
```
