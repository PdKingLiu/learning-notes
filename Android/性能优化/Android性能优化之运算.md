[TOC]

# Android性能优化——运算
[原Udacity视频链接](https://classroom.udacity.com/courses/ud825/lessons/3764448604/concepts/37343690490923)

## 1. 计算和内存问题
Android中的Java代码，会需要经过编译优化在执行的过程，代码的不同写法会影响Java编译器的优化效率。不同的for循环写法就会对编译器优化这段代码产生不同的效率，当程序中包含大量这种可优化的代码的时候，运算性能就会出现问题。当我们想要大幅提升计算性能的时，就意味着我们需要理解每段代码是如何在硬件上执行的。
## 2. 缓慢的函数性能
代码运行时间过长，很大程度上归咎于编程语言，当然还有相关硬件执行代码的方式，例如在一些老旧的硬件上，执行浮点比较算法分支语句所花的时间，几乎是整数或布尔值的四倍。现代硬件通常不需要处理这种细微问题，但是这也说明了一个重要的观点，编码方式会影响性能，具体视硬件上执行的编程语言而有所不同，这个问题甚至可以追溯到芯片架构。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723162550154.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072316263718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
为了优化我们的代码，我们需要理解系统如何运行代码，缓慢的函数执行通常是由两方面的问题造成的，第一种是执行速度很慢的函数，这种函数很容易发现，当函数执行速度比预想的慢数倍、数十倍时，我们只需要找到运行很慢的函数，查看代码然后想办法解决就可以了。更难发现的事第二种类型，想方设法也难以发现，尤其是当我们有数以千计的函数时，每个函数的时间都额外增加一毫秒，从而导致整个程序变慢数百毫秒，这种类型的问题很难追踪。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072316271824.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723162745408.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
但是Android SDK 有一些很不错的工具可以帮助我们找到这些有问题的代码。

##  3. TraceView
打开Android Device Monitor，切换至DDMS窗口，点击右边上面想要追踪的进程，再点击如图按钮
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723190438668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
然后再再APP里面进行一些操作，做一些你想要跟踪的事件，然后再次点击按钮停止。

现在Traceview已弃用，如果使用的是Android Studio 3.2或更高版本，Google官方推荐使用 CPU Profiler 进行检测跟踪。

[传送门](https://developer.android.com/studio/profile/traceview)

## 4. 批处理(batching）和缓存(caching)
Batching是在真正执行运算操作之前对数据进行批量预处理。
视频中举一个例子
在查找集合中的值时，最有效的是进行排序，然后进行二分搜索等等。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723192910781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
我们写了一个函数，提供一个集合和一个值，对集合排序查看值是否在集合中。对于少量数据，这样完全是可以的，但是假如有10000个值，而且共需要数百万组数据，排序所花费的时间将会增加很多倍。

这并不是明智的方法，这时我们就需要用到批处理，对这组数据一次性完成排序，然后查找所有10,000个值。

我们找到重复的运算，找到之后，进行批处理。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723193250244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
缓存是与批处理相似的概念。

以计算机为例，内存的作用是用来存储信息，让CPU能够更快地访问数据，其速度远快于访问硬盘数据。或者以网络为例，世界各地存在大型服务器仓库，它们被称为数据中心。它们的作用是存储或缓冲被频繁访问的内容。这样，你的计算机就不必每次都访问远在12,000英里之外的服务器。以代码为例，最常见的缓存优化通常涉及多次计算，但是结果始终相同的数据。例如，在循环计算中，你计算一个4x4数列的导数，结算始终是相同的，每次重新计算循环迭代，实际是在浪费计算机资源，相反，在循环流程的外部存储导数的结果，并让你的内部循环语句引用缓存结果，可以极大地提升效率。
在这里插入图片描述
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723193726189.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 5. 阻塞UI线程

提升代码的运算效率是改善性能的一方面，让代码执行在哪个线程也同样很重要。我们都知道Android的Main Thread也是UI Thread，它需要承担用户的触摸事件的反馈，界面视图的渲染等操作。这就意味着，我们不能在Main Thread里面做任何非轻量级的操作，类似I/O操作会花费大量时间，这很有可能会导致界面渲染发生丢帧的现象，甚至有可能导致ANR。防止这些问题的解决办法就是把那些可能有性能问题的代码移到非UI线程进行操作。

## 6. 容器的性能
另外我们需要注意运算性能问题是基础算法和容器的选择，例如冒泡排序和快速排序的性能差异。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723194339386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

Java 提供了很多现成的容器Vector，ArrayList，LinkedList，HashMap等等，我们需要了解这些基础容器的性能差异以及适用场景。这样才能够选择合适的容器，达到最佳的性能。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072319450148.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)