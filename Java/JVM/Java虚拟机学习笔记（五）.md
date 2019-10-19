[toc]

# 类加载器
JVM 中内置了三个重要的 ClassLoader，除了 **BootstrapClassLoader** 其他类加载器均由 Java 实现且全部继承自java.lang.ClassLoader：

## 类加载器总结

- **BootstrapClassLoader(启动类加载器)** ：最顶层的加载类，由C++实现，负责加载 %JAVA_HOME%/lib目录下的jar包和类或者或被 -Xbootclasspath参数指定的路径中的所有类。
- **ExtensionClassLoader(扩展类加载器)** ：主要负责加载目录 %JRE_HOME%/lib/ext 目录下的jar包和类，或被 java.ext.dirs 系统变量所指定的路径下的jar包。
- **AppClassLoader(应用程序类加载器)** :面向我们用户的加载器，负责加载当前应用classpath下的所有jar包和类。

## 双亲委派模型
每一个类都有一个对应它的类加载器。系统中的 ClassLoder 在协同工作的时候会默认使用 **双亲委派模型** 。即在类加载的时候，系统会首先**判断当前类是否被加载过。已经被加载的类会直接返回**，否则才会尝试加载。加载的时候，首先会把该请求**委派该父类加载器的 loadClass() 处理**，因此所有的**请求最终都应该传送到顶层的启动类加载器 BootstrapClassLoader 中**。当父类加载器无法处理时，才由自己来处理。**当父类加载器为null时，会使用启动类加载器 BootstrapClassLoader 作为父类加载器**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018100516257.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
每个类加载都有一个父类加载器 。

```csharp
public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println("ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader());
        System.out.println("The Parent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent());
        System.out.println("The GrandParent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent().getParent());
    }
}
```

```csharp
ClassLodarDemo's ClassLoader is sun.misc.Launcher$AppClassLoader@18b4aac2
The Parent of ClassLodarDemo's ClassLoader is sun.misc.Launcher$ExtClassLoader@1b6d3586
The GrandParent of ClassLodarDemo's ClassLoader is null
```
**AppClassLoader的父类加载器为ExtClassLoader ExtClassLoader的父类加载器为null，null并不代表ExtClassLoader没有父类加载器，而是 BootstrapClassLoader 。**

## 双亲委派模型实现源码分析
双亲委派模型的实现代码非常简单，逻辑非常清晰，都集中在 java.lang.ClassLoader 的 loadClass() 中，相关代码如下所示。

```csharp
private final ClassLoader parent; 
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException{
    	synchronized (getClassLoadingLock(name)) {
        // 首先，检查请求的类是否已经被加载过
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                    c = parent.loadClass(name, false);
                } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
               //抛出异常说明父类加载器无法完成加载请求
            }
            
            if (c == null) {
                long t1 = System.nanoTime();
                //自己尝试加载
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
## 双亲委派模型的好处
双亲委派模型保证了Java程序的**稳定运行**，可以**避免类的重复加载**（**JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类**），也**保证了 Java 的核心 API 不被篡改**。**如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 java.lang.Object 类的话，那么程序运行的时候，系统就会出现多个不同的 Object 类**。

## 自定义类加载器
除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自java.lang.ClassLoader。如果我们要自定义自己的类加载器，需要**继承 ClassLoader**。

```java
public class ClassLoadDemo{

    public static void main(String[] args) throws Exception {

        ClassLoader clazzLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String clazzName = name.substring(name.lastIndexOf(".") + 1) + ".class";

                    InputStream is = getClass().getResourceAsStream(clazzName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };

        String currentClass = "com.classloader.ClassLoadDemo";
        Class<?> clazz = clazzLoader.loadClass(currentClass);
        Object obj = clazz.newInstance();

        System.out.println(obj.getClass());
        System.out.println(obj instanceof com.classloader.ClassLoadDemo);
		//class com.classloader.ClassLoadDemo
		//false 即使都是来自同一个Class文件，加载器不同，仍然是两个不同的类，所以返回值是false
    }
}
```
# 补充：Android中的ClassLoader
Android 的 Dalvik/ART 虚拟机如同标准 Java 的 JVM 虚拟机一样，也是同样需要加载 class 文件到内存中来使用，但是**在 ClassLoader 的加载细节上会有略微的差别**。

## Android中的dex
Android 应用打包成 apk 文件时，class 文件会被打包成**一个或者多个 dex 文件**。将一个 apk 文件后缀改成 .zip 格式解压后（也可以直接解压，apk 文件本质是个 zip 文件），里面就有 class.dex 文件，由于 Android 的 64K 问题，**使用 MultiDex 就会生成多个 dex 文件**。

Android 中的 Dalvik/ART **无法像 JVM 那样 直接 加载 class 文件和 jar 文件中的 class**，需要通过 dx 工具来优化转换成 Dalvik byte code 才行，**只能通过 dex 或者 包含 dex 的jar、apk 文件来加载** ，因此 **Android 中的 ClassLoader 工作就交给了 BaseDexClassLoader 来处理**。

## BaseDexClassLoader 及其子类
在 Android 开发者官网上的 ClassLoader 的文档说明中我们可以看到，**ClassLoader 是个抽象类，其具体实现的子类有 BaseDexClassLoader 和 SecureClassLoader** 。

SecureClassLoader 的子类是 URLClassLoader ，其只能用来加载 jar 文件，这在 Android 的 Dalvik/ART 上没法使用的。

**BaseDexClassLoader** 的子类是 **PathClassLoader 和 DexClassLoader** 。

### PathClassLoader
PathClassLoader 在应用启动时创建，**从 data/app/… 安装目录下加载 apk 文件**。

```java
public PathClassLoader(String dexPath, ClassLoader parent) {
    super(dexPath, null, null, parent);
}

public PathClassLoader(String dexPath, String libraryPath,
        ClassLoader parent) {
    super(dexPath, null, libraryPath, parent);
}
```
他有两个构造方法。

- dexPath : **包含 dex 的 jar 文件或 apk 文件的路径集**，多个以文件分隔符分隔，默认是“：”
- libraryPath : **包含 C/C++ 库的路径集**，多个同样以文件分隔符分隔，可以为空

PathClassLoader 里面除了这 2 个构造方法以外就没有其他的代码了，具体的实现都是在 BaseDexClassLoader 里面，其 **dexPath 比较受限制，一般是已经安装应用的 apk 文件路径**。

在 Android 中，App 安装到手机后，**apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的**。

### DexClassLoader
> A class loader that loads classes from .jar and .apk filescontaining a classes.dex entry. This can be used to execute code notinstalled as part of an application.

对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，**DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件**，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的 dex 的加载。

**构造方法**

```java
public DexClassLoader(String dexPath, String optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(dexPath, new File(optimizedDirectory), libraryPath, parent);
}
```
- **String dexPath** : 包含 class.dex 的 apk、jar 文件路径 ，多个用文件分隔符(默认是 ：)分隔

- **String optimizedDirectory** : 用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径（实际上也可以使用外部存储空间，但是这样的话就存在代码注入的风险）。

- **String libraryPath** : 存储 C/C++ 库文件的路径集

- **ClassLoader parent** : 父类加载器，遵从双亲委托模型

## BaseDexClassLoader 

PathClassLoader 和 DexClassLoader，这两者都是对 BaseDexClassLoader 的一层简单封装，真正的实现都在 BaseDexClassLoader 内。

BaseDexClassLoader 有个很重要的字段` private final DexPathList pathList`他继承ClassLoader，findClass()、findResource()均是基于pathList实现的。

```java
 @Override
 protected Class<?> findClass(String name) throws ClassNotFoundException {
     List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
     Class c = pathList.findClass(name, suppressedExceptions);
     ...
     return c;
 }
 @Override
 protected URL findResource(String name) {
     return pathList.findResource(name);
 }
 @Override
 protected Enumeration<URL> findResources(String name) {
     return pathList.findResources(name);
 }
 @Override
 public String findLibrary(String name) {
     return pathList.findLibrary(name);
 }
```

**DexPathList的构造方法和前面介绍的类似，接受之前传进来的包含 dex 的 apk/jar/dex 的路径集、native 库的路径集和缓存优化的 dex 文件的路径**

```java
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    ...
}
```
 然后调用 makePathElements() 方法生成一个 Element[] dexElements 数组，Element 是 DexPathList 的一个嵌套类，其有以下字段

```java
static class Element {
	private final File dir;
	private final boolean isDirectory;
	private final File zip;
	private final DexFile dexFile;
	private ZipFile zipFile;
	private boolean initialized;
}
```
makePathElements() 生成 Element 数组的过程

```java
private static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                          List<IOException> suppressedExceptions) {
    List<Element> elements = new ArrayList<>();
    // 遍历所有的包含 dex 的文件
    for (File file : files) {
        File zip = null;
        File dir = new File("");
        DexFile dex = null;
        String path = file.getPath();
        String name = file.getName();
        // 判断是不是 zip 类型
        if (path.contains(zipSeparator)) {
            String split[] = path.split(zipSeparator, 2);
            zip = new File(split[0]);
            dir = new File(split[1]);
        } else if (file.isDirectory()) {
            // 如果是文件夹,则直接添加 Element,这个一般是用来处理 native 库和资源文件
            elements.add(new Element(file, true, null, null));
        } else if (file.isFile()) {
            // 直接是 .dex 文件,而不是 zip/jar 文件(apk 归为 zip),则直接加载 dex 文件
            if (name.endsWith(DEX_SUFFIX)) {
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else {
                // 如果是 zip/jar 文件(apk 归为 zip),则将 file 值赋给 zip 字段,再加载 dex 文件
                zip = file;
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException suppressed) {
                    suppressedExceptions.add(suppressed);
                }
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(dir, false, zip, dex));
        }
    }
    // list 转为数组
    return elements.toArray(new Element[elements.size()]);
}
```
loadDexFile() 方法最终会调用 JNI 层的方法来读取 dex 文件。

DexPathList 的 findClass()方法，传入的完整的类名来加载对应的 class：

```java
public Class findClass(String name, List<Throwable> suppressed) {
	// 遍历 dexElements 数组，依次寻找对应的 class，一旦找到就终止遍历
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    // 抛出异常
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
} 
```
这里有关于热修复实现的一个点，就是将补丁 dex 文件放到 dexElements 数组前面，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的。

BaseDexClassLader 寻找 class 的路线

- 当传入一个完整的类名，调用 `BaseDexClassLader 的 findClass(String name) `方法
- BaseDexClassLader 的 findClass 方法会交给 DexPathList 的` findClass(String name, List<Throwable> suppressed `方法处理
- 在 DexPathList 方法的内部，会遍历 dexFile ，通过 DexFile 的 `dex.loadClassBinaryName(name, definingContext, suppressed) `来完成类的加载
实际使用

需要注意的是，在项目中使用 BaseDexClassLoader 或者 DexClassLoader 去加载某个 dex 或者 apk 中的 class 时，是**无法调用 findClass() 方法的**，因为该方法是**包访问权限**，你**需要调用 loadClass(String className)** ，该方法其实是 BaseDexClassLoader 的父类 ClassLoader 内实现的：

```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
    return loadClass(className, false);
}

protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> clazz = findLoadedClass(className);
    if (clazz == null) {
        ClassNotFoundException suppressed = null;
        try {
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            suppressed = e;
        }
        if (clazz == null) {
            try {
                clazz = findClass(className);
            } catch (ClassNotFoundException e) {
                e.addSuppressed(suppressed);
                throw e;
            }
        }
    }
    return clazz;
}
```
上面这段代码结合之前提到的双亲委托模型就很好理解了，先查找当前的 ClassLoader 是否已经加载过，如果没有就交给父 ClassLoader 去加载，如果父 ClassLoader 没有找到，才调用当前 ClassLoader 来加载，此时就是调用上面分析的 findClass() 方法了。