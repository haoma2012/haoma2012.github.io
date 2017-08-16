---
title: 'Android frameWork层(二):Android应用程序安装过程PMS源代码分析'
date: 2017-08-09 22:10:37
tags:
description： PMS:PackageManagerService 包管理服务类，保存了apk相关信息，安装过程会扫描apk目录，使用PackageParse解析apk的
manifest文件，并保存Package信息;
---

<!-- more -->

- [Android应用程序安装过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6766010)

Android系统在启动的过程中，会启动一个应用程序管理服务PackageManagerService，这个服务负责扫描系统中特定的目录，找到里面的应用程序文
件，即以Apk为后缀的文件，然后对这些文件进解析，得到应用程序的相关信息。（其实最主要的是解析应用程序配置文件：AndroidManifest.xml的过
程）应用程序管理服务PackageManagerService是系统启动的时候由SystemServer组件启动的，启后它就会执行应用程序安装的过程


#### 1.源码分析

- 1.SystemServer组件是由Zygote进程负责启动的，启动的时候就会调用SystemServer.java 中的main函数：

SystemServer 是一个服务类，位于：/frameworks/base/services/java/com/android/server/SystemServer.java，管理了：PowerManagerService、 ActivityManagerService、PackageManagerService，等等
```
new SystemServer().run(); 具体查看调用的run()方法到底
执行的是什么；

// Enable the sampling profiler.
if (SamplingProfilerIntegration.isEnabled()) {启用分析器
SamplingProfilerIntegration.start();
mProfilerSnapshotTimer = new Timer();
mProfilerSnapshotTimer.schedule(new TimerTask() {
@Override
public void run() {
SamplingProfilerIntegration.writeSnapshot("system_server", null);
}
}, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);

启动线程执行写入“system_server” 进程到内存中；
SamplingProfilerIntegration.writeSnapshot("system_server", null);
...
BinderInternal.setMaxThreads(sMaxBinderThreads);//在Sysytem_Server进程中
设置binder的最大线程数，以及准备主线程
Looper.prepareMainLooper();
System.loadLibrary("android_servers");//初始化本地服务
...
mSystemServiceManager 创建SSM对象之后调用addService();
把服务加入服务列表中

...
//核心 启动服务
startBootstrapServices();
startCoreServices();
startOtherServices();
```

在类SamplingProfilerIntegration.java中，会执行相应进程写入内存文件中；

```
writeSnapshotFile(processName, packageInfo);

把进程name写入内存文件中，保存路径：/data/snapshots

这个类里面还有一个 writeZygoteSnapshot()方法，应该是把zygote进程保存
内存快照；

```

启动相应的服务：
startBootstrapServices：

```
启动：
1.mActivityManagerService活动管理器服务；
2.mPowerManagerService电源管理器服务；
3.mDisplayManagerService显示管理器服务；
4.mSystemServiceManager系统管理器服务；
5.mPackageManagerService包管理器服务；
等等
```
startCoreServices()：

```
电源服务，webview服务，usagestate 服务
mSystemServiceManager.startService(BatteryService.class);
mSystemServiceManager.startService(UsageStatsService.class);
mSystemServiceManager.startService(WebViewUpdateService.class);
```
startOtherServices()：启动其他服务相关类；

```
ConnectivityService
NetworkScoreService
WindowManagerService
NetworkManagementService
InputManagerService 
ConsumerIrService
..等等还有很多服务，比如 状态栏管理服务，复制黏贴板服务，usb服务...
执行： 启动相应服务
mSystemServiceManager.startService(“Server_name”);
```

- 2.启动PackageManagerService服务执行PackageManagerService.main；
执行路径：/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```
// Start the package manager.启动包管理服务
traceBeginAndSlog("StartPackageManagerService"); 输出log
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();
Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

//1.检查初始化设置
PackageManagerServiceCompilerMapping.checkProperties();
//2.创建PackageManagerService实例
PackageManagerService m = new PackageManagerService(context, installer, factoryTest, onlyCore);
...
//3.把这个服务添加到ServiceManager中去
ServiceManager.addService("package", m);
return m;
```

ServiceManager是Android系统Binder进程间通信机制的守护进程，负责管理系统中的Binder对象，具体可以参考第一篇binder机制分析；

下面分析创建PackageManagerService实例方法：

```
public PackageManagerService(Context context, Installer installer,boolean factoryTest, boolean onlyCore) {
...
synchronized (mInstallLock) {  
synchronized (mPackages) {  
......  
File dataDir = Environment.getDataDirectory();
mAppInstallDir = new File(dataDir, "app");
mAppLib32InstallDir = new File(dataDir, "app-lib");
mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
mAsecInternalPath = new File(dataDir,
"app-asec").getPath();
mDrmAppPrivateInstallDir = new File(dataDir, "app-private");    
.....
mFrameworkDir = new File(Environment.getRootDirectory(),
"framework");  
mDalvikCacheDir = new File(dataDir, "dalvik-cache");  

......

//最终调用下面的扫面各个文件下面的apk文件:scanDirTracedLI

scanDirTracedLI(privilegedAppDir, mDefParseFlags
| PackageParser.PARSE_IS_SYSTEM
| PackageParser.PARSE_IS_SYSTEM_DIR
| PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

.....        
}
```

- 3.这里会调用scanDirTracedLI函数来扫描移动设备下面六个目录的APK文件：

1. /vendor/app
1. /system/framework
1. /system/app
1. /data/priv-app
1. /data/app
1. /vendor/overlay


```
private void scanDirTracedLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
......
scanDirLI(dir, parseFlags, scanFlags, currentTime);
......
}

private void scanDirLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
final File[] files = dir.listFiles();
.....
for (File file : files) {
//判断是否是package路径
final boolean isPackage = (isApkFile(file) || file.isDirectory()) && !PackageInstallerService.isStageName(file.getName());
if(!isPackage){
continue;
}
try{
scanPackageTracedLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK,
scanFlags, currentTime, null);
}catch(PackageManagerException e){
......
}
}
}
```
对于目录中的每一个文件，如果是以后Apk作为后缀名，那么先调用scanPackageTracedLI函数，最终调用scanPackageLI来对它进行解析和安装。

```
private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
long currentTime, UserHandle user) throws PackageManagerException {
//1.创建一个PackageParser实例
PackageParser pp = new PackageParser();
......
//2.调用parsePackage来对apk路径解析
pkg = pp.parsePackage(scanFile, parseFlags);
......
return scanPackageLI(pkg, scanFile, parseFlags, scanFlags, currentTime, user);
}
```
这个函数首先会为这个Apk文件创建一个PackageParser实例，接着调用这个实例的parsePackage函数来对这个Apk文件进行解析。这个函数最后还会调用另外一个版本的scanPackageLI函数把来解析后得到的应用程序信息保存在PackageManagerService中。

- 4.PackageParser.parsePackage这个函数定义在frameworks/base/core/java/android/content/pm/PackageParser.java文件中：
```
public class PackageParser {  
......  

public Package parsePackage(File packageFile, int flags)
if (packageFile.isDirectory()) {
return parseClusterPackage(packageFile, flags);
} else {
return parseMonolithicPackage(packageFile, flags);
}
} 
```

首先判断当前文件是否是一个文件夹是执行parseClusterPackage，否则执行parseMonolithicPackage；

我们先看是一个文件夹的情况调用parseClusterPackage：

```
private Package parseClusterPackage(File packageDir, int flags){
//1.解析文件夹下面的apk文件信息返回一个PackageLite对象
final PackageLite lite = parseClusterPackageLite(packageDir, 0);
.......
final AssetManager assets = new AssetManager();//2.创建AssentManager对象
//3.把apk基本信息载入到assets里，作用是解析manifest文件时资源可以快速重写
loadApkIntoAssetManager(assets, lite.baseCodePath, flags);
......
for (String path : lite.splitCodePaths) {
loadApkIntoAssetManager(assets, path, flags);
}

......
//4.新建一个baseApk 文件，把assets信息解析保存到pkg中
final File baseApk = new File(lite.baseCodePath);
final Package pkg = parseBaseApk(baseApk, assets, flags);
//5.pkg重新赋值
pkg.splitNames = lite.splitNames;
pkg.splitCodePaths = lite.splitCodePaths;
pkg.splitRevisionCodes = lite.splitRevisionCodes;
pkg.splitFlags = new int[num];
pkg.splitPrivateFlags = new int[num];
}
```

loadApkIntoAssetManager函数：主要是确保在assent 路径返回的唯一性使用cookie，如果文件在AssetManager存在，调用 int cookie = assets.addAssetPath(apkPath); 就会返回唯一的cookie值；

parseBaseApk函数：创建XmlResourceParser对象，去解析AndroidManifest.xml文件
```
private Package parseBaseApk(File apkFile, AssetManager assets, int flags){
.......

XmlResourceParser parser = null;
res = new Resources(assets, mMetrics, null);
parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
//调用parseBaseApk方法解析，参数不一样；返回package信息
final Package pkg = parseBaseApk(res, parser, flags, outError);
.......
pkg.setVolumeUuid(volumeUuid);
pkg.setVolumeUuid(volumeUuid);
pkg.setVolumeUuid(volumeUuid);
pkg.setVolumeUuid(volumeUuid);

return pkg;
}
```

- 5.继续调用：parseBaseApk(Resources res, XmlResourceParser parser, int flags, String[] outError) 这里才是真正对apk的manifest文件做解析；

每一个Apk文件都是一个归档文件，它里面包含了Android应用程序的配置文件AndroidManifest.xml，这里主要就是要对这个配置文件就行解析了，从Apk归档文件中得到这个配置文件后，就调用另一外版本的parseBaseApk函数对这个应用程序进行解析了：

Parse the manifest of a <em>base APK</em>.解析apk的manifest文件
```
private Package parseBaseApk(Resources res, XmlResourceParser parser, int flags, String[] outError) {
......
//1.拿到manifest自定义属性得到mVersionCode，mVersionName等等
final Package pkg = new Package(pkgName);
boolean foundApp = false;

TypedArray sa = res.obtainAttributes(attrs,
com.android.internal.R.styleable.AndroidManifest);
pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
pkg.baseRevisionCode = sa.getInteger(
com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
pkg.mVersionName = sa.getNonConfigurationString(
com.android.internal.R.styleable.AndroidManifest_versionName, 0);
if (pkg.mVersionName != null) {
pkg.mVersionName = pkg.mVersionName.intern();
}
......

//2.XmlResourceParser 解析xml文件
while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
&& (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
continue;
}
String tagName = parser.getName();
if (tagName.equals("application")) {
......
foundApp = true;
//3.解析application节点
if (!parseBaseApplication(pkg, res, parser, attrs, flags, outError)) {
return null;
}
}else if (tagName.equals("key-sets")){
......
}else if (tagName.equals("permission-group")) {  
......  
} else if (tagName.equals("permission")) {  
......  
} else if (tagName.equals("permission-tree")) {  
......  
} else if (tagName.equals("uses-permission")) {  
......  
} else if (tagName.equals("uses-configuration")) {  
......  
} else if (tagName.equals("uses-feature")) {  
......  
} else if (tagName.equals("uses-sdk")) {  
......  
} else if (tagName.equals("supports-screens")) {  
......  
} else if (tagName.equals("protected-broadcast")) {  
......  
} else if (tagName.equals("instrumentation")) {  
......  
} else if (tagName.equals("original-package")) {  
......  
} else if (tagName.equals("adopt-permissions")) {  
......  
} else if (tagName.equals("uses-gl-texture")) {  
......  
} else if (tagName.equals("compatible-screens")) {  
......  
} else if (tagName.equals("eat-comment")) {  
......  
} else if (RIGID_PARSER) {  
......  
} else {  
......  
} 
.......
}
return pkg;
}
```
这里就是对AndroidManifest.xml文件中的各个标签进行解析了，各个标签的含义可以参考官方文档http://developer.android.com/guide/topics/manifest/manifest-intro.html，简单看下parseBaseApplication解析application节点


```
private boolean parseBaseApplication(Package owner, Resources res,
XmlPullParser parser, AttributeSet attrs, int flags, String[] outError)
throws XmlPullParserException, IOException {
final ApplicationInfo ai = owner.applicationInfo;  
final String pkgName = owner.applicationInfo.packageName;  

TypedArray sa = res.obtainAttributes(attrs,  
com.android.internal.R.styleable.AndroidManifestApplication);  

......  

int type;  
while ((type=parser.next()) != parser.END_DOCUMENT  
&& (type != parser.END_TAG || parser.getDepth() > innerDepth)) {  
if (type == parser.END_TAG || type == parser.TEXT) {  
continue;  
}  

String tagName = parser.getName();  
if (tagName.equals("activity")) {  
Activity a = parseActivity(owner, res, parser, attrs, flags, outError, false);  
......  

owner.activities.add(a);  

} else if (tagName.equals("receiver")) {  
Activity a = parseActivity(owner, res, parser, attrs, flags, outError, true);  
......  

owner.receivers.add(a);  
} else if (tagName.equals("service")) {  
Service s = parseService(owner, res, parser, attrs, flags, outError);  
......  

owner.services.add(s);  
} else if (tagName.equals("provider")) {  
Provider p = parseProvider(owner, res, parser, attrs, flags, outError);  
......  

owner.providers.add(p);  
} else if (tagName.equals("activity-alias")) {  
Activity a = parseActivityAlias(owner, res, parser, attrs, flags, outError);  
......  

owner.activities.add(a);  
} else if (parser.getName().equals("meta-data")) {  
......  
} else if (tagName.equals("uses-library")) {  
......  
} else if (tagName.equals("uses-package")) {  
......  
} else {  
......  
}  
}  

return true;  
}  

......  

}
```
这里就是对AndroidManifest.xml文件中的application标签进行解析了，我们常用到的标签就有activity、service、receiver和provider; 这是文件夹下面对apk文件的解析；如果不是则会执行 parseMonolithicPackage方法


```
if (mOnlyCoreApps) {
final PackageLite lite = parseMonolithicPackageLite(apkFile, flags);
if (!lite.coreApp) {
throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
"Not a coreApp: " + apkFile);
}
}
final AssetManager assets = new AssetManager();
try {
final Package pkg = parseBaseApk(apkFile, assets, flags);
pkg.codePath = apkFile.getAbsolutePath();
return pkg;
} finally {
IoUtils.closeQuietly(assets);
}
```
里面最后也会执行parseBaseApk(apkFile, assets, flags)方法，得到package信息，然后就会和之前分析的一样会执行四个参数的parseBaseApk真正解析manifest.xml文件得到包信息；得到的PackageParser.Package对象保存到PackageManagerService服务中；

#### 分析汇总

- 1.SystemServer组件由Zygote进程负责启动；
- 2.SystemServer 是一个管理服务类；负责启动服务:startBootstrapServices();
startCoreServices();startOtherServices();启动PackageManagerService
- 3.把包服务加入到ServiceManager中，ServiceManager是Android系统Binder进程间通信机制的守护进程，保存的也是ServiceManager.addService("package", m);服务相对的package的name；
- 4.PackageManagerService调用scanDirTracedLI负责扫描相应几个目录下的apk文件
- 5.PackageParser的parsePackage方法就是为了解析apk文件中的manifest.xml文件并且保存到PMS中；


------
这样，在Android系统启动的时候安装应用程序的过程就介绍完了，但是，这些应用程序只是相当于在PackageManagerService服务注册好了，如果我们想要在Android桌面上看到这些应用程序，还需要有一个Home应用程序，负责从PackageManagerService服务中把这些安装好的应用程序取出来，并以友好的方式在桌面上展现出来，例如以快捷图标的形式。在Android系统中，负责把系统中已经安装的应用程序在桌面中展现出来的Home应用程序就是Launcher了；在下一篇文章中，我们将介绍Launcher是如何启动的以及它是如何从PackageManagerService服务中把系统中已经安装好的应用程序展现出来的。



//old version pic；供参考
应用程序管理服务PackageManagerService从启动到安装应用程序的过程如下图所示：
![image](http://hi.csdn.net/attachment/201109/10/0_1315661784a77A.gif)
