[toc]

# Android之自定义View
## 继承特定View控件
这种自定义 View 在系统控件的基础上进行拓展，一般是添加新的功能或者修改显示的效果，一般情况下在onDraw()方法中进行处理。

我们写一个自定义View，继承自TextView：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020205118113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
- 这个自定义View继承了TextView，并且在onDraw()方法中画了一条红色的横线。

## 继承View
与上面的继承系统控件的自定义View不同，继承View的自定义View实现起来要稍微复杂一些。其**不只是要实现onDraw()方法**，而且**在实现过程中还要考虑到wrap_content属性以及padding 属性的设置**；为了方便配置自己的自定义 View，还会**对外提供自定义的属性**。另外，如果**要改变触控的逻辑，还要重写 onTouchEvent()等触控事件的方法**。按照上面的例子我们再写一个**RectView类继承View来画一个正方形**，代码如下所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021143730194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021143909694.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
效果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144025561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 处理padding

只需要在onDraw()方法中稍加修改，在绘制正方形的时候考虑padding属性即可，代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144144574.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
运行后效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144223311.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 处理wrap_content属性
为何会产生这种情况，详细内容参考View的工作流程。

这种情况需要我们在onMeasure方法中指定一个默认的宽和高，在设置wrap_content属性时设置此默认的宽和高就可以了：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144445803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 自定义属性
首先在values目录下创建 attrs.xml：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144800656.png)
这个配置文件定义了名为RectView的自定义属性组合。我们定义了rect_color属性，它的格式为color。接下来在RectView的构造方法中解析自定义属性的值，如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021144830635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
用 TypedArray 来获取自定义的属性集 R.styleable.RectView，这个 RectView 就是我们在XML中定义的name的值，然后通过TypedArray的getColor方法来获取自定义的属性值。

## View滑动冲突
在界面中只要内外两层同时可以滑动，这个时候就会产生滑动冲突。

### 常见的滑动冲突
- 场景一： 外部滑动方向和内部方向不一致
- 场景二：外部滑动方向和内部滑动方向一致
- 场景三：上面两种情况的嵌套

先说场景1，主要是将ViewPager和Fragment配合使用所组成的页面滑动效果，
主流应用几乎都会使用这个效果。在这种效果中，可以通过左右滑动来切换页面，而
每个页面内部往往又是一个ListView. 本来这种情况下是有滑动冲突的，但是
ViewPager内部处理了这种滑动冲突，因此采用ViewPager时我们无须关注这个问题，
如果我们采用的不是ViewPager而是ScrolIView等，那就必须手动处理滑动冲突了，
否则造成的后果就是内外两层只能有一层能够滑动，这是因为两者之间的滑动事件有冲突。除了这种典型情况外，还存在其他情况，比如外部上下滑动、内部左右滑动等，
但是它们属于同一类滑动冲突。

再说场景2,这种情况就稍微复杂些， 当内外两层都在同一个方向可以滑动的时候，
显然存在逻辑问题。因为当手指开始滑动的时候，系统无法知道用户到底是想让哪- -层滑
动，所以当手指滑动的时候就会出现问题，要么只有一层能滑动，要么就是内外两层都滑
动得很卡顿。在实际的开发中，这种场景主要是指内外两层同时能上下滑动或者内外两层
同时能左右滑动。


最后说下场景3，场景3是场景1和场景2两种情况的嵌套，因此场景3的滑动冲突
看起来就更加复杂了。比如在许多应用中会有这么一个效果:内层有一个场景1中的滑动
效果，然后外层又有一个场景2中的滑动效果。具体说就是，外部有一个SlideMenu效果，
然后内部有一个ViewPager, ViewPager 的每-一个 页面中又是一个ListView。虽然说场景3
的滑动冲突看起来更复杂，但是它是几个单- -的滑动冲突的叠加，因此只需要分别处理内
层和中层、中层和外层之间的滑动冲突即可，而具体的处理方法其实是和场景1.场景2
相同的。

### 处理规则
对于场景1,它的处理规则是:当用户左右滑动时，需要让外部的View
拦截点击事件，当用户上下滑动时，需要让内部View拦截点击事件。这个时候我们就可以
根据它们的特征来解决滑动冲突，具体来说是:根据滑动是水平滑动还是竖直滑动来判断
到底由谁来拦截事件，如图，根据滑动过程中两个点之间的坐标就可以得出到底
是水平滑动还是竖直滑动。如何根据坐标来得到滑动的方向呢?这个很简单，有很多可以
参考，比如可以依据滑动路径和水平方向所形成的夹角，也可以依据水平方向和竖直方向
上的距离差来判断，某些特殊时候还可以依据水平和竖直方向的速度差来做判断。这里我
们可以通过水平和竖直方向的距离差来判断，比如竖直方向滑动的距离大就判断为竖直滑
动，否则判断为水平滑动。根据这个规则就可以进行下-步的解决方法制定了。


![在这里插入图片描述](https://img-blog.csdnimg.cn/20191021182007945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
对于场景2来说，比较特殊，它无法根据滑动的角度、距离差以及速度差来做判断，
但是这个时候- -般 都能在业务上找到突破点，比如业务上有规定:当处于某种状态时需要
外部Vicw响应用户的滑动，而处于另外种状态时则需 要内部View来响应View的滑动，
根据这种业务上的需求我们也能得出相应的处理规则，有了处理规则同样可以进行下一步
处理。

对于场景3来说，它的滑动规则就更复杂了，和场景2一样，它也无法直接根据滑动
的角度、距离差以及速度差来做判断，同样还是只能从业务上找到突破点，具体方法和场
景2一样，都是从业务的需求上得出相应的处理规则。
