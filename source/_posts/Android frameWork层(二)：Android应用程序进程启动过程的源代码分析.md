---
title: 'Android frameWork层(二):Android应用程序进程启动过程的源代码分析'
date: 2017-08-09 22:10:37
tags:
description: Android进程启动过程，入口ActivityThread.main函数，binder进程间通信
---

<!-- more -->


#### 1.简介
Android应用程序框架层创建的应用程序进程具有两个特点：

- 一是进程的入口函数是ActivityThread.main；
- 二是进程天然支持Binder进程间通信机制；
这两个特点都是在进程的初始化过程中实现的，本文将详细分析android应用程序进程创建过程中是如何实现这两个特点的。

Android应用程序框架层创建的应用程序进程的入口函数是ActivityThread.main比较好理解，即进程创建完成之后，Android应用程序框架层就会在这个进程中将ActivityThread类加载进来，然后执行它的main函数，这个main函数就是进程执行消息循环的地方了；

Android应用程序框架层创建的应用程序进程天然支持Binder进程间通信机制这个特点应该怎么样理解呢？我们在第一篇[Android frameWork层 (一)：Android 进程间通信学习](),它具有四个组件：分别是驱动程序、守护进程、Client以及Server，其中Server组件在初始化时必须进入一个循环中不断地与Binder驱动程序进行到交互，以便获得Client组件发送的请求。

#### 2.启动过程分析

在Android应用程序框架层中，是由ActivityManagerService(AMS)组件负责为Android应用程序创建新的进程的，它本来也是运行在一个独立的进程之中，不过这个进程是在系统启动的过程中创建的，当系统决定要在一个新的进程中启动Activity或者service的时候，就会创建一个进程。

ActivityManagerService源码分析：

- 1.ActivityManagerService实例调用

AMS创建启动过程是在SystemServer负责的，而SystemServer是Android系统服务的管理类，负责服务的启动：包括AMS,PMS，DMS等等分别会执行：startBootstrapServices(),startCoorServices(),startOtherServices();ActivityManagerService在startBootstrapServices启动，这里也包括：PowerManagerService、DisplayManagerService、PackageManagerService

```
private void startBootstrapServices() {
.......
//1.获取ActivityManagerService服务的实例
mActivityManagerService = mSystemServiceManager.startService(              
ActivityManagerService.Lifecycle.class).getService(); 
//把AMS实例加入到服务管理类SystemServiceManager统一管理
mActivityManagerService.setSystemServiceManager(mSystemServiceManager); 
//执行安装器先关类(Installer/InstallConnection)安装apk路径的apk文件
mActivityManagerService.setInstaller(installer);
.......

}
```

我们看下Lifecycle类：继承SystemService，它是一个控制低优先级服务的实用工具

Lifecycle是ActiityManagerService的内部类，里面是对AMS创建以及启动过程，所以调用getService()方法会获得AMS的实例

```
public static final class Lifecycle extends SystemService {
private final ActivityManagerService mService;

public Lifecycle(Context context) {//创建AMS实例
super(context);
mService = new ActivityManagerService(context);
}

@Override
public void onStart() {//启动服务
mService.start();
}

public ActivityManagerService getService() {
return mService;//返回AMS实例
}
}

```

- 2.ActivityManagerService 源码分析
先看看构造函数具体初始化了什么：

```
public ActivityManagerService(Context systemContext) {
mContext = systemContext;//context
mSystemThread = ActivityThread.currentActivityThread();//当前线程
.....

mHandlerThread = new ServiceThread(TAG,
android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
mHandlerThread.start();//启动
mHandler = new MainHandler(mHandlerThread.getLooper());
mUiHandler = new UiHandler();
//前后台广播消息队列由MainHander处理分发
mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
"foreground", BROADCAST_FG_TIMEOUT, false);
mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
"background", BROADCAST_BG_TIMEOUT, true);
.......
//创建进程CPU运行线程监听cpu运行；
mProcessCpuThread = new Thread("CpuTracker")

}
```

创建了mHandlerThread：具有Looper的循环处理消息的线程；继续创建了MainHandler和UiHandler处理消息；mProcessCpuThread，当然还有初始化电池mBatteryStatsService服务，获取耗电量信息，进程状态服务ProcessStatsService，AppOpsService服务，读取AtomicFile文件等等。

[Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/Luoshengyang/article/details/6747696)

ActivityManagerService启动新的进程是从其成员函数startProcessLocked开始的，在深入分析这个过程之前，我们先来看一下进程创建过程的序列图，然后再详细分析每一个步骤。

![image](http://hi.csdn.net/attachment/201109/5/0_1315236533f7n7.gif)

- 3.ActivityManagerService.startProcessLocked

```

public final class ActivityManagerService extends ActivityManagerNative    
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {    

......    

private final void startProcessLocked(ProcessRecord app,    
String hostingType, String hostingNameStr) {    

......    

try {    
int uid = app.info.uid;    
int[] gids = null;    
try {    
gids = mContext.getPackageManager().getPackageGids(    
app.info.packageName);    
} catch (PackageManager.NameNotFoundException e) {    
......    
}    

......    

int debugFlags = 0;    

......    
if (entryPoint == null) entryPoint = "android.app.ActivityThread";
//1.启动进程：Zygote进程负责启动新的应用程序进程
checkTime(startTime, "startProcess: asking zygote to start proc");    
//2.Process.start参数：ActivityThread字符串，进程name
//uid，gids，保存应用数据路径
Process.ProcessStartResult startResult = Process.start(entryPoint,
app.processName, uid, uid, gids, debugFlags, mountExternal,
app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
app.info.dataDir, entryPointArgs);   

......    

} catch (RuntimeException e) {    

......    

}    
}    

......    

}  
```
它调用了Process.start函数开始为应用程序创建新的进程，注意，它传入一个第一个参数为"android.app.ActivityThread"，这就是进程初始化时要加载的Java类了，把这个类加载到进程之后，就会把它里面的静态成员函数main作为进程的入口点，后面我们会看到。

- 4.Process.start 

这个函数定义在frameworks/base/core/java/android/os/Process.java文件中：
这个类的作用是：管理Android系统进程的工具类；


//管理系统进程类：包括各个功能UID、线程优先级、信号量定义、Zygote进程状态ZygoteState类

```
public class Process {

.......
public static final ProcessStartResult start(final String processClass,
final String niceName,
int uid, int gid, int[] gids,
int debugFlags, int mountExternal,
int targetSdkVersion,
String seInfo,
String abi,
String instructionSet,
String appDataDir,
String[] zygoteArgs) {
try {
return startViaZygote(processClass, niceName, uid, gid, gids,
debugFlags, mountExternal, targetSdkVersion, seInfo,
abi, instructionSet, appDataDir, zygoteArgs);
} catch (ZygoteStartFailedEx ex) {
Log.e(LOG_TAG,
"Starting VM process through Zygote failed");
throw new RuntimeException(
"Starting VM process through Zygote failed", ex);
}
}
........

}
```

很简单直接调用startViaZygote方法：Starts a new process via the zygote mechanism.t通过Zygote进程启动一个新的进程；


```
public class Process {

.......
//通过Zygote进程启动新的进程
private static ProcessStartResult startViaZygote(final String processClass,
final String niceName,
final int uid, final int gid,
final int[] gids,
int debugFlags, int mountExternal,
int targetSdkVersion,
String seInfo,
String abi,
String instructionSet,
String appDataDir,
String[] extraArgs)
throws ZygoteStartFailedEx {
synchronized(Process.class) {//同步锁操作
ArrayList<String> argsForZygote = new ArrayList<String>();
//Zygote启动进程需要的参数
// --runtime-args, --setuid=, --setgid=,
// and --setgroups= must go first
argsForZygote.add("--runtime-args");
argsForZygote.add("--setuid=" + uid);
argsForZygote.add("--setgid=" + gid);
if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
argsForZygote.add("--enable-jni-logging");
}
if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
argsForZygote.add("--enable-safemode");
}
if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
argsForZygote.add("--enable-debugger");
}
if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
argsForZygote.add("--enable-checkjni");
}
if ((debugFlags & Zygote.DEBUG_ENABLE_JIT) != 0) {
argsForZygote.add("--enable-jit");
}
if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
argsForZygote.add("--generate-debug-info");
}
if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
argsForZygote.add("--enable-assert");
}
if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
argsForZygote.add("--mount-external-default");
} else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
argsForZygote.add("--mount-external-read");
} else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
argsForZygote.add("--mount-external-write");
}
argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

//TODO optionally enable debuger
//argsForZygote.add("--enable-debugger");

// --setgroups is a comma-separated list
if (gids != null && gids.length > 0) {
StringBuilder sb = new StringBuilder();
sb.append("--setgroups=");

int sz = gids.length;
for (int i = 0; i < sz; i++) {
if (i != 0) {
sb.append(',');
}
sb.append(gids[i]);
}

argsForZygote.add(sb.toString());
}

if (niceName != null) {
argsForZygote.add("--nice-name=" + niceName);
}

if (seInfo != null) {
argsForZygote.add("--seinfo=" + seInfo);
}

if (instructionSet != null) {
argsForZygote.add("--instruction-set=" + instructionSet);
}

if (appDataDir != null) {
argsForZygote.add("--app-data-dir=" + appDataDir);
}

argsForZygote.add(processClass);

if (extraArgs != null) {
for (String arg : extraArgs) {
argsForZygote.add(arg);
}
}
//把这些进程需要的参数由Zygote进程fork并获得创建进程的结果
return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
}
}

.......
}

```

这个函数将创建进程的参数放到argsForZygote列表中去，如参数"--runtime-init"表示要为新创建的进程初始化运行时库，然后调用zygoteSendArgsAndGetResult函数进一步操作。

- 5.Process.zygoteSendArgsAndGetResult()：
它将启动一个新的子进程，并返回该子pid。请注意:当前的实现用空格替代了参数列表中的新行。


```
private static ProcessStartResult zygoteSendArgsAndGetResult(
ZygoteState zygoteState, ArrayList<String> args)
throws ZygoteStartFailedEx {
.......
//获取BufferedWriter、DataInputStream输入流
final BufferedWriter writer = zygoteState.writer;
final DataInputStream inputStream = zygoteState.inputStream;
//
writer.write(Integer.toString(args.size()));
writer.newLine();

//for循环取得传入的参数并写入到Zygote进程中
int sz = args.size();
for (int i = 0; i < sz; i++) {
String arg = args.get(i);
if (arg.indexOf('\n') >= 0) {
throw new ZygoteStartFailedEx(
"embedded newlines not allowed");
}
writer.write(arg);
writer.newLine();
}

.....
// Should there be a timeout on this?
ProcessStartResult result = new ProcessStartResult();
result.pid = inputStream.readInt();
if (result.pid < 0) {//pid 小于0 新进程fork失败
throw new ZygoteStartFailedEx("fork() failed");
}
result.usingWrapper = inputStream.readBoolean();
return result;
.......

}
```

这里由ZygoteState类与Zygote进程通信交流的状态我们看下这个ZygoteState内部类：内部维护着LocalSocket，DataInputStream，BufferedWriter，abiList

```
public static class ZygoteState {
final LocalSocket socket;
final DataInputStream inputStream;
final BufferedWriter writer;
final List<String> abiList;

......

public static ZygoteState connect(String socketAddress) throws IOException {
......

try {
zygoteSocket.connect(new LocalSocketAddress(socketAddress,
LocalSocketAddress.Namespace.RESERVED));
......

String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
Log.i("Zygote", "Process: zygote socket opened, supported ABIS: " + abiListString);

return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
Arrays.asList(abiListString.split(",")));
}
......
}
```

调用LocalSocket 的connect方法通信:

```
public class LocalSocket implements Closeable {
.......
//调用这个方法会将这个socket套接字连接到一个端点，这个只能被连接到未连接的实例
public void connect(LocalSocketAddress endpoint) throws IOException {
synchronized (this) {
if (isConnected) {//如果连接了 抛异常
throw new IOException("already connected");
}
//是否需要创建LocalSocketImpl实例
implCreateIfNeeded();
//连接socket
impl.connect(endpoint, 0);
isConnected = true;
isBound = true;
}
}

......
}
```
LocalSocketImpl这个实现类用作LocalSocket和LocalServerSocket，implCreateIfNeeded()方法就是创建Socket；


```
public void create (int sockType) throws IOException {
.......
fd = Os.socket(OsConstants.AF_UNIX, osType, 0);
......
}
```
调用OS创建socket通信，而维护这个Socket由frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中的ZygoteInit类在registerZygoteSocket函数侦听的。


```
public class ZygoteInit {
........
private static void registerZygoteSocket(String socketName) {
if (sServerSocket == null) {//判空
int fileDesc;
final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;//serversocket的name
try {
String env = System.getenv(fullSocketName);
fileDesc = Integer.parseInt(env);
} catch (RuntimeException ex) {
throw new RuntimeException(fullSocketName + " unset or invalid", ex);
}

try {
FileDescriptor fd = new FileDescriptor();
fd.setInt$(fileDesc);
sServerSocket = new LocalServerSocket(fd);
} catch (IOException ex) {
throw new RuntimeException(
"Error binding to local socket '" + fileDesc + "'", ex);
}
}

}
........
//等待命令连接ServerSocket
private static ZygoteConnection acceptCommandPeer(String abiList) {
try {
return new ZygoteConnection(sServerSocket.accept(), abiList);
} catch (IOException ex) {
throw new RuntimeException(
"IOException during accept()", ex);
}
}
}
```

注册服务socket监听Zygote命令连接，创建LocalServerSocket实例监听socket连接，内部会调用ZygoteConnection,等待socket的连接；

ZygoteInit类:它的入口main函数


```
1.registerZygoteSocket(name)注册ServerSocket，连接，
关闭closeServerSocket操作
2.preload();预加载类，资源，OpenGL，SharedLibraries，文本资源
，prepareWebViewInZygote，preloadColorStateLists，preloadDrawables
图片资源，颜色资源
3.Zygote进程初始化成功： SamplingProfilerIntegration.writeZygoteSnapshot();
4.zygote进程启动后进行内存回收调用gc：
gcAndFinalize();
//这对调用fork()出新子进程之前进程回收内存有用；
static void gcAndFinalize() {
final VMRuntime runtime = VMRuntime.getRuntime();

/* runFinalizationSync() lets finalizers be called in Zygote,
* which doesn't have a HeapWorker thread.
*/
System.gc();
runtime.runFinalizationSync();
System.gc();
}
5.startSystemServer，准备fork出进程的System server进程，所以
我们知道新创建的进程由Zygote进程fork出来的，同时也会为这个进程
fork唯一的SystemServer进程管理服务；
```

- 6.ZygoteConnection.runOnce，真正创建子进程的地方：调用Zygote.forkAndSpecialize,返回进程的pid；


```
class ZygoteConnection {  
......  

boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {  
String args[];  
Arguments parsedArgs = null;  
FileDescriptor[] descriptors;  

try {  
args = readArgumentList();  
descriptors = mSocket.getAncillaryFileDescriptors();  
} catch (IOException ex) {  
......  
return true;  
}  

......  

/** the stderr of the most recent request, if avail */  
PrintStream newStderr = null;  

if (descriptors != null && descriptors.length >= 3) {  
newStderr = new PrintStream(  
new FileOutputStream(descriptors[2]));  
}  

int pid;  

try {  
parsedArgs = new Arguments(args);  

applyUidSecurityPolicy(parsedArgs, peer);  
applyDebuggerSecurityPolicy(parsedArgs);  
applyRlimitSecurityPolicy(parsedArgs, peer);  
applyCapabilitiesSecurityPolicy(parsedArgs, peer);  

int[][] rlimits = null;  

if (parsedArgs.rlimits != null) {  
rlimits = parsedArgs.rlimits.toArray(intArray2d);  
}  

pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
parsedArgs.appDataDir);  
} catch (IllegalArgumentException ex) {  
......  
} catch (ZygoteSecurityException ex) {  
......  
}  

if (pid == 0) {  
// in child  
handleChildProc(parsedArgs, descriptors, newStderr);  
// should never happen  
return true;  
} else { /* pid != 0 */  
// in parent...pid of < 0 means failure  
return handleParentProc(pid, descriptors, parsedArgs);  
}  
}  

......  
}  
```
真正创建进程的地方就是在这里了：

```
pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
parsedArgs.appDataDir);
```
这个函数会创建一个进程，而且有两个返回值，一个是在当前进程中返回的，一个是在新创建的进程中返回，即在当前进程的子进程中返回，在当前进程中的返回值就是新创建的子进程的pid值，而在子进程中的返回值是0


```
int pid = nativeForkAndSpecialize(
uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
instructionSet, appDataDir);
if (pid == 0) {
Trace.setTracingEnabled(true);

// Note that this event ends at the end of handleChildProc,
Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
}
```

这里会调用nativeForkAndSpecialize函数创建进程，pid==0的时候创建子进程，我们只看这里回调 Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");，相当于一个消息机制，发出Postfork；


- 7.ZygoteInit.startSystemServer()，fork出SystemServer进程；


```
private static boolean startSystemServer(String abiList, String socketName)
throws MethodAndArgsCaller, RuntimeException {
......
//fork 出SystemServer进程，得到进程的pid
pid = Zygote.forkSystemServer(
parsedArgs.uid, parsedArgs.gid,
parsedArgs.gids,
parsedArgs.debugFlags,
null,
parsedArgs.permittedCapabilities,
parsedArgs.effectiveCapabilities);
......


}
```

- 8.RuntimeInit.zygoteInit
这个函数定义在frameworks/base/core/java/com/android/internal/os/RuntimeInit.java文件中：

```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
throws ZygoteInit.MethodAndArgsCaller {
if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
redirectLogStreams();

commonInit();
nativeZygoteInit();
applicationInit(targetSdkVersion, argv, classLoader);
}
```

这里有两个关键的函数调用，一个是zygoteInitNative函数调用，一个是invokeStaticMain函数调用，这个在applicationInit方法里面，前者就是执行Binder驱动程序初始化的相关工作了，正是由于执行了这个工作，才使得进程中的Binder对象能够顺利地进行Binder进程间通信，而后一个函数调用，就是执行进程的入口函数，这里就是执行startClass类的main函数了，而这个startClass即是我们在Step 1中传进来的"android.app.ActivityThread"值，表示要执行android.app.ActivityThread类的main函数。我们先来看一下zygoteInitNative函数的调用过程，然后再回到RuntimeInit.zygoteInit函数中来，看看它是如何调用android.app.ActivityThread类的main函数的。

zygoteInitNative是一个native函数，实现在frameworks/base/core/jni/AndroidRuntime.cpp文件中：


```
static AndroidRuntime* gCurRuntime = NULL; 
.......
static void com_android_internal_os_RuntimeInit_zygoteInit(JNIEnv* env, jobject clazz)  
{  
gCurRuntime->onZygoteInit();  
}  
......
```

- 9.AppRuntime.onZygoteInit
这个函数定义在frameworks/base/cmds/app_process/app_main.cpp文件中：


```
class AppRuntime : public AndroidRuntime  
{  
......  

virtual void onZygoteInit()  
{  
sp<ProcessState> proc = ProcessState::self();  
if (proc->supportsProcesses()) {  
LOGV("App process: starting thread pool.\n");  
proc->startThreadPool();  
}  
}  

......  
}; 
```
这里它就是调用ProcessState::startThreadPool启动线程池了，这个线程池中的线程就是用来和Binder驱动程序进行交互的了。ProcessState类是Binder进程间通信机制的一个基础组件，它会创建一个PoolThread线程类，然后执行它的run函数，最终就会执行PoolThread类的threadLoop函数了。这个线程会告诉binder机制它要进入循环，不断与binder驱动进行交互，获取client端的进程间调用，当线程退出后会调用

也会告诉Binder驱动程序，它退出了，这样Binder驱动程序就不会再在Client端的进程间调用分发给它了：
```
mOut.writeInt32(BC_EXIT_LOOPER);  
talkWithDriver(false);  
```
调用talkWithDriver和binder驱动交互，还有线程池，我们在开发Android应用程序的时候，当我们要和其它进程中进行通信时，只要定义自己的Binder对象，然后把这个Binder对象的远程接口通过其它途径传给其它进程后，其它进程就可以通过这个Binder对象的远程接口来调用我们的应用程序进程的函数了，它不像我们在C++层实现Binder进程间通信机制的Server时，必须要手动调用IPCThreadState.joinThreadPool函数来进入一个无限循环中与Binder驱动程序交互以便获得Client端的请求，这样就实现了我们在文章开头处说的Android应用程序进程天然地支持Binder进程间通信机制。


RuntimeInit.zygoteInit函数中，在初始化完成Binder进程间通信机制的基础设施后，它接着就要进入进程的入口函数了。
这个函数定义在frameworks/base/core/java/com/android/internal/os/RuntimeInit.java文件中：

- 10. RuntimeInit.invokeStaticMain

```
public class ZygoteInit {  
......  

static void invokeStaticMain(ClassLoader loader,  
String className, String[] argv)  
throws ZygoteInit.MethodAndArgsCaller {  
Class<?> cl;  

try {  
cl = loader.loadClass(className);  
} catch (ClassNotFoundException ex) {  
......  
}  

Method m;  
try {  
m = cl.getMethod("main", new Class[] { String[].class });  
} catch (NoSuchMethodException ex) {  
......  
} catch (SecurityException ex) {  
......  
}  

int modifiers = m.getModifiers();  
......  

/* 
* This throw gets caught in ZygoteInit.main(), which responds 
* by invoking the exception's run() method. This arrangement 
* clears up all the stack frames that were required in setting 
* up the process. 
*/  
throw new ZygoteInit.MethodAndArgsCaller(m, argv);  
}  

......  
}  
```
前面我们说过，这里传进来的参数className字符串值为"android.app.ActivityThread"，这里就通ClassLoader.loadClass函数将它加载到进程中：


```
cl = loader.loadClass(className);  
```
然后获得它的静态成员函数main：

```
m = cl.getMethod("main", new Class[] { String[].class });  
```
最后调用ActivityThread类的入口main函数：


```
public final class ActivityThread {  
......  

public static final void main(String[] args) {  
SamplingProfilerIntegration.start();  

Process.setArgV0("<pre-initialized>");  

Looper.prepareMainLooper();  
if (sMainThreadHandler == null) {  
sMainThreadHandler = new Handler();  
}  

ActivityThread thread = new ActivityThread();  
thread.attach(false);  

if (false) {  
Looper.myLooper().setMessageLogging(new  
LogPrinter(Log.DEBUG, "ActivityThread"));  
}  
Looper.loop();  

if (Process.supportsProcesses()) {  
throw new RuntimeException("Main thread loop unexpectedly exited");  
}  

thread.detach();  
String name = (thread.mInitialApplication != null)  
? thread.mInitialApplication.getPackageName()  
: "<unknown>";  
Slog.i(TAG, "Main thread of " + name + " is now exiting");  
}  

......  
}  
```
创建ActivityThread 进入消息循环，这样，我们以后就可以在这个进程中启动Activity或者Service了。至此，Android应用程序进程启动过程的源代码就分析完成了，它除了指定新的进程的入口函数是ActivityThread的main函数之外，还为进程内的Binder对象提供了Binder进程间通信机制的基础设施，由此可见，Binder进程间通信机制在Android系统中是何等的重要，而且是无处不在。
