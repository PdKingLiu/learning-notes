[TOC]

# ARouter
## 概述

在每个需要对其他module提供调用的Activity中，都要声明类似下面@Route注解，我们称之为路由地址。

```java
@Route(path = "/main/main")
public class MainActivity extends AppCompatActivity {}
@Route(path = "/module1/module1main")
public class Module1MainActivity extends AppCompatActivity {}
```

现在我们有三个module

- APP——主工程
- QRCode——扫码Module
- Gallery——项目模块

对于其他module需要使用ARouter跳转至QRCode Activity时

我们需要声明一个QRCode Activity的路由地址

	public static final String QRCODE_SCAN_PATH = "/qrcode/scan";

然后在对应的QRCode的Activity上添加注解

	@Route(path = RouteHub.QRCode.QRCODE_SCAN_PATH)

然后调用以下方法就可以完成跳转了

	ARouter.getInstance().buildRouteHub.QRCode.QRCODE_SCAN_PATH).navigation(this)

这样，两个模块不用相互有任何直接的依赖，就可以进行转跳，模块与模块之间就相互独立了。

## 原理
### APT
ARouter的使用非常方便，得益于APT。

APT的作用是在编译阶段扫描并处理代码中的注解，然后根据注解输出Java文件。

ARouter为了方便实现注解处理器还额外用了两个库。

一个是JavaPoet，他提供了调用对象方法的方式生成需要的代码，而不再需要人为的用StringBuilder去拼接代码，再使用IO写入文件。

第二个是Auto-Service，他提供了简便的方式去注册APT，避免了原本繁琐的注册步骤。

### @Route
Route的定义：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Route{
    /**
     * Path of route
     */
    String path();
    ……
}
```
- @Target({ElementType.TYPE})——表示这个annotation是修饰类的
- @Retention(RetentionPolicy.CLASS)——表示需要保留到编译时

Route中有一个主要的参数path，他表示了Activity的路由地址。

	@Route(path = RouteHub.QRCode.QRCODE_SCAN_PATH)

这样编译时能获取到Route所注解的类，并且能获取到path路径。

### RouteProcessor
RouteProcessor是对@Route处理的地方。

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})
public class RouteProcessor extends BaseProcessor
```
**解释**

- auto-service——这个库为Processor完成了自动注册
- @SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})——表明了当前Processor是出里那些注释的

RouteProcessor继承于BaseProcessor，在init方法中获取到了每个模块的moduleName。

```java

// Attempt to get user configuration [moduleName]
Map<String, String> options = processingEnv.getOptions();
if (MapUtils.isNotEmpty(options)) {
  moduleName = options.get(KEY_MODULE_NAME);
  …………
}
```
RouteProcessor的process方法是对注解处理的地方，它直接获取了所有Route注解的元素。

	Set<? extends Element> routeElements = roundEnv.getElementsAnnotatedWith(Route.class);


拿到被标注的元素后就会进入this.parseRoutes(routeElements);方法。这个方法使用**JavaPoet**生成Java文件。如果不用这个库我们也可以使用StringBuilder去写Java文件的内容。

我们先来看看他生成的类。

### IRouteGroup 

```java

public class ARouter$$Group$$qrcode implements IRouteGroup {
  @Override
  public void loadInto(Map<String,RouteMeta> atlas) {
    atlas.put("/qrcode/scan",
      RouteMeta.build(
        RouteType.ACTIVITY, 
        ScanQRActivity.class,
        "/qrcode/scan", 
        "qrcode", 
        new java.util.HashMap<String,Integer>(){{put("tag", 8); }}, -1, -2147483648));
  }
}
```
**关于RouteMeta：** RouteMeta是包含了@Route所注解的元素的必要信息，最明显的就是ScanQRActivity.class，有了它，我们就可以通过Intent跳转到这个Activity了。

`ARouter$$Group$$qrcode `这个类继承自IRouteGroup ，他实现了接口中的loadInto方法。

loadInto方法逻辑很简单，传入一个map，将注解的path值作为key，将元素（RouteMeta）作为value作为Value放入map。

如果完成了这个方法，就完成了Activity的注册。

### IRouteRoot 
RouteProcessor还产生了`ARouter$$Root$$qrcode`这个类

```java
public class ARouter$$Root$$qrcode implements IRouteRoot {
  @Override
  public void loadInto(Map<String,Class<? extends IRouteGroup>> routes) {
    routes.put("qrcode",ARouter$$Group$$qrcode.class);
  }
}
```
它实现了IRouteRoot接口，内容非常相似。通过loadInto方法，往Map中插入以group名为Key，IRouteGroup实现类为Value的内容。

group默认就是path中第一个斜杠之后的内容（@Route(path="/group/xxx")）

如果调用了这个方法，那么可以通过group拿到IRouteGroup实现类的class，有了class实例化之后就能通过前面所说的拿到Activity.class了。

整体的结构是下图的样子

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191111160158176.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

我们看到，其实直接可以从IRouteGroup 传入的Map获得我们的RouteMeta，那么为什么还需要IRouteRoot的loadInto再把IRouteGroup放到Map里呢。这个大家后面就会知道。

### process

回过头来再看RouteProcessor的process方法：

```java
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironmentroundEnv) {
  if(CollectionUtils.isNotEmpty(annotations)) {
    //获取所有Route注解元素
    Set<? extends Element>routeElements = roundEnv.getElementsAnnotatedWith(Route.class);
    try {
      this.parseRoutes(routeElements);
    } catch (Exception e) {
      logger.error(e);
    }
    return true;
  }
  return false;
}
```

process方法将所有的Route注解元素放进了parseRoutes方法用于生成IRouteGroup和IRouteRoot。

这里面使用JavaPoet提供的类，通过方法调用的形式生成代码。

### 生成代码

使用了JavaPoet提供的类，通过方法调用的形式生成代码，这里看的不是太懂，简单过一遍。

构建IRootGroup的loadInto方法

```java
/*
Build method : 'loadInto'
*/
MethodSpec.BuilderloadIntoMethodOfRootBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
  .addAnnotation(Override.class)
  .addModifiers(PUBLIC)
  .addParameter(rootParamSpec);
```
方法体内容生成

```java

for (Element element : routeElements) {
  …………
  if (types.isSubtype(tm, type_Activity)) {                 
    // Activity
    logger.info(">>> Found activity route: " + tm.toString() + " <<<");
    // Get all fields annotation by @Autowired
    Map<String, Integer> paramsType = new HashMap<>();
    Map<String, Autowired> injectConfig = new HashMap<>();
    for (Element field : element.getEnclosedElements()) {
      if (field.getKind().isField() && field.getAnnotation(Autowired.class) != null && !types.isSubtype(field.asType(), iProvider)) {
        // It must be field, then it has annotation, but it not be provider.
        Autowired paramConfig = field.getAnnotation(Autowired.class);
        String injectName = StringUtils.isEmpty(paramConfig.name()) ? field.getSimpleName().toString() : paramConfig.name();
        paramsType.put(injectName, typeUtils.typeExchange(field));
        injectConfig.put(injectName, paramConfig);
      }
    }
    routeMeta = new RouteMeta(route, element, RouteType.ACTIVITY, paramsType);
    routeMeta.setInjectConfig(injectConfig);
  }
  …………
  categories(routeMeta);
}
```
这里完成了RouteMeta类的构建，还获取了Activity通过@AutoWired注解的接收参数，但是不影响我们理解跳转。然后通过categories(routeMeta)方法，对所有RouteMeta进行分组。

```java
private void categories(RouteMeta routeMete) {
    if (routeVerify(routeMete)) {
      logger.info(">>> Start categories, group = " + routeMete.getGroup() + ", path = " + routeMete.getPath() + " <<<");
      Set<RouteMeta> routeMetas = groupMap.get(routeMete.getGroup());
        if (CollectionUtils.isEmpty(routeMetas)) {
          Set<RouteMeta> routeMetaSet = new TreeSet<>(new Comparator<RouteMeta>() {
            @Override
            public int compare(RouteMeta r1, RouteMeta r2) {
              try {
                return r1.getPath().compareTo(r2.getPath());
              } catch (NullPointerException npe) {
                logger.error(npe.getMessage());
                return 0;
              }
            }
          });
          routeMetaSet.add(routeMete);
          groupMap.put(routeMete.getGroup(), routeMetaSet);
        } else {
          routeMetas.add(routeMete);
        }
    } else {
        logger.warning(">>> Route meta verify error, group is " + routeMete.getGroup() + " <<<");
    }
}
```
所有的RouteMeta会根据其group作为Key放入一个Map中。RouteMeta本身会放入一个Set，Set中所有RouteMeta的group都是相同的，作为Map的Value。

我们依据将所有的Route根据group进行了分类，并存在的Map中，结构如下。


![在这里插入图片描述](https://img-blog.csdnimg.cn/20191111162616734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
有了这个Map，再进行IRouteGroup和IRouteRoot的方法体实现就方便了。对分组的Map进行遍历，根据不同分组，再对Set中的RouteMeta进行遍历，依次添加loadInto代码实现。

```java
//遍历Map
for (Map.Entry<String, Set<RouteMeta>> entry : groupMap.entrySet()) {
    String groupName = entry.getKey();
    MethodSpec.Builder loadIntoMethodOfGroupBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
            .addAnnotation(Override.class)
            .addModifiers(PUBLIC)
            .addParameter(groupParamSpec);
    …………
    // Build group method body
    Set<RouteMeta> groupData = entry.getValue();
    //遍历Set
    for (RouteMeta routeMeta : groupData) {
        …………
        ClassName className = ClassName.get((TypeElement) routeMeta.getRawType());
        …………
        // Make map body for paramsType
        StringBuilder mapBodyBuilder = new StringBuilder();
        Map<String, Integer> paramsType = routeMeta.getParamsType();
        Map<String, Autowired> injectConfigs = routeMeta.getInjectConfig();
        if (MapUtils.isNotEmpty(paramsType)) {
            …………
            for (Map.Entry<String, Integer> types : paramsType.entrySet()) {
                mapBodyBuilder.append("put(\"").append(types.getKey()).append("\", ").append(types.getValue()).append("); ");
                …………
            }
            …………
        }
        String mapBody = mapBodyBuilder.toString();
        //生成IRouteGroup 的loadInto方法体实现
        loadIntoMethodOfGroupBuilder.addStatement(
                "atlas.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, " + (StringUtils.isEmpty(mapBody) ? null : ("new java.util.HashMap<String, Integer>(){{" + mapBodyBuilder.toString() + "}}")) + ", " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                routeMeta.getPath(),
                routeMetaCn,
                routeTypeCn,
                className,
                routeMeta.getPath().toLowerCase(),
                routeMeta.getGroup().toLowerCase());
                …………
    }
    // Generate groups
    //生成ARouter$$Group$$qrcode implements IRouteGroup类和Java文件
    //NAME_OF_GROUP定义在Consts，为"Arouter$$Group$$"
    String groupFileName = NAME_OF_GROUP + groupName;
    //groupFileName在我们的案例中为"Arouter$$Group$$qrcode"
    //PACKAGE_OF_GENERATE_FILE为包名，定义在Consts，为"com.alibaba.android.arouter.routes"
    //代码文件全部生成完成
    JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
            TypeSpec.classBuilder(groupFileName)
                    .addJavadoc(WARNING_TIPS)
                    .addSuperinterface(ClassName.get(type_IRouteGroup))
                    .addModifiers(PUBLIC)
                    .addMethod(loadIntoMethodOfGroupBuilder.build())
                    .build()
    ).build().writeTo(mFiler);

    logger.info(">>> Generated group: " + groupName + "<<<");
    //分组名和文件名存入Map，以便后续生成IRouteRoot
    rootMap.put(groupName, groupFileName);
    …………
}
```

到这里，我们已经生成了`ARouter$$Group$$qrcode`的Java文件。同样的逻辑会生成`ARouter$$Root$$qrcode`的Java文件。

```java
if (MapUtils.isNotEmpty(rootMap)) {
  // Generate root meta by group name, it must be generated before root, then I can find out the class of group.
  for (Map.Entry<String, String> entry : rootMap.entrySet()) {
        //生成loadInto具体实现
        loadIntoMethodOfRootBuilder.addStatement("routes.put($S, $T.class)", entry.getKey(), ClassName.get(PACKAGE_OF_GENERATE_FILE, entry.getValue()));
    }
}
……
// Write root meta into disk.
// NAME_OF_ROOT为"ARouter$$Root$$"
String rootFileName = NAME_OF_ROOT + SEPARATOR + moduleName;
//rootFileName在我们的案例中为ARouter$$Root$$qrcode
//代码文件全部生成完成
JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
        TypeSpec.classBuilder(rootFileName)
                .addJavadoc(WARNING_TIPS)
                .addSuperinterface(ClassName.get(elementUtils.getTypeElement(ITROUTE_ROOT)))
                .addModifiers(PUBLIC)
                .addMethod(loadIntoMethodOfRootBuilder.build())
                .build()
).build().writeTo(mFiler);    
```
IRouteRoot生成的过程中和IRouteGroup类不同的地方是文件名最后用的是moduleName，这个变量之前分析过就是在gradle中ARouter要求配置的参数，用module名字的原因是能够避免工程庞大后命名冲突的可能。

到这里就完成了两个类的生成工作。那么ARouter是怎么去用这两个类的呢？前面提出的问题，有了IRouteGroup就能拿到路由表，为什么还需要IRouteRoot再吧IRouteGroup包一层呢？

### 路由跳转
ARouter.init(this)是一个初始化的地方。一路下去，会调用到LogisticsCenter的init方法。

路由表的生成有两种方式，我们看ARouter最传统的方式。如果是设置了debug模式或者App版本与记录的不同的情况下，扫描所有Dex文件查找所有com.alibaba.android.arouter.routes开头的文件，然后更新到SharePreferences。否则直接从SharePreferences读缓存，减少解析时间。

```java

if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
    logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
    // These class was generated by arouter-compiler.
    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
    if (!routerMap.isEmpty()) {
        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
    }
    PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
} else {
    logger.info(TAG, "Load router map from cache.");
    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
}
```
解析完成后，进行类名遍历，将所有com.alibaba.android.arouter.routes.Arouter$$Root开头的类反射出来，实例化，并调用loadInto方法加到Warehouse.groupsIndex这个Map中，也就是拿到了所有group和对应的IRouteGroup。

```java

if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
  //com.alibaba.android.arouter.routes.Arouter$$Root
  // This one of root elements, load root.
  ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
}
```
Warehouse.groupsIndex结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191111163441953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

这些IRouteGroup所对应的类什么时候解析呢？我们来看会发生跳转的navigation方法，最终会走到navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback)，里边会立马调用LogisticsCenter.completion(postcard)方法。

方法里会先尝试根据path去取解析过的RouteMeta，但是很明显，我们这边还没有解析过。

	RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());

routeMeta 为null，于是从Warehouse.groupsIndex去取对应的IRouteGroup，拿到他就可以通过他的loadInto去加载对应的Activity信息了。

```java
Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
iGroupInstance.loadInto(Warehouse.routes);
Warehouse.groupsIndex.remove(postcard.getGroup());
```

可以看到，会把我们请求跳转的path所在group的内容全部加载到Warehouse.routes下，然后从Warehouse.groupsIndex移除group对应的IRouteRoot，因为他已经没有用了，该分组下的所有内容都被加载上来了。于是移除，省出内存。

可以看出，同一分组下的所有RouteMeta的注册是在其中一个第一次被请求跳转的时候完成的。Warehouse.routes的结构如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191111163657864.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**有了IRouteGroup就能拿到路由表，为什么还需要IRouteRoot再吧IRouteGroup包一层呢？**

启动应用，然后立马遍历所有com.alibaba.android.arouter.routes开头的文件，分析出是Root的文件，反射实例化，调用loadInto方法，将IRouteGroup.class载入。这几步里好多是耗时的方法，如果工程很庞大，像淘宝这样，如果一上来直接加载所有的IRouteGroup中的内容，恐怕要非常长的时间，会导致启动慢，或者ANR。

这也是为什么ARouter需要IRouteRoot存在的原因，他可以延时加载所需要的分组，第一次使用的时候加载可以分摊加载时间。

**路由表加载有两种方式，我们分析的是最传统的一种。那另一种呢**

就是使用com.alibaba:arouter-register了，它底下使用的是arouter-auto-register这个库，它能够在编译阶段扫描IRouteRoot的实现类，注册到LogisticsCenter。用这种方式能够避免应用开启后再扫描生成路由表的开销。而且能够避免在某些加壳工具加固后无法获取到dex文件中的com.alibaba.android.arouter.routes包下的内容，导致无法生成注册表的问题。

继续看最后的跳转工作。

这里拿到完整信息的Postcard会创建Intent，进行跳转

	final Intent intent = new Intent(currentContext, postcard.getDestination());

```java
// Navigation in main looper.
runInMainThread(new Runnable() {
  @Override
  public void run() {
    startActivity(requestCode, currentContext, intent, postcard, callback);
  }
});
```
跳转会被安排到主线程。

> 参考：https://mp.weixin.qq.com/s?src=11&timestamp=1573461757&ver=1967&signature=t0yrs2jjQBJf9V-xMEdHkuT85qPnsuoaR*JDjxF5Csj4PYh00rVEv3wx33ECWQn-2N5tZBSY02P*pgwZbuFy6wV-p9M3yIvT9QeFnzS7hW0OHQ6iSmblE5CVekRDhxHj&new=1