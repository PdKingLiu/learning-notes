[toc]

# 概述
第二种方式思路非常清晰，直接Hook Instrumentation。

由activity启动流程知道，startActivity会交给Activity的mInstrumentation.execStartActivity()来处理。

最终会调用ActivityThread里面的performLaunchActivity()方法。performLaunchActivity()方法进而调用 mInstrumentation.newActivity()会创建启动Activity。

所以步骤很简单。

因为有Application.startActivity()和Activity.startActivity()。等会说Application.startActivity()，先说Activity.startActivity()。

1. hook Activity的mInstrumentation变量
2. hook ActivityThread的mInstrumentation变量
3. 在我们的Instrumentation类里面拦截execStartActivity方法，在里面将Intent替换成我们的占坑Activity。
4. 在我们的Instrumentation类里面拦截newActivity方法，将Intent替换回来，再完成Activity的创建。

# 代码
首先是我们的HookedInstrumentation 类，我们通过反射得到原有的Instrumentation  放入我们的类中，在execStartActivity方法中替换Intent，完成AMS的验证。newActivity方法中再将Intent替换回来。这里需要注意一下，因为在newActivity中传入了我们自己的ClassLoader，关于ReflectUtil.setField(ContextThemeWrapper.class, activity, Constants.FIELD_RESOURCES, pluginApp.mResources);这个代码先不要管，下篇会说怎样加载插件APK的资源。

```java
public class HookedInstrumentation extends Instrumentation   {
    public static final String TAG = "Lpp";
    protected Instrumentation mBase;
    private PluginManager mPluginManager;

    public HookedInstrumentation(Instrumentation base, PluginManager pluginManager) {
        mBase = base;
        mPluginManager = pluginManager;
    }

    /**
     * 覆盖掉原始Instrumentation类的对应方法,用于插件内部跳转Activity时适配
     *
     * @Override
     */
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        mPluginManager.hookToStubActivity(intent);

        try {
            Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                    "execStartActivity", Context.class, IBinder.class, IBinder.class,
                    Activity.class, Intent.class, int.class, Bundle.class);
            execStartActivity.setAccessible(true);
            return (ActivityResult) execStartActivity.invoke(mBase, who,
                    contextThread, token, target, intent, requestCode, options);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("do not support!!!" + e.getMessage());
        }
    }

    @Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        if (mPluginManager.hookToPluginActivity(intent)) {
            String targetClassName = intent.getComponent().getClassName();
            PluginApp pluginApp = mPluginManager.getLoadedPluginApk();
            Activity activity = mBase.newActivity(pluginApp.mClassLoader, targetClassName, intent);
            Log.d(TAG, "newActivity: " + activity);
            Log.d(TAG, "newActivity getApplicationContext: " + activity.getApplication());
            ReflectUtil.setField(ContextThemeWrapper.class, activity, Constants.FIELD_RESOURCES, pluginApp.mResources);
            return activity;
        }
        return super.newActivity(cl, className, intent);
    }

}
```
完成我们的Instrumentation类后，接下来就是反射设置给ActivityThread和Activity了。

**Activity**

```java
 public void hookCurrentActivityInstrumentation(Activity activity) {
     ReflectUtil.setActivityInstrumentation(activity, sInstance);
 }
```

```java
 public static void setActivityInstrumentation(Activity activity, PluginManager manager) {
     try {
         sActivityInstrumentation = (Instrumentation) sActivityInstrumentationField.get(activity);
         HookedInstrumentation instrumentation = new HookedInstrumentation(sActivityInstrumentation, manager);
         sActivityInstrumentationField.set(activity, instrumentation);
     } catch (IllegalAccessException e) {
         e.printStackTrace();
     }
 }
```
**ActivityThread**

```java
 public void hookInstrumentation() {
     try {
         Instrumentation baseInstrumentation = ReflectUtil.getInstrumentation();
         final HookedInstrumentation instrumentation =
                 new HookedInstrumentation(baseInstrumentation, this);

         Object activityThread = ReflectUtil.getActivityThread();
         ReflectUtil.setInstrumentation(activityThread, instrumentation);
     } catch (Exception e) {
         e.printStackTrace();
     }
 }
```
下面是初始化的代码，得到ActivityThread等等

```java
 public static final String METHOD_currentActivityThread = "currentActivityThread";
 public static final String CLASS_ActivityThread = "android.app.ActivityThread";
 public static final String FIELD_mInstrumentation = "mInstrumentation";
 public static final String TAG = "Lpp";

 private static Instrumentation sInstrumentation;
 private static Instrumentation sActivityInstrumentation;
 private static Field sActivityThreadInstrumentationField;
 private static Field sActivityInstrumentationField;
 private static Object sActivityThread;

 public static boolean init() {
     //获取当前的ActivityThread对象
     Class<?> activityThreadClass = null;
     try {
         activityThreadClass = Class.forName(CLASS_ActivityThread);
         Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod(METHOD_currentActivityThread);
         currentActivityThreadMethod.setAccessible(true);
         Object currentActivityThread = currentActivityThreadMethod.invoke(null);

         //拿到在ActivityThread类里面的原始mInstrumentation对象
         Field instrumentationField = activityThreadClass.getDeclaredField(FIELD_mInstrumentation);
         instrumentationField.setAccessible(true);
         sActivityThreadInstrumentationField = instrumentationField;

         sInstrumentation = (Instrumentation) instrumentationField.get(currentActivityThread);
         sActivityThread = currentActivityThread;


         sActivityInstrumentationField =  Activity.class.getDeclaredField(FIELD_mInstrumentation);
         sActivityInstrumentationField.setAccessible(true);
         return true;
     } catch (ClassNotFoundException
             | NoSuchMethodException
             | IllegalAccessException
             | InvocationTargetException
             | NoSuchFieldException e) {
         e.printStackTrace();
     }

     return false;
 }
```

# 补充
上面说的是通过Activity.startActivity()完成跳转时的情况。

当我们使用Application.startActivity()完成跳转时，只需要hook ActivityThread里面的Instrumentation就可以了。

让我们来看为什么。

```java
    getApplication().startActivity();
    
    public final Application getApplication() {
        return mApplication;
    }
```
Application继承的是ContextWrapper，ContextWrapper又继承自Context，所以我们在ContextWrapper里面找startActivity方法。

```java
	public class ContextWrapper extends Context {
	    Context mBase;
	    ······
	    @Override
	    public void startActivity(Intent intent) {
	        mBase.startActivity(intent);
	    }
	}		
```
调用了mBase的startActivity，但是mBase是Context类型的，Context是一个抽象类，所以我们寻找他的实现类。

他的实现类是ContextImpl 。寻找他的startActivity方法。

```java
class ContextImpl extends Context {
	······
    final ActivityThread mMainThread;
   
    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in.
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
}
```

可以很清楚的看到，他从mMainThread中得到了Instrumentation对象，并调用了他的execStartActivity方法。而这个mMainThread就是ActivityThread 。

**所以说通过Application启动活动时，只需要hook ActivityThread里面的Instrumentation就可以了**
