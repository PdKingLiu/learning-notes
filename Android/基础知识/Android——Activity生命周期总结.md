[toc]

# 各个生命周期
- **onCreate——表示Activity正在创建**，是生命周期的第一个方法，我们可以在这个方法中做一些初始化工作。
- **onRestart——表示Activity正在重新启动**，一般情况下，当当前的Activity从不可见变为可见状态时，onRestart就会被调用。这种行为一般是用户行为所导致的，比如用户点击home键切换到桌面或者用户打开了一个新的Activity，这时当前Activity就会暂停。也就是onPause和onStop执行了，接着用户又回到了这个Activity，就会出现这种情况。
- **onStart——表示Activity正在被启动，即将开始**，这时Activity已经可见了，但是**还没有出现在前台**，还无法和用户交互。可以理解为Activity已经显示出来了，但是我们还看不到。
- **onResume——表示Activity已经可见了，并且出现在前台并开始活动**。注意这个和onStart的区别：onStart和onResume都表示Activity已经可见，但是onStart的时候Activity还在后台，onResume的时候Activity才显示到前台。
- **onPause——表示Activity正在停止，正常情况下，onStop就会被调用。**在特殊情况下，如果这个时候快速的再回到当前Activity，那么onResume就会被调用。任玉刚猜测这种情况属于几段情况，用户操作很难重现这一场景，此时可以做一些存储数据、停止动画等工作，不能太耗时，因为会影响到新Activity的显示。
- **onStop——表示Activity即将停止**，可以做一些稍微重量级的回收工作，但是不能太耗时
- **onDestroy——表示Activity即将销毁**，这是Activity声明周期中最后一个回调。我们可以做一些回收工作和最终的资源释放。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031194400540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

# 情景分析
- 对于一个**第一次启动的Activity**，调用如下：onCreate——onStart——onResume
- 当**打开新Activity**、**切换到桌面**，调用如下：onPause——onStop，当Activity采用了透明主题，那么就不会回调onStop。
- **再次回到原Activity**时，调用如下：onRestart——onStart——onResume
- **点击返回键时**，调用如下：onPause——onStop——onDestroy
- Activity回收重建时的生命周期和过程一样。

--------
- 对于整个生命周期：onCreate和onDestroy是配对的，分别对应创建和销毁。
- 从Activity可见来说，onStart和onStop是配对的。
- 从Activity是否在前台来说，onResume和onPause是配对的。

# 两个问题
**问题1: onStart 和onResume  onPause 和onStop从描述上来看差不多，对我们来说有
什么实质的不同呢?**

从实际使用过程来说，onStart和onResume、onPause和onStop看起来的确差不多。

系统为什么还要提供看起来重复的接口呢？根据上面的分析，我们知道这两个回调表示不同的意义，onStart和onStop是从Activity是否可见这个角度回调的，onResume和onPause是从Activity是否位于前台这个角度回调的。

**问题2:假设当前Activity为A,如果这时用户打开一个新Activity B,那么B的onResume
和A的onPause哪个先执行呢?**

对于这个问题可以直接回答：先执行A的onPause，在执行B的onResume。

这个就要追溯到源码了，有点啰嗦，这里就直接给出结论。

**源码中在新Activity启动之前，栈顶的Activity需要先onPause后，新Activity才能启动。**

# 异常情况下的周期
## 情况1
**资源相关配置发生改变导致Activity被杀死并重新创建**

默认情况下，如果我们的Activity不做特殊处理，那么当系统配置发生改变后，Activity就会被销毁并重新创建。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031210949541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
当系统配置发生改变后，Activity 会被销毁，其onPause、onStop. onDestroy 均会被调
用，同时由于Activity 是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前
Activity的状态。

这个方法的调用时机是在onStop之前，它和onPause没有既定的时序关系，它既可能在onPause 之前调用，也可能在onPause 之后调用。需要强调的一点是，这个方法只会出现在Activity 被异常终止的情况下，正常情况下系统不会回调这个方法。

当Activity被重新创建后，系统会调用onRestoreInstanceState, 并且把Activity 销毁时onSaveInstanceState方法所保存的Bundle对象作为参数同时传递给onRestorelnstanceState
和onCreate方法。因此,我们可以通过onRestoreInstanceState和onCreate方法来判断Activity
是否被重建了，如果被重建了，那么我们就可以取出之前保存的数据并恢复，从时序上来
说，onRestoreInstanceState 的调用时机在onStart 之后。

另外，我们要知道，在onSaveInstanceState和onRestoreInstanceState方法中，系统自动
为我们做了一-定的恢复工作。当Activity 在异常情况下需要重新创建时，系统会默认为我
们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框
中用户输入的数据、ListView 滚动的位置等。

## 情况2
**资源内存不足导致低优先级Activity被杀死**

Activity按照优先级从高到低可以分为如下三种：

- 前台Activity——正在交互的Activity
- 可见但非前台——比如Activity中弹出了一个对话框，导致不能与其交互
- 后台Activity——已经被暂停的Activity，比如执行了onStop

当系统内存不足时，系统就会按照上述优先级去杀死目标Activity 所在的进程，并在
后续通过onSaveInstanceState和onRestoreInstanceState来存储和恢复数数据。

一些后台工作不适合脱离四大组件而独自运行在后台中，这样进程很容易被杀死。比较好的方法是将后台工作放入Service中从而保证进程有一定的优先级， 这样就不会轻易地被系统杀死。

我们添加这个属性可以避免在屏幕方向发生改变的时候进行重建。也可以设置其他选项。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031223902841.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031224017935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
加入上面的代码后，系统属性更改后Activity便不会重启，但是会回调onConfigurationChanged方法，这个时候我们就可以做一些自己的特殊处理。
