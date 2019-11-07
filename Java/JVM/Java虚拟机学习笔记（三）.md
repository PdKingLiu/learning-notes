[TOC]

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
这个算法的基本思想是通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链（即GC Roots到对象不可达）时，则证明此对象是不可用的。

基本思想就是选定一些对象作为GC roots，并组成跟对象集合，然后以这些GCRoots的对象最为起始点，向下搜索，如果目标对象到GCRoots是连接着的，我们则称该目标对象是可达的，如果不可达则说明目标对象是可以被回收的对象。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012223238266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

上图中Obj5、Obj6、Obj7都是不可达的对象，其中虽然Obj5和Obj6相互引用，但是因为他们到GCRoots是不可达的，所以他们仍旧被判定为可回收的对象。

可以作为GC Roots的对象主要有以下几种：

- Java栈中引用的对象
- 本地方法栈中JNI引用的对象
- 方法区运行时常量池引用的对象
- 方法去中静态属性引用的对象

简单来说：**就是所有被引用的对象（包括静态对象和非静态对象）+线程+Native方法中的对象，都可以作为GC Root的对象。**

# Java对象在虚拟机中的生命周期
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

# GC触发条件
**Minor GC触发条件**

- 当Eden区满时，触发Minor GC。

**Full GC触发条件**

- 调用System.gc时，系统建议执行Full GC，但是不是必然执行
- 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
	当要触发一次Minor GC时，如果发现统计数据中Minor GC的平均晋升大小比目前old gen剩余的空间大，则不会触发Minor GC，而是直接转为Full GC。
- 方法区空间不足
	运行时数据区域中的方法区，在HotSpot虚拟机中又被习惯称为永生代或者永生区，Permanet Generation中存放的为一些class的信息、常量、静态变量等数据，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下也会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：
java.lang.OutOfMemoryError: PermGen space
为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。 
- 老年代空间不足，和第二点类似
	旧生代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误： java.lang.OutOfMemoryError: Java heap space 为避免以上两种状况引起的FullGC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。 

# OopMap

OopMap 用于枚举GCRoots

当垃圾回收时，收集线程会对栈上的内存进行扫描，看看那些位置上存储了Reference类型。如果发现了某个位置上存储的是Reference类型，就意味着这个引用所指向的对象在这一次垃圾回收过程中不能够回收。

栈上的本地变量表里面只有一部分数据是Reference类型的，为了避免每次都扫描整个栈，所以采用空间换时间的策略。在某个时候(安全点)把栈上代表引用的位置记录下来，GC时直接读取，避免了全部扫描。 HotSpot虚拟机采用了一种叫做OopMap的数据结构来记录这些引用（OopMap也帮助HotSpot实现了准确式GC），OopMap记录了栈上本地变量到堆上对象的引用关系，这些引用指向的对象不能够回收，并且可以作为根节点来进行可达性分析，查找出不能够回收的对象。


# 安全点

safepoint 安全点是指一些特定的位置，当线程运行到这些位置时，线程的一些状态可以被确定，比如记录OopMap的状态，从而确定GC Root的信息，使JVM可以安全的进行一些操作，比如开始GC。

上面讲到了为了快点进行可达性的分析，使用了一个引用类型的映射表，可以快速的知道对象内或者栈和寄存器中哪些位置是引用了。

但是随着而来的又有一个问题，就是在方法执行的过程中， 可能会导致引用关系发生变化，那么保存的OopMap就要随着变化。如果每次引用关系发生了变化都要去修改OopMap的话，这又是一件成本很高的事情。所以这里就引入了安全点的概念。

什么是安全点？OopMap的作用是为了在GC的时候，快速进行可达性分析，所以OopMap并不需要一发生改变就去更新这个映射表。只要这个更新在GC发生之前就可以了。所以OopMap只需要在预先选定的一些位置上记录变化的OopMap就行了。这些特定的点就是SafePoint（安全点）。由此也可以知道，程序并不是在所有的位置上都可以进行GC的，只有在达到这样的安全点才能暂停下来进行GC。

既然安全点决定了GC的时机，那么安全点的选择就至为重要了。安全点太少，会让GC等待的时间太长，太多会浪费性能。所以安全点的选择是以程序“是否具有让程序长时间执行的特征”为标准的，所以我们这里了解一下结果就行了。一般会在如下几个位置选择安全点：

1. 循环的末尾 (防止大循环的时候一直不进入safepoint，而其他线程在等待它进入safepoint)
2. 方法返回前
3. 调用方法的call指令之后
4. 抛出异常的位置 

# 新生代收集器
**并行、并发**

- 并行：指多条垃圾收集线程并行工作，此时用户线程任然处于等待状态
- 并发：指用户线程与垃圾收集线程同时执行。

**吞吐量**

吞吐量 = **CPU运行在用户代码时间 / CPU总消耗时间**

CPU总消耗时间 = **运行用户代码时间 + 垃圾收集时间**

## Serial（串行）收集器
Serial（串行）收集器，它采用**复制算法**的**新生代**收集器，它是一个**单线程**的收集器，只会使用一个CPU或一条收集线程完成垃圾收集工作。

它在进行垃圾回收时，必须暂停其他所有的工作线程，直至收集结束（Stop The World）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107111039506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
上图是Serial 收集器（老年代采用Serial Old收集器）的运行过程。

特点：**HotSpot虚拟机运行在Client模式下的默认的新生代收集器。简单高效（与其他收集器的单线程相比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得更高的单线程收集效率。**

## ParNew 收集器
ParNew收集器是Serial收集器的**多线程**版本，他也是一个**新生代**的收集器。除了使用多线程进行垃圾回收外，其余行为和Serial收集器完全相同。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107111439519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
上图是ParNew的工作过程。

除了Serial收集器外，目前只有它能和CMS收集器配合工作。

特点：单CPU效率低于Serial收集器，因为存在线程交互带来的开销。多CPU环境下，能更好的利用系统资源。

## Parallel Scavenge 收集器
Parallel Scavenge收集器是一个并行的多线程新生代收集器，它使用复制算法。

Parallel Scavenge收集器的目标是**达到一个可控制的吞吐量。**

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。

Parallel Scavenge收集器除了提供可以精确控制吞吐量的参数，还提供了一个参数-XX:+UseAdaptiveSizePolicy，打开参数后，就不需要手工指定新生代的大小、Eden和Survivor区的比例、晋升老年代对象年龄等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种方式称为GC自适应的调节策略（GC Ergonomics）。自适应调节策略也是Parallel Scavenge收集器与ParNew收集器的一个重要区别。



# 老年代收集器
## Serial Old收集器

Serial Old 是 Serial收集器的**老年代版本**，它同样是一个**单线程收集器**，使用**标记-整理**算法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107113241323.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

## Parallel Old收集器
Parallel Old收集器是**Parallel Scavenge收集器的老年代版本**，使用多线程和**标记-整理**算法。

在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107113404761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## CMS收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，非常重视服务的响应速度。它是基于**标记-清除**算法实现的。

收集流程：

- 初始标记：仅仅标记GC Roots能直接关联到的对象，速度快
- 并发标记：：进行GC Roots Tracing的过程，耗时长
- 重新标记：为了修正并发标记期间程序继续运作导致标记产生变动的部分对象标记记录
- 并发清除

并发标记和并发清除过程收集器线程都可以与用户线程一起工作，总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107114502855.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
**优点：并发收集、低停顿**

**缺点：**

- 并发阶段，它虽然不会导致用户线程停顿，但会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。
- 由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生。这一部分垃圾出现在标记过程之后，CMS无法再当次收集中处理掉它们，只好留待下一次GC时再清理掉。
- 标记-清除算法导致的空间碎片

## G1收集器
与其他GC收集器相比有以下特点：
- 并行与并发：G1能充分利用多CPU多核优势，使用多个CPU缩短Stop The World停顿时间。他可以并发的方式让Java程序继续执行。
- 分代收集
- 空间整合：G1基于标记-整理算法，G1运行期不会产生内存空间碎片。
- 可预测的停顿：降低停顿时间是G1和CMS共同的关注点，但G1除了降低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在GC上的时间不得超过N毫秒


**跨越整个堆内存**

G1不再像其他收集器那样收集的范围是新生代或老年代。G1的内存布局和其他收集器有很大的区别。

他将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留新生代和老年代，但是他们不再是物理隔离的了，而是一部分Region的集合。

**建立可预测的时间模型**

G1收集器有计划的在整个Java堆中进行全区域的垃圾收集。

G1追踪各个Region里面的垃圾堆积的价值大小（回收所获空间大小以及回收所需时间），在后台维护一个优先列表，优先回收价值最大的Region。

这种Region的划分内存空间以及优先级区域的回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。

**避免全堆扫描 Remembered Set**

Region不可能是孤立的，一个对象分配在某个Region中，可以与整个Java堆任意的对象发生引用关系。在做可达性分析确定对象是否存活的时候，需要扫描整个Java堆才能保证准确性，这会减低GC的效率。

为了避免全堆扫描发生，虚拟机为G1每个Region维护了一个与之对应的Remembered Set。虚拟机发现程序在对Reference类型的数据进行写操作时，会检查Reference引用对象是否处于不同的Region中，如果是，就把相关引用记录到被引用对象所属的Region的Remembered Set中。当进行内存回收时，在GC根节点的枚举范围中加入RememberedSet 即可保证不对全堆扫描也不会有遗漏。


**G1收集器运作步骤**

- 初始标记：仅仅标记GC roots能直接关联到的对象。
- 并发标记：从GC Root 开始对堆对象进行可达性分析，可与用户程序并发执行。
- 最终标记：修正并发期间用户程序继续运作导致标记产生变动的那一部分标记内容。
- 筛选回收：首先对各个Region中的回收价值和成本排序，根据用户期望的GC停顿时间进行制定回收计划。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107165756625.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191107165855260.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)