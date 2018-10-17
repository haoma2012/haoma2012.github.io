---
title: Android Multidex分包体验
date: 2018-05-15 11:13:19
categories:
- Android进阶
tags: 
- Android进阶
---

Android 的 `classLoader` 在加载 APK 的时候限制了`class.dex` 包含的 Java 方法数，其总数不能超过65535（64k)解决方案

<!-- more -->


> Android 的 `classLoader` 在加载 APK 的时候限制了`class.dex` 包含的 Java 方法数，其总数不能超过65535（64K，**不要再说成 65K** 了，1K = 2^10 = 1024 , 64 * 1024 = 65535），Google 官方给出的解决方案是使用 [Multidex](https://developer.android.com/studio/build/multidex.html) 。

1.修改gradle脚本来产生多dex。

```
defaultConfig {
        applicationId "com.ebmob.testluaview"
        minSdkVersion 14
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        multiDexEnabled true
        ndk {
            abiFilters "armeabi-v7a", "armeabi", "x86"
        }
    }
dependencies {
    ...
    ...
   //multidex 分包
    implementation 'com.android.support:multidex:1.0.2'
}
```

2.修改manifest 使用MulitDexApplication 或者重写attachBaseContext。

```
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    MultiDex.install(this);     
 
}
```

**MultiDex带来的问题**

> 1.在冷启动时因为需要安装DEX文件，如果DEX文件过大时，处理时间过长，很容易引发ANR（Application Not Responding）；
> 2.采用MultiDex方案的应用可能不能在低于Android 4.0 (API level 14) 机器上启动，这个主要是因为Dalvik linearAlloc的一个bug ([Issue 22586](http://b.android.com/22586));
> 3. 采用MultiDex方案的应用因为需要申请一个很大的内存，在运行时可能导致程序的崩溃，这个主要是因为Dalvik linearAlloc 的一个限制([Issue 78035](http://b.android.com/78035)). 这个限制在 Android 4.0 (API level 14)已经增加了, 应用也有可能在低于 Android 5.0 (API level 21)版本的机器上触发这个限制；

```
MultiDex的基本原理是把通过DexFile来加载Secondary DEX，并存放在
BaseDexClassLoader的DexPathList中。

```

- [Android Dex分包之旅](http://yydcdut.com/2016/03/20/split-dex/)

> INSTALL_FAILED_DEXOPT 出现的原因大部分都是两种，一种是 65536 了，另外一种是 LinearAlloc 太小了。两者的限制不同，但是原因却是相似，那就是App太大了，导致没办法安装到手机上。

*为什么是65536*
```
是因为 Dalvik 的 invoke-kind 指令集中，method reference index 只留了 
16 bits，最多能引用 65535 个方法
```

**解决 LinearAlloc**
```
afterEvaluate { 
  tasks.matching { 
    it.name.startsWith('dex') 
  }.each { dx -> 
    if (dx.additionalParameters == null) { 
      dx.additionalParameters = []
    }  
    dx.additionalParameters += '--set-max-idx-number=48000' 
  } 
}
```
--set-max-idx-number= 用于控制每一个 dex 的最大方法个数。

**dexopt && dex2oat**
![image](http://upload-images.jianshu.io/upload_images/4267785-d5af831c696a6127.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### dexopt
> 当 Android 系统安装一个应用的时候，有一步是对 Dex 进行优化，这个过程有一个专门的工具来处理，叫 DexOpt。DexOpt 是在第一次加载 Dex 文件的时候执行的，将 dex 的依赖库文件和一些辅助数据打包成 odex 文件，即 Optimised Dex，存放在 cache/dalvik_cache 目录下。保存格式为 apk路径 @ apk名 @ classes.dex 。执行 ODEX 的效率会比直接执行 Dex 文件的效率要高很多。
首次启动时Dalvik虚拟机会对classes.dex执行dexopt操作，生成ODEX文件，这个过程非常耗时，而执行MultiDex.install()必然会再次对classes2.dex执行dexopt等操作，所有这些操作必须在5秒内完成，否则就ANR给你看！

更多可查看 [Dalvik Optimization and Verification With *dexopt*](http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html) 。


#### dex2oat
> Android Runtime 的 dex2oat 是将 dex 文件编译成 oat 文件。而 oat 文件是 elf 文件，是可以在本地执行的文件，而 Android Runtime 替换掉了虚拟机读取的字节码转而用本地可执行代码，这就被叫做 AOT(ahead-of-time)。dex2oat 对所有 apk 进行编译并保存在 dalvik-cache 目录里。PackageManagerService 会持续扫描安装目录，如果有新的 App 安装则马上调用 dex2oat 进行编译。

更多可查看 [Android运行时ART简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/39256813) 。

> Multidex的实现原理是将class编译进不同的classes.dex文件中，一般情况下，一个APK文件中只包含了一个classes.dex文件。分包之后就存在一个主的classes.dex，多个副的classes2.dex，classes3.dex，以此类推。通过查看MultiDex的源码，可以发现MultiDex在冷启动时，因为会同步的反射安装Dex文件，进行IO操作，容易导致ANR

1.在冷启动时因为需要安装Dex文件，如果Dex文件过大时，处理时间过长，很容易引发ANR
2.采用MultiDex方案的应用因为linearAlloc的BUG，可能不能在2.x设备上启动

*看下Multidex的编译过程，它由三个不同的gradle task组成:*

```
1.collect{variant}MultiDexComponents task:
读取项目的AndroidManifest.xml文件中注册的application、Activity、service、receiver、provider、
instrumentation相关类，并将其class文件路径写到文件
buidl/intermediates/multi-dex/${variant.dirName}/manifest_keep.txt中

2.shrink{variant}MultiDexComponents task:
调用ProGuard并根据上一步生成的manifest_keep.txt文件内容去压缩class，剔除没有用到的class，
生成一个精简的jar包buidl/intermediates/multi-dex/${variant.dirName}/componentClasses.jar

3.create{variant}MainDexClassList task:
根据上一步生成的componentClasses.jar去寻找这里面的各个class文字中依赖的class，比如一个class中有一成员变量X，那么X就是依赖的class，
componentClasses.jar中所有的class和依赖的class路径都会被写入到文件buidl/intermediates/multi-dex/${variant.dirName}/maindexlist.txt中，
这个文件中的类都会被编译进主的classes.dex中去。

编译成功后可以查看到：
componentClasses.jar
components.flags
manifest_keep.txt
maindexlist.txt
```

#### 参考文章

- [其实你不知道MultiDex到底有多坑](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1218/3789.html)
- [Android傻瓜式分包插件](https://blog.csdn.net/amazing_alex/article/details/52064837)

- [美团多dex拆包方案](http://tech.meituan.com/mt-android-auto-split-dex.html)

> 精简主dex+异步加载secondary.dex 。对异步化执行速度的不确定性，他们的解决方案是重写Instrumentation execStartActivity 方法，hook跳转Activity的总入口做判断，如果当前secondary.dex 还没有加载完成，就弹一个loading Activity等待加载完成

FB的解决思路特别赞，让Launcher Activity在另外一个进程启动！当然这个Launcher Activity就是用来load dex 的 ，load完成就启动Main Activity。
```
 class LoadDexTask extends AsyncTask {

        @Override
        protected void onPreExecute() {
        }

        @Override
        protected Object doInBackground(Object[] params) {
            try {
                Log.d("loadDex", "install start");
                MultiDex.install(getApplication());
                Log.d("loadDex", "install finish");
                //((LinganApplication)getApplication()).installFinish(getApplication());

                LinganApplicationTinker.installFinish(getApplication());
            } catch (Exception e) {
                Log.e("loadDex", e.getLocalizedMessage());
            }
            return null;
        }

        @Override
        protected void onPostExecute(Object o) {
            Log.d("loadDex", "get install finish");
            finish();
            System.exit(0);
        }
    }

android:process=":mini"注册
```
*Android的打包流程示意图：*
![打包流程](http://upload-images.jianshu.io/upload_images/4267785-cb3e8ce06232bb89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)






[百转千回的 too many classes in --main-dex-list
](https://juejin.im/entry/58d54e2d44d90400686856f3)

-  1.首先是manifest_keep.txt大小的控制:collect task任务进行拦截和替换

```
afterEvaluate {
        android.applicationVariants.each {
            variant ->
                def collectTask = tasks.findByName("collect${variant.name.capitalize()}MultiDexComponents")//collectZroTestDebugMultiDexComponents
                if (collectTask != null) {
                    List<Action<? super Task>> list = new ArrayList<>()
                    list.add(new Action<Task>() {
                        @Override
                        void execute(Task task) {
                            println "collect${variant.name.capitalize()}MultiDexComponents action execute!---------XXXXXXX mini main dex生效了!!!!$projectDir"
                            def dir = new File("$projectDir/build/intermediates/multi-dex/${variant.dirName}");
                            if (!dir.exists()) {
                                println "$dir 不存在,进行创建"
                                dir.mkdirs()
                            }
                            def manifestkeep = new File(dir.getAbsolutePath() + "/manifest_keep.txt")
                            manifestkeep.delete()
                            manifestkeep.createNewFile()
                            println "先删除,后创建manifest_keep"
                            def backManifestListFile = new File("$projectDir/manifest_keep.txt")
                            backManifestListFile.eachLine {
                                line ->
                                    manifestkeep << line << '\n'
                            }
                        }
                    })
                    collectTask.setActions(list)
                }
        }
    }
```

- 2.然后在本地的projectDir/manifest_keep.txt配置最精简的主dex所需要依赖的类：


```
-keep class com.ebmob.framework.ui.LoadResActivity { <init>(); }
-keep class com.lingan.seeyou.messagein.NotificationTranslucentActivity{ <init>(); }
-keep class com.j256.ormlite.field.**
-keep class com.ebmob.testluaview.LuaViewApplication {
    *;
}
-keep class com.ebmob.framework.ui.LinganApplicationTinker {
    *;
}
保留了application和启动页之类的入口；
```

- 3.使用了一个可以干预maindexlist.txt文件内容的项目[DexKnifePlugin](https://juejin.im/entry/58d54e2d44d90400686856f3 "https://github.com/ceabie/DexKnifePlugin")
配置如下：
```
# 全局过滤, 如果没设置 -filter-suggest 并不会应用到 建议的maindexlist.
# 如果你想要某个已被排除的包路径在maindex中，则使用 -keep 选项，即使他已经在分包的路径中.
# 注意，没有split只用keep时，miandexlist将仅包含keep指定的类。
#-keep android.support.v4.view.**
# 这条配置可以指定这个包下类在第二dex中.（注意，未指定的类会在被认为在maindexlist中）
#android.support.v?.**
#
# 使用.class后缀，代表单个类.
#-keep android.support.v7.app.AppCompatDialogFragment.class
#
# 不包含Android gradle 插件自动生成的miandex列表.
#-donot-use-suggest
#
# 将 全局过滤配置应用到 建议的maindexlist中, 但 -donot-use-suggest 要关闭.
-filter-suggest
# 不进行dex分包， 直到 dex 的id数量超过 65536.
-auto-maindex
# dex 扩展参数, 例如 --set-max-idx-number=50000
# 如果出现 DexException: Too many classes in --main-dex-list, main dex capacity exceeded，则需要调大数值
#-dex-param --set-max-idx-number=48000
# 显示miandex的日志.
-log-mainlist
#过滤日志。Recommend：在maindexlist中（由推荐列表确定）；Global：在maindexlist中，由全局过滤确定；true，前两者都成立的；false，不在maindexlist中
-log-filter
```



- [Intellij / Android Studio 调试 Gradle Plugin](https://blog.csdn.net/ceabie/article/details/55271161)

```
1.创建remote调试任务：
2.打开Terminal窗口输入：
./gradlew(or gradle) assembleDebug -Dorg.gradle.daemon=false -Dorg.gradle.debug=true。
```








