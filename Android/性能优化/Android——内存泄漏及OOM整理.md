[toc]

# 内存泄漏
## 静态变量引用Activity
**静态变量引用Activty对象时，会导致Activty对象所占内存内漏。**

原因：**静态变量是驻扎在JVM的方法区，因此，静态变量引用的对象是不会被GC回收的，因为它们所引用的对象本身就是GC ROOT，即最终导致Activity对象不被回收，从而也就造成内存泄漏。**

常见的错误用法：

**将Context或Activity赋值给某个静态变量**

解决：
- 去掉 static 关键字，使用别的方法来实现想要的功能。
- 在 onDestroy 方法中置空 Activity 静态引用
- 也可以使用到软引用解决，确保在 Activity 销毁时，垃圾回收机制可以将其回收。

## static间接修饰Activity
有时，当一个Activity经常启动，但是对应的View读取非常耗时，我们可以通过静态View变量来保持对该Activity的rootView引用。这样就可以不用每次启动Activity都去读取并渲染View了。这确实是一个提高Activity启动速度的好方法！但是要注意，一旦View attach到我们的Window上，就会持有一个Context(即Activity)的引用。而我们的View有事一个静态变量，所以导致Activity不被回收。当然了，也不是说不能使用静态View，但是在使用静态View时，需要确保在资源回收时，将静态View detach掉。 

常见问题
```java
public class LoadingDialog extends Dialog {
    private static LoadingDialog mDialog;
    private TextView mText;
}
```
这里 static 虽然没有直接修饰 TextView（拥有 Context 引用），但是修饰了 mDialog 成员变量，mDialog 是 一个 LoadingDialog 对象， LoadingDialog 对象 包含一个 TextView 类型的成员变量，所以 mText 变量的生命周期也是全局的，和应用一样。这样，mText 持有的 Context 对象销毁时，没有 GC 回收，导致内存泄露。 

解决：
- 不使用static修饰
- 在适当的地方置空 mDialog

## 单例引用Context

```java
public class AppManager {
    private static AppManager instance;
    private Context context;

    private AppManager(Context context) {
        this.context = context;
    }

    public static AppManager getInstance(Context context) {
        if (instance == null) {
            instance = new AppManager(context);
        }
        return instance;
    }
} 
```
AppManager是一个单例类，他需要Context作为成员变量，Context 如果是 Activity Context 的话，必然会引起内存泄漏。

解决
- 使用 Applicaion Context 代替 Activity Context 
- 在调用的地方使用弱引用

## 匿名内部类执行耗时任务

```java
public class MainActivity extends Activity {  
    @Override
    protected void onCreate(Bundle savedInstanceState) {    
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        test();
    }  
    public void test() {    
        new Thread(new Runnable() {     
            @Override
            public void run() {        
                while (true) {          
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
} 
```
原因
- 非静态内部类或匿名类会拥有所在外部类的引用，new Thread是匿名内部类，它一直在执行当Activity消耗后，该匿名内部类还在执行任务，导致外部的Activity不能回收。

解决
- 静态匿名内部类，使用static修饰
	```java
	public static void test() {
	    new Thread(new Runnable() {     
	            @Override
	            public void run() {        
	                while (true) {          
	                    try {
	                        Thread.sleep(1000);
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	        }).start();
	} 
	```

## 非静态内部类
如果一个变量，既是静态变量，而且是非静态的内部类对象，那么也会造成内存泄漏。

```java
public class LeakActivity extends AppCompatActivity {
    
    private static Hello sHello;
    
    @Override    
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        
        sHello = new Hello();
    }
    
    public class Hello {}
}
```
这里我们定义的 Hello 虽然是空的，但它是一个非静态的内部类，所以它必然会持有外部类即 LeakActivity.this 引用，导致 sHello 这个静态变量一直持有这个 Activity，Activity 无法被回收。

## Handler引起的内存泄漏
这与非静态匿名内部类执行耗时任务一样，错误代码如下。

```java
public class MainActivity extends Activity {
    private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // ...
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mLeakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                // ...
            }
        }, 1000 * 60 * 10);
        finish();
    }
} 
```
在活动中发送了一个延时十分钟消息的message，handler会将消息放入MessageQueue里面。当活动被finish()时，Message还会继续存在于主线程，Handler是非静态内部类，会持有该活动的引用，所以此时finish()掉的活动就不会回收。

解决方法：

自定义静态Handler

```java
public class MainActivity extends Activity {

    private final MyHandler mHandler = new MyHandler(this);

    private static final Runnable mRunnable = new Runnable() {
        @Override
        public void run() { /* ... */ }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mHandler.postDelayed(mRunnable, 1000 * 60 * 10);
        finish();
    }

    private static class MyHandler extends Handler {
        private final WeakReference<MainActivity> mActivityReference;

        public MyHandler(MainActivity activity) {
            mActivityReference = new WeakReference<MainActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = mActivityReference.get();
            if (activity != null) {
                // ...
            }
        }
    }
} 
```
解释：
- 声明一个静态的MyHandler类，他不会隐式的持有MainActivity的引用
- 内部利用弱引用获取外部类的引用，若Activity回收，弱引用不会影响Activity的回收

## 资源对象没有关闭
资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于 java虚拟机内，还存在于java虚拟机外。如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄漏。 

# OOM
原因
- 内存泄漏导致，频繁的内存泄漏将会引发内存溢出。
- 占用内存较多的对象，保存了多个耗用内存较多的对象（如Bitmap），加载超大的图片。

加载图片OOM处理

- 等比例缩小图片
使用setImageBitmap或setImageResource或BitmapFactory.decodeResource设置大图时，这些函数在完成decode后，最终都是通过java层的createBitmap来完成的，需要消耗更多内存。 
	```java
	public static Bitmap scaleImage(Bitmap bitmap, int newWidth, int newHeight) {
	        if (bitmap == null) {
	            return null;
	        }
	        float scaleWidth = (float) newWidth / bitmap.getWidth();
	        float scaleHeight = (float) newHeight / bitmap.getHeight();
	        Matrix matrix = new Matrix();
	        matrix.postScale(scaleWidth, scaleHeight);
	        return Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(), bitmap.getHeight(), matrix, true);
	    }
	```
- 对图片采用软引用，及时地进行recycle()操作。
虽然系统能够确认Bitmap分配的内存最终会被销毁，但是由于它占用的内存过多，所以很可能会超过java堆的限制。因此，在用完Bitmap时要及时的recycle掉。recycle并不能确定立即就会将Bitmap释放掉，但是会给虚拟机一个暗示：“该图片可以释放了”。 
	```java
	SoftReference<Bitmap> bitmap;
	    bitmap = new SoftReference<>(pBitmap);
	    if(bitmap != null){
	        if(bitmap.get() != null && !bitmap.get().isRecycled()){
	            bitmap.get().recycle();
	            bitmap = null;
	        }
	    }
	```

其他OOM可使用LeakCanary检测是否发生了内存泄漏。