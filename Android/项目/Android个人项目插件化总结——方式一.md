[toc]

# 整体流程
本篇主要基于Hook代理对象IActivityManager，Activity启动流程不清楚的可以看看我分析Activity流程的文章。

```csharp
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
启动Activity交给了代理对象ActivityManagerNative，他接着会调用AMS启动Activity。

下面checkStartActivityResult(result, intent);是对result和intent做一个检测，也就是看要启动的intent是不是合法。

所以我们需要hook这个ActivityManagerNative的startActivity方法。

我们需要提前创建一个占坑的Activity，用于欺骗系统。

在这个方法里面将intent的信息替换成占坑的Activity信息。剩下流程等就交给AMS，因为我们启动的Activity是合法的，所以不会出现问题。

根据activity的启动流程我们也可以知道，到最后启动activity的准备工作做完后，会调用ApplicationThread的scheduleLaunchActivity方法会给主线程发送一个message。

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

通过handleLaunchActivity方法就正式创建启动Activity了。

在这里就又要借助Handler的消息处理机制，将我们的Intent替换回来了。

```csharp
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
handler最先调用message的CallBack，然后会调用自己的mCallBack，最后才调用handleMessage(msg)。

H类中实现了handleMessage方法，并没有实现mCallBack，所以我们需要通过反射创建我们的mCallback 并赋值给H，这样我们就可以在他处理消息之前做一些我们的处理，也就是将Intent替换回来。

还有一个问题，就是系统如何才能从我们提供的APK中加载class文件呢？

这里我们就要使用ClassLoader了。

这里直接给出结论，Android里面PathClassLoader用于加载系统和主dex文件，DexClassLoader用于其他Dex文件。

所以我们只要创建我们的ClassLoader，加载我们的apk。然后通过反射将最后的结果合并至PathClassLoader里面就可以。

基本流程就是如此

接着我们看一下实现

# 实现
首先创建我们的classloader

```java
    private void hook() throws IllegalAccessException, NoSuchFieldException,
            ClassNotFoundException, NoSuchMethodException, InvocationTargetException {
        String cachePath = getCacheDir().getAbsolutePath();
        String apkPath = Environment.getExternalStorageDirectory().getAbsolutePath() + "/chajian" +
                ".apk";
        DexClassLoader dexClassLoader = new DexClassLoader(apkPath, cachePath, cachePath,
                getClassLoader());
        RejectPluginHelper.loadPlugin(dexClassLoader, getApplicationContext());
        HookHelper.hookActivityManager(this);
        HookHelper.hookActivityThread();
    }
```

在loadPlugin(dexClassLoader, getApplicationContext());里面，我们合并两个dex的list

```java
    //加载插件apk
    public static void loadPlugin(DexClassLoader dexClassLoader, Context applicationContext) throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {

        //获取PathClassLoader
        PathClassLoader pathClassLoader = (PathClassLoader) applicationContext.getClassLoader();

        //分别获取宿主和插件的PathList
        Object suzhuPathList = getPathList(pathClassLoader);
        Object chajianPathList = getPathList(dexClassLoader);

        //获得PathList的element并合并
        Object newElements = mergeElements(getElements(suzhuPathList),
                getElements(chajianPathList));
        Log.d("Lpp", "newElements: " + Array.getLength(newElements));

        //重新设置给宿主的dexElement
        Field field = suzhuPathList.getClass().getDeclaredField("dexElements");
        field.setAccessible(true);
        field.set(suzhuPathList, newElements);
    }

    private static Object mergeElements(Object elements2, Object elements1) {
        Log.d("Lpp", "suzhuPathList: " + Array.getLength(elements1));
        Log.d("Lpp", "chajianPathList: " + Array.getLength(elements2));
        int len1 = Array.getLength(elements1);
        int len2 = Array.getLength(elements2);
        Object newArr = Array.newInstance(elements1.getClass().getComponentType(), len1 + len2);
        for (int i = 0; i < len1; i++) {
            Array.set(newArr, i, Array.get(elements1, i));
        }
        for (int i = len1; i < len1 + len2; i++) {
            Array.set(newArr, i, Array.get(elements2, i - len1));
        }
        return newArr;
    }

    // 获取DexPathList 中的dexElements
    private static Object getElements(Object suzhuPathList) throws NoSuchFieldException,
            IllegalAccessException {
        Class cl = suzhuPathList.getClass();
        Field field = cl.getDeclaredField("dexElements");
        field.setAccessible(true);
        return field.get(suzhuPathList);
    }

    // 获取DexPathList
    private static Object getPathList(Object loader) throws ClassNotFoundException,
            NoSuchFieldException, IllegalAccessException {
        Class cl = Class.forName("dalvik.system.BaseDexClassLoader");
        Field field = cl.getDeclaredField("pathList");
        field.setAccessible(true);
        return field.get(loader);
    }
```

然后通过

```java
    HookHelper.hookActivityManager(this);
    HookHelper.hookActivityThread();
```
分别hook ActivityManager 和 Handler的CallBack

对Hook不熟悉的得先了解了解hook。

```java
    public static void hookActivityManager(Context context) throws ClassNotFoundException, NoSuchFieldException,
            IllegalAccessException {

        //得到ActivityManagerNative的class对象
        Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");
        //得到他的gDefault字段，他是Singleton类型
        Field field = activityManagerNativeClass.getDeclaredField("gDefault");
        field.setAccessible(true);
        //静态直接传null即可
        Object gDefaultObj = field.get(null);

        //得到Singleton class
        Class singletonCLass = Class.forName("android.util.Singleton");
        //得到他的成员mInstance 他是IActivityManager
        Field mInstanceField = singletonCLass.getDeclaredField("mInstance");
        mInstanceField.setAccessible(true);
        //得到gDefault的 mInstance
        Object activityManagerObj = mInstanceField.get(gDefaultObj);

        //给IActivityManager设置代理
        Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{Class.forName("android.app.IActivityManager")},
                new IActivityManagerProxy(activityManagerObj, context));

        //将代理对象设置给gDefault
        mInstanceField.set(gDefaultObj, proxy);
        Log.d("Lpp", "已经替换掉了Intent");
    }
```

代理类

```java
/**
 * @author liupeidong
 * Created on 2019/10/9 11:08
 */
public class IActivityManagerProxy implements InvocationHandler {

    private Context context;

    private Object obj;

    public IActivityManagerProxy(Object obj, Context context) {
        this.context = context;
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("startActivity")) {
            //获得传入的Intent
            Intent targetIntent = null;
            int index;

            for (index = 0; index < args.length; index++) {
                if (args[index] instanceof Intent) {
                    targetIntent = (Intent) args[index];
                    break;
                }
            }

            //创建假Intent
            Intent fakeIntent = new Intent(context, FakeActivity.class);
            //把真的先存里面
            fakeIntent.putExtra("targetIntent", targetIntent);

            //设置到参数里面
            args[index] = fakeIntent;
            return method.invoke(obj, args);
        }
        return method.invoke(obj, args);
    }
}
```

主要做的任务就是Intent替换，然后反射赋值代理类。

接着通过反射，给H赋值我们封装的CallBack

```java
    public static void hookActivityThread() throws ClassNotFoundException, NoSuchMethodException,
            InvocationTargetException, IllegalAccessException, NoSuchFieldException {

        //先得到ActivityThread对象，他有一个返回自己本身的方法
        Class activityThreadClass = Class.forName("android.app.ActivityThread");
        Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod(
                "currentActivityThread");
        currentActivityThreadMethod.setAccessible(true);
        Object activityThreadObj = currentActivityThreadMethod.invoke(null);

        //得到他的成员 mH
        Field mHField = activityThreadClass.getDeclaredField("mH");
        mHField.setAccessible(true);
        Object handleObj = mHField.get(activityThreadObj);

        //给mH设置mCallback
        Field callBackField = Handler.class.getDeclaredField("mCallback");
        callBackField.setAccessible(true);
        ChangeCallBack changeCallBack = new ChangeCallBack((android.os.Handler) handleObj);
        Log.d("Lpp", "hookActivityThread 1: " + changeCallBack);
        callBackField.set(handleObj, changeCallBack);
        Log.d("Lpp", "替换回Intent");

        Class activityThreadClass2 = Class.forName("android.app.ActivityThread");
        Method currentActivityThreadMethod2 = activityThreadClass2.getDeclaredMethod(
                "currentActivityThread");
        currentActivityThreadMethod2.setAccessible(true);
        Object activityThreadObj2 = currentActivityThreadMethod2.invoke(null);

        //得到他的成员 mH
        Field mHField2 = activityThreadClass2.getDeclaredField("mH");
        mHField2.setAccessible(true);
        Object handleObj2 = mHField2.get(activityThreadObj2);

        //给mH设置mCallback
        Field callBackField2 = Handler.class.getDeclaredField("mCallback");
        callBackField2.setAccessible(true);
        Log.d("Lpp", "hookActivityThread 2: " + callBackField2.get(handleObj2));

    }
```

我们的CallBack

```java
/**
 * @author liupeidong
 * Created on 2019/10/9 13:05
 */
public class ChangeCallBack implements Handler.Callback {

    private Handler handler;

    public ChangeCallBack(Handler handler) {
        this.handler = handler;
    }

    @Override
    public boolean handleMessage(Message msg) {
        switch (msg.what) {
            case 100:
                try {
                    handleLaunchActivity(msg);
                } catch (NoSuchFieldException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
                break;
        }
        handler.handleMessage(msg);
        return true;
    }

    private void handleLaunchActivity(Message msg) throws NoSuchFieldException,
            IllegalAccessException {
        Object obj = msg.obj;

        Field intent = obj.getClass().getDeclaredField("intent");
        intent.setAccessible(true);
        Intent fakeIntent = (Intent) intent.get(obj);
        Intent targetIntent = fakeIntent.getParcelableExtra("targetIntent");
        fakeIntent.setComponent(targetIntent.getComponent());
//        intent.set(obj, targetIntent);
    }
```
