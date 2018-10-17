---
title: Android frameWork层(五)：Android硬件抽象层(HAL)学习
date: 2017-08-23 13:02:09
tags:
---
  
   Android的硬件抽象层，简单来说，就是对Linux内核驱动程序的封装，向上提供接口，屏蔽低层的实现细节。也就是说，把对硬件的支持分成了两层，
一层放在用户空间（User Space），一层放在内核空间（Kernel Space），其中，硬件抽象层运行在用户空间，而linux内核驱动程序运行在内核空
间。

<!-- more -->


Android的硬件抽象层，简单来说，就是对Linux内核驱动程序的封装，向上提供接口，屏蔽底层的实现细节。也就是说，把对硬件的支持分成了两层，一层
放在用户空间（User Space），一层放在内核空间（Kernel Space），其中，硬件抽象层运行在用户空间，而linux内核驱动程序运行在内核空间。

[Android硬件抽象层（HAL）概要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6567257)

撇开这些争论，学习Android硬件抽象层，对理解整个Android整个系统，都是极其有用的，因为它从下到上涉及到了Android系统的硬件驱动层、硬件抽
象层、运行时库和应用程序框架层等等，下面这个图阐述了硬件抽象层在Android系统中的位置，以及它和其它层的关系：

![image](http://hi.csdn.net/attachment/201106/25/0_1308977488PkP8.gif)

- 1.HAL硬件抽象层起到承上启下的作用，对底层驱动访问，对上提供读取底层信息；
- 2.硬件驱动程序一方面分布在Linux内核中，另一方面分布在用户空间的硬件抽象层中
- 3.上层Application访问HAL:编写JNI方法和在Android的Application Frameworks层增加API接口
- 4.在Android系统中，硬件服务一般是运行在一个独立的进程中为各种应用程序提供服务。因此，调用这些硬件服务的应用程序与这些硬件服务之间的通信需要通过代理来进行。
- 编写IHelloService.aidl文件，编译会生成.stub接口，

```
package android.os;  

interface IHelloService {  
void setVal(int val);  
int getVal();  
}  
```
就会根据IHelloService.aidl生成相应的IHelloService.Stub接口。

- 新增HelloService.java文件;继承stub，实现setVal 和getVal函数；
- 通过调用IHelloService.Stub.asInterface(ServiceManager.getService("hello")); 获得helloService实例，过IHelloService.Stub.asInterface函数转换为IHelloService接口；


一. [在Android内核源代码工程中编写硬件驱动程序。](http://blog.csdn.net/luoshengyang/article/details/6568411)

二. [在Android系统中增加C可执行程序来访问硬件驱动程序。](http://blog.csdn.net/luoshengyang/article/details/6571210)

三. [在Android硬件抽象层增加接口模块访问硬件驱动程序。](http://blog.csdn.net/luoshengyang/article/details/6573809)

四. [在Android系统中编写JNI方法在应用程序框架层提供Java接口访问硬件](http://blog.csdn.net/luoshengyang/article/details/6575988)。

五. [在Android系统的应用程序框架层增加硬件服务接口。](http://blog.csdn.net/luoshengyang/article/details/6578352)

六. [在Android系统中编写APP通过应用程序框架层访问硬件服务](http://blog.csdn.net/luoshengyang/article/details/6580267)。

#### Android系统硬件抽象层（HAL）原理

在android开发过程中，我们经常看到HAL这个概念，这就android的硬件抽象层的（Hardwaere Abstraction Layer）缩写,它是Goolge应某些厂商
不希望公开源码所添加的一个适配层，能以封闭源码的方式提供硬件驱动模块，目的就是把android framework层和Linux kernel层隔离开来，使
android系统不至于过度的依赖linux kernel层的实现，也就是说android framework可以不考虑linux kernel如何实现的情况下独立开发

设备驱动分为内核空间和用户空间两部分：

- 内核空间主要负责硬件访问逻辑（GPL）
- 用户空间主要负责参数和访问流程控制（Apache License）
![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110655866-1722630862.jpg)


Android硬件驱动程序开发：与传统的Linux硬件驱动程序开发是一样的
Android硬件驱动程序验证
Android硬件抽象层模块开发

![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110658350-1227133419.png)
Android硬件访问服务开发：

![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110700272-504028242.png)


#### Android进程机制

![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110844069-1234308723.jpg)

- 1.Zygote进程由Init进程启动
在Linux系统中，所有的进程都是init进程的子孙进程，也就是说，所有的进程都是直接或者间接地由init进程fork出来的。Zygote进程也是在系统启动的过程，由init进程创建的。

```
1.系统启动时init进程会创建Zygote进程，Zygote进程负责后续Android
应用程序框架层的其它进程的创建和启动工作。
2.Zygote进程会首先创建一个SystemServer进程，SystemServer进程负
责启动系统的关键服务，如包管理服务PackageManagerService和应用程
序组件管理服务ActivityManagerService。
3.当我们需要启动一个Android应用程序时，ActivityManagerService会
通过Socket进程间通信机制，通知Zygote进程为这个应用程序创建一个新
的进程。

```
Zygote进程启动完成后的地址空间：
![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110845397-548539053.jpg)

- 2.Android应用程序进程启动过程

```
1.在Android系统中，所有的应用程序进程以及系统服务进程SystemServer
都是由Zygote进程孕育（fork）出来的。
2.系统中的两个重要服务PackageManagerService和ActivityManagerService，
都是由SystemServer进程来负责启动的，而SystemServer进程本身是Zygote进
程在启动的过程中fork出来的。
3.当ActivityManagerService启动一个应用程序的时候，就会通过Socket与
Zygote进程进行通信，请求它fork一个子进程出来作为这个即将要启动的应
用程序的进程；
4.ActivityManagerService组件一般会在什么情况下会为应用程序创建一个
新的进程呢？：当系统决定要在一个新的进程中启动一个Activity或者Service
时，它就会创建一个新的进程了，然后在这个新的进程中启动这个Activity
或者Service。
5.Android应用程序进程启动过程：指定新的进程的入口函数是Activity
Thread的main函数；为进程内的Binder对象提供了Binder进程间通信机制
的基础设施（定义了Binder线程池：我们在开发Android应用程序的时候，
当我们要和其它进程中进行通信时，只要定义自己的Binder对象，然后把
这个Binder对象的远程接口通过其它途径传给其它进程后，其它进程就可
以通过这个Binder对象的远程接口来调用我们的应用程序进程的函数了）


```
System Server进程启动完成后的地址空间：
![image](http://images2015.cnblogs.com/blog/947400/201605/947400-20160526110846678-215170336.jpg)

- 3.Android应用程序进程回收机制

1. Linux的内存回收机制--Out of Memory Killer

```
1.每一个进程都有一个oom_adj值，取值范围[-17,15]，可以通过
/proc/<pid>/oom_adj访问
2.每一个进程的oom_adj初始值都等于其父进程的oom_adj值
3.oom_adj值越小，越不容易被杀死，其中，-17表示不会被杀死
4.内存紧张时，OOM Killer综合进程的内存消耗量、CPU时间、
存活时间和oom_adj值来决定是否要杀死一个进程来回收内存
```

2. Android的内存回收机制—Low Memory Killer

```
1.进程的oom_adj值由ActivityManagerService根据运行在进程里面的组件
的状态来计算
2.进程的oom_adj值取值范围为[-16,15]， oom_adj值越小，就不容易被杀死
3.内存紧张时， LMK基于oom_adj值来决定是否要回收一个进程
4.ActivityManagerService和WindowManagerService在特定情况下也会进行进程回收
5.LMK的进程回收策略：当系统内存小于i时，在oom_adj值大于等于
j的进程中，选择一个oom_adj值最大并且消耗内存最多的进程来回收
```
** 应用程序进程的oom_adj值：**

- SYSTEM_ADJ(-16)：System Server进程
- PERSISTENT_PROC_ADJ(-12)：android:persistent属性为true的系统App进程，如PhoneApp
- FOREGROUND_APP_ADJ(0)：包含前台Activity的进程
- VISIBLE_APP_ADJ(1)：包含可见Activity的进程
- PERCEPTIBLE_APP_ADJ(2)：包含状态为Pausing、Paused、Stopping的Activity的进程，以及运行有Foreground Service的进程
- HEAVY_WEIGHT_APP_ADJ(3)：重量级进程，android:cantSaveState属性为true的进程，目前还不开放
- BACKUP_APP_ADJ(4)：正在执行备份操作的进程
- SERVICE_ADJ(5)：最近有活动的Service进程
- HOME_APP_ADJ(6)：HomeApp进程
- PREVIOUS_APP_ADJ(7)：前一个App运行在的进程
- SERVICE_B_ADJ(8)：SERVICE_ADJ进程数量达到一定值时，最近最不活动的Service进程
- HIDDEN_APP_MIN_ADJ(9)和HIDDEN_APP_MAX_ADJ(15)：含有不可见Activity的进程，根据LRU原则赋予[9,15]中的一个值
- Init进程的oom_adj值被设置为-16，由Init进程所启动的daemon和service进程的oom_adj值也等于-16
- 如果运行在进程A中的Content Provider或者Service被绑定到进程B，并且进程B的oom_adj值比进程A的oom_adj小，那么进程A的oom_adj值就会被设置为进程B的oom_adj值，但是不能小于FOREGROUND_APP_ADJ

ActivityManagerService在以下四种情况下会更新应用程序进程的oom_adj值，以及杀掉那些已经被卸载了的App所运行在的应用程序进程：

- activityStopped：停止Activity
- setProcessLimit：设置进程数量限制
- unregisterReceiver：注销Broadcast Receiver
- finishReceiver：结束Broadcast Receiver

WindowManagerService在处理窗口的过程中发生Out Of Memroy时，也会通知ActivityManagerService杀掉那些包含有窗口的应用程序进程


http://www.cnblogs.com/Doing-what-I-love/p/5530291.html




