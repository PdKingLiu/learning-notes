[toc]

# 单向依赖的module

组件化中两个单向依赖的module之间需要互相启动对方的Activity时，因为没有相互引用，startActivity()是实现不了的，必须需要一个协定的通信方式，此时类似ARouter的路由框架就派上用场了。

# 互不依赖的module

两个module之间没有依赖，便不能通过startActivity()的显示启动进行通信，那么如何进行通信呢？

- 隐式跳转：这是一种解决方法，但是一个项目中不可能所有的跳转都是隐式的，这样Manifest文件会有很多过滤配置，而且非常不利于后期维护。
- 反射：当然你用反射拿到Activity的class文件也可以实现跳转，但是大量的使用反射跳转对性能会有影响。
- 路由框架。

# ARouter
## 概述

**在每个需要对其他module提供调用的Activity中，都会声明类似下面@Route注解，我们称之为路由地址。**

```java
@Route(path = "/main/main")
public class MainActivity extends AppCompatActivity {}
@Route(path = "/module1/module1main")
public class Module1MainActivity extends AppCompatActivity {}
```

**编译期时生成路由映射**

路由框架会在项目的编译期通过注解处理器apt扫描所有添加@Route注解的Activity类，然后将Route注解中的path地址和Activity.class文件映射关系保存到它自己生成的java文件中，只要拿到了映射关系便能拿到Activity.class。

```java
route.put("/main/main", MainActivity.class);
route.put("/module1/module1main", Module1MainActivity.class);
route.put("/login/login", LoginActivity.class);
```

```java
public void login(String name, String password) {
    LoginActivity.class classBean = route.get("/login/login");
    Intent intent = new Intent(this, classBean);
    intent.putExtra("name", name);
    intent.putExtra("password", password);
    startActivity(intent);
}
```
ARouter背后的实现原理其实跟上面讲解是一样的。

## apt & javapoet
apt是在编译期对代码中指定的注解进行解析，然后做一些其他处理（如通过javapoet生成新的Java文件）。我们常用的ButterKnife，其原理就是通过注解处理器在编译期扫描代码中加入的@BindView、@OnClick等注解进行扫描处理，然后生成XXX_ViewBinding类，实现了view的绑定。javapoet是用来生成java文件的一个library，它提供了简便的api供你去生成一个java文件。

## 路由映射文件生成原理
### 定义注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Route {
    //Path of route
    String path();

    //Used to merger routes
    String group() default "";
}
```

ARouter中会要求路由地址至少需要两级，如"/xx/xx"，其中第一级是group。因为当项目变得越来越庞大的时候，为了便于管理和减小首次加载路由表过于耗时的问题，我们对所有的路由进行分组。

### 使用注解

```java
@Route(path = "/main/main")
public class MainActivity extends AppCompatActivity {}
@Route(path = "/main/main2")
public class Main2Activity extends AppCompatActivity {}
```
### 注解处理器

```java
@AutoService(Processor.class)
@SupportedOptions({"AROUTER_MODULE_NAME", "AROUTER_GENERATE_DOC" })
@SupportedSourceVersion(SourceVersion.RELEASE_7)
@SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})
public class RouteProcessor extends AbstractProcessor {
// ModuleName and routeMeta.
//分组 key:组名 value:对应组的路由信息
    private Map<String, Set<RouteMeta>> groupMap = new HashMap<>();
    // Map of root metas, used for generate class file in order.
    private Map<String, String> rootMap = new TreeMap<>();  

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        //
        // Attempt to get user configuration [moduleName]
        Map<String, String> options = processingEnv.getOptions();
        if (MapUtils.isNotEmpty(options)) {
            moduleName = options.get("AROUTER_MODULE_NAME");  
        }
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //
    }
}
```

说明如下

- @AutoService(Processor.class)——AutoService会自动在META-INF文件夹下生成Processor配置信息文件，该文件里就是实现Processor接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。
- @SupportedOptions({KEY_MODULE_NAME, KEY_GENERATE_DOC_NAME})——SupportedOptions指定支持的选项（即处理器接收的参数），AbstractProcessor的getSupportedOptions()会得到它支持的选项。
- @SupportedSourceVersion(SourceVersion.RELEASE_7)——指定使用的Java版本。
- @SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})——指定注解处理器是注册给哪一个注解的，它是一个字符串的集合，意味着可以支持多个类型的注解，并且字符串是合法全名。
- process()方法里的set集合就是编译期扫描代码得到的加入了Route注解的文件集合。
- moduleName = options.get("AROUTER_MODULE_NAME");——以上代码会得到一个moduleName，这个moduleName需要我们在要使用注解处理器生成路由映射文件的模块gradle文件里配置，而@SupportedOptions("AROUTER_MODULE_NAME")会拿到当前module的名字，用来生成唯一对应module下存放分组信息的类文件名。我们需要在每个组件module的gradle下配置如下：
    ```java
    android {
        defaultConfig {
            javaCompileOptions {
                annotationProcessorOptions {
                    arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
                }
            }
        }
    }
    ```
### 使用javapoet生成java类
**定义变量**

```java
    /**
     * key:组名   value:该组名对应的group类名
     */
    private Map<String, String> rootMap = new TreeMap<>();

    /**
     * key:组名   value:该组下的路由信息
     */
    private Map<String, List<RouteMeta>> groupMap = new HashMap<>();
```
**定义接口IRouteGroup**

```java
public interface IRouteGroup {
    void loadInto(Map<String, RouteMeta> atlas);
}
```
**定义接口IRouteRoot**

```java
public interface IRouteRoot {
    void loadInto(Map<String, Class<? extends IRouteGroup>> routes);
}
```
**生成group类**

```java
private void generatedGroup(TypeElement iRouteGroup) {
        //创建参数类型 Map<String, RouteMeta>
        ParameterizedTypeName parameterizedTypeName = ParameterizedTypeName.get(
                ClassName.get(Map.class),
                ClassName.get(String.class),
                ClassName.get(RouteMeta.class));
        ParameterSpec altas = ParameterSpec.builder(parameterizedTypeName, "atlas").build();

        for (Map.Entry<String, List<RouteMeta>> entry : groupMap.entrySet()) {
            MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder(Constant.METHOD_LOAD_INTO)
                    .addModifiers(Modifier.PUBLIC)
                    .addAnnotation(Override.class)
                    .addParameter(altas);

            String groupName = entry.getKey();
            List<RouteMeta> groupData = entry.getValue();
            for (RouteMeta routeMeta : groupData) {
                //函数体的添加
                methodBuilder.addStatement("atlas.put($S,$T.build($T.$L,$T.class,$S,$S))",
                        routeMeta.getPath(),
                        ClassName.get(RouteMeta.class),
                        ClassName.get(RouteMeta.Type.class),
                        routeMeta.getType(),
                        ClassName.get(((TypeElement) routeMeta.getElement())),
                        routeMeta.getPath(),
                        routeMeta.getGroup());
            }
            String groupClassName = Constant.NAME_OF_GROUP + groupName;
            TypeSpec typeSpec = TypeSpec.classBuilder(groupClassName)
                    .addSuperinterface(ClassName.get(iRouteGroup))
                    .addModifiers(Modifier.PUBLIC)
                    .addMethod(methodBuilder.build())
                    .build();
            JavaFile javaFile = JavaFile.builder(Constant.PACKAGE_OF_GENERATE_FILE, typeSpec).build();
            try {
                javaFile.writeTo(filerUtils);
            } catch (IOException e) {
                e.printStackTrace();
            }
            rootMap.put(groupName, groupClassName);

        }
    }
```
概括：
- 根据groupMap分组信息，每一组生成一个group类，每一个group类里包含该组所有<path, routemeta>信息。
- rootMap.put(groupName, groupClassName); //存放组名和该组的group类名的映射关系。

**生成Root类**

```java
private void generatedRoot(TypeElement iRouteRoot, TypeElement iRouteGroup) {
        //创建参数类型 Map<String,Class<? extends IRouteGroup>> routes>
        //Wildcard 通配符
        ParameterizedTypeName parameterizedTypeName = ParameterizedTypeName.get(
                ClassName.get(Map.class),
                ClassName.get(String.class),
                ParameterizedTypeName.get(
                        ClassName.get(Class.class),
                        WildcardTypeName.subtypeOf(ClassName.get(iRouteGroup))
                ));
        //参数 Map<String,Class<? extends IRouteGroup>> routes> routes
        ParameterSpec parameter = ParameterSpec.builder(parameterizedTypeName, "routes").build();
        //函数 public void loadInfo(Map<String,Class<? extends IRouteGroup>> routes> routes)
        MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder(Constant.METHOD_LOAD_INTO)
                .addModifiers(Modifier.PUBLIC)
                .addAnnotation(Override.class)
                .addParameter(parameter);
        //函数体
        for (Map.Entry<String, String> entry : rootMap.entrySet()) {
            methodBuilder.addStatement("routes.put($S, $T.class)", entry.getKey(), ClassName.get(Constant.PACKAGE_OF_GENERATE_FILE, entry.getValue()));
        }
        //生成$Root$类
        String className = Constant.NAME_OF_ROOT + moduleName;
        TypeSpec typeSpec = TypeSpec.classBuilder(className)
                .addSuperinterface(ClassName.get(iRouteRoot))
                .addModifiers(Modifier.PUBLIC)
                .addMethod(methodBuilder.build())
                .build();
        try {
            JavaFile.builder(Constant.PACKAGE_OF_GENERATE_FILE, typeSpec).build().writeTo(filerUtils);
            log.i("Generated RouteRoot：" + Constant.PACKAGE_OF_GENERATE_FILE + "." + className);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

概括：
- 根据rootMap信息，生成一个root类，该root类以moduleName命名。
- root类里包含<groupName, groupClass>信息（即组名与组对应的group类的映射关系）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026225107810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 路由跳转
ARouter.init(this)是一个初始化的地方。一路下去，会调用到LogisticsCenter的init方法。路由表的生成有两种方式，我们看ARouter最传统的方式。如果是设置了debug模式或者App版本与记录的不同的情况下，扫描所有Dex文件查找所有com.alibaba.android.arouter.routes开头的文件，然后更新到SharePreferences。否则直接从SharePreferences读缓存，减少解析时间。

解析完成后，进行类名遍历，将所有com.alibaba.android.arouter.routes.Arouter$$Root开头的类反射出来，实例化，并调用loadInto方法加到Warehouse.groupsIndex这个Map中，也就是拿到了所有group和对应的IRouteGroup。

Warehouse.groupsIndex结构如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026225442409.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

这些IRouteGroup所对应的类什么时候解析呢？我们来看会发生跳转的navigation方法，最终会走到navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback)，里边会立马调用LogisticsCenter.completion(postcard)方法。方法里会先尝试根据path去取解析过的RouteMeta，但是很明显，我们这边还没有解析过。

    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());

routeMeta 为null，于是从Warehouse.groupsIndex去取对应的IRouteGroup，拿到他就可以通过他的loadInto去加载对应的Activity信息了。

```java
Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
iGroupInstance.loadInto(Warehouse.routes);
Warehouse.groupsIndex.remove(postcard.getGroup());
```

可以看出，同一分组下的所有RouteMeta的注册是在其中一个第一次被请求跳转的时候完成的。Warehouse.routes的结构如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026225703470.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
有了IRouteGroup就能拿到路由表，为什么还需要IRouteRoot再吧IRouteGroup包一层呢？

就解释了这个问题，我们就当App是首次运行的最坏情况。启动应用，然后立马遍历所有com.alibaba.android.arouter.routes开头的文件，分析出是Root的文件，反射实例化，调用loadInto方法，将IRouteGroup.class载入。这几步里好多是耗时的方法，如果工程很庞大，像淘宝这样，如果一上来直接加载所有的IRouteGroup中的内容，恐怕要非常长的时间，会导致启动慢，或者ANR。这也是为什么ARouter需要IRouteRoot存在的原因，他可以延时加载所需要的分组，第一次使用的时候加载可以分摊加载时间，很细腻。

最后拿到完整信息的Postcard会创建Intent，进行跳转。

    final Intent intent = new Intent(currentContext, postcard.getDestination());

跳转会被安排到主线程

```java
// Navigation in main looper.
runInMainThread(new Runnable() {
  @Override
  public void run() {
    startActivity(requestCode, currentContext, intent, postcard, callback);
  }
});
```
以上就是ARouter路由大致的实现逻辑。好烦，有些地方还没彻底弄明白。。。