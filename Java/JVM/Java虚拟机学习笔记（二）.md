[TOC]

# 对象的创建
通常情况下，我们是通过new指令完成一个对象的创建的。

虚拟机在接受到一个new指令会做如下操作：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017205903796.png)

1. **判断对象的类是否加载、链接、初始化**，	虚拟机在接收到一条new指令时，首先会检查这个指令的参数**是否在常量池中定位到一个类的引用**，并且检查这个符号引用代表的类**是否被类加载器加载、链接和初始化**。如果没有，那必须先**执行相应的类加载过程**。

2. **为对象分配内存**
	类加载完成后，接着**会在Java堆中划分一块内存区域分配给对象**。内存分配根据**Java堆是否规整**，有两种方式，而 Java 堆内存是否规整，**取决于 GC 收集器的算法是"标记-清除"，还是"标记-整理"** ，值得注意的是，复制算法内存也是规整的。：
	
	- **指针碰撞**：如果Java堆整的内存是规整的，则分配内存时将位于中间的指针指示器向空闲的内存移动一段与对象大小相等的距离。
	- **空闲列表**：如果Java堆是不规整的，则需要Java虚拟机维护一个列来记录哪些内存是可用的，在分配的时候从列表中查询到足够大的内存分配给对象，并更新列表记录。
	
3. **处理并发安全问题**
	创建对象是一个频繁的操作，所以要解决并发问题，有两种方式：
 	- 对分配空间的动作进行同步，保证操作的原子性
 	- 每个线程在Java堆预先分配一小块内存，这块内存叫做本地分配缓冲（TLAB）。线程需要分配内存时，就在对应线程的的TLAB上分配内存，当TLAB用完并且分配到了新的TLAB时，这时才需要同步锁定。

4. **初始化分配到的内存空间**
	将分配到的内存，除了对象头都初始化为零值。

5. **设置对象的对象头**
	初始化零值完成之后，**虚拟机要对对象进行必要的设置**，将对象的所属类、对象的HashCode和对象的GC分带年龄等数据**存储在对象的对象头**。
	
6. **执行init方法进行初始化**
	执行init方法，初始化对象的成员变量、调用类的构造方法，这样就完成了一个对象的创建。

# 对象的堆内存分布
Java对象在堆内存中内存布局分为三个区域，分别是对象头、实例数据、对齐填充。

- **对象头**：对象头包括两部分信息，分别是**Mark World和元数据指针**。
	**Mark World**用于存储运行时数据，比如hashcode，锁状态标志、GC分带年龄、线程持有的锁等等。
	**元数据用**于指向方法区中目标类元数据的指针，通过元数据可以确定对象的具体类型。
- **实例填充**：用于存储对象中的各种类型的字段信息。
- **对齐填充**：不一定存在，通常起到一个占位的作用。

Mark world在HotSpot中的实现类为markOop.hpp，markOop被设计成一个非固定的数据结构，这是为了在极小的控件中存储尽量对的数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101116051690.png)
数据类型解释：

- hash：对象的哈希码
- age：对象的分代年龄
- biased_lock：偏向锁标示位
- lock：锁状态标示位
- JavaThread*持有偏向锁的线程ID
- epoch：偏向时间戳

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191011164942714.png)

# 对象的访问定位
建立对象就是为了使用对象，我们的 Java 程序通过栈上的 **reference 数据来操作堆上的具体对象**。对象的访问方式由虚拟机实现而定，目前主流的访问方式有使用**句柄**和**直接指针**两种 。

 **1. 句柄**

如果使用句柄的话，那么 **Java 堆中将会划分出一块内存来作为句柄池**，**reference 中存储的就是对象的句柄地址**，而**句柄中包含了对象实例数据与类型数据各自的具体地址信息**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017221449283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

**2. 直接指针**

如果使用直接指针访问，那么 Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而 reference 中存储的直接就是对象的地址。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017221614107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

# oop-klass模型
oop-klass模型是用来描述Java实例对象的一种数据，它分为两个部分：

**Klass：** 包含元数据和方法信息，用来描述Java类。一般jvm在加载class文件时，会在方法区创建instanceKlass，表示其元数据，包括常量池、字段、方法等。

**Ordinary Object Pointer （普通对象指针）**，它用来表示对象的实例信息，看起来像个指针实际上是藏在指针里的对象。Klass是在class文件在加载过程中创建的，OOP则是在Java程序运行过程中new对象时创建的。

之所以采用这个模型是因为HotSopt JVM的设计者不想让每个对象中都含有一个vtable（虚函数表），所以就把对象模型拆成klass和oop，其中oop中不含有任何虚函数，而Klass就含有虚函数表，可以进行method dispatch。

oop是一个家族，Java虚拟机内存会定义很多oop类型：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012191631930.png)

其中oopDesc是所有oop的顶级父类

arrayOopDesc是objArrayOopDesc和typeArrayOopDesc的父类。

**instanceOopDesc和arrayOopDesc都可以用来描述对象头。**

klass家族：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012192718429.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

其中Klass是klass家族的父类，ArrayKlass是ObjArrayKlass和TypeArrayKlass的父类。

instanceOopDesc的定义如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012193813137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

父类：oopDesc

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012193859846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

oopDesc包含两个数据成员：mark和_metadata。其中markoop类型的mark对象指的是对象头Mark World。metadata是一个共用体，其中klass是普通指针，_compressed_klass是压缩类指针，这两个指针根据对应关系都会指向instanceKlass，instanceKlass可以用来描述元数据。

instanceKlass代码：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012194250231.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
ClassState用来标识对象的加载进度。instance继承自Klass，Klass中定义的部分字段如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101219442066.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
可以看到Klass描述了元数据，具体来说就是Java类在Java虚拟机中对等的C++类型描述，这样继承自klass的instanceKlass同样可以用来描述元数据，了解oop-klass模型我们就可以分析Java虚拟机是如何通过栈帧对象引用找到对应的对象实例的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012195053235.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

Java虚拟机通过栈帧的对象引用找到Java堆中的instanceOopDesc，这样就可以访问Java对象的实例信息，当需要访问具体的类型等信息是，可以通过instanceOopDesc中的元数据指针来找到方法区中对应的instanceKlass。

**实例**

```java
class Model
{
    public static int a = 1;
    public int b;

    public Model(int b) {
        this.b = b;
    }
}

public static void main(String[] args) {
    int c = 10;
    Model modelA = new Model(2);
    Model modelB = new Model(3);
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012201044384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

# 关于的String的一些补充

**1. String类和常量池**

**String 对象的两种创建方式：**

```java
String str1 = "abcd";//先检查字符串常量池中有没有"abcd"，如果字符串常量池中没有，则创建一个，然后 str1 指向字符串常量池中的对象，如果有，则直接将 str1 指向"abcd""；
String str2 = new String("abcd");//堆中创建一个新的对象
String str3 = new String("abcd");//堆中创建一个新的对象
System.out.println(str1==str2);//false
System.out.println(str2==str3);//false
```
这两种不同的创建方法是有差别的。

- 第一种方式是在**常量池**中拿对象；
- 第二种方式是直接在**堆内存空间创建一个新的对象**。

记住一点：只要使用 new 方法，便需要创建新的对象。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017222524526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**String 类型的常量池比较特殊。它的主要使用方法有两种：**

- 直接使用**双引号声明出来的 String 对象会直接存储在常量池中**。
- 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern 方法。String.intern() 是一个 Native 方法，它的作用是：**如果运行时常量池中已经包含一个等于此 String 对象内容的字符串，则返回常量池中该字符串的引用**；如果没有，JDK1.7之前（不包含1.7）的处理方式是**在常量池中创建与此 String 内容相同的字符串，并返回常量池中创建的字符串的引用**，JDK1.7以及之后的处理方式是在常量池中记录此字符串的引用，并返回该引用。

```java
  String s1 = new String("计算机");
  String s2 = s1.intern();
  String s3 = "计算机";
  System.out.println(s2);//计算机
  System.out.println(s1 == s2);//false，因为一个是堆内存中的 String 对象一个是常量池中的 String 对象，
  System.out.println(s3 == s2);//true，因为两个都是常量池中的 String 对象
```
**字符串拼接**

```java
 String str1 = "str";
 String str2 = "ing";
 String str3 = "str" + "ing";//常量池中的对象
 String str4 = str1 + str2; //在堆上创建的新的对象	  
 String str5 = "string";//常量池中的对象
 System.out.println(str3 == str4);//false
 System.out.println(str3 == str5);//true
 System.out.println(str4 == str5);//false
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101722305648.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
尽量**避免多个字符串拼接**，**因为这样会重新创建对象**。如果需要改变字符串的话，可以使用 StringBuilder 或者 StringBuffer。

**String s1 = new String("abc");这句话创建了几个字符串对象？**

将创建 1 或 2 个字符串。如果池中已存在字符串常量“abc”，则只会在堆空间创建一个字符串常量“abc”。如果池中没有字符串常量“abc”，那么它将首先在池中创建，然后在堆空间中创建，因此将创建总共 2 个字符串对象。

```java
String s1 = new String("abc");// 堆内存的地址值
String s2 = "abc";
System.out.println(s1 == s2);// 输出 false,因为一个是堆内存，一个是常量池的内存，故两者是不同的。
System.out.println(s1.equals(s2));// 输出 true
```

**8 种基本类型的包装类和常量池**
- **Java 基本类型的包装类的大部分都实现了常量池技术，即 Byte,Short,Integer,Long,Character,Boolean；这 5 种包装类默认创建了数值[-128，127] 的相应类型的缓存数据**，但是超出此范围仍然会去创建新的对象。 为啥把缓存设置为[-128，127]区间？性能和资源之间的权衡。
- 两种浮点数类型的包装类 Float,Double 并没有实现常量池技术。

```java
Integer i1 = 33;
Integer i2 = 33;
System.out.println(i1 == i2);// 输出 true
Integer i11 = 333;
Integer i22 = 333;
System.out.println(i11 == i22);// 输出 false
Double i3 = 1.2;
Double i4 = 1.2;
System.out.println(i3 == i4);// 输出 false
```

> 参考
> 安卓进阶解密
> [https://www.jianshu.com/p/424a920771a3](https://www.jianshu.com/p/424a920771a3)
> [https://www.jianshu.com/p/0009aaac16ed](https://www.jianshu.com/p/0009aaac16ed)