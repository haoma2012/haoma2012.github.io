---
title: Android frameWork层(四)： Android系统默认Home应用程序（Launcher）Zygote进程的启动过程源代码分析
date: 2017-08-20 23:22:34
tags:
---
在前面一篇文章中，我们分析了Android系统在启动时安装应用程序的过程，这些应用程序安装好之后，还需要有一个Home应用程序来负责把它们在桌面上
展示出来，在android系统中，这个默认的Home应用程序就是Launcher了，本文将详细分析Launcher应用程序的启动过程。

在Android系统中，所有的应用程序进程以及系统服务进程SystemServer都是由Zygote进程孕育（fork）出来的，这也许就是为什么要把它称为
Zygote（受精卵）的原因吧。由于Zygote进程在android系统中有着如此重要的地位，后面将详细分析它的启动过程。

<!-- more -->

在前面一篇文章中，我们分析了Android系统在启动时安装应用程序的过程，这些应用程序安装好之后，还需要有一个Home应用程序来负责把它们在桌面上展示出来，在android系统中，这个默认的Home应用程序就是Launcher了，本文将详细分析Launcher应用程序的启动过程。

[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/Luoshengyang/article/details/6767736)

Android系统的Home应用程序Launcher是由ActivityManagerService启动的，而ActivityManagerService和PackageManagerService一样，都是在开机时由SystemServer组件启动的，SystemServer组件首先是启动ePackageManagerService，由它来负责安装系统的应用程序，具体可以参考前面一篇文章[Android应用程序安装过程源代码分析](https://haoma2012.github.io/2017/08/17/Android%20frameWork%E5%B1%82(%E4%B8%89)%EF%BC%9AAndroid%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8BPMS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)，系统中的应用程序安装好了以后，SystemServer组件接下来就要通过ActivityManagerService来启动Home应用程序Launcher了，Launcher在启动的时候便会通过PackageManagerServic把系统中已经安装好的应用程序以快捷图标的形式展示在桌面上，这样用户就可以使用这些应用程序了，整个过程如下图所示：

![image](http://hi.csdn.net/attachment/201109/11/0_1315757033NNGT.gif)

#### 源码分析

- 1.SystemServer组件调用过程

```
1.SystemServer由Zygote进程fork出来的，管理系统所有的服务；
2.创建SystemServiceManager管理系统服务；
3.SystemServer.main方法里面子方法分别调用startBootstrapServices，
startCoreServices，startOtherServices启动不同的服务；
4.调用：获取AMS服务实例 Lifecycle是AMS的内部类
mActivityManagerService = mSystemServiceManager.startService(
ActivityManagerService.Lifecycle.class).getService();
```

- 2.分析ActivityManagerService的创建过程

创建过程是在Lifecycle类的构造函数：new 一个ActivityManagerService实例；
```
public Lifecycle(Context context) {
super(context);
mService = new ActivityManagerService(context);
}

........
//构造函数
public ActivityManagerService(Context systemContext) {
mContext = systemContext;//context
mFactoryTest = FactoryTest.getMode();
mSystemThread = ActivityThread.currentActivityThread();//ActivityThread线程
//
mHandlerThread = new ServiceThread(TAG,
android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
mHandlerThread.start();
mHandler = new MainHandler(mHandlerThread.getLooper());
mUiHandler = new UiHandler();
........//电池状态服务实例
mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
mBatteryStatsService.getActiveStatistics().readLocked();
mBatteryStatsService.scheduleWriteToDisk();


}
.......
```

构造函数创建了mHandlerThread，MainHandler，UiHandler消息传递；保存信息：创建ProcessStatsService实例，procstats文件保存，AppOpsService实例：保存appops.xml文件等等

- 3.PackageManagerService.main 创建PMS实例

frameworks/base/services/java/com/android/server/PackageManagerService.java
```
//systemserver 类方法
private void startBootstrapServices() {
.......
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();
.......
}
```

PackageManagerService.java文件主要功能就是扫描Android系统安装apk文件目录，并解析apk文件的manifest文件获取package信息保存到PMS中；执行完这一步之后，系统中的应用程序的所有信息都保存在PackageManagerService中了，后面Home应用程序Launcher启动起来后，就会把PackageManagerService中的应用程序信息取出来，然后以快捷图标的形式展示在桌面上，后面我们将会看到这个过程。

- 4. ActivityManagerService.setSystemProcess

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerServcie.java文件中：

```
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
......  
public void setSystemProcess() {
try {
ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
ServiceManager.addService("meminfo", new MemBinder(this));
ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
ServiceManager.addService("dbinfo", new DbBinder(this));
if (MONITOR_CPU_USAGE) {
ServiceManager.addService("cpuinfo", new CpuBinder(this));
}
ServiceManager.addService("permission", new PermissionController(this));
ServiceManager.addService("processinfo", new ProcessInfoService(this));

ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
"android", STOCK_PM_FLAGS);
//把应用程序框架层的android包加载进来
mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

synchronized (this) {
ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
app.persistent = true;
app.pid = MY_PID;
app.maxAdj = ProcessList.SYSTEM_ADJ;
app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
synchronized (mPidsSelfLocked) {
mPidsSelfLocked.put(app.pid, app);
}
updateLruProcessLocked(app, false, null);
updateOomAdjLocked();
}
} catch (PackageManager.NameNotFoundException e) {
throw new RuntimeException(
"Unable to find android system package", e);
}
}
.......
```
这个函数首先是将这个ActivityManagerService实例添加到ServiceManager中去托管，这样其它地方就可以通过ServiceManager.getService接口来访问这个全局唯一的ActivityManagerService实例了，接着又通过调用mSystemThread.installSystemApplicationInfo函数来把应用程序框架层下面的android包加载进来，这里的mSystemThread是一个ActivityThread类型的实例变量,在前面步骤创建，后面就是一些其它的初始化工作了。

- 5.ActivityManagerService.systemReady 最后启动home activity

系统中的一系列服务都初始化完毕之后才调用的，它定义在frameworks/base/services/java/com/android/server/am/ActivityManagerServcie.java文件中：

```
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
......  

public void systemReady(final Runnable goingCallback) {  
......  

synchronized (this) {  
......  
// Start up initial activity.
mBooting = true;
startHomeActivityLocked(mCurrentUserId, "systemReady");
......

mStackSupervisor.resumeTopActivitiesLocked();  
}  
}  

......  
}  

```
这个函数的内容比较多，这里省去无关的部分，主要关心启动Home应用程序的逻辑，这里就是通过
一：调用startHomeActivityLocked（）函数来启动Home应用程序的，
二：mStackSupervisor.resumeTopActivitiesLocked()了，mStackSupervisor类ActivityStackSupervisor是运行所有的ActivityStacks，activity堆栈；

- 6.ActivityManagerService.startHomeActivityLocked


```
boolean startHomeActivityLocked(int userId, String reason) {
.......
//1.获取home的 intent
Intent intent = getHomeIntent();
ActivityInfo aInfo =
resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
if (aInfo != null) {
intent.setComponent(new ComponentName(
aInfo.applicationInfo.packageName, aInfo.name));
// Don't do this if the home app is currently being
// instrumented.
aInfo = new ActivityInfo(aInfo);
aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
ProcessRecord app = getProcessRecordLocked(aInfo.processName,
aInfo.applicationInfo.uid, true);
if (app == null || app.instrumentationClass == null) {
intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
//调用startHomeActivity打开home
mStackSupervisor.startHomeActivity(intent, aInfo, reason);
}
}

return true;
}

....

Intent getHomeIntent() {
Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
intent.setComponent(mTopComponent);
if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
intent.addCategory(Intent.CATEGORY_HOME);
}
return intent;
}

......
private ActivityInfo resolveActivityInfo(Intent intent, int flags, int userId) {
ActivityInfo ai = null;
ComponentName comp = intent.getComponent();
try {
if (comp != null) {
// Factory test.
ai = AppGlobals.getPackageManager().getActivityInfo(comp, flags, userId);
} else {
ResolveInfo info = AppGlobals.getPackageManager().resolveIntent(
intent,
intent.resolveTypeIfNeeded(mContext.getContentResolver()),
flags, userId);

if (info != null) {
ai = info.activityInfo;
}
}
} catch (RemoteException e) {
// ignore
}

return ai;
}
```
函数首先创建一个CATEGORY_HOME类型的Intent，然后通过Intent.resolveActivityInfo函数向PackageManagerService查询Category类型为HOME的Activity，最后调用 mStackSupervisor.startHomeActivity(intent, aInfo, reason);这里我们假设只有系统自带的Launcher应用程序注册了HOME类型的Activity（见packages/apps/Launcher2/AndroidManifest.xml文件）：


```
<manifest  
xmlns:android="http://schemas.android.com/apk/res/android"  
package="com.android.launcher"  
android:sharedUserId="@string/sharedUserId"  
>  

......  

<application  
android:name="com.android.launcher2.LauncherApplication"  
android:process="@string/process"  
android:label="@string/application_name"  
android:icon="@drawable/ic_launcher_home">  

<activity  
android:name="com.android.launcher2.Launcher"  
android:launchMode="singleTask"  
android:clearTaskOnLaunch="true"  
android:stateNotNeeded="true"  
android:theme="@style/Theme"  
android:screenOrientation="nosensor"  
android:windowSoftInputMode="stateUnspecified|adjustPan">  
<intent-filter>  
<action android:name="android.intent.action.MAIN" />  
<category android:name="android.intent.category.HOME" />  
<category android:name="android.intent.category.DEFAULT" />  
<category android:name="android.intent.category.MONKEY"/>  
</intent-filter>  
</activity>  

......  
</application>  
</manifest>  
```
因此，这里就返回com.android.launcher2.Launcher这个Activity了。由于是第一次启动这个Activity，接下来调用函数getProcessRecordLocked返回来的ProcessRecord值为null，于是，就调用mStackSupervisor.startHomeActivity(intent, aInfo, reason);函数启动com.android.launcher2.Launcher这个Activity了，这里的mStackSupervisor是一个管理所有ActivityStack类型的成员变量。执行Launcher的onCreate方法

1.mContext变量中获得PackageManagerService；
2.PackageManagerService.queryIntentActivities接口来取回所有Action类型为Intent.ACTION_MAIN，并且Category类型为Intent.CATEGORY_LAUNCHER的Activity了
3.函数setApps首先清空mAllAppsList列表，然后调用addApps函数来为上一步得到的每一个应用程序创建一个ApplicationInfo实例了，有了这些ApplicationInfo实例之后，就可以在桌面上展示系统中所有的应用程序了。
4.系统默认的Home应用程序Launcher就把PackageManagerService中的应用程序加载进来了，当我们在屏幕上点击home这个图标时，就会把刚才加载好的应用程序以图标的形式展示出来了；

- [Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/Luoshengyang/article/details/6768304)

1.在Android系统中，所有的应用程序进程以及系统服务进程SystemServer都是由Zygote进程孕育（fork）出来的，这也许就是为什么要把它称为Zygote（受精卵）的原因吧。由于Zygote进程在android系统中有着如此重要的地位，本文将详细分析它的启动过程。

2.分析前面的文章知道：当ActivityManagerService启动一个应用程序的时候，就会通过Socket与Zygote进程进行通信，请求它fork一个子进程出来作为这个即将要启动的应用程序的进程；而系统中的两个重要服务PackageManagerService和ActivityManagerService，都是由SystemServer进程来负责启动的，而SystemServer进程本身是Zygote进程在启动的过程中fork出来的。

3.Android系统是基于Linux内核的，而在linux系统中，所有的进程都是init进程的子孙进程，也就是说，所有的进程都是直接或者间接地由init进程fork出来的。Zygote进程也不例外，它是在系统启动的过程，由init进程创建的。在系统启动脚本system/core/rootdir/init.rc文件中，我们可以看到启动Zygote进程的脚本命令：

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server  
socket zygote stream 666  
onrestart write /sys/android_power/request_state wake  
onrestart write /sys/power/state on  
onrestart restart media  
onrestart restart netd  
最后的一系列onrestart关键字表示这个zygote进程重启时需要执行的命令。
```
前面的关键字service告诉init进程创建一个名为"zygote"的进程，这个zygote进程要执行的程序是/system/bin/app_process，后面是要传给app_process的参数。接下来的socket关键字表示这个zygote进程需要一个名称为"zygote"的socket资源，这样，系统启动后，我们就可以在/dev/socket目录下看到有一个名为zygote的文件。这里定义的socket的类型为unix domain socket，它是用来作本地进程间通信用的，ActivityManagerService就是通这个socket来和zygote进程通信请求fork一个应用程序进程的了。

![image](http://hi.csdn.net/attachment/201109/16/0_1316190384ZuU0.gif)

1. app_process.main创建一个AppRuntime变量，然后调用它的start成员函数
2. AndroidRuntime.start这个函数的作用是启动Android系统运行时库，它主要做了三件事情，一是调用函数startVM启动虚拟机，二是调用函数startReg注册JNI方法，三是调用了com.android.internal.os.ZygoteInit类的main函数。
3. ZygoteInit.main它主要作了三件事情，一个调用registerZygoteSocket函数创建了一个socket接口，用来和ActivityManagerService通讯，二是调用startSystemServer函数来启动SystemServer组件，三是调用runSelectLoopMode函数进入一个无限循环在前面创建的socket接口上等待ActivityManagerService请求创建新的应用程序进程。
4. ActivityManagerService是通过Process.start函数来创建一个新的进程的，而Process.start函数会首先通过Socket连接到Zygote进程中，最终由Zygote进程来完成创建新的应用程序进程，而Process类是通过openZygoteSocketIfNeeded函数来连接到Zygote进程中的Socket的；
5. ZygoteInit.startSystemServer，Zygote进程通过Zygote.forkSystemServer函数来创建一个新的进程来启动SystemServer组件，返回值pid等0的地方就是新的进程要执行的路径，即新创建的进程会执行handleSystemServerProcess函数。
6. ZygoteInit.handleSystemServerProcess调用RuntimeInit.zygoteInit函数来进一步执行启动SystemServer组件的操作。
7.  RuntimeInit.zygoteInit一个是调用zygoteInitNative函数来执行一个Binder进程间通信机制的初始化工作，这个工作完成之后，这个进程中的Binder对象就可以方便地进行进程间通信了，另一个是调用上面Step5传进来的com.android.server.SystemServer类的main函数。
8.  RuntimeInit.zygoteInitNative native方法初始化
9.  ZygoteInit.runSelectLoopMode等待ActivityManagerService来连接这个Socket，然后调用ZygoteConnection.runOnce函数来创建新的应用程序


#### 汇总
```
1. 系统启动时init进程会创建Zygote进程，Zygote进程负责后续
Android应用程序框架层的其它进程的创建和启动工作。
2. Zygote进程会首先创建一个SystemServer进程，SystemServer
进程负责启动系统的关键服务，如包管理服务PackageManagerService和应用程序组件管理服务ActivityManagerService。
3. 当我们需要启动一个Android应用程序时，ActivityManagerService
会通过Socket进程间通信机制，通知Zygote进程为这个应用程序创建一个新的进程。

```

