[toc]

# View的工作流程
View的工作流程，指的就是measure、layout和draw。其中，measure用来测量View的宽和高，layout用来确定View的位置，draw则用来绘制View。

## View的工作流程入口
## DecorView被加载到Window中
当DecorView创建完毕，要加载到Window中时，我们需要先了解一下Activity的创建过程。当我们调用Activity的startActivity方法时，最终是调用ActivityThread的handleLaunchActivity方法来创建Activity的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174326745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

调用 performLaunchActivity 方法来创建 Activity，在这里面会调用到Activity的onCreate方法，从而完成DecorView的创建。

**handleResumeActivity方法**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174356534.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

注释1处的performResumeActivity方法中会调用Activity的onResume方法。接着往下看，注释2处得到了DecorView。注释3处得到了WindowManager，WindowManager是一个接口并且继承了接口ViewManager。在注释4处调用WindowManager的addView方法，WindowManager 的实现类是WindowManagerImpl，所以实际调用的是 WindowManagerImpl 的addView方法。具体代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174430704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

在 WindowManagerImpl 的 addView 方法中，又调用了 WindowManagerGlobal 的 addView方法，代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174450728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

注释1处创建了ViewRootImpl实例，在注释2处调用了ViewRootImpl的setView方法并将DecorView作为参数传进去，这样就把DecorView加载到了Window中。当然界面仍不会显示出什么来，因为View的工作流程还没有执行完，还需要经过measure、layout以及draw才会把View绘制出来。

将 DecorView 加载到 Window 中，是通过 ViewRootImpl 的 setView 方法。ViewRootImpl还有一个方法PerformTraveals，这个方法使得ViewTree开始View的工作流程，代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174509973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

主要执行了3个方法，分别是performMeasure、performLayout和performDraw，在其方法的内部又会分别调用View的measure、layout和draw方法。需要注意的是，performMeasure方法中需要传入两个参数，分别是 childWidthMeasureSpec 和 childHeightMeasureSpec。要了解这两个参数，需要了解MeasureSpec。

## MeasureSpec
MeasureSpec是View的内部类，其封装了一个View的规格尺寸，包括View的宽和高的信息，它的作用是在Measure流程中，系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，然后在onMeasure方法中根据这个MeasureSpec来确定View的宽和高。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174554426.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174615150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

从MeasureSpec的常量可以看出，它代表了32位的int值，其中高2位代表了SpecMode，低30位则代表SpecSize。SpecMode指的是测量模式，SpecSize指的是测量大小。SpecMode有3种模式，如下所示。
- UNSPECIFIED：未指定模式，View想多大就多大，父容器不做限制，一般用于系统内部的测量。
- AT_MOST：最大模式，对应于wrap_comtent属性，子View的最终大小是父View指定的SpecSize值，并且子View的大小不能大于这个值。
- EXACTLY：精确模式，对应于 match_parent 属性和具体的数值，父容器测量出 View所需要的大小，也就是SpecSize的值。

对于每一个View，都持有一个MeasureSpec，而该MeasureSpec则保存了该View的尺寸规格。在View的测量流程中，通过makeMeasureSpec来保存宽和高的信息。通过getMode或getSize得到模式和宽、高。MeasureSpec是受自身LayoutParams和父容器的MeasureSpec共同影响的。作为顶层View的DecorView来说，其并没有父容器，那么它的MeasureSpec是如何得来的呢？为了解决这个疑问，我们再回到ViewRootImpl的PerformTraveals方法，如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174642680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
注释 1 处调用了 getRootMeasureSpec(mWidth，lp.width)方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020174911641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
getRootMeasureSpec方法的第一个参数windowSize指的是窗口的尺寸，所以对于DecorView来说，它的MeasureSpec由自身的LayoutParams和窗口的尺寸决定，这一点和普通View是不同的。

注释2处的performMeasure方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020180025830.png)
##  View的measure流程
measure 用来测量 View 的宽和高，它的流程分为 View 的 measure 流程和 ViewGroup 的measure流程，只不过ViewGroup的measure流程除了要完成自己的测量，还要遍历地调用子元素的measure()方法。
### View的measure流程
View的onMeasure方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020181949274.png)
setMeasuredDimension方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020182010499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
getDefaultSize()方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020182117627.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
根据不同的SpecMode值来返回不同的result值，也就是SpecSize。在AT_MOST和EXACTLY模式下，都返回SpecSize这个值，即View在这两种模式下的测量宽和高直接取决于SpecSize。也就是说，对于一个直接继承自View的自定义View来说，它的wrap_content 和 match_parent 属性的效果是一样的。

对于一个直接继承自View的自定义View来说，它的wrap_content 和 match_parent 属性的效果是一样的。因此如果要实现自定义 View 的wrap_content，则要重写onMeasure方法，并对自定义View的wrap_content属性进行处理。

而在 UNSPECIFIED 模式下返回的是 getDefaultSize 方法的第一个参数 size 的值，size 的值从onMeasure方法来看是getSuggestedMinimumWidth方法或者getSuggestedMinimumHeight方法得到的。

getSuggestedMinimumWidth方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020182435997.png)

如果 View 没有设置背景，则取值为 mMinWidth，mMinWidth 是可以设置的，它对应于Android：minWidth这个属性设置的值或者View的setMinimumWidth的值；如果不指定的话，则默认为0。

总结一下：getSuggestedMinimumWidth方法就是：如果View没有设置背景，则返回mMinWidth；如果设置了背景，就返回mMinWidth和Drawable的最小宽度之间的最大值。

### ViewGroup的measure流程
对于ViewGroup，它不只要测量自身，还要遍历地调用子元素的measure()方法。ViewGroup中没有定义onMeasure()方法，但却定义了measureChildren()方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020183000477.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
遍历子元素并调用measureChild方法，measureChild方法如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020183402195.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
调用 child.getLayoutParams()方法来获得子元素的 LayoutParams 属性，获取子元素的MeasureSpec 并调用子元素的 measure()方法进行测量。

getChildMeasureSpec()方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020183507852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 根据父容器的MeasureSpec模式再结合子元素的LayoutParams属性来得出的子元素的 MeasureSpec 属性。
- 需要注意的是，如果父容器的 MeasureSpec 属性为AT_MOST，子元素的LayoutParams属性为WRAP_CONTENT，那根据上面代码注释1处的代码，我们会发现子元素的MeasureSpec属性也为AT_MOST，它的SpecSize值为父容器的SpecSize减去padding的值。换句话说，这和子元素设置LayoutParams属性为MATCH_PARENT效果是一样的。为了解决这个问题，需要在LayoutParams属性为WRAP_CONTENT时指定一下默认的宽和高。

ViewGroup并没有提供onMeasure 方法，而是让其子类来各自实现测量的方法，究其原因就是ViewGroup有不同布局的需要，很难统一。比如ViewGroup的子类LinearLayout的measure流程：

LinearLayout定义mTotalLength用来存储LinearLayout在垂直方向的高度，然后遍历子元素，根据子元素的MeasureSpec模式分别计算每个子元素的高度。如果是WRAP_CONTENT，则将每个子元素的高度和margin垂直高度等值相加并赋值给mTotalLength。当然，最后还要加上垂直方向padding的值。如果布局高度设置为MATCH_PARENT 或者具体数值，则和View的测量方法是一样的。

## View的layout流程
layout方法的作用是确定元素的位置。ViewGroup中的layout方法用来确定子元素的位置，View中的layout方法则用来确定自身的位置。首先我们看看View的layout方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020193955157.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
layout方法的4个参数l、t、r、b分别是View从左、上、右、下相对于其父容器的距离。接着来查看setFrame方法里做了什么，代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020194014706.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- setFrame方法用传进来的l、t、r、b这4个参数分别初始化mLeft、mTop、mRight、mBottom这4个值，这样就确定了该View在父容器中的位置。在调用setFrame方法后，会调用onLayout方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020194718948.png)
onLayout方法是一个空方法，这和onMeasure方法类似。

LinearLayout的onLayout方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020194759698.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020194915929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
方法会遍历子元素并调用setChildFrame方法。其中childTop值是不断累加的，这样子元素才会依次按照垂直方向一个接一个排列下去而不会是重叠的。接着看setChildFrame方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020195026388.png)
在setChildFrame方法中调用子元素的layout方法来确定自己的位置。

## View的draw流程
官方注释清楚地说明了每一步的做法，它们分别是：
- 如果需要，则绘制背景。
- 保存当前canvas层。
- 绘制View的内容。
- 绘制子View。
- 如果需要，则绘制View的褪色边缘，这类似于阴影效果。
- 绘制装饰，比如滚动条。
### 步骤1：绘制背景
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020195708387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
从上面代码注释1 处可看出绘制背景考虑了偏移参数 scrollX 和scrollY。如果有偏移值不为0，则会在偏移后的canvas绘制背景。
### 步骤3：绘制View的内容
步骤3调用了View的onDraw方法。这个方法是一个空实现，因为不同的View有着不同的内容，这需要我们自己去实现，即在自定义View中重写该方法来实现：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020195847134.png)
### 步骤4：绘制子View
步骤4调用了dispatchDraw方法，这个方法也是一个空实现，如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020195916777.png)
ViewGroup重写了这个方法，紧接着看看ViewGroup的dispatchDraw方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020195947593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
源码很长，这里截取了关键的部分，在 dispatchDraw 方法中对子类 View 进行遍历，并调用drawChild方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020200044169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
这里主要调用了View的draw方法，代码如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020200132386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
在上面代码注释1处判断是否有缓存，如果没有则正常绘制，如果有则利用缓存显示。

### 步骤6：绘制装饰
绘制装饰的方法为View的onDrawForeground方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020200252180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
很明显这个方法用于绘制ScrollBar以及其他的装饰，并将它们绘制在视图内容的上层。
