[TOC]

# Android性能优化之渲染

[原Udacity视频链接](https://classroom.udacity.com/courses/ud825/lessons/3753178711/concepts/37669287820923)

## 1. 为什么渲染性能很重要
现在有不少App为了达到很华丽的视觉效果，会需要在界面上层叠很多的视图组件，但是这会很容易引起性能问题。如何平衡Design与Performance就很需要智慧了。

## 2. Defining 'Jank'

渲染功能 Rendering 是应用程序最普遍的功能，一方面设计师要求为用户展现可用性最高的体验，另一方面那些华丽的图片和动画，并不是在所有的设备上都能流畅运行，所以我们首先得了解一下什么是渲染性能。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzkzNzBCMUIyNTc0OTQ3QjU5RjQ5MzYxOUZGMEQ5QzU2LzMwMzE)

Android系统每16ms重新绘制一次activity，也就是说你的应用必须在16ms内完成屏幕刷新的全部逻辑操作，这样才能达到每秒60帧。

这个每秒的帧数参数实际来源于手机硬件，现在大多数手机屏幕刷新率大概在60赫兹，这意味着你有60ms的时间去完成每一帧的绘制逻辑操作。一帧可以看做是一张的独立图片，60帧每秒就意味着:16ms=1000/60Hz，相当于60fps。这就是上面说的16ms，这也是为什么Android系统每隔16ms就会发出一次VSYNC信号触发对UI进行渲染，如果这16ms内我们没有完成对视图的绘制，那么就会出现丢帧的情况。


![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzRCNTEzRTlBQTE5MDRBOEE4NzEyNzYxQjMyMkIyMkY1LzMwMzk)
安卓系统尝试在屏幕上绘制新的一帧，但是这一帧还没有准备好，所以画面就不会刷新，所以用户盯着同一张图看了32ms而不是16ms，丢帧状况下运行的任何动画，用户很容易察觉出卡顿感，如果出现多次掉帧，用户就会开始抱怨，如果此时用户正和系统进行交互，例如侧滑列表或者输入数据，那么卡顿感就会更加明显，这样造成的后果便会是用户毫不留情的卸载APP，

## 3. CPU和GPU

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzc3MDI0Q0U3RDcxNzQ0RTZBQUEwMzBBODRBNzlBMzQxLzMwNzQ)
Android系统的渲染管线分为两个关键组件CPU和GPU，两者共同工作在屏幕上绘制图片,必须遵守特定的规则才能达到效果。

CPU方面，最常见的性能问题是不必要的布局，这些内容必须在视图层次结构中进行测量、清楚并重新创建，造成这个的原因有两个，一是重建显示列表的次数太多，二是花费太多时间作废视图层次，并进行不必要的重绘。这两个原因在更新显示列表或者其他缓存GPU资源时导致CPU工作过度。

第二个主要问题出在GPU上面。就是我们所说的透支（overdraw），通常是在色素着色过程照中、通过其他工具进行后期着色时，浪费了GPU处理时间。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlL0ZBN0QyODkyM0VBMzRBQzk5M0VENTBGOTYxQTM5NDM5LzMxMTU)

## 4. Android UI 和 GPU

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzBkZWEyZDQ4LnBuZw)


那些复杂的xml布局文件是如何转换成用户能看懂的图像的呢，实际上，这是由格栅化操作完成的。格式化将诸如字符串、按钮、路径或者形状的一些高级对象（Button，Shape，Path，String，Bitmap），拆分到不同的像素上在屏幕上进行显示，格栅化是一个非常耗时的过程，手机有一块特殊的硬件，就是为了加速这个过程，也就是GPU。GPU使用一些指定的基础指令集，主要是多边形和纹理，也就是图片，CPU在绘制图线前，会向GPU输入这些指令，这一过程通常使用的api就是Android的OpenGL ES


![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzE3NDdCNkQ2NEI2RDQyRERBODhCRTMxNUY3QTA1QjYzLzMyMjc)

这就是说。在屏幕上绘制UI对象时，不管是按钮、路径或者复选框，都需要在CPU中首先转换为多边形或者纹理（Textures），然后再传递给GPU进行格栅化，一个UI对象转化为多边形和纹理的过程非常耗时，从CPU传数据给GPU也相当耗时，所以要尽可能的减少传递的次数，OpenGL ES API 允许数据上传到GPU后，可以对数据进行保存，当你下次绘制一个按钮的时候，只需要在GPU存储器引用它然后告诉OpenGL如何绘制即可。

所以说渲染性能优化就是，尽可能快的上传数据到GPU，然后尽可能长的在不修改的条件下保存数据。在Android Honeycomb （3.0）之后渲染系统性能方面有很多改进，Android系统在降低、重新利用GPU资源方面做了很大改进，例如，任何主题提供的资源（BitMap、Drawable等）都是一起打包到统一的纹理当中，然后使用网格工具上传到GPU例如Nine Patches，这样每当需要绘制这些资源的时候，他们已经存储在GPU中了，大大加快了这类视图的显示，
## 5. Overdraw
Overdraw（过度绘制）意指屏幕上某个像素在同一时间内被绘制了多次，在多层重叠的UI结构里面。如果不可见的UI也在做绘制操作，会导致某些像素区域被绘制了多次，这样会造成大量的CPU和GPU以及资源

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzBlNzI3ZWJmLnBuZw)

当设计更华丽的效果的时候，我们就很容易采用复杂的多层次重叠视图。这很容易导致大量的性能问题，为了获得最佳性能，我们必须减少overdraw的发生。

我们可通过以下步骤查看Overdraw情况。

    1. Go to Settings > Developer Options 
    2. Turn on Debug GPU Overdraw. 
    3. In the popup, choose Show overdraw areas.

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzEwYzNlYjJhLnBuZw)

蓝色，淡绿，淡红，深红代表了4种不同程度的Overdraw情况，我们的目标就是尽量减少红色Overdraw，看到更多的蓝色区域。

## 6. 修复overdraw

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzExMGEzMjMwLnBuZw)

通过查看xml文件我们可以看到有好几处非必需的background，移除非必需的background，可以显著减少布局的过度绘制

其中一个比较有意思的地方是：针对ListView中的ImageView的设置，在getView的代码里面，判断是否获取到对应的Bitmap，在获取到图像之后，把ImageView的Background设置为Transparent，只有当图像没有获取到的时候才设置对应的Background占位图片，这样可以避免因为给Avatar设置背景图而导致的过度渲染。

总结一下，优化的步骤如下：
 - 移除window默认的background
 - 移除非必需的background
 - 按需显示占位背景图片

## 7. 关于overdraw的另一件事

对于有些过度绘制对于运行性能可能是必要的，也可能是可以接受的，比如ActionBar

## 8. clipRect 和 quickReject

Android知道，过度绘制是个麻烦，所以Android会设法避免在最终图片中不显示UI组件，这种优化类型叫做clipping，他对UI性能非常重要。

但是不幸的是这一技术无法应对复杂的自定义view，系统无法检测onDraw具体会执行什么操作。

针对这一情况，我们可以使用具有一些特别方法的Canva类识别被遮挡不需要绘制的部分，最有用的方法是Canvas.clipRect(),他可以帮助你识别给定View的图片边界，边界区域之外的任何绘制会被忽视，我们可以使用Canvas.quickReject方法，判断给定区域是否完全在剪辑矩形之外。

## 9. 练习使用

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzEyMzc2YjRhLnBuZw)

这是一个呈现多张重叠的卡片View，下面是onDraw方法

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE0NTczNTdmLnBuZw)

打开开发者选项后显示过度渲染。

## 10. 使用Canvas修复

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE1MDIxODQ3LnBuZw)

上面代码显示了如何解决自定义View的过度绘制问题。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE1Njg2MWZhLnBuZw)

优化后的效果

## 11. Layouts, Invalidations and Perf

为了在屏幕上绘制某个东西，Android通常将高级xml文件转换为GPU能够识别的对象，然后显示在屏幕上，这个操作时在DisplayList的帮助先完成的，

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlL0YzRUY1NUU0MjgxNDQxMkJCRDdCRDY1MUFFQ0YxNDBGLzM0NDk)


DisplayList 持有所有要交给GPU绘制到屏幕上的数据信息、包含GPU要绘制的全部对象的信息列表、和执行绘制操作的OpenGL命令列表。

在某个view第一次需要被渲染时，DisplayList会因此被创建，当这个View要显示到屏幕上时，我们将绘制指令提交给GPU来执行DisplayList。

当下次渲染这个View时，比如说位置发生了变化，我们仅需要执行DisplayList就可以了，但是如果修改了View的某些可见组件的内容，那么之前的DisplayList就无法继续使用了，这时我们要重建一个DisplayList，重新执行渲染指令并更新在屏幕上。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzBFMkM0QjhCOTE5QjRGNzY4MTAyMTQyQTAyMjczQjBCLzM0NjQ)


需要注意的是，任何时候View的绘制内容发生变化时，都会重新创建DisplayList，更新到屏幕上等一系列操作。这个表现性能取决于你的View的复杂程度、View的状态变化以及渲染管道的执行性能。

举个例子，如果某个Button的大小需要增大到目前的两倍，在增大之前，需要通过父View重新计算并摆放其他子View的位置，修改View的大小会触发HierarchyVIew的重新计算大小的操作，如果是修改View的位置，则会重新计算其他View的位置，如果布局很复杂，这就会很容易导致性能问题。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE2YTVmYjk5LnBuZw)

## 12. Hierarchy Viewer

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9mMTJiZmE1NDVlOTliODViYTE1MDhlNTcyYjcxZTY2Yi94bWxub3RlLzdFRjgxQTgzM0FGNzRGRDZCNkUyRDhCOTJGNjlGRTI4LzM1MTE)

Hierarchy Viewer可以显示布局的层次关系，我们可以通过红黄绿三种不同的颜色来区分布局的measure、layout、executive的相对性能如何。

现在Google已经不推荐使用层次结构查看器。如果使用的是Android Studio 3.1或更高版本，推荐使用Layout Inspector在运行时检查应用程序的视图层次结构。

[https://developer.android.com/studio/profile/hierarchy-viewer](https://developer.android.com/studio/profile/hierarchy-viewer)

## 13. 嵌套层次结构和性能

提升布局性能的关键点是尽量保持布局层级的扁平化，避免出现重复的嵌套布局。例如下面的例子，有2行显示相同内容的视图，分别用两种不同的写法来实现，他们有着不同的层级。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE2ZGUzNmFkLnBuZw)
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE3OGEwZjQzLnBuZw)

下图显示了使用2种不同的写法，在Hierarchy Viewer上呈现出来的性能测试差异。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE4M2VlZGE3LnBuZw)

## 14.  优化布局

下图举例演示了如何优化ListItem的布局，其主要通过RelativeLayout替代旧方案中的嵌套LinearLayout来优化布局。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ib3gua2FuY2xvdWQuY24vMjAxNS0wOC0yMV81NWQ2YzE4OWVjYTJjLnBuZw)