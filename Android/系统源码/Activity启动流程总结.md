学习插件化首先要清楚Activity启动流程。此篇文章对Activity整体的启动流程做一个总结。SDK版本基于25，项目的插件化适配也只做了API 25的。

通常使用startActivity来启动一个Activity

```java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
```
继续跟进

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
} 
```

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
} 
```
最终来到了startActivityForResult

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            mStartedActivity = true;
        }
        cancelInputsAndStartExitTransition(options);
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
} 
```
其中有个很关键的调用

	mInstrumentation.execStartActivity()

可以看到启动交给了Instrumentation类，这个类也是插件化的一个hook点。

跟进execStartActivity方法

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    ...
    int result = ActivityManagerNative.getDefault()
        .startActivity(whoThread, who.getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token, target != null ? target.mEmbeddedID : null,
                requestCode, 0, null, options);
    checkStartActivityResult(result, intent);
    ...
} 
```
对于这个方法的几个参数：
- who：启动Activity的上下文
- contextThread：启动Activity的上下文线程
- token：正在启动Activity的标识
- target：启动该Activity的Activity（回调Activity）

下面调用了checkStartActivityResult方法

这个方法是干什么的呢

```java
public static void checkStartActivityResult(int res, Object intent) {
    if (res >= ActivityManager.START_SUCCESS) {
        return;
    }
    switch (res) {
        case ActivityManager.START_INTENT_NOT_RESOLVED:
        case ActivityManager.START_CLASS_NOT_FOUND:
            if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                throw new ActivityNotFoundException(
                        "Unable to find explicit activity class "
                        + ((Intent)intent).getComponent().toShortString()
                        + "; have you declared this activity in your AndroidManifest.xml?");
            throw new ActivityNotFoundException(
                    "No Activity found to handle " + intent);
        case ActivityManager.START_PERMISSION_DENIED:
            throw new SecurityException("Not allowed to start activity "
                    + intent);
        case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
            throw new AndroidRuntimeException(
                    "FORWARD_RESULT_FLAG used while also requesting a result");
        case ActivityManager.START_NOT_ACTIVITY:
            throw new IllegalArgumentException(
                    "PendingIntent is not an activity");
        case ActivityManager.START_NOT_VOICE_COMPATIBLE:
            throw new SecurityException(
                    "Starting under voice control not allowed for: " + intent);
        case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                    "Session calling startVoiceActivity does not match active session");
        case ActivityManager.START_VOICE_HIDDEN_SESSION:
            throw new IllegalStateException(
                    "Cannot start voice activity on a hidden session");
        case ActivityManager.START_CANCELED:
            throw new AndroidRuntimeException("Activity could not be started for "
                    + intent);
        default:
            throw new AndroidRuntimeException("Unknown error code "
                    + res + " when starting " + intent);
    }
} 
```
很明显，就是根据得到的返回码去做一些合法性的判断。

这个是插件化需要欺骗过的方法。

继续看我们的execStartActivity，它调用了ActivityManagerNative.getDefault()。

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        ...
        IActivityManager am = asInterface(b);
        ...
        return am;
    }
};
static public IActivityManager asInterface(IBinder obj) {
    ...
    return new ActivityManagerProxy(obj);
} 
```
它通过gDefault得到了一个IActivityManager实例ActivityManagerProxy。gDefault是Singleton类型，这里可以理解为ActivityManagerProxy是一个单例。

所以Activity的启动又交给了ActivityManagerProxy。

看一下他的startActivity方法。

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    ...
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    ...
    reply.recycle();
    data.recycle();
    return result;
} 
```

这里如果熟悉Binder的话就知道transact方法。

transact的作用是通过Binder远程调用服务端的方法。

对于参数简单说一下，详细情况可我看我的AIDL和Binder的博客。

transact的参数
- 第一个参数相当于一个code——表明要调用的方法
- 第二个参数是我们要传递的参数
- 第三个是返回值

mRemote.transact()实际上就是调用了AMS的startActivity方法。

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,resulUserHandle.getCallingUserId());
} 
```
关于他的参数
- caller——就是我们的whoThread
- callingPackage——包名
- intent——意图
- resolvedType——注册文件注册时的MIME类型
- resultTo——token
- resultWho——target的id
- requestCode——请求码
- startFlags——启动标志
- profilerInfo——默认null
- options——。。。

AMS中的startActivity方法直接调用了startActivityAsUser方法。

```java
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,profilerInfo, null, null, bOptions, false, userId, null, null);
} 
```

其中enforceNotlsolatedCaller方法是检查是否属于被隔离对象

mUserController.handleIncomingUser检查操作权限然后返回一个用户id

startActivityAsUser方法大概就是检测下权限。

然后返回由mActivityStarter 调用的startActivityMayWait方法。

ActivityStarter：可以说关于Activity启动的所有信息都在这了，然后根据信息把Activity具体分配到哪个任务栈

继续跟进：

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
            Bundle bOptions, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
    ...
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
    ...
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    ...
    final ProcessRecord heavy = mService.mHeavyWeightProcess;
    if (heavy != null && (heavy.info.uid != aInfo.applicationInfo.uid
            || !heavy.processName.equals(aInfo.processName))) {
        ...
    }
    ...
    int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor,
            resultTo, resultWho, requestCode, callingPid,
            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, outRecord, container,
            inTask);
    ...
} 
```
mSupervisor(ActivityStackSupervisor)是用来管理ActivityStack的。

resolveIntent和resolveActivity方法用来确定此次启动activity的信息。

关于heavy(ResolveInfo)涉及到重量级进程(SystemServer, MediaServer,ServiceManager)，如果当前系统中已经存在的重量级进程且不是即将要启动的这个，那么就要给Intent赋值。

startActivityLocked方法

```java
final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask) {
    ...
    ProcessRecord callerApp = null;
    ...
    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    ...
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
            intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
            requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
            options, sourceRecord);
    ...
    err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true, options, inTask);
    ...
    return err;
}
```

callerApp、sourceRecord、resultRecord都为r服务，让记录着整个方法的各种结果，然后带着r调用了startActivityUnchecked方法

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
    voiceInteractor);
    computeLaunchingTaskFlags();
    ...
    mIntent.setFlags(mLaunchFlags);
    mReusedActivity = getReusableIntentActivity();
    ...
    boolean newTask = false;
    ...
     if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
        && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        ...
    }
    ...
    mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
    ...
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
    ...
} 
```
先看setInitialState方法。

```java
private void setInitialState(ActivityRecord r, ActivityOptions options, TaskRecord inTask,
            boolean doResume, int startFlags, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor) {
    reset();
    mStartActivity = r;
    mIntent = r.intent;
    mOptions = options;
    mCallingUid = r.launchedFromUid;
    mSourceRecord = sourceRecord;
    mVoiceSession = voiceSession;
    ...
    mLaunchSingleTop = r.launchMode == LAUNCH_SINGLE_TOP;
    mLaunchSingleInstance = r.launchMode == LAUNCH_SINGLE_INSTANCE;
    mLaunchSingleTask = r.launchMode == LAUNCH_SINGLE_TASK;
    mLaunchFlags = adjustLaunchFlagsToDocumentMode(
            r, mLaunchSingleInstance, mLaunchSingleTask, mIntent.getFlags());
     ...
} 
```
这里面初始了比较重要的mStartActivity，mIntent，mCallingUid，
mSourceRecord，mLaunchFlags。

mLaunchFlags用来记录我们Activity的启动方式，省略的部分都是根据启动方式来初始化一堆变量或进行操作。

下面的方法computeLaunchingTaskFlags还是用来初始化启动标志位的。

getReusableIntentActivity方法中，我们需要去找一个可复用的ActivityRecord，那么只有启动模式为SingleInstance才能真正的复用，因为整个系统就只有一个实例。

下来看后面的方法startActivityLocked

```java
final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
            ActivityOptions options) {
    ...
    mWindowManager.setAppVisibility(r.appToken, true);
    ...
} 
```

mWindowManager.setAppVisibility(r.appToken, true);这句话表示这个Activity已经具备了显示的条件。

接着会调用ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法，接着调ActivityStack的resumeTopActivityUncheckedLocked方法，再调自己的resumeTopActivityInnerLocked方法 


然后，然后，如果Activity已存在需要可见的状态，那么会调用IApplicationThread的scheduleResumeActivity方法

反之调用ActivityStackSupervisor的startSpecificActivityLocked的方法，那么继续会调realStartActivityLocked的方法


终于调用ApplicationThread的scheduleLaunchActivity方法了

```java
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    updateProcessState(procState, false);
    ActivityClientRecord r = new ActivityClientRecord();
    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;
    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;
    r.startsNotResumed = notResumed;
    r.isForward = isForward;
    r.profilerInfo = profilerInfo;
    r.overrideConfig = overrideConfig;
    updatePendingConfiguration(curConfig);
    sendMessage(H.LAUNCH_ACTIVITY, r);
} 
```
通过Handler发送message。

这个handler是H

```java
private class H extends Handler {
    ...
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            ...
    }
} 
```
接收到消息后，他调用了ActivityThread的handleLaunchActivity方法。

```java
WindowManagerGlobal.initialize();//初始化WMS
Activity a = performLaunchActivity(r, customIntent);//创建并启动
```

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;//取出组件信息，并一顿操作
    ...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
         //开始创建Activity实例，通过类加载器创建，看参数就知道了
         ...
         Application app = r.packageInfo.makeApplication(false, mInstrumentation);
         //获取Application
         ...
         activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
         //与window建立关联
         ...
         mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
         //callActivityOnCreate->activity.performCreate->onCreate，之后就很熟悉了
         ...
    }
} 
```
在这里，通过mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
创建了Activity实例。

这实际上也是插件化的第二个Hook点。即hook newActivity方法。

```java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }
    //如果mApplication不为空则直接返回，这也是为什么Application为单例
    ...
    Application app = null;
    ...
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
    //创建Application，跟Activity一样，都是用类加载器创建
    ...
    mApplication = app;//保存下来
    ...
    instrumentation.callApplicationOnCreate(app);
    //callApplicationOnCreate->onCreate
    ...
} 
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191108003434677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)