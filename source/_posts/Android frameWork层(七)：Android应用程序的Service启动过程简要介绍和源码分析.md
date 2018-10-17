---
title: Android frameWork层(七)：  Android应用程序的Service启动过程简要介绍和源码分析
date: 2017-08-30 22:53:05
tags:
---

在编写Android应用程序时，我们一般将一些计算型的逻辑放在一个独立的进程来处理，这样主进程仍然可以流畅地响应界面事件，提高用户体验。android系统为我们提供了一个Service类，我们可以实现一个以Service为基类的服务子类，在里面实现自己的计算型逻辑，然
后在主进程通过startService函数来启动这个服务。在本文中，将详细分析主进程是如何通过startService函数来在新进程中启动自定义服务的。

在主进程调用startService函数时，会通过Binder进程间通信机制来通知ActivitManagerService来创建新进程，并且启动指定的服务。

<!-- more -->

#### [Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)


先来看看Activity的类图：
![image](http://hi.csdn.net/attachment/201108/10/0_1312984689zu5P.gif)

- 从图中可以看出，Activity继承了ContextWrapper类，而在ContextWrapper类中，实现了startService函数。在ContextWrapper类中，有一个成员变量mBase，它是一个ContextImpl实例，而ContextImpl类和ContextWrapper类一样继承于Context类，ContextWrapper类的startService函数最终过调用ContextImpl类的startService函数来实现。这种类设计方法在设计模式里面，就称之为装饰模式（Decorator），或者包装模式（Wrapper）。
- 在ContextImpl类的startService类，最终又调用了ActivityManagerProxy类的startService来实现启动服务的操作，看到这里的Proxy关键字，回忆一下前面Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析这篇文章，就会知道ActivityManagerProxy是一个Binder对象的远程接口了，而这个Binder对象就是我们前面所说的ActivityManagerService了。

#### ContextImpl类  startService(Intent service)->startServiceCommon(Intent service, UserHandle user)
```
ComponentName cn = ActivityManagerNative.getDefault().startService(
mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
getContentResolver()), getOpPackageName(), user.getIdentifier());
```
ActivityManagerNative.getDefault()得到的就是：系统全局的activity manager，返回值IActivityManager，这个接口的实现类就是ActivityManagerProxy
我们在binder机制文章有ServiceManagerProxy：
![image](http://hi.csdn.net/attachment/201107/28/0_1311872523tLjI.gif)

- IServiceManager提供了getService和addService两个成员函数来管理系统中的Service。从ServiceManagerProxy类的构造函数可以看出，它需要一个BinderProxy对象的IBinder接口来作为参数。因此，要获取Service Manager的Java远程接口ServiceManagerProxy，首先要有一个BinderProxy对象。
- ActivityManagerProxy是一个Binder对象的远程接口了，而这个Binder对象就是我们前面所说的ActivityManagerService了。
- ActivityManagerService它是Binder进程间通信机制中的Server角色，它是SystemServer启动的。
- 首先是调用ActivityManagerService.main函数来创建一个ActivityManagerService实例，然后通过调用ActivityManagerService.setSystemProcess函数把这个Binder实例添加Binder进程间通信机制的守护进程ServiceManager中去
- ActivityManagerProxy类的startService函数把这三个参数(参数service是一个Intent实例，它里面指定了要启动的服务的名称；参数caller是一个IApplicationThread实例，它是一个在主进程创建的一个Binder对象；参数resolvedType是一个字符串，它表示service这个Intent的MIME类型，)写入到data本地变量去，接着通过mRemote.transact函数进入到Binder驱动程序，然后Binder驱动程序唤醒正在等待Client请求的ActivityManagerService进程


**ActivityManagerService的startService函数的处理流程如下图所示**：

![image](http://hi.csdn.net/attachment/201108/10/0_13129910510yBK.gif)

- 一. Step 1至Step 7，从主进程调用到ActivityManagerService进程中，完成新进程的创建；
- 二. Step 8至Step 11，从新进程调用到ActivityManagerService进程中，获取要在新进程启动的服务的相关信息；
- 三. Step 12至Step 20，从ActivityManagerService进程又回到新进程中，最终将服务启动起来。


#### [Android应用程序绑定服务（bindService）的过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6745181)

Android应用程序组件Service与Activity一样，既可以在新的进程中启动，也可以在应用程序进程内部启动；前面我们已经分析了在新的进程中启动Service的过程，本文将要介绍在应用程序内部绑定Service的过程，这是一种在应用程序进程内部启动Service的方法。

绑定服务过程：整个调用过程为MainActivity.bindService->CounterService.onCreate->CounterService.onBind->MainActivity.ServiceConnection.onServiceConnection->CounterService.CounterBinder.getService。

![image](http://hi.csdn.net/attachment/201109/3/0_1315064144wnUc.gif)

- 1.ContextWrapper.bindService 返回mBase.bindService(service, conn, flags)，mBase是一个ContextImpl实例变量，于是就调用ContextImpl的bindService函数来进一步处理。
- ContextImpl.bindService：

```
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
UserHandle user) {
IServiceConnection sd;
if (conn == null) {
throw new IllegalArgumentException("connection is null");
}
if (mPackageInfo != null) {
sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
mMainThread.getHandler(), flags);
} else {
throw new RuntimeException("Not supported in system context");
}
validateServiceIntent(service);
try {
IBinder token = getActivityToken();
if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
&& mPackageInfo.getApplicationInfo().targetSdkVersion
< android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
flags |= BIND_WAIVE_PRIORITY;
}
service.prepareToLeaveProcess();
int res = **ActivityManagerNative.getDefault().bindService(
mMainThread.getApplicationThread(), getActivityToken(), service,
service.resolveTypeIfNeeded(getContentResolver()),
sd, flags, getOpPackageName(), user.getIdentifier());**
if (res < 0) {
throw new SecurityException(
"Not allowed to bind to service " + service);
}
return res != 0;
} catch (RemoteException e) {
throw new RuntimeException("Failure from system", e);
}
}
```

mPackageInfo.getServiceDispatcher函数来获得一个IServiceConnection接口，这里的mPackageInfo的类型是LoadedApk，我们来看看它的getServiceDispatcher函数的实现，然后再回到ContextImpl.bindService函数来。

在getServiceDispatcher函数中，传进来的参数context是一个MainActivity实例，先以它为Key值在mServices中查看一下，是不是已经存在相应的ServiceDispatcher实例，如果有了，就不用创建了，直接取出来。在我们这个情景中，需要创建一个新的ServiceDispatcher。在创建新的ServiceDispatcher实例的过程中，将上面传下来ServiceConnection参数c和Hanlder参数保存在了ServiceDispatcher实例的内部，并且创建了一个InnerConnection对象，这是一个Binder对象，一会是要传递给ActivityManagerService的，ActivityManagerServic后续就是要通过这个Binder对象和ServiceConnection通信的。

函数getServiceDispatcher最后就是返回了一个InnerConnection对象给ContextImpl.bindService函数。回到ContextImpl.bindService函数中，它接着就要调用ActivityManagerService的远程接口来进一步处理了。

- 3.ActivityManagerProxy.bindService：这个函数通过Binder驱动程序就进入到ActivityManagerService的bindService函数去了。mRemote就是一个binder对象

```
public int bindService(IApplicationThread caller, IBinder token,
Intent service, String resolvedType, IServiceConnection connection,
int flags,  String callingPackage, int userId) throws RemoteException {
Parcel data = Parcel.obtain();
Parcel reply = Parcel.obtain();
data.writeInterfaceToken(IActivityManager.descriptor);
data.writeStrongBinder(caller != null ? caller.asBinder() : null);
data.writeStrongBinder(token);
service.writeToParcel(data, 0);
data.writeString(resolvedType);
data.writeStrongBinder(connection.asBinder());
data.writeInt(flags);
data.writeString(callingPackage);
data.writeInt(userId);
mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);
reply.readException();
int res = reply.readInt();
data.recycle();
reply.recycle();
return res;
}
```
- 4. ActivityManagerService.bindService：调用ActiveServices.bindServiceLocked

```
synchronized(this) {
return mServices.bindServiceLocked(caller, token, service,
resolvedType, connection, flags, callingPackage, userId);
}


int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
String resolvedType, IServiceConnection connection, int flags,
String callingPackage, int userId) throws TransactionTooLargeException {
final ProcessRecord callerApp = getRecordForAppLocked(caller);  
......  

ActivityRecord activity = null;  
if (token != null) {  
int aindex = mMainStack.indexOfTokenLocked(token);  
......  
activity = (ActivityRecord)mMainStack.mHistory.get(aindex);  
}  

......  

ServiceLookupResult res =  
retrieveServiceLocked(service, resolvedType,  
Binder.getCallingPid(), Binder.getCallingUid());  

......  

ServiceRecord s = res.record;  

final long origId = Binder.clearCallingIdentity();  

......  

AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);  
ConnectionRecord c = new ConnectionRecord(b, activity,  
connection, flags, clientLabel, clientIntent);  

IBinder binder = connection.asBinder();  
ArrayList<ConnectionRecord> clist = s.connections.get(binder);  

if (clist == null) {  
clist = new ArrayList<ConnectionRecord>();  
s.connections.put(binder, clist);  
}  
clist.add(c);  
b.connections.add(c);  
if (activity != null) {  
if (activity.connections == null) {  
activity.connections = new HashSet<ConnectionRecord>();  
}  
activity.connections.add(c);  
}  
b.client.connections.add(c);  
clist = mServiceConnections.get(binder);  
if (clist == null) {  
clist = new ArrayList<ConnectionRecord>();  
mServiceConnections.put(binder, clist);  
}  

clist.add(c);  

if ((flags&Context.BIND_AUTO_CREATE) != 0) {  
......  
if (!bringUpServiceLocked(s, service.getFlags(), false)) {  
return 0;  
}  
}  

......  
}  

return 1;  
}             

......      

}
```

函数首先根据传进来的参数token是MainActivity在ActivityManagerService里面的一个令牌，通过这个令牌就可以将这个代表MainActivity的ActivityRecord取回来了。

接着通过retrieveServiceLocked函数，得到一个ServiceRecord，这个ServiceReocrd描述的是一个Service对象，这里就是CounterService了，这是根据传进来的参数service的内容获得的。

接下来，就是把传进来的参数connection封装成一个ConnectionRecord对象。注意，这里的参数connection是一个Binder对象，它的类型是LoadedApk.ServiceDispatcher.InnerConnection，是在Step 4中创建的，后续ActivityManagerService就是要通过它来告诉MainActivity，CounterService已经启动起来了，因此，这里要把这个ConnectionRecord变量c保存下来，它保在在好几个地方，都是为了后面要用时方便地取回来的，这里就不仔细去研究了，只要知道ActivityManagerService要使用它时就可以方便地把它取出来就可以了，具体后面我们再分析。

最后，传进来的参数flags的位Context.BIND_AUTO_CREATE为1（参见上面MainActivity.onCreate函数调用bindService函数时设置的参数），因此，这里会调用bringUpServiceLocked函数进一步处理。

- 5.ActivityManagerService.bringUpServiceLocked：

```
if (app != null && app.thread != null) {
try {
app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
realStartServiceLocked(r, app, execInFg);
return null;
} catch (TransactionTooLargeException e) {
throw e;
} catch (RemoteException e) {
Slog.w(TAG, "Exception when starting service " + r.shortName, e);
}
```
我们没有在程序的AndroidManifest.xml配置文件中设置CounterService的process属性值，因此，它默认就为application标签的process属性值，而application标签的process属性值也没有设置，于是，它们就默认为应用程序的包名了，这里取回来的app和app.thread均不为null，于是，就执行realStartServiceLocked函数来执行下一步操作了。
- 6.ActivityManagerService.realStartServiceLocked

```
private final void realStartServiceLocked(ServiceRecord r,  
ProcessRecord app) throws RemoteException {  
......  
r.app = app;  
......  

app.services.add(r);  
......  

try {  
......  
app.thread.scheduleCreateService(r, r.serviceInfo);  
......  
} finally {  
......  
}  

requestServiceBindingsLocked(r);  

......  
}  

......  
```
这个函数执行了两个操作，一个是操作是调用app.thread.scheduleCreateService函数来在应用程序进程内部启动CounterService，这个操作会导致CounterService的onCreate函数被调用；另一个操作是调用requestServiceBindingsLocked函数来向CounterService要一个Binder对象，这个操作会导致CounterService的onBind函数被调用。

app.thread是一个Binder对象的远程接口，类型为ApplicationThreadProxy。每一个Android应用程序进程里面都有一个ActivtyThread对象和一个ApplicationThread对象，其中是ApplicationThread对象是ActivityThread对象的一个成员变量，是ActivityThread与ActivityManagerService之间用来执行进程间通信的，ApplicationThreadProxy.scheduleCreateService通过Binder驱动程序就进入到ApplicationThread的scheduleCreateService函数去了。然后调用ActivityThread的queueOrSendMessage函数把一个H.CREATE_SERVICE类型的消息放到ActivityThread的消息队列中去。mH.sendMessage(target)最终消息处理：

- 7.ActivityThread.handleCreateService

```
private void handleCreateService(CreateServiceData data) {
// If we are getting ready to gc after going to the background, well
// we are back active so skip it.
unscheduleGcIdler();

LoadedApk packageInfo = getPackageInfoNoCheck(
data.info.applicationInfo, data.compatInfo);
Service service = null;
try {
java.lang.ClassLoader cl = packageInfo.getClassLoader();
service = (Service) cl.loadClass(data.info.name).newInstance();
} catch (Exception e) {
if (!mInstrumentation.onException(service, e)) {
throw new RuntimeException(
"Unable to instantiate service " + data.info.name
+ ": " + e.toString(), e);
}
}

try {
if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
context.setOuterContext(service);

Application app = packageInfo.makeApplication(false, mInstrumentation);
service.attach(context, this, data.info.name, data.token, app,
ActivityManagerNative.getDefault());
service.onCreate();
mServices.put(data.token, service);
try {
ActivityManagerNative.getDefault().serviceDoneExecuting(
data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
} catch (RemoteException e) {
// nothing to do.
}
} catch (Exception e) {
if (!mInstrumentation.onException(service, e)) {
throw new RuntimeException(
"Unable to create service " + data.info.name
+ ": " + e.toString(), e);
}
}
}
```
个函数的工作就是把CounterService类加载到内存中来，然后调用它的onCreate函数。

- 8. CounterService.onCreate():创建Service，这样CounterService就创建起来了；至此，应用程序绑定服务过程中的第一步MainActivity.bindService->CounterService.onCreate就完成了。
- 9.ActivityManagerService.requestServiceBindingsLocked这个调用是用来执行CounterService的onBind函数的。


```
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
throws TransactionTooLargeException {
for (int i=r.bindings.size()-1; i>=0; i--) {
IntentBindRecord ibr = r.bindings.valueAt(i);
if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
break;
}
}
}

private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
boolean execInFg, boolean rebind) throws TransactionTooLargeException {
......
if ((!i.requested || rebind) && i.apps.size() > 0) {
try {
bumpServiceExecutingLocked(r, execInFg, "bind");
r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind, r.app.repProcState);
if (!rebind) {
i.requested = true;
}
i.hasBound = true;
i.doRebind = false;
} catch (TransactionTooLargeException e) {
.....
} catch (RemoteException e) {
.....
}
}
return true;
}
```
这里的参数r就是我们在前面的前面中创建的ServiceRecord了，它代表刚才已经启动了的CounterService。函数requestServiceBindingsLocked调用了requestServiceBindingLocked函数来处理绑定服务的操作，而函数requestServiceBindingLocked又调用了app.thread.scheduleBindService函数执行操作，前面我们已经介绍过app.thread，它是一个Binder对象的远程接口，类型是ApplicationThreadProxy。

- 10.调用ApplicationThreadProxy.scheduleBindService,[调用ApplicationThreadProxy 是调用ApplicationThreadNative 内部类]方法里面会Binder驱动程序就进入
到ActivityThread的scheduleBindService函数去了。

- 11.ActivityThread.scheduleBindService :调用ActivityThread.queueOrSendMessage函数来发送消息。


```
public final void scheduleBindService(IBinder token, Intent intent,
boolean rebind, int processState) {
updateProcessState(processState, false);
BindServiceData s = new BindServiceData();
s.token = token;
s.intent = intent;
s.rebind = rebind;

if (DEBUG_SERVICE)
Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
+ Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
sendMessage(H.BIND_SERVICE, s);
}


处理消息： mH.sendMessage(msg); mH就是一个handler消息处理

case BIND_SERVICE:
Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
handleBindService((BindServiceData)msg.obj);
Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
break;
```

- 12.handleBindService((BindServiceData)msg.obj)

```
private void handleBindService(BindServiceData data) {
Service s = mServices.get(data.token);
if (s != null) {
try {
data.intent.setExtrasClassLoader(s.getClassLoader());
data.intent.prepareToEnterProcess();
try {
if (!data.rebind) {
IBinder binder = s.onBind(data.intent);
ActivityManagerNative.getDefault().publishService(
data.token, data.intent, binder);
} else {
s.onRebind(data.intent);
ActivityManagerNative.getDefault().serviceDoneExecuting(
data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
}
ensureJitEnabled();
} catch (RemoteException ex) {
}
} catch (Exception e) {
if (!mInstrumentation.onException(s, e)) {
throw new RuntimeException(
"Unable to bind to service " + s
+ " with " + data.intent + ": " + e.toString(), e);
}
}
}

}
```
在前面执行ActivityThread.handleCreateService函数中，已经将这个CounterService实例保存在mServices中，因此，这里首先通过data.token值将它取回来，保存在本地变量s中，接着执行了两个操作，一个操作是调用s.onBind，即CounterService.onBind获得一个Binder对象，另一个操作就是把这个Binder对象传递给ActivityManagerService。

- 13.IBinder binder = s.onBind(data.intent);完成执行service的绑定；

接下来会执行：MainActivity.ServiceConnection.onServiceConnection和第四步CounterService.CounterBinder.getService


#### 总结

这样，Android应用程序绑定服务（bindService）的过程的源代码分析就完成了，总结一下这个过程：

- 1. Step 1 -  Step14，MainActivity调用bindService函数通知ActivityManagerService，它要启动CounterService这个服务，ActivityManagerService于是在MainActivity所在的进程内部把CounterService启动起来，并且调用它的onCreate函数；

- 2. Step 15 - Step21，ActivityManagerService把CounterService启动起来后，继续调用CounterService的onBind函数，要求CounterService返回一个Binder对象给它；

- 3. Step 22 - Step29，ActivityManagerService从CounterService处得到这个Binder对象后，就把它传给MainActivity，即把这个Binder对象作为参数传递给MainActivity内部定义的ServiceConnection对象的onServiceConnected函数；

- 4. Step 30，MainActivity内部定义的ServiceConnection对象的onServiceConnected函数在得到这个Binder对象后，就通过它的getService成同函数获得CounterService接口。
