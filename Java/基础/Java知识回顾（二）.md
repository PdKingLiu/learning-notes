[toc]

### 泛型是什么
集合存储对象并在使用先进行类型转换很不方便，泛型是为了那种情况的发生。他提供了编译器的类型安全，确保你只能把正确类型的对象放入集合中，避免了在运行时出现ClassCastException。
### 泛型如何工作
泛型是通过类型擦除来实现的，编译器在编译时擦除了所有相关类型的信息，所以在运行时不存在任何类型的相关信息。
### 泛型中的限定通配符和非限定通配符
有两种限定通配符

一种是<? extends T>它通过确保类型必须是T的子类来设定类型的上界

一种是<? super T>它通过确保类型必须是T的父类来设定类型的下界

另一方面<?>表示了非限定通配符，<?>可以用任意类型来替代。

### 编写泛型方法
泛型方法需要用泛型类型来替代原始类型，比如使用T, E or K,V等被广泛认可的类型占位符。

```java
public V put(K key, V value) {
	return cache.put(key, value);
}
```

### 异常
异常是在程序执行期间可能发生的错误事件，并且会中断它的正常流程。

### 异常关键字
- throw——throw关键字用于向运行时抛出异常来处理它。
- throws——在方法中抛出任何已检查的异常而不处理它时，我们需要在方法签名中使用throws关键字让调用者程序知道该方法可能抛出的异常。
- try-catch——在代码中使用try-catch块进行异常处理
- finally——finally块总是被执行，无论是否发生异常。

### 异常的层次结构
Throwable是异常的父类，他有Error和Exception两个子类。

**Error** 是超出应用程序范围的特殊情况，并且无法预测并从中恢复，例如硬件故障，JVM崩溃或内存不足错误。

**Exception** 是程序本身可以处理的异常。Exception 类有一个重要的子类 RuntimeException。运行时异常是由错误的编程引起的，例如尝试从Array中检索超出下标的元素。

### throw和throws
throws关键字与方法签名一起用于声明方法可能抛出的异常，而throw关键字用于破坏程序流并将异常对象移交给运行时来处理它。

### OutOfMemoryError
Java中的OutOfMemoryError是java.lang.VirtualMachineError的子类，当JVM用完堆内存时，它会抛出它。

### 注解
注解是绑定到程序源代码元素的元数据，对运行​​代码的操作没有影响。

Java注解又称为标注，是支持加入源码的特殊语法元数据；Java中的类、方法、变量、参数、包都可以被注解。这里提到的元数据是描述数据的数据。

典型用例：

- 编译器的信息 ——使用注解，编译器可以检测错误或抑制警告
- 编译时处理——软件工具可以处理注解并生成代码，配置文件等
- 运行时处理——可以在运行时检查注解以自定义程序的行为

### 注解分类
**标准注解**：包括 Override, Deprecated, SuppressWarnings

标准 Annotation 是指 Java 自带的几个 Annotation。

上面三个分别表示**重写函数**，**不鼓励使用** ，**忽略Warning**。

**元注解**

**1. @Retention**

用于指定被修饰的注解可以保留多长时间，只能修饰Annotation定义。

- RetentionPolicy.CLASS——编译器将把Annotation记录在class文件中。当运行java程序时，JVM不可获取Annotation信息。
- RetentionPolicy.RUNTIME——编译器将把Annotation记录在class文件中。当运行java程序时，JVM也可获取Annotation信息，程序可以通过反射获取该Annotation信息。
- RetentionPolicy.SOURCE: Annotation只保留在源代码中（.java文件中），编译器直接丢弃这种Annotation。

**@Target**

用于指定被修饰的Annotation能用于修饰哪些程序单元。

- @Target(ElementType.ANNOTATION_TYPE)： 指定该策略的Annotation只能修饰Annotation。
- @Target(ElementType.TYPE) ： 接口、类、枚举、注解
- @Target(ElementType.FIELD) ： 成员变量（字段、枚举的常量）
- @Target(ElementType.METHOD) ： 方法
- @Target(ElementType.PARAMETER)： 方法参数
- @Target(ElementType.CONSTRUCTOR)： 构造函数
- @Target(ElementType.LOCAL_VARIABLE)： 局部变量
- @Target(ElementType.PACKAGE)： 修饰包定义 

**3. Documented**

用于指定被修饰的Annotation将被javadoc工具提取成文档。即说明该注解将被包含在javadoc中。

**4. @Inherited**

用于指定被修饰的Annotation具有继承性。即子类可以继承父类中的该注解。

**5. Repeatable**

表示这个注解可以在同一处多次声明


### 反射
**反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用对象方法的功能称为java语言的反射机制。**

通过反射可以在程序运行时动态创建对象并调用类中的方法。

Java的反射机制可以做3件事：运行时创建对象、运行时调用方法、运行时读写属性。

### 动态代理
代理类在程序运行时创建的代理方式被称为动态代理。
代理类并不是在Java代码中定义的，而是在运行时根据我们在Java代码中的指示动态生成的。
相比于静态代理， 动态代理的优势在于可以很方便的对代理类的函数进行统一的处理，而不用修改每个代理类的函数。 

例如：

在执行委托类中的方法之前输出“before”，在执行完毕后输出“after”。

```java
public class Vendor implements Sell {
    public void sell() {
        System.out.println("In sell method");
    }
    public void ad() {
        System, out.println("ad method")
    }
} 
```

**使用静态代理**

```java
public class BusinessAgent implements Sell {
    private Vendor mVendor;
    public BusinessAgent(Vendor vendor) {
        this.mVendor = vendor;
    }
    public void sell() {
        System.out.println("before");
        mVendor.sell();
        System.out.println("after");
    }

    public void ad() {
        System.out.println("before");
        mVendor.ad();
        System.out.println("after");
    }
} 
```

**动态代理**

需要定义一个位于代理类与委托类之间的中介类，这个中介类被要求实现InvocationHandler接口。


```java
public class DynamicProxy implements InvocationHandler {

    private Object obj; //obj为委托类对象; 
    public DynamicProxy(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before");
        Object result = method.invoke(obj, args);
        System.out.println("after");
        return result;
    }

} 
```

```java
public class Main {
    public static void main(String[] args) {
        //创建中介类实例 
        DynamicProxy inter = new DynamicProxy(new Vendor());
        //加上这句将会产生一个$Proxy0.class文件，这个文件即为动态生成的代理类文件 
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        //获取代理类实例sell 
        Sell sell = (Sell) (Proxy.newProxyInstance(Sell.class.getClassLoader(), new Class[]{Sell.class}, inter));
        //通过代理类对象调用代理类方法，实际上会转到invoke方法调用 
        sell.sell();
        sell.ad();
    }
} 
```
### 线程进程区别
进程是程序运行和资源分配的基本单位，一个程序至少有一个进程，一个进程至少有一个线程。

进程在执行过程中拥有独立的内存单元，而多个线程共享内存资源，减少切换次数，从而效率更高。

线程是进程的一个实体，是cpu调度和分派的基本单位，是比程序更小的能独立运行的基本单位，同一进程中的多个线程之间可以并发执行。

### 守护线程
用户线程：平常创建的普通线程
守护线程：用来服务于用户线程；不需要上层逻辑介入。

当线程只剩下守护线程的时候,JVM就会退出；如果还有其他的任意一个用户线程还在，JVM就不会退出。守护线程最典型的例子就是GC线程。

### 多线程上下文切换
多线程的上下文切换是指CPU控制权由一个已经正在运行的线程切换到另外一个就绪并等待获取CPU执行权的线程的过程。

### 创建线程的方式

- 实现Runnable——实现Runnable接口的类还可能扩展另一个类.
- 扩展Thread——扩展Thread类就代表这个子类不能扩展其他类
- 通过Callable和FutureTask创建线程——有返回值

### FutureTask
FutureTask表示一个异步运算的任务。

可以对这个异步运算的任务的结果进行等待获取、判断是否已经完成、取消任务等操作。
### wait()与sleep()的区别

- 调用sleep()方法的过程中，线程不会释放对象锁。而 调用 wait 方法线程会释放对象锁
- sleep()睡眠后不出让系统资源，wait让其他线程可以占用CPU
- sleep(milliseconds)需要指定一个睡眠时间，时间一到会自动唤醒.而wait()需要配合notify()或者notifyAll()使用

### synchronized和ReentrantLock的区别

synchronized是和if、else、for、while一样的关键字，ReentrantLock是类，这是二者的本质区别。

ReentrantLock是类，它提供了比synchronized更多更灵活的特性，可以被继承、可以有方法、可以有各种各样的类变量。

扩展性体现在：

- ReentrantLock可以对获取锁的等待时间进行设置，这样就避免了死锁
- ReentrantLock可以获取各种锁的信息
- ReentrantLock可以灵活地实现多路通知
- 二者的锁机制其实也是不一样的:ReentrantLock底层调用的是Unsafe的park方法加锁，synchronized操作的应该是对象头中mark word. 

### 两个线程间共享数据
通过在线程之间共享对象就可以了，然后通过wait/notify/notifyAll、await/signal/signalAll进行唤起和等待，比方说阻塞队列BlockingQueue就是为线程之间共享数据而设计的。

### ThreadLocal

ThreadLocal就是一种以空间换时间的做法在每个Thread里面维护了一个ThreadLocal.ThreadLocalMap把数据进行隔离，数据不共享，自然就没有线程安全方面的问题了。

