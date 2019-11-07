[toc]

### 面向对象和面向过程的区别
- **面相过程**：**面相过程性能比面向对象高**， 因为类调用时需要实例化，开销比较大。
- **面向对象**：**面向对象更易维护、易复用、易扩展**，因为面向对象有封装、继承、多态性的特性。

### Java语言有哪些特点
- 面向对象（封装、继承、多态）
- 平台无关性
- 支持多线程
- 支持网络编程

### JVM JDK 和 JRE
- JVM：Java虚拟机（JVM）是运行Java字节码的虚拟机。JVM有针对不同系统的特定的实现，目的是使用相同的字节码，他们都能给出相同的结果。字节码和不同系统的 JVM 实现是 Java 语言一次编译，随处可以运行的关键所在。
- JDK：JDK是Java Development Kit，它是功能齐全的Java SDK。它拥有JRE所拥有的一切，还有编译器（javac）和工具（如javadoc和jdb）。它能够创建和编译程序。
- JRE：JRE 是 Java运行时环境。它是运行已编译 Java 程序所需的所有内容的集合，包括 Java虚拟机（JVM），Java类库，java命令和其他的一些基础构件。但是，它不能用于创建新程序。

###  Java和C++的区别
- 都是面向对象语言，都支持封装、继承、多态
- Java不提供指针来直接访问内存，程序内存更加安全
- Java类是单继承，C++支持多继承，接口可以多继承
- Java有自动内存管理机制。

### 字符型常量和字符串常量的区别
- 形式上：字符常量是单引号引起的一个字符，字符常量是双引号引起的若干字符。
- 含以上：字符常量相当于一个整型值，字符串常量代表一个地址值。

### 构造器是否可以被override
父类的私有属性和构造方法并不能被继承，所以 Constructor 也就不能被 override（重写）,但是可以 overload（重载）
### 重载和重写的区别
- 重载：发生在同一个类中，方法名相同，参数类型、个数、顺序不同，返回值和访问修饰符可不同。
- 重写：发生在父子类中，方法名、参数列表必须相同。

### 封装 继承 多态
- 封装：把一个对象的属性私有化。同时提供一些可以被外界访问的属性和方法。
- 继承：在已有类的基础上建立新类，可以增加新的功能，也可以用父类的功能。
- 多态：指程序中定义的**引用变量类型**和**通过该引用变量调用的方法**在编程时并不确定，在程序运行期间才能确定。

实现多态的两种形式：**继承、接口**

### String StringBuffer StringBuilder
**线程安全**
- String对象不可变，线程安全
- StringBuffer对方法加了同步锁，线程安全
- StringBuilder非线程安全

**性能**
- String 在改变时，都会生成一个新String对象。
- StringBuffer每次都对StringBuffer对象本身操作，不生成新对象
- StringBuilder性能相比StringBuffer高，但是线程不安全。

**总结**
- 少量数据：String
- 单线程操作大量数据：StringBuilder
- 多线程操作大量数据：StringBuffer
### 自动装箱与拆箱
装箱：将基本类型用引用类型包装起来
拆箱：将包装类型转换为基本数据类型

### 接口和抽象类的区别
- 接口方法默认是public，所有方法在接口中不能有实现，抽象类可以有非抽象的方法
- 接口只能有static、final变量
- 类可以实现多个接口，只能实现一个抽象类
- 设计层面来说，抽象是对类的抽象，是一种模板设计，而接口是对行为的抽象，是一种行为的规范。
### == 与 equals
- == 判断两个对象是不是相等（是不是同一对象）

对于 equals()，有两种情况
- 没有覆盖equals方法，等价于使用==比较
- 覆盖了equals方法，若他们内容相等则返回true


String 的equals被重写过，object比较的是对象地址，String比较的是值。

创建String 类型的对象时，首先会在常量池中查找有没有和要创建值相同的对象，否则就在常量池中创建一个对象。

### hashCode 与 equals 
hashCode作用是获取哈希码。实际上返回一个int整数。这个哈希码可以确定对象在哈希表中的索引位置。hashCode定义在Object中，Java中任何类都有hashCode函数。

- 如果两个对象相对，则hashCode一定相等
- 两个对象相等，对两个对象分别调用equals方法都返回true
- 两个对象有相同的hashCode，他们也不一定相等。
- 因此，equals 方法被覆盖过，则 hashCode 方法也必须被覆盖 

### 线程的基本状态
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191104194800320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191104195010250.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 当线程执行 wait()方法之后，线程进入 **WAITING（等待）** 状态。
- 进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而TIME_WAITING(超时等待) 状态相当于在等待状态的基础上增加了超时限制，比如通过 sleep（long millis）方法或 wait（long millis）方法可以将 Java 线程置于 TIMED WAITING 状态。当超时时间到达后 Java 线程将会返回到 RUNNABLE 状态。
- 当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到 BLOCKED（阻塞） 状态。
- 线程在执行 Runnable 的run()方法之后将会进入到 TERMINATED（终止） 状态。

###  final 关键字
final关键字可以修饰**变量、方法、类**

- 修饰变量：如果是基本类型，一旦初始化不能更改；如果是引用类型，则是不能指向其他的对象
- 修饰类：表明不能被继承
- 修饰方法：可以避免继承类修改他的定义。private都隐式的指定为final

### Java异常
**异常层次**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191104195625854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

Java中所有的异常都是Throwable类的子类。

- **Error**：是程序无法处理的错误，表示运行应用程序中较严重的问题。大多数错误与代码编写者执行的操作无关，而表示代码运行时 JVM出现的问题。比如OutOfMemoryError。这些异常发生时，Java虚拟机一般会选择线程终止。
- **Exception**：是程序本身可以处理的异常。	Exception 类有一个重要的子类 RuntimeException。RuntimeException 异常由Java虚拟机抛出。比如NullPointerException、ArithmeticException、ArrayIndexOutOfBoundsException 

**异常处理**

- try——用于捕获异常
- catch——用于处理try捕获到的异常
- finally——无论是否捕获到，都会执行的代码块
	try或cache中出现return时，finally语句块将在方法返回之前被执行，若finally里面有返回，他会覆盖原始的返回值。


### 有些字段不想进行序列化
使用transient关键字修饰。当对象被反序列化时，被transient修饰的变量值不会被持久化和恢复。
