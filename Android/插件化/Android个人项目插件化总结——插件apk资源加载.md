[toc]

# 为什么加载不了资源

我们知道获取资源时实际都是使用了Resource。

mResource这个变量在Activity父类ContextThemeWrapper中。

ContextThemeWrapper的mResource的获得是通过Context的实现类得到的。

Context实现类ContextImpl中获得Resources 是通过

	 Resources resources = packageInfo.getResources(mainThread);

packageInfo 是一个LoadedApk对象。

这个对象相当于是一个apk在内存中的表示， Apk文件的相关信息、资源都可以通过此对象获取。

这个LoadedApk是怎么创建的呢。

在ActivityThread的getPackageInfo方法中：

```java
private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,  ClassLoader baseLoader, boolean securityViolation, boolean includeCode,boolean registerPackage) {
    synchronized (mResourcesManager) {
        WeakReference<LoadedApk> ref;
        if (includeCode) {
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }
        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
            if (localLOGV) Slog.v(TAG, (includeCode ? "Loading code package "
                    : "Loading resource-only package ") + aInfo.packageName
                    + " (in " + (mBoundApplication != null
                            ? mBoundApplication.processName : null)
                    + ")");
                    //在这里创建
            packageInfo = new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode 
                        && aInfo.flags & ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);
          ······
        }
        return packageInfo;
    }
} 
```
可以看到构造参数里面有个参数是ApplicationInfo 。

所以说不管前面是hook IActivityManager合并dex还是hook Instrumentation 传入我们的ClassLoader，这个LoadedApk里传入的ApplicationInfo其实还是我们宿主的，并不是插件apk中的。

所以我们在插件apk中直接使用资源，等到插件apk被宿主调用器后，使用的是宿主的资源库，而宿主的资源中并没有我们插件apk中的资源，所以一运行的时候就会报错。

那么怎么解决呢。

我们创建一个Resource给我们的插件Activity就可以了。

# 如何加载资源

我们知道一般得到资源调用的是getResource方法，最终调用了Context.getResources()方法。

我们看看Context实现类ContextImpl对应的方法。

```java
 @Override
public Resources getResources() {
    return mResources;
}
```
再看看mResource是如何赋值的。

```java
private ContextImpl(ContextImpl container, ActivityThread mainThread,
        LoadedApk packageInfo, IBinder activityToken, UserHandle user, int flags,
        Display display, Configuration overrideConfiguration, int createDisplayWithId) {

    ...

    mPackageInfo = packageInfo;

    //这里拿到了一个ResourcesManager,单例的，说明我们应用当中使用的都是同一套资源
    mResourcesManager = ResourcesManager.getInstance();

    ...

    //LoadedApk对象中得到Resources对象
    Resources resources = packageInfo.getResources(mainThread);
    if (resources != null) {
        if (activityToken != null
                || displayId != Display.DEFAULT_DISPLAY
                || overrideConfiguration != null
                || (compatInfo != null && compatInfo.applicationScale
                        != resources.getCompatibilityInfo().applicationScale)) {
            resources = mResourcesManager.getTopLevelResources(packageInfo.getResDir(),
                    packageInfo.getSplitResDirs(), packageInfo.getOverlayDirs(),
                    packageInfo.getApplicationInfo().sharedLibraryFiles, displayId,
                    overrideConfiguration, compatInfo, activityToken);
        }
    }
    //给mResources赋值
    mResources = resources;
    ...

} 
```
ResourcesManager是一个单例，通过它获得mResources，保证了我们每个Context获取的都是同样的资源。

resources通过getTopLevelResources方法赋值，我们具体看看是如何创建的

```java
public Resources getTopLevelResources(String resDir, String[] splitResDirs,
            String[] overlayDirs, String[] libDirs, int displayId,
            Configuration overrideConfiguration, CompatibilityInfo compatInfo, IBinder token) {

		······
        //创建一个AssetManager对象
        AssetManager assets = new AssetManager();
        // resDir can be null if the 'android' package is creating a new Resources object.
        // This is fine, since each AssetManager automatically loads the 'android' package
        // already.
        if (resDir != null) {'
            //添加资源路径
            if (assets.addAssetPath(resDir) == 0) {
                return null;
            }
        }

        if (splitResDirs != null) {
            for (String splitResDir : splitResDirs) {
                if (assets.addAssetPath(splitResDir) == 0) {
                    return null;
                }
            }
        }

        if (overlayDirs != null) {
            for (String idmapPath : overlayDirs) {
                assets.addOverlayPath(idmapPath);
            }
        }

        if (libDirs != null) {
            for (String libDir : libDirs) {
                if (assets.addAssetPath(libDir) == 0) {
                    Slog.w(TAG, "Asset path '" + libDir +
                            "' does not exist or contains no resources.");
                }
            }
        }

        //Slog.i(TAG, "Resource: key=" + key + ", display metrics=" + metrics);
        DisplayMetrics dm = getDisplayMetricsLocked(displayId);
        Configuration config;
        boolean isDefaultDisplay = (displayId == Display.DEFAULT_DISPLAY);
        final boolean hasOverrideConfig = key.hasOverrideConfiguration();
        if (!isDefaultDisplay || hasOverrideConfig) {
            config = new Configuration(getConfiguration());
            if (!isDefaultDisplay) {
                applyNonDefaultDisplayMetricsToConfigurationLocked(dm, config);
            }
            if (hasOverrideConfig) {
                config.updateFrom(key.mOverrideConfiguration);
            }
        } else {
            config = getConfiguration();
        }'
        //创建一个Resource对象
        r = new Resources(assets, dm, config, compatInfo, token);
        if (false) {
            Slog.i(TAG, "Created app resources " + resDir + " " + r + ": "
                    + r.getConfiguration() + " appScale="
                    + r.getCompatibilityInfo().applicationScale);
        }

        synchronized (this) {
            WeakReference<Resources> wr = mActiveResources.get(key);
            Resources existing = wr != null ? wr.get() : null;
            if (existing != null && existing.getAssets().isUpToDate()) {
                // Someone else already created the resources while we were
                // unlocked; go ahead and use theirs.
                r.getAssets().close();
                return existing;
            }

            // XXX need to remove entries when weak references go away
            mActiveResources.put(key, new WeakReference<Resources>(r));
            return r;
        }
    } 
```

从上述源码可以得到如下创建Resource的步骤
- 创建一个AssetManager对象
- 通过addAssetPath方法将资源路径添加给AssetManager，需要注意的是，这个方法是hide的，需要反射调用。
- 通过AssetManager对象创建一个Resource对象，构造方法中，需要一个AssetManager，其他的可以使用宿主的。

这样就好办了。

我们在activity刚创建好时，将我们的Resource通过反射添加到ContextThemeWrapper中。

这个时机该是什么时候呢？

那就是Instrumentation的newActivity方法中。

看一下代码实现：

```java

    public PluginApp loadPluginApk(String apkPath) {
        String addAssetPathMethod = "addAssetPath";
        PluginApp pluginApp = null;
        try {
            AssetManager assetManager = AssetManager.class.newInstance();
            Method addAssetPath = assetManager.getClass().getMethod(addAssetPathMethod,
                    String.class);
            addAssetPath.invoke(assetManager, apkPath);
            Resources pluginRes = new Resources(assetManager,
                    mContext.getResources().getDisplayMetrics(),
                    mContext.getResources().getConfiguration());
            pluginApp = new PluginApp(pluginRes);
            pluginApp.mClassLoader = createDexClassLoader(apkPath);
        } catch (IllegalAccessException
                | InstantiationException
                | NoSuchMethodException
                | InvocationTargetException e) {
            e.printStackTrace();
        }
        return pluginApp;
    }


    @Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {

        if (mPluginManager.hookToPluginActivity(intent)) {
            String targetClassName = intent.getComponent().getClassName();
            PluginApp pluginApp = mPluginManager.getLoadedPluginApk();
            Activity activity = mBase.newActivity(pluginApp.mClassLoader, targetClassName, intent);
            ReflectUtil.setField(ContextThemeWrapper.class, activity, Constants.FIELD_RESOURCES, pluginApp.mResources);
            return activity;
        }

        return super.newActivity(cl, className, intent);
    }
```

先创建好我们的Resource，然后在newActivity方法内通过反射赋值。