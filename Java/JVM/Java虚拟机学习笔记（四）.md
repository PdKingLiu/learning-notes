[toc]

# 类加载过程
## 概述
Class 文件需要加载到虚拟机中之后才能运行和使用，那么虚拟机是如何加载这些 Class 文件呢？

系统加载 Class 类型的文件主要三步：**加载 => 连接 => 初始化**。连接过程又可分为三步：**验证 => 准备 => 解析**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017194724649.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 加载
类加载过程的第一步，主要完成下面3件事情：
- 通过全类名获取定义此类的**二进制字节流**
- 将字节流所代表的**静态存储结构**转换为**方法区的运行时数据结构**
- 在内存中生成一个代表该类的 **Class 对象**，作为**方法区这些数据的访问入口**

**数组类型**不通过类加载器创建，它由 Java 虚拟机直接创建。

**加载阶段和连接阶段**的部分内容是**交叉进行**的，加载阶段尚未结束，连接阶段可能就已经开始了。

## 连接
### 验证
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017195545603.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 准备
准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。

- 这时候进行内存分配的**仅包括类变量（static），而不包括实例变量**，实例变量会在对象实例化时随着对象一块分配在 Java 堆中。
- 这里所设置的初始值"通常情况"下是数据类型**默认的零值（如0、0L、null、false等）**，比如我们定义了public static int value = 111 ，那么 value 变量在准备阶段的初始值就是 0 而不是111（初始化阶段才会复制）。特殊情况：比如给 value 变量加上了 fianl 关键字public static final int value=111 ，那么准备阶段 value 的值就被复制为 111。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101720003839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 解析
解析阶段是虚拟机将常量池内的**符号引用替换为直接引用的过程**。

**符号引用**就是一组符号来描述目标，可以是任何字面量。**直接引用**就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

## 初始化
初始化是类加载的最后一步，也是真正执行类中定义的 Java 程序代码(字节码)，初始化阶段是执行类构造器  < clinit >  ()方法的过程。