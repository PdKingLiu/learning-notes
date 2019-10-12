[toc]
# 垃圾标记算法
垃圾收集器，通常称为GC。

GC 主要做了两个工作，一个是内存的划分和分配，另一个是对垃圾的回收。

内存的划分依赖于GC设计，为了能够更快的进行垃圾回收。

GC是采用的分代垃圾回收算法。

## Java中的引用
**1.强引用**

当新建一个对象时就创建了一个具有强引用的对象，当他还存在引用链时，垃圾回收器绝不会回收他。

**2.软引用**

如果一个对象只具有软引用，当内存不足时，会直接回收这些对象的内存。Java提供了SoftReference类来实现软引用。

**3.弱引用**

弱引用比起软引用有更短的生命周期，垃圾回收器若发现只具有若引用的对象，不管当前内存是否够用，都会回收他。Java提供WeakReference来实现弱引用。

**4.虚引用**

虚引用并不会决定对象的生命周期，如果一个对象仅持有虚引用，这就和没有任何引用一样，在任何时候都可能被收集。Java提供PhantomReference类实现虚引用。

## 引用计数法
引用计数法基本思想就是每个对象都有一个引用计数器，当对象在某处被引用时，引用计数就加一，引用失效时就减一，当引用计数值为0，就不能被使用并变成垃圾。

现代虚拟机没有选择引用计数法来标记垃圾的，主要原因是没有办法解决对象之间相互循环引用的问题。

```java
public class Main {
    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new A();
        a1.reference = a2;
        a2.reference = a1;
    }
}
class A {
    Object reference;
} 
```
上面例子a1和a2相互引用。如果虚拟机采用引用计数法，垃圾回收器就无法回收他们。

## 跟搜索法
基本思想就是选定一些对象作为GC roots，并组成跟对象集合，然后以这些GCRoots的对象最为起始点，向下搜索，如果目标对象到GCRoots是连接着的，我们则称该目标对象是可达的，如果不可达则说明目标对象是可以被回收的对象。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012223238266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

上图中Obj5、Obj6、Obj7都是不可达的对象，其中虽然Obj5和Obj6相互引用，但是因为他们到GCRoots是不可达的，所以他们仍旧被判定为可回收的对象。

可以作为GC Roots的对象主要有以下几种：

- Java栈中引用的对象
- 本地方法栈中JNI引用的对象
- 方法区运行时常量池引用的对象
- 方法去中静态属性引用的对象
- 运行中的线程
- 有引导类加载器加载的对象
- GC控制的对象 

# Java对象在虚拟机中的声明周期
在对象在被类加载器加载到虚拟机后，对象在Java虚拟机中有7个阶段。

**1.创建阶段**

- 为对象分配空间
- 构造对象
- 从超类到子类对static成员进行初始化
- 递归调用超类的构造方法
- 调用子类的构造方法

**2.应用阶段**

当对象创建，并分配给变量赋值时，状态就进入到了应用阶段。

**3.不可见阶段**

在程序找不到任何的强引用，比如程序已经超出了该对象的作用域。在不可见阶段，对象可能被特殊的强引用GC roots持有着，比如对象本地方法栈中JNI引用或运行中的线程引用等。

**4.不可达阶段**

在程序找不到任何对象的强引用，并且发现对象不可达。

**5.收集阶段**

垃圾收集器发现对象不可达，并且垃圾收集器已经准备好要对该对象的内存空间进行重新分配，这个时候如果对象重写了finalize方法，则会调用该方法。

**6.终结阶段**

在对象执行完finalize方法仍然处于不可达状态，或者没有重写该方法，则该对象进入终结阶段，等待被回收。

**7.对象空间重新分配阶段**

当垃圾收集器对对象空间进行回收或者在分配时，这个对象就会彻底消失。

# 垃圾收集算法
## 标记——清除算法
标记清除算法是一种常见的基础垃圾收集算法，他将垃圾收集分为两个阶段。

- 标记阶段：标记出可以清除回收的对象
- 清除阶段：回收被标记的对象所占用的空间

标记清除算法是基础的算法，其他垃圾收集算法是在此算法的基础上进行改进的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012225004701.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

两个缺点：

- 低效率
- 产生大量不连续内存碎片，导致没有足够连续内存分配给较大的对象

## 复制算法
为了解决标记清除效率不高的问题，产生了复制算法。他把内存划分为两个相等的区域，每次只使用其中一个区域。在垃圾收集时，把存活的对象复制到另外一个区域，最后将当前使用的区域可回收对象进行回收。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012225417352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

这种算法不考虑内存碎片的问题，代价就是使用内存原来的一半。

算法效率取决于存活对象的数目多少，如果存活对象很少，复制效率就会很高。

复制算法广泛应用在新生代中。

## 标记——压缩算法
在新生代中可以使用复制算法，但是在老年代就不能使用复制算法了，因为老年代对象的存活率较高，这样就会有较多的复制操作，效率变得底下。

标记清除算法可以应用到老年代中，但是他的效率不高，在内存回收后容易产生大量内存碎片。

因此出现了标记压缩算法：

与标记清除算法不同的是，在标记可回收对象后将所有存活的对象压缩到内存的一端，然后对边界外的内存进行回收。

标记压缩算法解决了标记——清除算法效率低和容易产生大量内存碎皮的问题，广泛应用于老年代。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012230356472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

## 分代收集算法
分代收集算法结合不同的收集算法来处理不同的空间。

Java虚拟机对象声明周期的长短将他们分别放到不同的区域，并在不同的区域采用不同的收集算法，这就是分代收集算法。

Java堆区基于分代的概念，分为新生代和老年代，新生代又细分为Eden、From Survivor、To Survivor。

Eden空间中的大多数对象声明周期很短，所以新生代的空间比例并不是均分的。HotSpot虚拟机默认的Eden空间和两个Survivor空间的比例为8:1。

### 分代收集
根据堆区的空间划分，垃圾收集的类型分为两种：

- Minor Collection：新生代垃圾收集
- Full Collection：老年代垃圾收集，通常会伴随一次Minor Collection，他的收集频率较低，耗时较长

当执行Minor Collection时，Edon空间存活的对象会被复制到To Survivor空间，并且之前经过一次Minor Collection并在From Survivor空间存活的仍然年轻的对象也会复制到To Survivor空间。

有两种情况Eden空间和From Survivor空间存活的对象不会复制到To Survivor空间，而是直接晋升老年代。

- 一种是存活的对象分代年龄超过设置的阈值（控制对象经历多少次Minor Survivor才晋升到老年代）
- 另一种是To Survivor空间容量达到阈值。

在存活的对象被复制到To Survivor或者晋升至老年代，就一位置Eden个To Survivor空间剩下的都是可回收的对象。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012232328191.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

这是Eden和From Survivor空间会被清空，新生代中存活的对象都存放在To Survivor空间。

接下来将From Survivor和To Survivor空间互换，每次都保证To Survivor是空的。

这就是复制算法在新生代的应用。

老年代则使用标记——压缩或标记——清除算法。