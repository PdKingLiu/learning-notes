[TOC]

# Service
Service 是一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。

## 生命周期
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191105141231950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
Service的生命周期分启动服务和绑定服务两种。

**启动方式（startService）**

通过startService启动服务，stopService停止服务。

生命周期为onCreate——onStartCommand——onDestroy

存活的时间位于onStartCommand和onDestroy之间

- onCreate——创建服务时调用，只会调用一次
- onStartCommand——启动服务时回调。一旦启动，服务即可在后台运行。每次调用startService时都会调用。
- onDestroy——停止服务时调用。用来清理所有的资源。需要调用stopSelf或stopService停止服务。 

**绑定方式（bindService）**

通过bindService绑定服务和unbindService解绑服务。

生命周期回调顺序为：onCreate——onBind——onUnbind——onDestroy

存活时间为onBind和onUnbind之间

- onCreate——创建服务时回调，只会调用一次
- onBind——绑定服务时回调，绑定服务提供了一个客户端-服务器接口，允许组件与服务进行交互、发送请求、获取结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作，如果不允许绑定，则应返回null。仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定后，该服务即会被销毁。只有在第一个客户端绑定时，系统才会调用服务的 onBind() 方法来检索 IBinder，系统随后无需再次调用 onBind()，便可将同一 IBinder 传递至任何其他绑定的客户端。
- onUnbind——解绑时回调。当所有客户端都与Service断开绑定时调用，默认返回false，当返回值为true时，后续有新Client绑定时会回调onRebind
- onRebind——重新绑定时回调。onUnbind返回true，且有新Client绑定时调用。
- 停止服务时调用——用来清理所有资源。服务与所有客户端绑定全部取消时，系统会销毁服务。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191105144115481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## IntentService
Service代码默认运行在主线程，如果做一些耗时操作，会出现ANR的情况。为了简单的创建一个异步的、可以自动停止的服务，Android专门提供了一个IntentService类。 只需要实现 onHandleIntent() 方法即可。任务执行完后会自动停止。
## 关于startService和bindService
使用startService启动服务后，服务就会启动，但是这个服务就好像和我们的Activity没有什么关系了。

为了可以和我们的Service进行通信，我们就可以使用bindService方法。

我们可以实现一个继承自Binder的类，在里面实现自己想要完成的方法。

然后在Service的onBind方法中返回这个自定义Binder的实例。

在我们的Activity中实现ServiceConnection的实例，在他的onServiceConnection中将Binder向下转型，然后调用里面的方法。

这样就实现了和Service的通信，当Service在其他进程时，Activity和Service则使用Binder进行进程间通信。

## AndroidManifest.xml
- android:name——服务类名
- android:label——服务的名字，如果此项不设置，那么默认显示的服务名则为类名
- android:icon——服务的图标
- android:permission——申明此服务的权限，这意味着只有提供了该权限的应用才能控制或连接此服务
- android:process——表示该服务是否运行在另外一个进程，如果设置了此项，那么将会在包名后面加上这段字符串表示另一进程的名字
- android:enabled——表示是否能被系统实例化，为true表示可以，为false表示不可以，默认为true
- android:exported——表示该服务是否能够被其他应用程序所控制或连接，不设置默认此项为 false

## 本地Service和远程Service
本地服务依附在主进程上，在一定程度上节约了资源。本地服务因为是在同一进程，因此不需要IPC，也不需要AIDL。相应bindService会方便很多。缺点是主进程被kill后，服务变会终止。

远程服务是独立的进程，对应进程名格式为所在包名加上你指定的android:process字符串。由于是独立的进程，因此在Activity所在进程被kill的是主进程，该服务依然在运行。缺点是该服务是独立的进程，会占用一定资源，并且使用AIDL进行IPC稍微麻烦一点。

startService来说，不管是本地服务还是远程服务，需要做的工作都一样。
## 关于bindService的特点
- bindService启动的service和调用者之间是典型的client-server，这里的调用者是Activity。客户端是一个或多个。
- 客户端可以通过Binder和Service进行通信
- startService启动的服务默认无限期执行（调用stopService和stopSelf除外），bindService的生命周期和client息息相关，当client销毁的时候会自动解绑，也可通过调用解绑的方法进行解绑。当没有任何绑定的时候，Service会自行销毁。
- 回调方法和startService不同。

# Content Provider
- Content Provider是Android系统中为开发者专门提供的不同应用间进行数据共享的组件。

- 提供一套标准的接口用来获取以及操作数据，允许开发者把自己的应用数据根据需求开放给其他应用进行增删改查。

- 系统预置了许多Content Provider用于获取用户数据，例如联系人、日程表等。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191106140036758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

**设计用意**

- 封装。对数据进行封装，提供统一的接口，使用者完全不必关心这些数据是在DB，XML、Preferences或者网络请求来的。当项目需求要改变数据来源时，使用我们的地方完全不需要修改。
- 提供一种跨进程数据共享的方式。

**数据更新通知机制**

数据是在多个应用程序中共享的，当其中一个应用程序改变了这些共享数据的时候，它有责任通知其它应用程序，让它们知道共享数据被修改了，这样它们就可以作相应的处理。

ContentResolver接口的notifyChange函数来通知那些注册了监控特定URI的ContentObserver对象，使得它们可以相应地执行一些处理。ContentObserver可以通过registerContentObserver进行注册。

ContentProvider的onCreate()是在UI线程运行。

------
ContentProvider所提供的query()，insert()，delete()，update()都是在ContentProvider进程的线程池中被调用执行的，而不是进程的主线程中。因为那些方法可能同时被多个线程所调用，所以他们都应该是线程安全的。 


# Broadcast Receiver
广播是一种应用间传输信息的机制，一个广播可以有任意个接受者，也可以不被任何应用程序接收。广播机制是一种典型的发布-订阅模式，即观察者模式。

广播最大的特点是发送方不需要关心接收方是否接收到这个数据，也不必关心接收方如何处理数据的。通过这种方式来实现接方收方完全解耦。

内部通信使用系统的Binder。

## 分类
**普通广播**

普通广播是完全异步的，通过 Context 的 sendBroadcast() 函数来发送，消息传递的效率比较高，但所有的 receivers（接收器）的执行顺序不确定。接收者不能将处理结果传递给下一个接收者，并且无法终止广播 Intent 的传播，直到没有与之匹配的广播接收器为止。

**系统广播**

Android系统中内置了多个系统广播，只要涉及到手机的基本操作，基本上都会发出相应的系统广播。如：开机启动，网络状态改变，拍照，屏幕关闭与开启，电量不足等等。

**本地广播**

有的时候我们并不需要把自己的应用内的信息广播给所有应用，而只是进程内使用，现在使用 Support v4 包中的 LocalBroadcastManager 就能够实现限于应用内的广播。

**有序广播**

有序广播通过 Context.sendOrderedBroadcast() 来发送，所以的广播接收器按照优先级依次执行，广播接收器的优先级通过 receiver 的 intent-filter 中的  android:priority 属性来设置，数值越大优先级越高（参数为 -1000 ~ 1000）。当广播接收器接收到广播后，可以使用 setResult() 函数来将结果传递给下一个广播接收器，然后通过 getResult() 函数来取得上一个广播接收器返回的结果，并可以使用 abortBroadcast() 函数来让系统丢弃该广播，使该广播不再传递到别的广播接收器。

**粘性广播**

粘性消息在发送后就一直存在于系统的消息容器里面，等待对应的处理器去处理，如果暂时没有处理器处理这个消息则一直在消息容器里面处于等待状态，粘性广播的Receiver如果被销毁，那么下次重建时会自动接收到消息数据。 

也就是说该广播会一直保留下去，即使已经有广播接收器处理了该广播，当再有匹配的广播接收器被注册时，此广播仍会被接收。如果你只想处理一遍该广播，可以通过removeStickyBroadcast() 函数实现。


## 注册
 **静态注册**：即在 AndroidManifest.xml 文件中进行注册

```java
<receiver
	android:name=".MyBroadcastReceiver"
	android:enabled="true"
	android:exported="true">
	<intent-filter>
		 <action android:name="com.cyy.broad" />
	</intent-filter>
</receiver>
```

**动态注册**：即在代码中进行注册

```java
public void registerHelloBroadcast() {
	receiver = new MyBroadcastReceiver();
	registerReceiver(receiver, new IntentFilter("com.cyy.broad"));
}
```
