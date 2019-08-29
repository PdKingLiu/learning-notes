[TOC]

# Android性能优化——电量
[原Udacity视频链接](https://classroom.udacity.com/courses/ud825/lessons/3758168642/concepts/37313589490923)

## 1. 理解电池消耗
手机各个硬件模块的耗电量是不一样的，有些模块非常耗电，而有些模块则相对显得耗电量小很多。

![android_perf_battery_drain](https://img-blog.csdnimg.cn/20190723201207501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)

电量消耗的计算与统计是一件麻烦而且矛盾的事情，记录电量消耗本身也是一个费电量的事情。唯一可行的方案是使用第三方监测电量的设备，这样才能够获取到真实的电量消耗。

当设备处于待机状态时消耗的电量是极少的，以N5为例，打开飞行模式，可以待机接近1个月。可是点亮屏幕，硬件各个模块就需要开始工作，这会需要消耗很多电量。

使用WakeLock或者JobScheduler唤醒设备处理定时的任务之后，一定要及时让设备回到初始状态。每次唤醒蜂窝信号进行数据传递，都会消耗很多电量，它比WiFi等操作更加的耗电。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723201249728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 2. Battery Historian
Battery Historian是Android 5.0开始引入的新API。通过下面的指令，可以得到设备上的电量消耗信息：

	adb shell dumpsys batterystats > xxx.txt  //得到整个设备的电量消耗信息
	adb shell dumpsys batterystats > com.package.name > xxx.txt //得到指定app相关的电量消耗信息

得到了原始的电量消耗数据之后，我们需要通过Google编写的一个[Python脚本](https://github.com/google/battery-historian)把数据信息转换成可读性更好的html文件，打开这个转换过后的html文件，可以看到生成的列表数据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723203533357.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 3. 充电状态和BatteryManager
我们可以通过如下代码获取现在的充电状态是否为AC充电。

```
IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
Intent batteryStatus = this.registerReceiver(null, filter);
int chargePlug = batteryStatus.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
boolean acCharge = (chargePlug == BatteryManager.BATTERY_PLUGGED_AC);
if (acCharge) {
    Log.v(LOG_TAG,“The phone is charging!”);
}
```
得到充电的状态之后我们可以有针对性的做一些操作，比如在充电时执行一些耗时的代码。

```
private boolean checkForPower() {
    // It is very easy to subscribe to changes to the battery state, but you can get the current
    // state by simply passing null in as your receiver.  Nifty, isn't that?
    IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
    Intent batteryStatus = this.registerReceiver(null, filter);

    // There are currently three ways a device can be plugged in. We should check them all.
    int chargePlug = batteryStatus.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
    boolean usbCharge = (chargePlug == BatteryManager.BATTERY_PLUGGED_USB);
    boolean acCharge = (chargePlug == BatteryManager.BATTERY_PLUGGED_AC);
    boolean wirelessCharge = false;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
        wirelessCharge = (chargePlug == BatteryManager.BATTERY_PLUGGED_WIRELESS);
    }
    return (usbCharge || acCharge || wirelessCharge);
}
```
上面代码可以得到现在是否充电。

## 4. Wakelock和电池消耗
Android会不断关闭各种硬件来延长大量的硬件来延长手机的待机时间，首先屏幕会变暗直至关闭，然后CPU进入睡眠。但是即使在这种睡眠的状态下，大多数应用还是会尝试进行工作，他们将不断的唤醒手机。

最简单的唤醒手机的方法是使用WakeLock的API来保持CPU工作。这样手机可以简单的被唤醒工作，但是及时释放WakeLock是非常重要的，不恰当的使用WakeLock会导致很严重的后果。例如如果应用在上传图片内容时不是用预期的10秒，而是花了60分钟来分析。或者服务器崩溃了怎么办，社交媒体的服务器再也没有回应，要一直等下去了。这会导致你的手机保持开启状态 ，一直等待这些内容，这样很快手机电量就耗完了。

这就是我们需要使用带时间参数的wakelock.acquire()的原因。

在上述意外情况出现时会强制释放唤醒锁。

但是这也不能完全解决问题。这些情况下我们该如何设置时间参数值呢，使用唤醒锁并重试前，等待多久才比较合适呢?

正确的解决方法可能是，使用不精准的计时器。通常情况下，我们会设定一个时间进行某个操作，但是动态修改这个时间也许会更好。例如，如果有另外一个程序需要比你设定的时间晚5分钟唤醒，最好能够等到那个时候，两个任务捆绑一起同时进行，这就是非精确定时器的核心工作原理。我们可以定制计划的任务，可是系统如果检测到一个更好的时间，它可以推迟你的任务，以节省电量消耗。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723212552776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
现实中，有很多情况，可以进行集中电池工作更好。比如手机充电时，或连接了WIFI，或已经被另一进程唤醒。如果你的工作可以延迟到将来某一时刻，延迟到到更理想的时刻 ，那么你会大大提高低电池寿命。这就是任务调度API的功能。这个API在规划未来工作，提升电池性能上发挥了很大作用。例如，非用户操作可等到手机连接电源，或接入WIFI时在进行，或者在某一时间内集中处理任务。

## 5. 网络和电池消耗
对电池来说，网络连接是最耗电的工作。手机里面有一个芯片，Ham radio，他的功能就是连接当地的电话信号塔并和它们进行大量的数据传输。但是这个芯片不是一直活跃的，一旦你发送了数据，无线芯片会在一定时间内保持开启状态再接收返回的数据。但是如果没有活动，这个硬件就会休眠以节省电量。

通过前面学习到的Battery Historian我们可以得到设备的电量消耗数据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723214725685.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
每一个红色的栏都代表一个活跃的移动无线网，空隙代表无线网休眠，这里面存在很多窄窄的栏，就说明存在问题：反复，频繁出现的唤醒和休眠。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190723215155373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
如果出现大段的红栏和大段的空隙，这代表已经通过降低网络连接提高了性能。用编写代码来分批处理、缓存并延迟这些网络连接工作非常困难，可以使用AndroidAPI替我们完成所有的网络连接工作。

## 6. 使用JobScheduler
我们将部分任务交给系统决定，因为系统知道什么时间执行最省电，我们的任务是辨别哪些任务可以通过API交给Android任务调度器。
[JobScheduler传送门](https://developer.android.com/reference/android/app/job/JobScheduler.html)
下面是一个示例，首先需要一个JobService
```
public class MyJobService extends JobService {
    private static final String LOG_TAG = "MyJobService";

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(LOG_TAG, "MyJobService created");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(LOG_TAG, "MyJobService destroyed");
    }

    @Override
    public boolean onStartJob(JobParameters params) {
        // This is where you would implement all of the logic for your job. Note that this runs
        // on the main thread, so you will want to use a separate thread for asynchronous work
        // (as we demonstrate below to establish a network connection).
        // If you use a separate thread, return true to indicate that you need a "reschedule" to
        // return to the job at some point in the future to finish processing the work. Otherwise,
        // return false when finished.
        Log.i(LOG_TAG, "Totally and completely working on job " + params.getJobId());
        // First, check the network, and then attempt to connect.
        if (isNetworkConnected()) {
            new SimpleDownloadTask() .execute(params);
            return true;
        } else {
            Log.i(LOG_TAG, "No connection on job " + params.getJobId() + "; sad face");
        }
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        // Called if the job must be stopped before jobFinished() has been called. This may
        // happen if the requirements are no longer being met, such as the user no longer
        // connecting to WiFi, or the device no longer being idle. Use this callback to resolve
        // anything that may cause your application to misbehave from the job being halted.
        // Return true if the job should be rescheduled based on the retry criteria specified
        // when the job was created or return false to drop the job. Regardless of the value
        // returned, your job must stop executing.
        Log.i(LOG_TAG, "Whelp, something changed, so I'm calling it on job " + params.getJobId());
        return false;
    }

    /**
     * Determines if the device is currently online.
     */
    private boolean isNetworkConnected() {
        ConnectivityManager connectivityManager =
                (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo networkInfo = connectivityManager.getActiveNetworkInfo();
        return (networkInfo != null && networkInfo.isConnected());
    }

    /**
     *  Uses AsyncTask to create a task away from the main UI thread. This task creates a
     *  HTTPUrlConnection, and then downloads the contents of the webpage as an InputStream.
     *  The InputStream is then converted to a String, which is logged by the
     *  onPostExecute() method.
     */
    private class SimpleDownloadTask extends AsyncTask<JobParameters, Void, String> {

        protected JobParameters mJobParam;

        @Override
        protected String doInBackground(JobParameters... params) {
            // cache system provided job requirements
            mJobParam = params[0];
            try {
                InputStream is = null;
                // Only display the first 50 characters of the retrieved web page content.
                int len = 50;

                URL url = new URL("https://www.google.com");
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                conn.setReadTimeout(10000); //10sec
                conn.setConnectTimeout(15000); //15sec
                conn.setRequestMethod("GET");
                //Starts the query
                conn.connect();
                int response = conn.getResponseCode();
                Log.d(LOG_TAG, "The response is: " + response);
                is = conn.getInputStream();

                // Convert the input stream to a string
                Reader reader = null;
                reader = new InputStreamReader(is, "UTF-8");
                char[] buffer = new char[len];
                reader.read(buffer);
                return new String(buffer);

            } catch (IOException e) {
                return "Unable to retrieve web page.";
            }
        }

        @Override
        protected void onPostExecute(String result) {
            jobFinished(mJobParam, false);
            Log.i(LOG_TAG, result);
        }
    }
}
```
然后Activity的Button模拟触发多个任务并交给JobService来处理。

```
public class FreeTheWakelockActivity extends ActionBarActivity {
    public static final String LOG_TAG = "FreeTheWakelockActivity";

    TextView mWakeLockMsg;
    ComponentName mServiceComponent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_wakelock);

        mWakeLockMsg = (TextView) findViewById(R.id.wakelock_txt);
        mServiceComponent = new ComponentName(this, MyJobService.class);
        Intent startServiceIntent = new Intent(this, MyJobService.class);
        startService(startServiceIntent);

        Button theButtonThatWakelocks = (Button) findViewById(R.id.wakelock_poll);
        theButtonThatWakelocks.setText(R.string.poll_server_button);

        theButtonThatWakelocks.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                    pollServer();
            }
        });
    }

    /**
     * This method polls the server via the JobScheduler API. By scheduling the job with this API,
     * your app can be confident it will execute, but without the need for a wake lock. Rather, the
     * API will take your network jobs and execute them in batch to best take advantage of the
     * initial network connection cost.
     *
     * The JobScheduler API works through a background service. In this sample, we have
     * a simple service in MyJobService to get you started. The job is scheduled here in
     * the activity, but the job itself is executed in MyJobService in the startJob() method. For
     * example, to poll your server, you would create the network connection, send your GET
     * request, and then process the response all in MyJobService. This allows the JobScheduler API
     * to invoke your logic without needed to restart your activity.
     *
     * For brevity in the sample, we are scheduling the same job several times in quick succession,
     * but again, try to consider similar tasks occurring over time in your application that can
     * afford to wait and may benefit from batching.
     */
    public void pollServer() {
        JobScheduler scheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
        for (int i=0; i<10; i++) {
            JobInfo jobInfo = new JobInfo.Builder(i, mServiceComponent)
                    .setMinimumLatency(5000) // 5 seconds
                    .setOverrideDeadline(60000) // 60 seconds (for brevity in the sample)
                    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY) // WiFi or data connections
                    .build();

            mWakeLockMsg.append("Scheduling job " + i + "!\n");
            scheduler.schedule(jobInfo);
        }
    }
}
```

**关于电量优化Google官方也发布了一些文章**

[Android Developers 传送门](https://developer.android.com/training/efficient-downloads/index.html#lessons)