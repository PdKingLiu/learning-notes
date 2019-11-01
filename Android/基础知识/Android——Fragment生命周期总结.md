[toc]

# 概述
Fragment的主要功能就是创建一个View，并且有一个生命周期来管理这个View。

Fragment的生命周期和Activity的生命周期类似，都有一些回调方法。

# 各个生命周期
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191101114934853.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
左侧是Activity的生命周期，右侧对应这个状态下执行Fragment的生命周期方法。

Fragment有的生命周期与Activity生命周期名字都是一样的，对应的功能也类似。

只不过在Created状态和Destroy状态多了一些方法。

**onAttach()**

当Fragment和它所在的Activity关联起来的时候调用。

**onCreateView()**

当需要创建一个与Fragment关联的View的时候会调用，这个方法会返回一个View。

inflate的三个参数含义：


**onDestroyView()**

当与Fragment关联的那个View与Fragment解除的关联，从View树中移除的时候调用。下次Fragment需要显示一个View的时候会重现调用onCreateView。

**onDetach()**

当Fragment与之前onAttach()关联的那个Activity解除关系的时候调用。

**和Activity类似，Fragment可以停留的三个状态：**

**Resumed**

Fragment的运行状态，此时的Fragment处于运行状态，可以用户进行交互。类似Activity的Resumed

**Paused**

**有其他的Activity获取焦点**，前台运行，Fragment所在的Activity失去焦点，部分显示在前台Activity下面。

**Stopped**

Fragment不在可见。**此时的Fragment所在的Activity可能已经stopped了**，**或者fragment从Activity中移除到Fragment的退回栈中**。

一个stopped状态的Fragment没有销毁，还在存活状态，他的状态被系统保存，只是不可见、不可交互，此时很可能被系统回收。

可以利用Bundle来记录Fragment的状态，当Activity被销毁需要记录Fragment状态，并且在Activity重新创建的时候恢复Fragment的状态。可以保存Fragment的状态在Fragment的nSaveInstanceState()回调方法中，在onCteat()、onCreatView()或者onActivityCreated()方法中进行恢复。


在生命周期中Activity与Fragment的最大不同之处是回退栈是相互独立的，Activity的回退栈是系统来管理的，Fragment的回退栈是被宿主Activity来管理的，也就是说我们可以自己来进行控制(调用addToBackStack()).

Fragment生命周期也没有啥多说的哈哈，导致这篇文章很短。😁

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20191101122946339.png)