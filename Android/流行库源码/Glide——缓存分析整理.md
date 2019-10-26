[toc]

# LruCache
## 概述

LruCache是Android 3.1所提供的一个缓存类，所以在Android中可以直接使用LruCache实现内存缓存。

**主要算法原理是把最近使用的对象用强引用（即我们平常使用的对象引用方式）存储在 LinkedHashMap 中。当缓存满时，把最近最少使用的对象从内存中移除，并提供了get和put方法来完成缓存的获取和添加操作。**

**简单使用**

```java
int maxMemory = (int) (Runtime.getRuntime().totalMemory()/1024);
int cacheSize = maxMemory/8;
mMemoryCache = new LruCache<String,Bitmap>(cacheSize){
        @Override
        protected int sizeOf(String key, Bitmap value) {
            return value.getRowBytes()*value.getHeight()/1024;
        }
    };
```

- 设置LruCache缓存大小
- 重写sizeOf方法计算每张图片大小

## 原理
维护一个缓存对象列表，其中对象列表的排列方式是按照访问顺序实现的，即一直没访问的对象，将放在队尾，即将被淘汰。而最近访问的对象将放在队头，最后被淘汰。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026105736278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
这个队列的维护基于LinkedHashMap。

**LruCache中维护了一个集合LinkedHashMap，该LinkedHashMap是以访问顺序排序的。当调用put()方法时，就会在结合中添加元素，并调用trimToSize()判断缓存是否已满，如果满了就用LinkedHashMap的迭代器删除队尾元素，即近期最少访问的元素。当调用get()方法访问缓存对象时，就会调用LinkedHashMap的get()方法获得对应集合元素，同时会更新该元素到队头。** 

[https://www.jianshu.com/p/b49a111147ee](https://www.jianshu.com/p/b49a111147ee)

# DiskLruCache
## 概述
DiskLruCache 的分析分为两部分：分别是日志，读、写缓存。

DiskLruCache 的日志就是一个操作记录。例如，删除一条缓存条目，就会在日志文件中记录一条 REMOVE 记录；新建一条缓存，缓存文件刚建立时会增加一条 DIRTY 记录，缓存写入成功后再增加一条 CLEAN 记录；缓存被读取，会增加 READ 记录。日志文件的格式如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026113637520.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

记录日志是为了在 DiskLruCache 初始化时建立起缓存的 LRU 结构。DiskLruCache 内部使用了 LinkHashMap 来存储所有的缓存项，在初始化的时候，DiskLruCache 会读日志文件上的记录，根据该日志文件的记录将所有历史操作“重做”一遍：在执行历史操作时，会根据日志记录对缓存项进行添加和删除操作。为什么这样能保证缓存项是按 LRU 的顺序排序的呢？因为构造 LinkedHashMap 的时候选择使用 Access Order（访问顺序）来保持元素的顺序，因此只需遍历日志根据日志记录向 LinkedHashMap 中放入缓存项（Entry）自然而然地就保持了 LRU 的顺序。

[http://liwenkun.me/2017/08/29/glide-disk-cache-strategy/](http://liwenkun.me/2017/08/29/glide-disk-cache-strategy/)
# Glide缓存概述
## 资源分类
Glide的缓存图片资源分为两类
- 原始图片（Source）——原始图片大小
- 转换后的图片（Result）——经过大小处理过的图片

使用Glide加载图片时，Glide默认会根据View对图片进行压缩转换。

## 缓存设计
- Glide缓存设计为二级缓存：内存缓存和硬盘缓存
- 缓存读取顺序：内存缓存 --> 磁盘缓存 --> 网络
- 内存缓存：防止应用重复将图片数据读取内存当中——只缓存转换过后的图片
- 硬盘缓存：防止应用重复从网络或其他地方重复下载和读取数据——可缓存原始图片、转换过后的图片

Glide内存缓存实现基于**LruCache 算法 +  弱引用机制**

- LruCache算法原理：将最近使用的对象，用强引用的方式 存储在LinkedHashMap中 ；当缓存满时 ，将最近最少使用的对象从内存中移除
- 弱引用：弱引用的对象具备更短生命周期，因为 当JVM进行垃圾回收时，一旦发现弱引用对象，都会进行回收（无论内存充足否）

磁盘缓存**使用Glide 自定义的DiskLruCache算法**

- 该算法基于 Lru 算法中的DiskLruCache算法，具体应用在磁盘缓存的需求场景中
- 该算法被封装到Glide自定义的工具类中（该工具类基于Android 提供的DiskLruCache工具类）

# Glide 缓存源码分析
## 1、生成key
Glide实现内存、磁盘缓存是**根据图片缓存key进行唯一标识**

**生成缓存 Key 的代码发生在Engine类的 load()中**

```java
    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        Util.assertMainThread();
        long startTime = LogTime.getLogTime();

        final String id = fetcher.getId();
        // 获得了一个id字符串，即需加载图片的唯一标识
        // 如，若图片的来源是网络，那么该id = 这张图片的url地址

        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),transcoder, loadProvider.getSourceEncoder());
        // Glide的缓存Key生成规则复杂：根据10多个参数生成
        // 将该id 和 signature、width、height等10个参数一起传入到缓存Key的工厂方法里，最终创建出一个EngineKey对象
        // 创建原理：通过重写equals() 和 hashCode()，保证只有传入EngineKey的所有参数都相同情况下才认为是同一个EngineKey对象
        // 该EngineKey 即Glide中图片的缓存Key

        ... 
```
概括一下

- 获得了一个id字符串，若图片的来源是网络，那么该id = 这张图片的url地址
- Glide的缓存Key生成规则：根据10多个参数生成，将该id 和 signature、width、height等10个参数一起传入到缓存Key的工厂方法里，最终创建出一个EngineKey对象

## 2、创建缓存对象LruResourceCache
**LruResourceCache对象是在创建 Glide 对象时创建的**

```java
public class GlideBuilder {
    ...
    Glide createGlide() {
        MemorySizeCalculator calculator = new MemorySizeCalculator(context);
        if (bitmapPool == null) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
                int size = calculator.getBitmapPoolSize();
                bitmapPool = new LruBitmapPool(size);
            } else {
                bitmapPool = new BitmapPoolAdapter();
            }
        }

        if (memoryCache == null) {
            memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
            // 创建一个LruResourceCache对象 并 赋值到memoryCache对象
            // 该LruResourceCache对象 = Glide实现内存缓存的LruCache对象

        }
        
        return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
    }
} 
```
这里没什么多说的。

## 3、获取内存缓存中的图片
读取内存缓存代码实在Engine类的load()方法中

 

```java
    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        Util.assertMainThread();

        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());
         // 上面讲解的生成图片缓存Key


        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        // 调用loadFromCache()获取内存缓存中的缓存图片

        if (cached != null) {
            cb.onResourceReady(cached);
        }
        // 若获取到，就直接调用cb.onResourceReady()进行回调

        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
        }
        // 若没获取到，就继续调用loadFromActiveResources()获取缓存图片
        // 获取到也直接回调

        // 若上述两个方法都没有获取到缓存图片，就开启一个新的线程准备加载图片
        // 即从上文提到的 Glide最基础功能：图片加载
        EngineJob current = jobs.get(key);
            return new LoadStatus(cb, current);
        }

        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        engineJob.start(runnable);

        return new LoadStatus(cb, engineJob);
    }

    ...
} 
```
概括一下
- Glide 将 内存缓存划分为两块：一块使用了LruCache算法机制；另一块使用了弱引用机制。
- 当 获取内存缓存时，会通过两个方法分别从上述两块区域进行缓存获取。
- 若没有找到，则新开一个线程重新获取

```java

<-- 方法1：loadFromCache() -->
// 原理：使用了 LruCache算法
    private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        // 若isMemoryCacheable = false就返回null，即内存缓存被禁用
        // 即 内存缓存是否禁用的API skipMemoryCache() - 请回看内存缓存的具体使用
        // 若设置skipMemoryCache(true)，此处的isMemoryCacheable就等于false，最终返回Null，表示内存缓存已被禁用
        }

        EngineResource<?> cached = getEngineResourceFromCache(key);
        // 获取图片缓存 ->>分析4

        // 从分析4回来看这里：
        if (cached != null) {
            cached.acquire();
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
            // 将获取到的缓存图片存储到activeResources当中
            // activeResources = 一个弱引用的HashMap：用于缓存正在使用中的图片
            // 好处：保护这些图片不会被LruCache算法回收掉。 ->>方法2

        }
        return cached;
    }

<<- 分析4：getEngineResourceFromCache（） ->>
// 作用：获取图片缓存
// 具体过程：根据缓存Key 从cache中 取值 
// 注：此处的cache对象 = 在构建Glide对象时创建的LruResourceCache对象，即说明使用的是LruCache算法
    private EngineResource<?> getEngineResourceFromCache(Key key) {
        Resource<?> cached = cache.remove(key);
        // 当从LruResourceCache中获取到缓存图片后，会将它从缓存中移除->>回到方法1原处
        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }

<-- 方法2：loadFromActiveResources() -->
// 原理：使用了 弱引用机制
// 具体过程：当在方法1中无法获取内存缓存中的缓存图片时，就会从activeResources中取值
// activeResources = 一个弱引用的HashMap：用于缓存正在使用中的图片
    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                activeResources.remove(key);
            }
        }
        return active;
    }

    ...
} 
```
概括一下
- **loadFromCache**调用了getEngineResourceFromCache获取，getEngineResourceFromCache从cache中获取，获取后会从cache中移除。
- loadFromCache获取完成后，会将此缓存放入activeResources中， 一个弱引用的HashMap：用于缓存正在使用中的图片。
- **loadFromActiveResources**方法：当在loadFromCache中无法获取内存缓存中的缓存图片时，就会从activeResources中取值

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026151625261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 4、开启加载图片线程
当无法从内存缓存中获得缓存的图片时，Glide就会开启加载图片的线程。

但是在开启此线程后并不会立即从网络上加载图片，而是使用Glide的第二级缓存——磁盘缓存。

```java
private Resource<?> decode() throws Exception {

// 在执行 加载图片 线程时（即加载图片时），分两种情况：
// 情况1：从磁盘缓存当中读取图片（默认情况下Glide会优先从缓存当中读取，没有才会去网络源读取图片）
// 情况2：不从磁盘缓存中读取图片

// 情况1：从磁盘缓存中读取缓存图片
    if (isDecodingFromCache()) {
    // 取决于在使用API时是否开启，若采用DiskCacheStrategy.NONE，即不缓存任何图片，即禁用磁盘缓存
        return decodeFromCache();
        // 读取磁盘缓存的入口就是这里，此处主要讲解 ->>直接看步骤4的分析9
    } else {

    // 情况2：不从磁盘缓存中读取图片        
    // 即上文讨论的从网络读取图片，此处不作过多描述
        return decodeFromSource();
    }
} 
```

概括一下

- 当磁盘缓存开启时，调用decodeFromCache()方法获取。
- 否则调用decodeFromSource();从网络加载

## 5、获取磁盘缓存

```java
<--分析9：decodeFromCache()  -->
private Resource<?> decodeFromCache() throws Exception {
    Resource<?> result = null;

        result = decodeJob.decodeResultFromCache();
        // 获取磁盘缓存时，会先获取 转换过后图片 的缓存
        // 即在使用磁盘缓存时设置的模式，如果设置成DiskCacheStrategy.RESULT 或DiskCacheStrategy.ALL就会有该缓存
        // 下面来分析decodeResultFromCache() ->>分析10

    }
    if (result == null) {
        result = decodeJob.decodeSourceFromCache();
        // 如果获取不到 转换过后图片 的缓存，就获取 原始图片 的缓存
        // 即在使用磁盘缓存时设置的模式，如果设置成DiskCacheStrategy.SOURCE 或DiskCacheStrategy.ALL就会有该缓存
        // 下面来分析decodeSourceFromCache() ->>分析12
    }
    return result;
}


<--分析10：decodeFromCache()  -->
public Resource<Z> decodeResultFromCache() throws Exception {
    if (!diskCacheStrategy.cacheResult()) {
        return null;
    }

    Resource<T> transformed = loadFromCache(resultKey);
    // 1. 根据完整的缓存Key（由10个参数共同组成，包括width、height等）获取缓存图片
    // ->>分析11

    Resource<Z> result = transcode(transformed);
    return result;
    // 2. 直接将获取到的图片 数据解码 并 返回
    // 因为图片已经转换过了，所以不需要再作处理
    // 回到分析9原处
}


<--分析11：decodeFromCache()  -->
private Resource<T> loadFromCache(Key key) throws IOException {
    File cacheFile = diskCacheProvider.getDiskCache().get(key);

    // 1. 调用getDiskCache()获取Glide自己编写的DiskLruCache工具类实例
    // 2. 调用上述实例的get() 并 传入完整的缓存Key，最终得到硬盘缓存的文件

    if (cacheFile == null) {
        return null;
        // 如果文件为空就返回null
    }
    Resource<T> result = null;
    try {
        result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
            } finally {
        if (result == null) {
            diskCacheProvider.getDiskCache().delete(key);
        }
    }
    return result;
    // 如果文件不为空，则将它解码成Resource对象后返回
    // 回到分析10原处
}



<--分析12：decodeFromCache()  -->
public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }

    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    // 1. 根据缓存Key的OriginalKey来获取缓存图片
    // 相比完整的缓存Key，OriginalKey只使用了id和signature两个参数，而忽略了大部分的参数
    // 而signature参数大多数情况下用不到，所以基本是由id（也就是图片url）来决定的Original缓存Key
    // 关于loadFromCache（）同分析11，只是传入的缓存Key不一样

    return transformEncodeAndTranscode(decoded);
    // 2. 先将图片数据 转换 再 解码，最终返回
    
} 
```
概括一下整个流程
- 首先调用decodeJob.decodeResultFromCache()，内部再调用loadFromCache(resultKey)，获得经转换过的缓存，其中resultKey是一个完成的key，包括十多个参数，loadFromCache(resultKey)调用getDiskCache()获得DiskLruCache实例，接着通过他得到磁盘缓存。
- 如果转换过的图片获得不到，调用decodeJob.decodeSourceFromCache()，获得原始图片缓存。接着又调用loadFromCache(resultKey.getOriginalKey());  不过这时的key只使用了id和signature两个参数，而忽略了大部分的参数，这也就意味着获得的缓存是原始的图片，最后调用transformEncodeAndTranscode(decoded);进行转码返回

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026153911147.png)
## 6、写入磁盘
写入时机：获得图片资源后，图片加载完成前。
写入类型分为：原始写入、转换后写入

```java
private Resource<?> decode() throws Exception {

// 在执行 加载图片 线程时（即加载图片时），分两种情况：
// 情况1：从磁盘缓存当中读取图片（默认情况下Glide会优先从缓存当中读取，没有才会去网络源读取图片）
// 情况2：不从磁盘缓存中读取图片

// 情况1：从磁盘缓存中读取缓存图片
    if (isDecodingFromCache()) {
        return decodeFromCache();
        // 读取磁盘缓存的入口就是这里，上面已经讲解
    } else {

    // 情况2：不从磁盘缓存中读取图片
        // 即上文讨论的从网络读取图片，不采用缓存
        // 写入磁盘缓存就是在 此处 写入的 ->>分析13
        return decodeFromSource();
    }
}


<--分析13：decodeFromSource()  -->
public Resource<Z> decodeFromSource() throws Exception {
    Resource<T> decoded = decodeSource();
    // 解析图片
    // 写入原始图片 磁盘缓存的入口 ->>分析14

     // 从分析16回来看这里
    return transformEncodeAndTranscode(decoded);
    // 对图片进行转码
    // 写入 转换后图片 磁盘缓存的入口 ->>分析17
}


<--分析14：decodeSource()  -->
private Resource<T> decodeSource() throws Exception {
    Resource<T> decoded = null;
    try {

        final A data = fetcher.loadData(priority);
        // 读取图片数据
        if (isCancelled) {
            return null;
        }
        decoded = decodeFromSourceData(data);
        // 对图片进行解码 ->>分析15
    } finally {
        fetcher.cleanup();
    }
    return decoded;
}

<--分析15：decodeFromSourceData()  -->
private Resource<T> decodeFromSourceData(A data) throws IOException {
    final Resource<T> decoded;
    // 判断是否允许缓存原始图片
    // 即在使用 硬盘缓存API时，是否采用DiskCacheStrategy.ALL 或 DiskCacheStrategy.SOURCE
    if (diskCacheStrategy.cacheSource()) {
        decoded = cacheAndDecodeSourceData(data);
        // 若允许缓存原始图片，则调用cacheAndDecodeSourceData()进行原始图片的缓存 ->>分析16

    } else {
        long startTime = LogTime.getLogTime();
        decoded = loadProvider.getSourceDecoder().decode(data, width, height);
    }
    return decoded;
}

<--分析16：cacheAndDecodeSourceData   -->
private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
   
    ...
    diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);
    // 1. 调用getDiskCache()获取DiskLruCache实例
    // 2. 调用put()写入硬盘缓存
    // 注：原始图片的缓存Key是用的getOriginalKey()，即只有id & signature两个参数
    // 请回到分析13

}

<--分析17：transformEncodeAndTranscode（） -->
private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    
    Resource<T> transformed = transform(decoded);
    // 1. 对图片进行转换

    writeTransformedToCache(transformed);
    // 2. 将 转换过后的图片 写入到硬盘缓存中 -->分析18

    Resource<Z> result = transcode(transformed);
    return result;
}

<-- 分析18：TransformedToCache（） -->
private void writeTransformedToCache(Resource<T> transformed) {
    if (transformed == null || !diskCacheStrategy.cacheResult()) {
        return;
    }

    diskCacheProvider.getDiskCache().put(resultKey, writer);
    // 1. 调用getDiskCache()获取DiskLruCache实例
    // 2. 调用put()写入硬盘缓存
    // 注：转换后图片的缓存Key是用的完整的resultKey，即含10多个参数
} 
```
概括一下上述过程
- 调用decodeFromSource()从网络读取图片
- 调用decodeSource();进行加载图片，然后调用decodeFromSourceData(data);进行解码，顺便进行检测是否允许缓存原始图片，调用cacheAndDecodeSourceData()进行原始图片的缓存。
- 接着调用transformEncodeAndTranscode(decoded);
- 首先调用transform(decoded);对图片进行转换
- 接着调用writeTransformedToCache(transformed)将转换过的图片写入缓存，这时的key是完整的resultKey，即含10多个参数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019102616085862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

##  7、写入内存缓存
缓存时机：图片加载完成后 、图片显示出来前

当图片加载完成后，会在EngineJob中通过Handler发送一条消息将执行逻辑切回到主线程当中，从而执行handleResultOnMainThread()

```java
    private void handleResultOnMainThread() {
        ... 
        // 关注1：写入 弱引用缓存
        engineResource = engineResourceFactory.build(resource, isCacheable);
        listener.onEngineJobComplete(key, engineResource);

        // 关注2：写入 LruCache算法的缓存
        engineResource.acquire();

        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        engineResource.release();
    } 
```

写入 内存缓存分为：写入弱引用缓存、LruCache算法的缓存，内存缓存只缓存 转换后的图片
### 写入弱引用缓存

```java
    private void handleResultOnMainThread() {
        ...

        // 写入 弱引用缓存
        engineResource = engineResourceFactory.build(resource, isCacheable);
        // 创建一个包含图片资源resource的EngineResource对象

        listener.onEngineJobComplete(key, engineResource);
        // 将上述创建的EngineResource对象传入到Engine.onEngineJobComplete() ->>分析6


        // 写入LruCache算法的缓存（先忽略）
        engineResource.acquire();

        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        engineResource.release();
    }


<<- 分析6：onEngineJobComplete()（） ->>
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
    ...    

    @Override
    public void onEngineJobComplete(Key key, EngineResource<?> resource) {
        Util.assertMainThread();

        if (resource != null) {
            resource.setResourceListener(key, this);
            if (resource.isCacheable()) {
                activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
                // 将 传进来的EngineResource对象 添加到activeResources（）中
                // 即写入了弱引用 内存缓存
            }
        }
        jobs.remove(key);
    } 
```
概括一下

- 首先创建一个包含图片资源resource的EngineResource对象 
- 将传进来的EngineResource对象添加到activeResources中

### 写入LruCache

```java
    private void handleResultOnMainThread() {
        ... 
        // 写入 LruCache算法的缓存
        engineResource.acquire();
        // 标记1 
        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                // 标记2
                cb.onResourceReady(engineResource);
            }
        }
        engineResource.release();
        // 标记3
    } 
```
缓存原理
- 用 一个 acquired 变量 记录图片被引用的次数
- 加载图片时：调用 acquire() ，变量加1

```java
<-- 分析7：acquire() -->
    void acquire() {
        if (isRecycled) {
            throw new IllegalStateException("Cannot acquire a recycled resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call acquire on the main thread");
        }
        ++acquired;
        // 当调用acquire()时，acquired变量 +1
    } 
```
不加载图片时，调用 release() 时，变量减1

```java
<-- 分析8：release()  -->
    void release() {
        if (acquired <= 0) {
            throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call release on the main thread");
        }
        if (--acquired == 0) {
            listener.onResourceReleased(key, this);
            // 若acquired变量 = 0，即说明图片已经不再被使用
            // 调用listener.onResourceReleased()释放资源
            // 该listener = Engine对象，Engine.onResourceReleased()->>分析9
        }
    }
}

<-- 分析9：onResourceReleased（）  -->

public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    private final MemoryCache cache;
    private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
    ...    

    @Override
    public void onResourceReleased(Key cacheKey, EngineResource resource) {
        Util.assertMainThread();
        activeResources.remove(cacheKey);
        // 步骤1：将缓存图片从activeResources弱引用缓存中移除

        if (resource.isCacheable()) {
            cache.put(cacheKey, resource);
            // 步骤2：将该图片缓存放在LruResourceCache缓存中
        } else {
            resourceRecycler.recycle(resource);
        }
    }
    ...
} 
```

概括一下
- 当 acquired 变量 >0 时，说明图片正在使用，即该图片缓存继续存放到activeResources弱引用缓存中
- 当 acquired变量 = 0，即说明图片已经不再被使用，就将该图片的缓存Key从 activeResources弱引用缓存中移除，并存放到LruResourceCache缓存中 
-  正在使用中的图片 采用 弱引用 的内存缓存
- 不在使用中的图片 采用 LruCache算法 的内存缓存

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026164859288.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026164920571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026171331881.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)