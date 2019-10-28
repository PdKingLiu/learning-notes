[toc]

# 概述
AsyncTask类内部封装了Handler和线程池。可以简化其他线程对UI的操作。

AsyncTask是一个抽象类，我们需要创建子类去继承它，并且重组一些方法。

**参数**

- Params：指定传给任务执行时的参数的类型

- Progress：指定后台任务执行时将任务进度返回给UI线程的参数类型

- Result：指定任务完成后返回的结果的类型

**方法**

- onPreExecute()：这个方法在UI线程调用，用于在任务执行前做一些初始化操作，如在界面上显示加载进度控件

- doInBackground：在onPreExecute()结束之后立即在后台线程调用，用于耗时操作。在这个方法中可调用publishProgress方法返回任务的执行进度

- onProgressUpdate：在doInBackground调用publishProgress后被调用，工作在UI线程

- onPostExecute：后台任务结束后被调用，工作在UI线程

# 源码

## 3.0前的AsyncTask
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028223339460.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 线程池使用的是ThreadPoolExecutor，核心线程数为5个，最大线程数为128，非核心线程空闲等待新任务的最长时间为1s。
- 阻塞队列是LinkedBlockQueue，他的容量是10
- 缺点：线程池最大线程数是128，加上阻塞队列的10个任务，最多容纳138个任务。

## 3.0后的AsyncTask

这个类的实现，主要有线程池以及Handler两部分。

构造方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028223849126.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- **WorkerRunnable实现了Callable接口，并实现了它的call方法，在call方法中调用了doInBackground（mParams）来处理任务并得到结果，并最终调用postResult将结果投递出去。**
- **FutureTask是一个可管理的异步任务，它实现了Runnable和Futrue这两个接口。它可以包装Runnable和Callable，并提供给Executor执行。**
- **WorkerRunnable作为参数传递给了FutureTask。**

执行一个任务的时候，调用的是execute方法：

```java
public final AsyncTask execute(Params... params){
    return executeOnExecutor(sDefaultExecutor, params);
}
public final AsyncTask executeOnExecutor(Executor exec,  
        Params... params) {  
    if (mStatus != Status.PENDING) {  
        switch (mStatus) {  
            case RUNNING:  
                throw new IllegalStateException("Cannot execute task:" + " the task is already running."); 
                          
            case FINISHED:  
                throw new IllegalStateException("Cannot execute task:"  + " the task has already been executed "  + "(a task can be executed only once)");    
                       
                       
        }  
    }  
  
    mStatus = Status.RUNNING;  
    //先执行 onPreExecute
    onPreExecute();  
  
    mWorker.mParams = params;  
  
    exec.execute(mFuture);  
    return this; 
} 
```
概述一下

- 先检测任务是否已经执行或者执行结束。然后将任务标记为running。
- 开始执行onPreExecute方法。
- 接着把参数赋值给mWorker对象，他是一个Callable对象，最终会被包装为FutureTask。
- 接下来会调用exec的execute方法，并将mFuture也就是前面讲到的FutureTask传进去。
- exec是传进来的参数sDefaultExecutor，它是一个串行的线程池 SerialExecutor，代码如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028224207264.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
概述一下：

- 当调用SerialExecutor 的execute方法时，会将FutureTask加入到mTasks中。
- 当任务执行完或者当前没有活动的任务时都会执行scheduleNext方法，它会从 mTasks 取出 FutureTask 任务并交由 THREAD_POOL_EXECUTOR 处理。
- 从这里可以看出SerialExecutor是串行执行的。注释2处执行了FutureTask的run方法，它最终会调用WorkerRunnable的call方法。

postReult方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028224619183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- postResult方法中会创建Message，将结果赋值给这个Message。
- 通过getHandler方法得到Handler，并通过这个Handler发送消息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028224701487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019102822472494.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
接收到MESSAGE_POST_RESULT消息后会调用AsyncTask的finish方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028224753372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 如果AsyncTask任务被取消了，则执行onCancelled方法，否则就调用onPostExecute方法。

线程池SerialExecutor主要用来处理排队，将任务串行处理。在SerialExecutor中调用 scheduleNext 方法时，将任务交给 THREAD_POOL_EXECUTOR。

THREAD_POOL_EXECUTOR同样是一个线程池，用来处理任务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191028225117637.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- THREAD_POOL_EXECUTOR 指的就是 threadPoolExecutor，其核心线程和线程池允许创建的最大线程数都是由CPU的核数来计算出来的。
- 采用的阻塞队列仍旧是LinkedBlockingQueue，容量为128。

**总结**

- **Android 3.0及以上**版本用SerialExecutor作为默认的线程，它将任务串行地处理，保证一个时间段只有一个任务执行；
- **而Android 3.0之前**的版本是并行处理的。
- Android 3.0之前版本的缺点在Android 3.0之后的版本中也不会出现，因为线程是一个接一个执行的，不会出现超过任务数而执行饱和策略的情况。
- 如果想要在Android 3.0及以上版本使用并行的线程处理，可以使用如下的代码：
	```java
	asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,"");
	```
