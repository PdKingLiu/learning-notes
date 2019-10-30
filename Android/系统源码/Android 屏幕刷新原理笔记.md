[toc]

# 概述
屏幕刷新包括三个步骤：
- CPU计算屏幕数据，，把计算好数据交给GPU。
- GPU会对图形数据进行渲染，渲染好后放到buffer里存起来。
- 接下来display负责把buffer里的数据呈现到屏幕上。

显示过程，简单的说就是CPU/GPU准备好数据，存入buffer，display每隔一段时间去buffer里取数据，然后显示出来。display读取的频率是固定的，比如每个16ms读一次，但是CPU/GPU写数据是完全无规律的。

对于Android而言：CPU 计算屏幕数据指的也就是 View 树的绘制过程，也就是 Activity 对应的视图树从根布局 DecorView 开始层层遍历每个 View，分别执行测量、布局、绘制三个操作的过程。

也就是说，我们常说的 Android 每隔 16.6ms 刷新一次屏幕其实是指：底层以固定的频率，比如每 16.6ms 将 buffer 里的屏幕数据显示出来。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191030082014525.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

Display 这一行可以理解成屏幕，所以可以看到，底层是以固定的频率发出 VSync 信号的，而这个固定频率就是我们常说的每 16.6ms 发送一个 VSync 信号，至于什么叫 VSync 信号，我们可以不用深入去了解，只要清楚这个信号就是屏幕刷新的信号就可以了。

CPU 蓝色的这行，上面也说过了，CPU 这块的耗时其实就是我们 app 绘制当前 View 树的时间，而这段时间就跟我们自己写的代码有关系了，如果你的布局很复杂，层次嵌套很多，每一帧内需要刷新的 View 又很多时，那么每一帧的绘制耗时自然就会多一点。

- 我们常说的 Android 每隔 16.6 ms 刷新一次屏幕其实是指底层会以这个固定频率来切换每一帧的画面。
- 这个每一帧的画面也就是我们的 app 绘制视图树（View 树）计算而来的，这个工作是交由 CPU 处理，耗时的长短取决于我们写的代码：布局复不复杂，层次深不深，同一帧内刷新的 View 的数量多不多。
- CPU 绘制视图树来计算下一帧画面数据的工作是在屏幕刷新信号来的时候才开始工作的，而当这个工作处理完毕后，也就是下一帧的画面数据已经全部计算完毕，也不会马上显示到屏幕上，而是会等下一个屏幕刷新信号来的时候再交由底层将计算完毕的屏幕画面数据显示出来。
- 当我们的 app 界面不需要刷新时（用户无操作，界面无动画），app 就接收不到屏幕刷新信号所以也就不会让 CPU 再去绘制视图树计算画面数据工作，但是底层仍然会每隔 16.6 ms 切换下一帧的画面，只是这个下一帧画面一直是相同的内容。


**为什么界面不刷新时 app 就接收不到屏幕刷新信号了？为什么绘制视图树计算下一帧画面的工作会是在屏幕刷新信号来的时候才开始的？**

# 源码
ViewRootImpl 与 DecorView 的绑定.

View#invalidate() 是请求重绘的一个操作，所以我们切入点可以从这个方法开始一步步跟下去。

Android 设备呈现到界面上的大多数情况下都是一个 Activity，真正承载视图的是一个 Window，每个 Window 都有一个 DecorView，我们调用 setContentView() 其实是将我们自己写的布局文件添加到以 DecorView 为根布局的一个 ViewGroup 里，构成一颗 View 树。

每个 Activity 对应一颗以 DecorView 为根布局的 View 树，但其实 DecorView 还有 mParent，而且就是 ViewRootImpl，而且每个界面上的 View 的刷新，绘制，点击事件的分发其实都是由 ViewRootImpl 作为发起者的，由 ViewRootImpl 控制这些操作从 DecorView 开始遍历 View 树去分发处理。

## ViewRootImpl 与 DecorView 的绑定

跟着 invalidate() 一步步往下走的时候，发现最后跟到了 ViewRootImpl#scheduleTraversals() 就停止了。

Android 设备呈现到界面上的大多数情况下都是一个 Activity，真正承载视图的是一个 Window，每个 Window 都有一个 DecorView，我们调用 setContentView() 其实是将我们自己写的布局文件添加到以 DecorView 为根布局的一个 ViewGroup 里，构成一颗 View 树。

每个 Activity 对应一颗以 DecorView 为根布局的 View 树，但其实 DecorView 还有 mParent，而且就是 ViewRootImpl，而且每个界面上的 View 的刷新，绘制，点击事件的分发其实都是由 ViewRootImpl 作为发起者的，由 ViewRootImpl 控制这些操作从 DecorView 开始遍历 View 树去分发处理。

View#invalidate() 时，也可以看到内部其实是有一个 do{}while() 循环来不断寻找 mParent，所以最终才会走到 ViewRootImpl 里去，那么可能大伙就会疑问了，为什么 DecorView 的 mParent 会是 ViewRootImpl 呢？换个问法也就是，在什么时候将 DevorView 和 ViewRootImpl 绑定起来？

Activity 的启动是在 ActivityThread 里完成的，handleLaunchActivity() 会依次间接的执行到 Activity 的 onCreate(), onStart(), onResume()。在执行完这些后 ActivityThread 会调用 WindowManager#addView()，而这个 addView() 最终其实是调用了 WindowManagerGlobal 的  addView() 方法，我们就从这里开始看：

```java
//WindowManagerGlobal#addView
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    ...
    synchronized (mLock) {
        ...
        //1. 实例化一个 ViewRootImpl对象
        root = new ViewRootImpl(view.getContext(), display);
        ...
        mViews.add(view);
        mRoots.add(root);
        ...
    }
    try {
        //2. 调用ViewRootImpl的setView()，并将DecorView作为参数传递进去
        root.setView(view, wparams, panelParentView);
    }...  
}
```

WindowManager 维护着所有 Activity 的 DecorView 和 ViewRootImpl。这里初始化了一个 ViewRootImpl，然后调用了它的 setView() 方法，将 DevorView 作为参数传递了进去。所以看看 ViewRootImpl 中的 setView() 做了什么：

```java
//ViewRootImpl#setView
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            //1. view 是 DecorView
            mView = view;
            ...
            //2.发起布局请求
            requestLayout();
            ...
            //3.将当前ViewRootImpl对象this,作为参数调用了DecorView的assignParent
            view.assignParent(this);
            ...
        }
    }
}
```
在 setView() 方法里调用了 DecorView 的 assignParent() 方法，所以去看看 View 的这个方法：

```java
//View#assignParent
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = null;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RunTimeException("view " + this + " is already has a parent")
    }
}
```
参数是 ViewParent，而 ViewRootImpl 是实现了 ViewParent 接口的，所以在这里就将 DecorView 和 ViewRootImpl 绑定起来了。每个Activity 的根布局都是 DecorView，而 DecorView 的 parent 又是 ViewRootImpl，所以在子 View 里执行 invalidate() 之类的操作，循环找 parent 时，最后都会走到 ViewRootImpl 里来。 

跟界面刷新相关的方法里应该都会有一个循环找 parent 的方法，或者是不断调用 parent 的方法，这样最终才都会走到 ViewRootImpl 里，也就是说实际上 View 的刷新都是由 ViewRootImpl 来控制的。

即使是界面上一个小小的 View 发起了重绘请求时，都要层层走到 ViewRootImpl，由它来发起重绘请求，然后再由它来开始遍历 View 树，一直遍历到这个需要重绘的 View 再调用它的 onDraw() 方法进行绘制。

重新看回 ViewRootImpl 的 setView() 这个方法，这个方法里还调用了一个 requestLayout() 方法：

```java
//ViewRootImpl#requestLayout
@Override
public void requestLayout() {
    if (!mHandingLayoutInLayoutRequest) {
        //1.检查该操作是否是在主线程中执行
        checkThread();
        mLayoutRequested = true;
        //2.安排一次遍历绘制View树的任务
        scheduleTraversals();
    }
}
```
这里调用了一个 scheduleTraversals()，还记得当 View 发起重绘操作 invalidate() 时，最后也调用了 scheduleTraversals() 这个方法么。其实这个方法就是屏幕刷新的关键，它是安排一次绘制 View 树的任务等待执行，具体后面再说。

也就是说，其实打开一个 Activity，当它的 onCreate---onResume 生命周期都走完后，才将它的 DecoView 与新建的一个 ViewRootImpl 对象绑定起来，同时开始安排一次遍历 View 任务也就是绘制 View 树的操作等待执行，然后将 DecoView 的 parent 设置成 ViewRootImpl 对象。

这也就是为什么在 onCreate---onResume 里获取不到 View 宽高的原因，因为在这个时刻 ViewRootImpl 甚至都还没创建，更不用说是否已经执行过测量操作了。

还可以得到一点信息是，一个 Activity 界面的绘制，其实是在 onResume() 之后才开始的。

## ViewRootImpl # scheduleTraversals
调用一个 View 的 invalidate() 请求重绘操作，内部原来是要层层通知到 ViewRootImpl 的 scheduleTraversals() 里去。

```java
//ViewRootImpl#scheduleTraversals
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreograhper.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```
mTraversalScheduled 这个 boolean 变量的作用等会再来看，先看看 mChoreographer.postCallback() 这个方法，传入了三个参数，第二个参数是一个 Runnable 对象，先来看看这个 Runnable：

```java
//ViewRootImpl$TraversalRunnable
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
//ViewRootImpl成员变量
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
做的事很简单，就调用了一个方法，doTraversal():

```java
//ViewRootImpl#doTraversal
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...

        //1. 遍历绘制View树
        performTraversals();
        ...
    }
}
```
看看这个方法做的事，跟 scheduleTraversals() 正好相反，一个将变量置成 true，这里置成 false，一个是 postSyncBarrier()，这里是 removeSyncBarrier()，具体作用等会再说，继续先看看 performTraversals()，这个方法也是屏幕刷新的关键：

```java
//ViewRootImpl#performTraversals
private void performTraversals() {
    //该方法实在太过复杂，所以将无关代码全部都省略掉，只留下关键代码和代码结构
    ...
    if (...) {
        ...
        if (...) {
            if (...) {
                ...
                //1.测量
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                ...
                layoutRequested = true;
            }
        }
    } ...
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    ...
    if (didLayout) {
        //2.布局
        performLayout(lp, mWidth, mHeight);
        ...
    }
    ...
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (!cancelDraw && !newSurface) {
        ...
        //3.绘制
        performDraw();
    }...

    ...
}
```
**View 的测量、布局、绘制三大流程都是交由 ViewRootImpl 发起，而且还都是在 performTraversals() 方法中发起的，所以这个方法的逻辑很复杂，因为每次都需要根据相应状态判断是否需要三个流程都走，有时可能只需要执行 performDraw() 绘制流程，有时可能只执行 performMeasure() 测量和 performLayout() 布局流程（一般测量和布局流程是一起执行的）。不管哪个流程都会遍历一次 View 树，所以其实界面的绘制是需要遍历很多次的，如果页面层次太过复杂，每一帧需要刷新的 View 又很多时，耗时就会长一点。**

测量、布局、绘制这些流程在遍历时并不一定会把整颗 View 树都遍历一遍，ViewGroup 在传递这些流程时，还会再根据相应状态判断是否需要继续往下传递。

了解了 performTraversals() 是刷新界面的源头后，接下去就需要了解下它是什么时候执行的，和 scheduleTraversals() 又是什么关系？

performTraversals() 是在 doTraversal() 中被调用的，而 doTraversal() 又被封装到一个 Runnable 里，那么关键就是这个 Runnable 什么时候被执行了？

## Choreographer

scheduleTraversals() 里调用了 Choreographer 的 postCallback() 将 Runnable 作为参数传了进去

```java
//Choreograhper#postCallback
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}
//Choreograhper#postCallbackDelayed
pubic void postCallbackDelayed(int callbackType, Runnable action, Object token, long delayMillis) {
    ...  

    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

//Choreograhper#postCallbackDelayedInternal
private void postCallbackDelayedInternal(int callbackType, Object action, Object token, long delayMillis) {
    ...

    synchronized (mLock) {
        //1.获取当前时间戳
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //2.根据时间戳将Runnable任务添加到指定的队列中
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        //3.因为postCallback默认传入delay = 0,所以代码会走进if里面
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {...}
    }
}
```
因为 postCallback() 调用 postCallbackDelayed() 时传了 delay = 0 进去，所以在 postCallbackDelayedInternal() 里面会先根据当前时间戳将这个 Runnable 保存到一个 mCallbackQueue 队列里，这个队列跟 MessageQueue 很相似，里面待执行的任务都是根据一个时间戳来排序。然后走了 scheduleFrameLocked() 方法这边，看看做了些什么：

```java
//Choreograhper#scheduleFrameLocked
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        //1.系统4.0之后该变量默认为true,所以会走进if里
        if (USE_VSYNC) {
            ...

            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } ...
    }
}
```
如果代码走了 else 这边来发送一个消息，那么这个消息做的事肯定很重要，因为对这个 Message 设置了异步的标志而且用了sendMessageAtFrontOfQueue() 方法，这个方法是将这个 Message 直接放到 MessageQueue 队列里的头部，可以理解成设置了这个 Message 为最高优先级，那么先看看这个 Message 做了些什么：

```java
//Choreograhper#scheduleFrameLocked
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        //1.系统4.0之后该变量默认为true,所以会走进if里
        if (USE_VSYNC) {
            ...

            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } ...
    }
}
```
这个 Message 设置了异步的标志而且用了sendMessageAtFrontOfQueue() 方法，这个方法是将这个 Message 直接放到 MessageQueue 队列里的头部，可以理解成设置了这个 Message 为最高优先级，先看看这个 Message ：

```java
//Choreograhper$FrameHandler#handleMessage
private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ...
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            ...
        }
    }
}

//Choreographer#doScheduleVsync
void doScheduleVsync() {
    synchronized (mLock) {
        if (mFrameScheduled) {
            scheduleVsyncLocked();
        }
    }
}
```
所以这个 Message 最后做的事就是 scheduleVsyncLocked()。我们回到 scheduleFrameLocked() 这个方法里，当走 if 里的代码时，直接调用了 scheduleVsyncLocked()，当走 else 里的代码时，发了一个最高优先级的 Message，这个 Message 也是执行 scheduleVsyncLocked()。既然两边最后调用的都是同一个方法，那么为什么这么做呢？

关键在于 if 条件里那个方法是用来判断当前是否是在主线程的，我们知道主线程也是一直在执行着一个个的 Message，那么如果在主线程的话，直接调用这个方法，那么这个方法就可以直接被执行了，如果不是在主线程，那么 post 一个最高优先级的 Message 到主线程去，保证这个方法可以第一时间得到处理。

那么这个方法是干嘛的呢，为什么需要在最短时间内被执行呢，而且只能在主线程？

```java
//Choreographer#scheduleVsyncLocked
private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
//DisplayEventReceiver#scheduleVsync
/**
 * Schedules a single vertical sync pulse to be delivered when the next
 * display frame begins.
 */
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event " + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```
调用了 native 层的一个方法。

到这里为止，我们知道一个 View 发起刷新的操作时，会层层通知到 ViewRootImpl 的 scheduleTraversals() 里去，然后这个方法会将遍历绘制 View 树的操作 performTraversals() 封装到 Runnable 里，传给 Choreographer，以当前的时间戳放进一个 mCallbackQueue 队列里，然后调用了 native 层的一个方法就跟不下去了。所以这个 Runnable 什么时候会被执行还不清楚。那么，下去的重点就是搞清楚它什么时候从队列里被拿出来执行了？

既然这个 Runnable 操作被放在一个 mCallbackQueue 队列里，那就从这个队列着手，看看这个队列的取操作在哪被执行了：

```java
//Choreographer$CallbackQueue
private final class CallbackQueue {
    private CallbackRecord mHead;
    ...
    //1.取操作
    public CallbackRecord extractDueCallbacksLocked(long now){...}  
    //2.入队列操作
    public void addCallbackLocked(long dueTime, Object action, Object token) {...}
    ...  
}

//Choreographer#doCallbacks
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized(mLock) {
        ...
        //1.这个队列跟MessageQueue很相似，所以取的时候需要传入一个时间戳，因为队头的任务可能还没到设定的执行时间
        callback = mCallbackQueues[callbackType].extractDueCallbacksLocked(now / TimeUtils.NANOS_PER_MS);
        ...
    }
}

//Choreographer#doFrame
void doFrame(long frameTimeNanos, int frame) {
    ...
    try {
        ...
        //1.这个参数跟 ViewRootImpl调用mChoreographer.postCallback()时传进的第一个参数是一致的
        doCallbacks(Choreograhper.CALLBACK_TRAVERSAL, frameTimeNanos);
        ...
    }...
}
```
我们说过在 ViewRootImpl 的 scheduleTraversals() 里会将遍历 View 树绘制的操作封装到 Runnable 里，然后调用 Choreographer 的 postCallback() 将这个 Runnable 放进队列里么，而当时调用 postCallback() 时传入了多个参数，这是因为 Choreographer 里有多个队列，而第一个参数 Choreographer.CALLBACK_TRAVERSAL 这个参数是用来区分队列的，可以理解成各个队列的 key 值。

那么这样一来，就找到关键的方法了：doFrame()，这个方法里会根据一个时间戳去队列里取任务出来执行，而这个任务就是 ViewRootImpl 封装起来的 doTraversal() 操作，而 doTraversal() 会去调用 performTraversals() 开始根据需要测量、布局、绘制整颗 View 树。所以剩下的问题就是 doFrame() 这个方法在哪里被调用了。

```java
//Choreographer$FrameDisplayEventReceiver
private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
    ...
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        //1.这个这里的this，该message做的事其实是下面的run()方法
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```
这个继承自 DisplayEventReceiver 的 FrameDisplayEventReceiver 类的作用很重要。

FrameDisplayEventReceiver继承自DisplayEventReceiver接收底层的VSync信号开始处理UI过程。VSync信号由SurfaceFlinger实现并定时发送。FrameDisplayEventReceiver收到信号后，**调用onVsync方法组织消息发送到主线程处理**。这个消息**主要内容就是run方法里面的doFrame**了，这里mTimestampNanos是信号到来的时间参数。

也就是说，onVsync() 是底层会回调的，可以理解成每隔 16.6ms 一个帧信号来的时候，底层就会回调这个方法，当然前提是我们得先注册，这样底层才能找到我们 app 并回调。当这个方法被回调时，内部发起了一个 Message，注意看代码对这个 Message 设置了 callback 为 this，Handler 在处理消息时会先查看 Message 是否有 callback，有则优先交由 Message 的 callback 处理消息，没有的话再去看看Handler 有没有 callback，如果也没有才会交由 handleMessage() 这个方法执行。

onVsync() 是由底层回调的，那么它就不是运行在我们 app 的主线程上，毕竟上层 app 对底层是隐藏的。但这个 doFrame() 是个 ui 操作，它需要在主线程中执行，所以才通过 Handler 切到主线程中。

前面分析 scheduleTraversals() 方法时，最后跟到了一个 native 层方法就跟不下去了，现在再回过来想想这个 native 层方法的作用是什么，应该就比较好猜测了。

```java
//DisplayEventReceiver#scheduleVsync
/**
 * Schedules a single vertical sync pulse to be delivered when the next
 * display frame begins.
 */
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event " + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```
大体上是说安排接收一个 vsync 信号。而根据我们的分析，如果这个 vsync 信号发出的话，底层就会回调 DisplayEventReceiver 的 onVsync() 方法。

如果只是这样的话，就有一点说不通了，首先上层 app 对于这些发送 vsync 信号的底层来说肯定是隐藏的，也就是说底层它根本不知道上层 app 的存在，那么在它的每 16.6ms 的帧信号来的时候，它是怎么找到我们的 app，并回调它的方法呢？


这就有点类似于观察者模式，或者说发布-订阅模式。既然上层 app 需要知道底层每隔 16.6ms 的帧信号事件，那么它就需要先注册监听才对，这样底层在发信号的时候，直接去找这些观察者通知它们就行了。

这是我的理解，所以，这样一来，scheduleVsync() 这个调用到了 native 层方法的作用大体上就可以理解成注册监听了，这样底层也才找得到上层 app，并在每 16.6ms 刷新信号发出的时候回调上层 app 的 onVsync() 方法。这样一来，应该就说得通了。

还有一点，scheduleVsync() 注册的监听应该只是监听下一个屏幕刷新信号的事件而已，而不是监听所有的屏幕刷新信号。比如说当前监听了第一帧的刷新信号事件，那么当第一帧的刷新信号来的时候，上层 app 就能接收到事件并作出反应。但如果还想监听第二帧的刷新信号，那么只能等上层 app 接收到第一帧的刷新信号之后再去监听下一帧。

**梳理一下目前的信息**

- 我们知道一个 View 发起刷新的操作时，最终是走到了 ViewRootImpl 的 scheduleTraversals() 里去，然后这个方法会将遍历绘制 View 树的操作 performTraversals() 封装到 Runnable 里，传给 Choreographer，以当前的时间戳放进一个 mCallbackQueue 队列里，然后调用了 native 层的方法向底层注册监听下一个屏幕刷新信号事件。
- 当下一个屏幕刷新信号发出的时候，如果我们 app 有对这个事件进行监听，那么底层它就会回调我们 app 层的 onVsync() 方法来通知。当 onVsync() 被回调时，会发一个 Message 到主线程，将后续的工作切到主线程来执行。
- 切到主线程的工作就是去 mCallbackQueue 队列里根据时间戳将之前放进去的 Runnable 取出来执行，而这些 Runnable 有一个就是遍历绘制 View 树的操作 performTraversals()。在这次的遍历操作中，就会去绘制那些需要刷新的 View。
- 所以说，当我们调用了 invalidate()，requestLayout()，等之类刷新界面的操作时，并不是马上就会执行这些刷新的操作，而是通过 ViewRootImpl 的 scheduleTraversals() 先向底层注册监听下一个屏幕刷新信号事件，然后等下一个屏幕刷新信号来的时候，才会去通过 performTraversals() 遍历绘制 View 树来执行这些刷新操作。

## 过滤一帧内重复的刷新请求
整体上的流程我们已经梳理出来，但还有几点问题需要解决。

我们在一个 16.6ms 的一帧内，代码里可能会有多个 View 发起了刷新请求，这是非常常见的场景了，比如某个动画是有多个 View 一起完成，比如界面发生了滑动等等。

按照我们上面梳理的流程，只要 View 发起了刷新请求最终都会走到 ViewRootImpl 中的 scheduleTraversals() 里去，是吧。而这个方法又会封装一个遍历绘制 View 树的操作 performTraversals() 到 Runnable 然后扔到队列里等刷新信号来的时候取出来执行。

那如果多个 View 发起了刷新请求，岂不是意味着会有多次遍历绘制 View 树的操作？

其实，这点不用担心，还记得我们在最开始分析 scheduleTraverslas() 的时候先跳过了一些代码么？现在我们回过来继续看看这些代码：

```java
//ViewRootImpl#scheduleTraversals
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        //1.注意这个boolean类型的变量
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreograhper.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```
我们上面分析的 scheduleTraversals() 干的那一串工作，前提是 mTraversalScheduled 这个 boolean 类型变量等于 false 才会去执行。那这个变量在什么时候被赋值被 false 了呢：

一个是上图的 doTraversal()，还有就是声明时默认为 false，剩下一个是在取消遍历绘制 View 操作 unscheduleTraversals() 里。

doTraversal()这个方法，就是在 scheduleTraversals() 中封装到 Runnable 里的那个方法。

当我们调用了一次 scheduleTraversals()之后，直到下一个屏幕刷新信号来的时候，doTraversal() 被取出来执行。在这期间重复调用 scheduleTraversals() 都会被过滤掉的。那么为什么需要这样呢？

View  就是在执行 performTraversals() 遍历绘制 View 树过程中层层遍历到需要刷新的 View，然后去绘制它的。既然是遍历，那么不管上一帧内有多少个 View 发起了刷新的请求，在这一次的遍历过程中全部都会去处理的。这也是我们从代码上看到的，每一个屏幕刷新信号来的时候，只会去执行一次 performTraversals()，因为只需遍历一遍，就能够刷新所有的 View 了。

## 同步屏障消息postSyncBarrier()

当我们的 app 接收到屏幕刷新信号时，来不及第一时间就去执行刷新屏幕的操作，这样一来，即使我们将布局优化得很彻底，保证绘制当前 View 树不会超过 16ms，但如果不能第一时间优先处理绘制 View 的工作，那等 16.6 ms 过了，底层需要去切换下一帧的画面了，我们 app 却还没处理完，这样也照样会出现丢帧了吧。而且这种场景是非常有可能出现的吧，毕竟主线程需要处理的事肯定不仅仅是刷新屏幕的事而已，那么这个问题是怎么处理的呢？

```java
//ViewRootImpl#scheduleTraversals
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //1.注意这行代码，往主线程的消息队列里发送了一个同步屏障消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreograhper.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}

//ViewRootImpl#doTraversal
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //1.注意这行代码，移除消息队列里的同步屏障消息
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...

        performTraversals();
        ...
    }
}
```
逻辑走进 Choreographer 前会先往队列里发送一个同步屏障，而当 doTraversal() 被调用时才将同步屏障移除。

这个同步屏障的作用可以理解成拦截同步消息的执行，主线程的 Looper 会一直循环调用 MessageQueue 的 next() 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。

当 next() 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 next() 方法陷入阻塞状态。

如果 next() 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除出队列，否则主线程就一直不会去处理同步屏幕后面的同步消息。

而所有消息默认都是同步消息，只有手动设置了异步标志，这个消息才会是异步消息。另外，同步屏障消息只能由内部来发送，这个接口并没有公开给我们使用。

最后，仔细看上面 Choreographer 里所有跟 message 有关的代码，你会发现，都手动设置了异步消息的标志，所以这些操作是不受到同步屏障影响的。这样做的原因可能就是为了尽可能保证上层 app 在接收到屏幕刷新信号时，可以在第一时间执行遍历绘制 View 树的工作。

## 刷新控制者 ViewRootImpl
所有跟界面刷新相关的操作，其实最终都会走到 ViewRootImpl 中的 scheduleTraversals() 去的。

跟界面刷新有关的操作大概就是下面几种场景吧：

- invalidate(请求重绘)

- requestLayout(重新布局)

- requestFocus(请求焦点)

- startActivity(打开新界面)

- onRestart(重新打开界面)

- KeyEvent(遥控器事件，本质上是焦点导致的刷新)

- Animation(各种动画，本质上是请求重绘导致的刷新)

- RecyclerView滑动（页面滑动，本质上是动画导致的刷新）

- setAdapter(各种adapter的更新)

```java
//ViewRootImpl#requestChildFocus
@Override
public void requestChildFocus(View child, View focused) {
    if (DEBUG_INPUT_RESIZE) {
        Log.v(mTag, "Request child focus: focus now " + focused);
    }
    checkThread();
    scheduleTraversals();
}

//ViewRootImpl#clearChildFocus
@Override
public void clearChildFocus(View child) {
    if (DEBUG_INPUT_RESIZE) {
        Log.v(mTag, "Clearing child focus");
    }
    checkThread();
    scheduleTraversals();
}

//ViewRootImpl#requestLayout
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```
# 总结
- 界面上任何一个 View 的刷新请求最终都会走到 ViewRootImpl 中的 scheduleTraversals() 里来安排一次遍历绘制 View 树的任务；
- scheduleTraversals() 会先过滤掉同一帧内的重复调用，在同一帧内只需要安排一次遍历绘制 View 树的任务即可，这个任务会在下一个屏幕刷新信号到来时调用 performTraversals() 遍历 View 树，遍历过程中会将所有需要刷新的 View 进行重绘；
- 接着 scheduleTraversals() 会往主线程的消息队列中发送一个同步屏障，拦截这个时刻之后所有的同步消息的执行，但不会拦截异步消息，以此来尽可能的保证当接收到屏幕刷新信号时可以尽可能第一时间处理遍历绘制 View 树的工作；
- 发完同步屏障后 scheduleTraversals() 才会开始安排一个遍历绘制 View 树的操作，作法是把 performTraversals() 封装到 Runnable 里面，然后调用 Choreographer 的 postCallback() 方法；
- postCallback() 方法会先将这个 Runnable 任务以当前时间戳放进一个待执行的队列里，然后如果当前是在主线程就会直接调用一个native 层方法，如果不是在主线程，会发一个最高优先级的 message 到主线程，让主线程第一时间调用这个 native 层的方法；
- native 层的这个方法是用来向底层注册监听下一个屏幕刷新信号，当下一个屏幕刷新信号发出时，底层就会回调 Choreographer 的onVsync() 方法来通知上层 app；
- onVsync() 方法被回调时，会往主线程的消息队列中发送一个执行 doFrame() 方法的消息，这个消息是异步消息，所以不会被同步屏障拦截住；
- doFrame() 方法会去取出之前放进待执行队列里的任务来执行，取出来的这个任务实际上是 ViewRootImpl 的 doTraversal() 操作；
- 上述第4步到第8步涉及到的消息都手动设置成了异步消息，所以不会受到同步屏障的拦截；
- doTraversal() 方法会先移除主线程的同步屏障，然后调用 performTraversals() 开始根据当前状态判断是否需要执行performMeasure() 测量、perfromLayout() 布局、performDraw() 绘制流程，在这几个流程中都会去遍历 View 树来刷新需要更新的View；

# 常见问题
**Android 每隔 16.6 ms 刷新一次屏幕到底指的是什么意思？是指每隔 16.6ms 调用 onDraw() 绘制一次么？**

**如果界面一直保持没变的话，那么还会每隔 16.6ms 刷新一次屏幕么？**

我们常说的 Android 每隔 16.6 ms 刷新一次屏幕其实是指底层会以这个固定频率来切换每一帧的画面，而这个每一帧的画面数据就是我们 app 在接收到屏幕刷新信号之后去执行遍历绘制 View 树工作所计算出来的屏幕数据。

而 app 并不是每隔 16.6ms 的屏幕刷新信号都可以接收到，只有当 app 向底层注册监听下一个屏幕刷新信号之后，才能接收到下一个屏幕刷新信号到来的通知。而只有当某个 View 发起了刷新请求时，app 才会去向底层注册监听下一个屏幕刷新信号。

也就是说，只有当界面有刷新的需要时，我们 app 才会在下一个屏幕刷新信号来时，遍历绘制 View 树来重新计算屏幕数据。如果界面没有刷新的需要，一直保持不变时，我们 app 就不会去接收每隔 16.6ms 的屏幕刷新信号事件了，但底层仍然会以这个固定频率来切换每一帧的画面，只是后面这些帧的画面都是相同的而已。

**界面的显示其实就是一个 Activity 的 View 树里所有的 View 都进行测量、布局、绘制操作之后的结果呈现，那么如果这部分工作都完成后，屏幕会马上就刷新么？**

我们 app 只负责计算屏幕数据而已，接收到屏幕刷新信号就去计算，计算完毕就计算完毕了。至于屏幕的刷新，这些是由底层以固定的频率来切换屏幕每一帧的画面。所以即使屏幕数据都计算完毕，屏幕会不会马上刷新就取决于底层是否到了要切换下一帧画面的时机了。

**网上都说避免丢帧的方法之一是保证每次绘制界面的操作要在 16.6ms 内完成，但如果这个 16.6ms 是一个固定的频率的话，请求绘制的操作在代码里被调用的时机是不确定的啊，那么如果某次用户点击屏幕导致的界面刷新操作是在某一个 16.6ms 帧快结束的时候，那么即使这次绘制操作小于 16.6 ms，按道理不也会造成丢帧么？这又该如何理解？ **

代码里调用了某个 View 发起的刷新请求，这个重绘工作并不会马上就开始，而是需要等到下一个屏幕刷新信号来的时候才开始。

**主线程耗时的操作会导致丢帧，但是耗时的操作为什么会导致丢帧？它是如何导致丢帧发生的？**

造成丢帧大体上有两类原因，一是遍历绘制 View 树计算屏幕数据的时间超过了 16.6ms；

二是，主线程一直在处理其他耗时的消息，导致遍历绘制 View 树的工作迟迟不能开始，从而超过了 16.6 ms 底层切换下一帧画面的时机。

第一个原因就是我们写的布局有问题了，需要进行优化了。而第二个原因则是我们常说的避免在主线程中做耗时的任务。

针对第二个原因，系统已经引入了同步屏障消息的机制，尽可能的保证遍历绘制 View 树的工作能够及时进行，但仍没办法完全避免，所以我们还是得尽可能避免主线程耗时工作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191030132347789.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)