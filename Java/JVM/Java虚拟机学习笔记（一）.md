[TOC]

# Java虚拟机结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191010141724468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
java虚拟机包括运行时`数据区域`、`执行引擎`、`本地接口`和`本地方法库`。

类加载系统不属于Java虚拟机结构。



## class文件结构
Java文件编译后生成了class文件，这种二进制文件不依赖于硬件和操作系统。
Class文件格式包含了很多关键的信息，其中的u4 u2 表示的事进本数据类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191010104246931.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

- u1: 1字节，无符号类型
- u2: 2字节，无符号类型
- u4: 4字节，无符号类型
- u8: 8字节，无符号类型

![https://camo.githubusercontent.com/b500a94627f72a03fd9ce4063cea156512110ade/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545372542312542422545362539362538372545342542422542362545352541442539372545382538412538322545372541302538312545372542422539332545362539452538342545372542422538342545372542422538372545372541342542412545362538342538462545352539422542452e706e67](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jYW1vLmdpdGh1YnVzZXJjb250ZW50LmNvbS9iNTAwYTk0NjI3ZjcyYTAzZmQ5Y2U0MDYzY2VhMTU2NTEyMTEwYWRlLzY4NzQ3NDcwNzMzYTJmMmY2ZDc5MmQ2MjZjNmY2NzJkNzQ2ZjJkNzU3MzY1MmU2ZjczNzMyZDYzNmUyZDYyNjU2OTZhNjk2ZTY3MmU2MTZjNjk3OTc1NmU2MzczMmU2MzZmNmQyZjMyMzAzMTM5MmQzNjJmMjU0NTM3MjU0MjMxMjU0MjQyMjU0NTM2MjUzOTM2MjUzODM3MjU0NTM0MjU0MjQyMjU0MjM2MjU0NTM1MjU0MTQ0MjUzOTM3MjU0NTM4MjUzODQxMjUzODMyMjU0NTM3MjU0MTMwMjUzODMxMjU0NTM3MjU0MjQyMjUzOTMzMjU0NTM2MjUzOTQ1MjUzODM0MjU0NTM3MjU0MjQyMjUzODM0MjU0NTM3MjU0MjQyMjUzODM3MjU0NTM3MjU0MTM0MjU0MjQxMjU0NTM2MjUzODM0MjUzODQ2MjU0NTM1MjUzOTQyMjU0MjQ1MmU3MDZlNjc?x-oss-process=image/format,png)
[https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/%E7%B1%BB%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84.md](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/%E7%B1%BB%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84.md)

## 类的生命周期
class文件加载至虚拟机内存到从内存中卸载称为类的声明周期。
类的声明周期阶段包括：`加载`、`链接`、`初始化`、`使用和卸载`。链接包括了三个阶段：`验证`、`准备`、`解析`。

- 加载：查找并加载class文件
- 链接：包括验证、准备、解析
	1. 验证：确保导入类型的正确性
	2. 准备：给类的静态字段分配字段，并初始化
	3. 解析：虚拟机将常量池内的符号引用替换为直接引用
- 初始化：将类变量初始化为正确的初始值。

加载阶段主要做了三件事：

- 根据类名查找二进制字节流
- 将二进制字节流所代表的的静态存储结构转化为方法区的运行时数据结构
- 在内存中生成一个代表这个类的Class对象，作为这方法区这个类的各种数据访问接口

## 类加载子系统
类加载子系统通过多种类查找Class文件加载到内存中。

Java虚拟机有两种类加载器：系统加载器和自定义加载器。

**Bootstrap ClassLoader（引导类加载器）**
用C/C++代码实现的类加载器，用于加载指定的JDK核心类库。

**Extensions ClassLoader（扩展类加载器）**
用于加载Java的扩展类

**Application ClassLoader（应用程序类加载器）**
又叫系统类加载器，用来加载当前应用程序Classpath目录、系统属性java.class.path指定的目录。
除了系统类加载器还有自定义加载器他是通过ClassLoader类来实现自己的类加载的。

## 运行时数据区域
Java内存区域划分为：

-	程序计数器
-	Java虚拟机栈
-	本地方法栈
-	Java堆
-	方法区

**JDK8 之前**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101720352178.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**JDK8** 
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101720362124.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

**1. 程序计数器**

程序计数器是一块较小的内存空间，可以看作是当前线程所执行的**字节码的行号指示器**。字节码解释器工作时**通过改变这个计数器的值来选取下一条需要执行的字节码指令**，**分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成**。

**为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。**

- 字节码解释器通过**改变程序计数器**来依次读取指令，从而**实现代码的流程控制**，如：顺序执行、选择、循环、异常处理。
- 在多线程的情况下，程序计数器用于**记录当前线程执行的位置**，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。

**2. Java虚拟机栈**
- 每一个Java虚拟机线程都一个**线程私有的Java虚拟机栈**。他的生命周期与线程相同。
- Java虚拟机栈存储线程中Java方法调用的状态，包括局部变量、参数、返回值以及运算的中间结构等。
- Java虚拟机栈**包含多个栈帧**，一个栈帧用来存储**局部变量表、操作数栈、动态链接、方法出口**等信息。当线程调用一个Java的方法时，虚拟机栈压入一个栈帧，该方法执行完后，这个栈帧就会弹出。

栈内存异常：

- 线程分配的栈容量超过Java虚拟机所允许的最大容量，Java虚拟机会抛出StackOverflowError
- 虚拟机栈可扩展，当扩展时无法再申请到足够的内存，会抛出OutOfMemoryError

**3. 本地方法栈**

他与本地方法栈类似，只不过本地方法栈是用于支撑native方法的。Java虚拟机不支持native方法，可以无需支持本地方法栈。类似的，本地方法栈也会抛出StackOverflowError和OutOfMemoryError

**4. Java堆**

Java堆是所有线程共享的内存区域，Java堆用来存放对象实例。**几乎所有的对象实例以及数组都在这里分配内存**。堆内存对象被垃圾回收器管理。

Java堆从内存回收分可以分为新生代和老年代。从内存分配可划分出多个线程私有的分配缓存区。

异常：当堆没有内存实例化，并且也无法扩展时，会抛出OutOfMemoryError。

**5. 方法区**

方法区是被**所有线程共享的运行时内存区域**，用来存储已经被Java虚拟机加载的**类的结构信息**，包括**运行时常量池，字段和方法信息，静态变量等数据**。方法区是Java堆的逻辑组成部分，他在物理上不需要连续，并且可以选择在方法区中不实现垃圾回收。

如果方法区内存空间不满足内存分配需求时，Java虚拟机会抛出OutOfMemoryError

**6. 运行时常量池**

运行时常量池不是运行时数据区域的其中一份子，而是方法区的一部分。

class文件不仅包含类的版本、接口、字段和方法等信息，还包含常量池，他用来**存放编译时期生成的字面量和符号引用**，这些内容会在类加载后存放在运行时常量池中。运行时常量池可以理解是类或接口的常量池运行时表现形式。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017204743986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

当创建类或接口时，如果运行时常量池所需的内存超过了方法区所能提供的最大值，Java虚拟机会抛出OutOfMemoryError异常。

**JDK1.7 及之后版本的 JVM 已经将运行时常量池从方法区中移了出来，在 Java 堆（Heap）中开辟了一块区域存放运行时常量池。**