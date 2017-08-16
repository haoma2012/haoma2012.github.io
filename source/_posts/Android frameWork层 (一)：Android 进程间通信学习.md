---
title: Android frameWork层 (一)：Android 进程间通信学习
date: 2017-08-03 15:21:13
description: Android 系统进程间通信Binder机制学习,client server SM Binder驱动程序之间的IPC通信机制
tags:
---

<!--more-->

#### [Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)

- [深入浅出binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)

分析的binder源码目录在：/frameworks/native/libs/binder/下

mediaService///frameworks/av/media/mediaserver/main_mediaserver.cpp
```
1.获得ProcessState 实例，返回的是一个实例指针，还会调用open_driver()打开dev/binder设备(驱动程序)，
映射fd到内存
2.defaultServiceManager，返回BpServiceManager,remote对象是BpBinder;SM Android OS 服务管理类
3实例化MediaPlayerservice，然后执行BpServiceManager发送了一个addService命令到BnServiceManager，
然后收到回复。
4.ServiceManager 的意义：Android系统中Service信息都是先add到ServiceManager中，由ServiceManager
来集中管理，这样就可以查询当前系统有哪些服务。
5.startThreadPool和joinThreadPool完后确实有两个线程，主线程和工作线程，而且都在做消息循环
```

-  MediaPlayerService向SM注册
-  MediaPlayerClient查询当前注册在SM中的MediaPlayerService的信息
-  根据这个信息，MediaPlayerClient和MediaPlayerService交互

- [Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589/) 

```
Android需要建立一套新的IPC机制来满足系统对通信方式，传输性能和安全性的要求，这就是Binder。
Binder基于Client-Server通信模式，传输过程只需一次拷贝，为发送方添加UID/PID身份，既支持
实名Binder也支持匿名Binder，安全性高。

Binder 驱动：用户通过/dev/binder访问，负责进程之间Binder通信的建立，Binder在进程之间的
传递，Binder引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。

ServiceManager 与实名Binder：SM的作用是将字符形式的Binder名字转化成Client中对该Binder
的引用，使得Client能够通过Binder名字获得对Server中Binder实体的引用。
Client 获得实名Binder的引用：Server向SM注册了Binder实体及其名字后，Client就可以通过名字
获得该Binder的引用了。Client也利用保留的0号引用向SMgr请求访问某个Binder
```

在Android系统中，每一个应用程序都是由一些Activity和Service组成的，这些Activity和Service有可能运行在同一个进程中，也有可能运行在不同的进程中。那么，不在同一个进程的Activity或者Service是如何通信的呢？这就是本文中要介绍的Binder进程间通信机制了。

![image](http://hi.csdn.net/attachment/201107/19/0_13110996490rZN.gif)

- Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中

- Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server

- Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信

- Client和Server之间的进程间通信通过Binder驱动程序间接实现

- Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力


2.[Service Manager是如何成为一个守护进程的？即Service Manager是如何告知Binder驱动程序它是Binder机制的上下文管理者。](http://blog.csdn.net/luoshengyang/article/details/6621566)

```
Service Manager，它是整个Binder机制的守护进程，用来管理开发者创建的各种Server，并且向Client提供
查询Server远程接口的功能。

SM：
一是打开Binder设备文件；open("/dev/binder", O_RDWR);
二是建立128K内存映射：mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
三是告诉Binder驱动程序自己是Binder上下文管理者，即我们前面所说的守护进程；
binder_become_context_manager(bs);
四是进入一个无穷循环，充当Server的角色，等待Client的请求。binder_loop(bs, svcmgr_handler);

Binder进程间通信机制的精髓所在内存映射，同一个物理页面，一方映射到进程虚拟地址空间，一方面映射到内核
虚拟地址空间，这样，进程和内核之间就可以减少一次内存拷贝了，提到了进程间通信效率。【1次内存拷贝】
```

3.[浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路](http://blog.csdn.net/luoshengyang/article/details/6627260)


gDefaultServiceManager = new BpServiceManager(new BpBinder(0));  SM创建远程接口本质是一个BpServiceManager，包含一个句柄值为0的Binder引用；

Service：调用IServiceManager::addService 与Binder驱动程序交互——>BpServiceManager::addService——>会调用通过其基类BpRefBase的成员函数remote获得原先创建的BpBinder实例，接着调用BpBinder::transact成员函数。在BpBinder::transact函数中，又会调用IPCThreadState::transact成员函数IPCThreadState有一个PorcessState类型的成中变量mProcess，而mProcess有一个成员变量mDriverFD，它是设备文件/dev/binder的打开文件描述符，因此，IPCThreadState就相当于间接在拥有了设备文件/dev/binder的打开文件描述符，于是，便可以与Binder驱动程序交互了。

Client:调用IServiceManager::getService这个接口来和Binder驱动程序交互了

具体查看下面这张调用关系图：

![image](http://img.my.csdn.net/uploads/201107/22/0_1311363642X5Cd.gif)

4.[Server是如何把自己的服务启动起来的？Service Manager在Server启动的过程中是如何为Server提供服务的？即IServiceManager::addService接口是如何实现的。](http://blog.csdn.net/luoshengyang/article/details/6629298)

MediaPlayerService的类图：

![image](http://hi.csdn.net/attachment/201107/24/0_1311479168os88.gif)

MediaPlayerService继承于BnMediaPlayerService类，熟悉Binder机制的同学应该知道BnMediaPlayerService是一个Binder Native类，用来处理Client请求的。BnMediaPlayerService继承于BnInterface<IMediaPlayerService>类，BnInterface是一个模板类，BnMediaPlayerService实际是继承了IMediaPlayerService和BBinder类。IMediaPlayerService和BBinder类又分别继承了IInterface和IBinder类，IInterface和IBinder类又同时继承了RefBase类。
BnMediaPlayerService并不是直接接收到Client处发送过来的请求，而是使用了IPCThreadState接收Client处发送过来的请求，而IPCThreadState又借助了ProcessState类来与Binder驱动程序交互。IPCThreadState接收到了Client处的请求后，就会调用BBinder类的transact函数，并传入相关参数，BBinder类的transact函数最终调用BnMediaPlayerService类的onTransact函数，于是，就开始真正地处理Client的请求了。


```
1.获取ProcessState实例：创建全局唯一变量gProcess
open_driver函数打开Binder设备文件/dev/binder；
mmap来把设备文件/dev/binder映射到内存中；

2.MediaPlayerService::instantiate函数把MediaPlayerService
添加到Service Manger中去了defaultServiceManager返回的实际
是一个BpServiceManger类实例

3.把Binder实体MediaPlayerService交给SM管理，也就是说
Service Manager要引用这个MediaPlayerService了，最后，
把待处理事务加入到target_list列表中去

4.最后执行IPCThreadState::talkWithDriver，与dev/binder驱动程序
交互返回binder实体

5.ProcessState::self()->startThreadPool();  
IPCThreadState::self()->joinThreadPool();  
Server启动起来之后，就会在一个无穷循环中等待Client的请求了
```

5.[Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6633311)

Client是通过IServiceManager::getService接口来获得MediaPlayerService这个Server的远程接口的。
![image](http://hi.csdn.net/attachment/201107/26/0_13117045717gMi.gif)
BpMediaPlayerService继承于BpInterface<IMediaPlayerService>类，即BpMediaPlayerService继承了IMediaPlayerService类和BpRefBase类，这两个类又分别继续了RefBase类。BpRefBase类有一个成员变量mRemote，它的类型为IBinder，实际是一个BpBinder对象。BpBinder类使用了IPCThreadState类来与Binder驱动程序进行交互，而IPCThreadState类有一个成员变量mProcess，它的类型为ProcessState，IPCThreadState类借助ProcessState类来打开Binder设备文件/dev/binder，因此，它可以和Binder驱动程序进行交互。

```
1.通过defaultServiceManager函数来获得ServiceManager的远程接
口，实际上就是获得BpServiceManager的IServiceManager接口,相当于
new BpServiceManager(new BpBinder(0));   
这里的0表示Service Manager的远程接口的句柄值是0。
2.while循环通过sm->getService接口来不断尝试获得名称为
“media.player”的Service，即MediaPlayerService。
3.在BpServiceManager::checkService中，首先是通过
Parcel::writeInterfaceToken往data写入一个RPC头，
Service Manager来处理CHECK_SERVICE_TRANSACTION请求之前，
会先验证一下这个RPC头，看看是否正确。这个name，就是service在SM注册的名字
```

6.[ Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析](http://blog.csdn.net/Luoshengyang/article/details/6642463)

1.获取Service Manager的Java远程接口的过程；
2.HelloService接口的定义；【aidl接口定义-->stub 和proxy类】
3.HelloService的启动过程；(SystemServer调用SM.addservice)
4.Client获取HelloService的Java远程接口的过程；IServiceManager.getService函数来获得HelloService的远程接口，实现IHelloService.Stub.Proxy对象
5.Client通过HelloService的Java远程接口来使用HelloService提供的服务的过程。


```
1.SM远程接口ServiceManagerProxy，前提是构造函数需要BinderProxy对象
2.SM:ServiceManager类有一个静态成员函数getIServiceManager，它的作用
就是用来获取Service Manager的Java远程接口了,最终是通过ServiceManagerNative
获取Java远程接口的。

就是在Java层，我们拥有了一个ServiceManager远程接口ServiceManagerProxy，
而这个ServiceManagerProxy对象在JNI层有一个句柄值为0的BpBinder对象与之
通过gBinderProxyOffsets关联起来。

```

#### Linux 和Android 使用通信比对

1.Linux 使用：
传统IPC：无名pipe, signal, trace, 有名管道
AT&T Unix 系统V：共享内存，信号灯，消息队列
BSD Unix：Socket
2.Android使用Binder机制

```
采用C/S的通信模式；
有更好的传输性能；
socket：是一个通用接口，导致其传输效率低，开销大；
管道和消息队列：因为采用存储转发方式，所以至少需要拷贝2次数据，效率低；
共享内存：虽然在传输时没有拷贝数据，但其控制机制复杂
安全性更高；
Linux的IPC机制在本身的实现中，并没有安全措施，得依赖上层协议
来进行安全控制。而Binder机制的UID/PID是由Binder机制本身在内
核空间添加身份标识，安全性高；并且Binder可以建立私有通道，这
是linux的通信机制所无法实现的 （Linux访问的接入点是开放的）。
```

#### Binder在Service服务中的作用：client-server-SM 间通信

Binder在C/S中的流程如下：

![image](http://www.2cto.com/uploadfile/Collfiles/20160607/20160607091016311.jpg)

- Server注册服务：Server作为众多Service的拥有者，当它想向Client提供服务时，得先去Service Manager（以后缩写成SM）那儿注册自己的服务。Server可以向SM注册一个或多个服务。

- Client申请服务：Client作为Service的使用者，当它想使用服务时，得向SM申请自己所需要的服务。Client可以申请一个或多个服务。当Client申请服务成功后，Client就可以使用服务了。

- SM一方面管理Server所提供的服务，同时又响应Client的请求并为之分配相应的服务。扮演的角色相当于月老，两边牵线。这种通信方式的好处是： 一方面，service和Client请求便于管理，另一方面在应用程序开发时，只需为Client建立到Server的连接，就可花很少时间和精力去实现Server相应功能。


总结：
1.Client和Server 还有SM是存在于用户空间,dev/binder驱动运行在内核空间
2.SM作为守护进程，处理客户端请求，管理所有服务项
3.Server 向SM注册服务
![image](http://www.2cto.com/uploadfile/Collfiles/20160607/20160607091017313.png)

- 1.XXXServer(XXX代表某个)在自己的进程中向Binder驱动申请创建一个XXXService的Binder的实体，
- 2.Binder驱动为这个XXXService创建位于内核中的Binder实体节点以及Binder的引用，注意，是将名字和新建的引用打包传递给SM（实体没有传给SM），通知SM注册一个名叫XXX的Service。
- 3.SM收到数据包后，从中取出XXXService名字和引用，填入一张查找表中。

此时，如果有Client向SM发送申请服务XXXService的请求，那么SM就可以在查找表中找到该Service的Binder引用，并把Binder引用(XXXBpBinder)返回给Client。

```
引用和实体：对于一个用于通信的实体可以有多个该实体的引用
如果一个进程持有某个实体，其他进程也想操作该实体，最高效
的做法是去获得该实体的引用，再去操作这个引 用。
```
4.如何获得SM的远程接口？

![image](http://www.2cto.com/uploadfile/Collfiles/20160607/20160607091017314.jpg)

- 针对Binder的通信机制，Server端拥有的是Binder的实体；Client端拥有的是Binder的引用。
- SM 可以作为特殊的Server端，在Binder驱动一运行起来时就有自己的Binder实体（代码中设置ServiceManager的Binder其handle值恒为0）。这个Binder实体没有名字也不需要注册，所有的client都认为handle值为0的binder引用是用来与SM通信 的
- 当一个进程调用Binder驱动时，使用BINDER_SET_CONTEXT_MGR命令（在驱动的binder_ioctl中）将自己注册成SM时，Binder驱动会自动为它创建Binder实体。这个Binder的引用对所有的Client都为0。

5.Client从SM获得Service的远程接口

![image](http://www.2cto.com/uploadfile/Collfiles/20160607/20160607091017315.png)

概括为：Server向SM注册了Binder实体及其名字后，Client就可以通过Service的名字在SM的查找表中获得该Binder的引用了（BpBinder）。Client也利用保留的handle值为0的引用向SM请求访问某个Service：我申请访问XXXService的引用。SM就会从请求数据包中获得XXXService的名字，在查找表中找到该名字对应的条目，取出Binder的引用打包回复给client。之后，Client就可以利用XXXService的引用使用XXXService的服务了。如果有更多的Client请求该Service，系统中就会有更多的Client获得这个引用。

6.建立C/S通路

![image](http://www.2cto.com/uploadfile/Collfiles/20160607/20160607091017316.png)

client拥有自己Binder的实体，以及Server的Binder的引用；Server拥有自己Binder的实体，以及Client的Binder的引用；

建立CS通路后的流程：（当接收方获得Binder的实体，发送方获得Binder的引用后）
发送方会通过Binder实体请求发送操作。Binder驱动会处理这个操作请求,把发送方的数据放入写缓存(binder_write_read.write_buffer)(对于接收方为读缓冲区)，并把read_size(接收方读数据)置为数据大小）；接收方之前一直在阻塞状态中，当写缓存中有数据，则会读取数据，执行命令操作接收方执行完后，会把返回结果同样用binder_transaction_data结构体封装，写入写缓冲区（对于发送方，为读缓冲区）
