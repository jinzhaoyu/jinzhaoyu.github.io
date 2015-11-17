title:  Android - 琢磨隐藏前台服务的过程-隐藏的!
tags: Android 前台服务 隐藏
---

在国内，Android平台下的应用因为种种原因，都必须要各自做各自的推送通道。然而，由于Android系统的特性，这些需要保持通道连接的应用，又要想方设法的防止被系统杀掉。（这时候真的觉得Apple的APNs 是一个神一样的存在！）

于是乎各种综合手段就都会用上了，比如分出一个守护进程，这个进程做的都是轻量的事情，甚至只检测主要进程的存活性。同时又在这个进程中启动一个前台服务（Foreground Service），尽可能减少这个守护进程被系统回收的机会。这里我就只说一下前台服务的事。

Android 系统的内存回收真正的执行者是 LowMemoryKiller(这里不做过多描述，可以参考： http://www.cnblogs.com/angeldevil/archive/2013/05/21/3090872.html) ，它将进程分了几个层次，其中最高层次的是：

	// This is the process running the current foreground app.  We'd really
	// rather not kill it!
	static final int FOREGROUND_APP_ADJ = 0;

也就是说，前台进程是最后了，逼不得已的情况下才会去释放它占用的内存的。前台进程就是当前正在前台显示的，用户正在操作的界面，或者是一个前台的服务。

Android系统要求前台服务必须要发送一个Notification，以便告知用户有应用一直在运行中，让用户感知到这个应用可能会耗费他的电池或流量。但是有些时候，我们确实想隐藏这种Notification，还要保持在前台服务的运行，因为常驻通知栏的图标，有的用户需要，有的用户则强烈要求去掉。

我们发现，手机QQ有一个前台服务，可是状态栏和通知中心都看不到QQ的任何通知图标，用如下命令可以dump出来系统中正在运行的所有的Service ：

`adb shell dumpsys activity services`

可以看到手机QQ有三个Service：

```
* ServiceRecord{43a6db20 u0 com.tencent.mobileqq/.app.CoreService}
intent={cmp=com.tencent.mobileqq/.app.CoreService}
packageName=com.tencent.mobileqq
processName=com.tencent.mobileqq
baseDir=/data/app/com.tencent.mobileqq-2.apk
dataDir=/data/data/com.tencent.mobileqq
app=ProcessRecord{435c1a98 8126:com.tencent.mobileqq/u0a10145}
createTime=-9m52s660ms lastActivity=-4m52s742ms
executingStart=-4m52s742ms restartTime=-9m52s660ms
startRequested=true stopIfKilled=false callStart=true lastStartId=7


* ServiceRecord{4378c818 u0 com.tencent.mobileqq/.app.CoreService$KernelService}
intent={cmp=com.tencent.mobileqq/.app.CoreService$KernelService}
packageName=com.tencent.mobileqq
processName=com.tencent.mobileqq
baseDir=/data/app/com.tencent.mobileqq-2.apk
dataDir=/data/data/com.tencent.mobileqq
app=ProcessRecord{435c1a98 8126:com.tencent.mobileqq/u0a10145}
isForeground=true foregroundId=537041609 foregroundNoti=Notification(pri=0 icon=7f020314 contentView=com.tencent.mobileqq/0x10900b4 vibrate=null sound=null defaults=0x0 flags=0x62 when=1429702926311 ledARGB=0x0 contentIntent=Y deleteIntent=N contentTitle=QQ正在执行中 contentText=触控来取得更多信息，或停止应用程序 tickerText=N kind=[null])
createTime=-9m51s902ms lastActivity=-9m51s902ms
executingStart=-9m51s897ms restartTime=-9m51s902ms
startRequested=true stopIfKilled=true callStart=true lastStartId=1


* ServiceRecord{43a12670 u0 com.tencent.mobileqq/.msf.service.MsfService}
intent={cmp=com.tencent.mobileqq/.msf.service.MsfService}
packageName=com.tencent.mobileqq
processName=com.tencent.mobileqq:MSF
baseDir=/data/app/com.tencent.mobileqq-2.apk
dataDir=/data/data/com.tencent.mobileqq
app=ProcessRecord{432900e0 3974:com.tencent.mobileqq:MSF/u0a10145}
createTime=-8d3h5m17s705ms lastActivity=-9m50s414ms
……………………

```

注意上面的标红的 .app.CoreService$KernelService ，其中有一个 isForeground=true 属性，其值为true，并且也设置了 foregroundNoti 的值，从这个Notification也看不出来有什么异常。
一开始以为他们用了什么手段搞了个看不见的通知，比如透明的图标或者高度为0的RemoteView，我也按这种思路去尝试了下，总有一个占位的Notification在。再 dump 出 Notification ：

` adb shell dumpsys statusbar`

查看 Notification list ，可以看到我做的透明的通知，却看不到有他们的任何 StatusBarNotification 那么它是怎么做到隐藏这个通知的呢？试试反编，看看他们的代码！


于是我就用 Android反编工具 试着反编手机QQ最新版 5.5.1.2435 ，直接就失败了，因为他们做了反apkTool的工作，无法用apkTool去解压他们的apk包。没关系，那我就直接解压缩QQ的安装包，毕竟apk也就是一种zip格式嘛。unzip解压后，可以看到目录中有一个 9.9M 的 classes.dex , 用 dex2jar 2.0 反编译它： 

`./decompileAndroid/dex2jar-2.0/d2j-dex2jar.sh -o source.jar classes.dex`

竟然成功了！！用JD—GUI，打开这个source.jar文件，找到 CoreService 和其内部类 KernelService ： com.tencent.mobileqq.app.CoreService 、 com.tencent.mobileqq.app.CoreService$KernelService ,代码内容我就不贴出来了，有兴趣的同学可以自己动动手或者从附件中下载。这里我只说下我理解的大致过程：

1、 CoreService 是一个假的Service，应用启动时即开始启动这个Service，这个Service 会在 onCreate() 中就 startForeground() ，传的 Notification 是 new 出的一个空的通知，通知 ID 为固定的值。

2、 启动完 CoreService 就开始启动真正的 KernelService

3、 KernelService中将 CoreService 再次 startForeground() ，然后再把自己 startForeground() ，最后再把 CoreService stopForeground() 掉。这是非常关键的一步。同时，要确保两个 Service startForeground()时使用的 Notification ID 都是同一个！


最终，仿照手机QQ的实现代码为： 

```java

public class MessageCenterService extends Service {

    private static final String sTAG = "MessageCenterService";

    //9521 就是你的终身代号 :)
    static final int NOTIFY_ID = 9521;

    private static MessageCenterService instance;

    public static MessageCenterService getInstance() {
        return instance;
    }

    /**
     * 启动前台服务
     */
    public static void start() {
        try {
            Intent intent = new Intent(App.getContext(), MessageCenterService.class);
            App.getContext().startService(intent);
        } catch (Exception e) {
            LogUtil.e(sTAG, "", e);
        }
    }

    /**
     * 终止前台服务。包含{@link MessageCenterService.KernelService}
     */
    public static void stop() {
        try {
            Intent intent = new Intent(App.getContext(), MessageCenterService.class);
            App.getContext().stopService(intent);

            stopKernel();
        } catch (Exception e) {
            LogUtil.e(sTAG, "", e);
        }
    }

    static void startKernel() {
        try {
            Intent intent = new Intent(App.getContext(), KernelService.class);
            App.getContext().startService(intent);
        } catch (Exception e) {
            LogUtil.e(sTAG, "", e);
        }
    }

    static void stopKernel() {
        try {
            Intent intent = new Intent(App.getContext(), KernelService.class);
            App.getContext().stopService(intent);
        } catch (Exception e) {
            LogUtil.e(sTAG, "", e);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        instance = this;
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
            startForeground(NOTIFY_ID, new Notification());
        }
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        //启动真正的Service
        startKernel();
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
            stopForeground(true);
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }


    /**
     *
     */
    public static class KernelService extends Service {
        private static KernelService instance;

        public static KernelService getInstance() {
            return instance;
        }

        @Override
        public void onCreate() {
            super.onCreate();
            instance = this;
        }

        @Override
        public int onStartCommand(Intent intent, int flags, int startId) {
            super.onStartCommand(intent, flags, startId);
            try {
                MessageCenterService fakeService = MessageCenterService.getInstance();
                fakeService.startForeground(NOTIFY_ID, new Notification());
                startForeground(NOTIFY_ID, new Notification());
                fakeService.stopForeground(true);
            } catch (Exception e) {
                LogUtil.e(sTAG, " **** Can not start foreground service !! ****", e);
            }
            return START_STICKY;
        }

        @Override
        public void onDestroy() {
            stopForeground(true);
            instance = null;
            super.onDestroy();
        }

        @Override
        public IBinder onBind(Intent intent) {
            return null;
        }
    }
}
 
```

里面有一些自己的工具类什么的，大家可以替换为自己的东西，同时别忘了在 AndroidManifest.xml中定义下这两个Service：

```xml
       <service
            android:name=".service.MessageCenterService$KernelService"
            android:label="@string/message_center_service"
            android:exported="false"/>
       <service
            android:name=".service.MessageCenterService"
            android:label="@string/message_center_service"
            android:exported="false"/>
``` 

今天又去stackOverflow上有针对性的查了下这个方法，还真有人说过这种处理方式：

[http://stackoverflow.com/a/18281520](http://stackoverflow.com/a/18281520)

以上就可以实现一个隐藏的前台服务，增加应用的存活率。看下LowMemoryKiller的分析后，你就应该知道，无论是那种级别的进程，都会有一个内存占用的阈值的，超过这个阈值同样会被杀，所以，优化好应用的内存使用也同样重要。


