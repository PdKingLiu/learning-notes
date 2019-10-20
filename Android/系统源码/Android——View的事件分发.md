[TOC]

# View的事件分发机制
> 参考《Android进阶之光》
## 解析Activity构成
点击事件用**MotionEvent**表示。

当一个点击事件产生后，**事件最先传递个Activity**。首先看一下setContentView()方法。

```java
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

调用了getWindow()对应的方法

```java
    public Window getWindow() {
        return mWindow;
    }
```

在Activity的attach()方法中发现了mWindow

```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        ······
    }
```

原来**mWindow指的就是PhoneWindow**，PhoneWindow是继承抽象类Window的，这样就知道了**getWindow()得到的是一个PhoneWindow**，因为Activity中setContentView()方法调用的是 getWindow().setContentView(layoutResID)。

```java
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }

```

调用了**installDecor()** 方法

```java
    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ······
```

跟进注释1处的generateDecor()方法

```java
    protected DecorView generateDecor(int featureId) {
        return new DecorView(context, featureId, this, getAttributes());
    }
```

这里**创建了一个DecorView**，这个DecorView就**是Activity中的根View**。接着查看DecorView的源码，发现**DecorView是PhoneWindow类的内部类**，并且继承了FrameLayout。我们再看看generateLayout(mDecor)做了什么：

```java
        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
            setCloseOnSwipeEnabled(true);
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            ······
```

PhoneWindow的generateLayout()方法比较长，这里只截取了一小部分关键的代码，其**主要内容就是根据不同的情况加载不同的布局给layoutResource**。现在查看上面代码注释1处的布局R.layout.screen_title，这个文件在frameworks，它的代码如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191020165911617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
查看源码，可得到以下结论：

- 上面的**ViewStub是用来显示Actionbar的**。下面的**两个FrameLayout：一个是title，用来显示标题；另一个是content，用来显示内容**。看到上面的源码，大家就知道了**一个Activity包含一个Window对象，这个对象是由PhoneWindow来实现的**。
- **PhoneWindow将DecorView作为整个应用窗口的根 View**，而**这个 DecorView 又将屏幕划分为两个区域：一个是 TitleView，另一个是ContentView**，而我们**平常做应用所写的布局正是展示在ContentView中的**。
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019163613171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 源码解析View的事件分发机制
### 概述
- 当我们点击屏幕时，就产生了点击事件，这个**事件被封装成了一个类：MotionEvent**。
- MotionEvent产生后，那么系统就会**将这个MotionEvent传递给View的层级**，**MotionEvent在View中的层级传递过程就是点击事件分发**。

点击事件有3个重要的方法：

- dispatchTouchEvent(MotionEvent ev)——**用来进行事件的分发**。
- onInterceptTouchEvent(MotionEvent ev)——**用来进行事件的拦截**，**在dispatchTouchEvent()中调用**，需要注意的是View没有提供该方法。
- onTouchEvent(MotionEvent ev)——**用来处理点击事件**，在dispatchTouchEvent()方法中进行调用。

### 分发机制
- 当点击事件产生后，事件**首先会传递给当前的 Activity**，这会调用 Activity 的**dispatchTouchEvent()方法**，具体工作都是**交由Activity中的PhoneWindow来完成**的，然后**PhoneWindow再把事件处理工作交给DecorView**，之后再由**DecorView将事件处理工作交给根ViewGroup**

**ViewGroup的dispatchTouchEvent()**

```java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }


```

- 首先**判断事件是否为DOWN事件**，如果是，则进行初始化，resetTouchState方法中会把mFirstTouchTarget的值置为null，为什么要进行初始化呢？原因就是**一个完整的事件序列是以DOWN开始，以UP结束的**。所以如果是DOWN事件，那么**说明这是一个新的事件序列**，故而需要初始化之前的状态。
- 代码注释1处的条件如果满足，则执行下面的句子，**mFirstTouchTarget 的意义是：当前 ViewGroup 是否拦截了事件，如果拦截了，mFirstTouchTarget=null；如果没有拦截并交由子View来处理，则mFirstTouchTarget！=null**。**假设当前的 ViewGroup 拦截了此事件**，mFirstTouchTarget！=null 则为 false，**如果这时触发ACTION_DOWN 事件**，**则会执 行 onInterceptTouchEvent(ev) 方法**；**如 果 触发的 是ACTION_MOVE、ACTION_UP事件，则不再执行onInterceptTouchEvent(ev)方法**，**而是直接设置intercepted=true，此后的一个事件序列均由这个ViewGroup处理**。
- 注释2 处出现了一个FLAG_DISALLOW_INTERCEPT 标志位，它主要是**禁止ViewGroup 拦截除了DOWN之外的事件**，**一般通过子View的requestDisallowInterceptTouchEvent来设置**

总结一下：

**当ViewGroup要拦截事件的时候，那么后续的事件序列都将交给它处理，而不用再调用onInterceptTouchEvent()方法了。所以，onInterceptTouchEvent()方法并不是每次事件都会调用的。**

**onInterceptTouchEvent()方法：**

```java
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```

- onInterceptTouchEvent()方法**默认返回false，不进行拦截**。如果**想要让ViewGroup拦截事件，那么应该在自定义的ViewGroup中重写这个方法**。

**dispatchTouchEvent()方法剩余的部分**

```java
  final View[] children = mChildren;
  for (int i = childrenCount - 1; i >= 0; i--) {
      final int childIndex = getAndVerifyPreorderedIndex(
              childrenCount, i, customOrder);
      final View child = getAndVerifyPreorderedView(
              preorderedList, children, childIndex);

      // If there is a view that has accessibility focus we want it
      // to get the event first and if not handled we will perform a
      // normal dispatch. We may do a double iteration but this is
      // safer given the timeframe.
      if (childWithAccessibilityFocus != null) {
          if (childWithAccessibilityFocus != child) {
              continue;
          }
          childWithAccessibilityFocus = null;
          i = childrenCount - 1;
      }

      if (!canViewReceivePointerEvents(child)
              || !isTransformedTouchPointInView(x, y, child, null)) {
          ev.setTargetAccessibilityFocus(false);
          continue;
      }

      newTouchTarget = getTouchTarget(child);
      if (newTouchTarget != null) {
          // Child is already receiving touch within its bounds.
          // Give it the new pointer in addition to the ones it is handling.
          newTouchTarget.pointerIdBits |= idBitsToAssign;
          break;
      }

      resetCancelNextUpFlag(child);
      if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
          // Child wants to receive touch within its bounds.
          mLastTouchDownTime = ev.getDownTime();
          if (preorderedList != null) {
              // childIndex points into presorted list, find original index
              for (int j = 0; j < childrenCount; j++) {
                  if (children[childIndex] == mChildren[j]) {
                      mLastTouchDownIndex = j;
                      break;
                  }
              }
          } else {
              mLastTouchDownIndex = childIndex;
          }
          mLastTouchDownX = ev.getX();
          mLastTouchDownY = ev.getY();
          newTouchTarget = addTouchTarget(child, idBitsToAssign);
          alreadyDispatchedToNewTouchTarget = true;
          break;
      }

      // The accessibility focus didn't handle the event, so clear
      // the flag and do a normal dispatch to all children.
      ev.setTargetAccessibilityFocus(false);
  }
```

- 注释1处我们看到了for循环。首先**遍历ViewGroup的子元素，判断子元素是否能够接收到点击事件，如果子元素能够接收到点击事件，则交由子元素来处理**。需要注意这个**for循环是倒序遍历**的，即从最上层的子View开始往内层遍历。
- 注释2处的代码，其意思是**判断触摸点位置是否在子View的范围内或者子View是否在播放动画**。如果均**不符合则执行continue语句**，表示这个子View不符合条件，开始遍历下一个子View。

注释3处的**dispatchTransformedTouchEvent**方法

```java
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
```


这里会**调用子view的dispatchTouchEvent(event)，如果子view的dispatchTouchEvent方法返回为true，dispatchTransformedTouchEvent方法返回为true，for循环结束**。

当子view的dispatchTouchEvent方法**返回为false，然后这个循环不会结束会接着调用dispatchTransformedTouchEvent方法**。

```java
   handled = dispatchTransformedTouchEvent(ev, canceled, null,
           TouchTarget.ALL_POINTER_IDS);
```
此时传入的child为null，会调用
	
```java
handled = super.dispatchTouchEvent(event);
```

也就是**View的dispatchTouchEvent方法(如下)**，也就意味值子View不处理，我自己处理，如果我自己的方法还没有处理返回给我的父亲为false，那么我的父亲和我一样也会调用他的super.dispatchTouchEvent(event); 接着调用onTouch或onTouchEvent

```java
    public boolean dispatchTouchEvent(MotionEvent event) { 
        boolean result = false;
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
```

- 如果**OnTouchListener不为null并且onTouch方法返回true**，则**表示事件被消费**，**就不会执行onTouchEvent(event)，否则就会执行onTouchEvent(event)。**
- 可以看出 **OnTouchListener中的onTouch()方法优先级要高于onTouchEvent(event)** 方法。

onTouchEvent()方法的部分源码：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019220722611.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019220732397.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

- 从上面的代码可以看到，只要**View的CLICKABLE和LONG_CLICKABLE有一个为true**，那么**onTouchEvent()就会返回true消耗这个事件**。CLICKABLE和LONG_CLICKABLE代表View可以被点击和长按点击，可以通过View的setClickable和setLongClickable方法来设置，也可以通过View的setOnClickListenter和setOnLongClickListener来设置，它们会自动将View设置为CLICKABLE和LONG_CLICKABLE。

接着在ACTION_UP事件中会调用performClick()方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019220918131.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

- 上面代码注释 1 处可以看出，**如果 View 设置了点击事件 OnClickListener，那么它的onClick()方法就会被执行**。View事件分发机制的源码分析就讲到这里了，接下来介绍点击事件分发的传递规则。

### 事件分发的传递规则

由前面事件分发机制的源码分析可知点击事件分发的这3个重要方法的关系，用伪代码来简单表示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019221108173.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
 **onInterceptTouchEvent方法和onTouchEvent方法都在dispatchTouchEvent方法中调用**。现在我们根据这段伪代码来分析一下点击事件分发的传递规则。

**点击事件由上而下的传递规则**

- 点击事件产生后会由 Activity 来处理，传递给PhoneWindow，再传递给DecorView，最后传递给顶层的ViewGroup。一般在事件传递中只考虑 ViewGroup 的 onInterceptTouchEvent 方法，因为一般情况下我们不会重写 dispatchTouch-Event()方法。对于根ViewGroup，点击事件首先传递给它的dispatchTouchEvent()方法，如果该ViewGroup的onInterceptTouchEvent()方法返回true，则表示它要拦截这个事件，这个事件就会交给它的onTouchEvent()方法处理；如果onInterceptTouchEvent()方法返回false，则表示它不拦截这个事件，则这个事件会交给它的子元素的dispatchTouchEvent()来处理，如此反复下去。如果传递给底层的View，View是没有子View的，就会调用View的dispatchTouchEvent()方法，一般情况下最终会调用View的onTouchEvent()方法。

**点击事件由下而上的传递**

- 当点击事件传给底层的 View 时，如果其onTouchEvent()方法返回true，则事件由底层的View消耗并处理；如果返回false则表示该View不做处理，则传递给父View的onTouchEvent()处理；如果父View的onTouchEvent()仍旧返回false，则继续传递给该父View的父View处理，如此反复下去。