---
title: Binder的框架
date: 2023-08-30 16:26:23
tags: Binder
categories: Binder
top_img: https://s2.loli.net/2023/08/30/d5z6I8PQev3DVN2.png
cover: https://s2.loli.net/2023/08/30/d5z6I8PQev3DVN2.png
---

## 1 Binder的框架

  在Android系统中，几乎所有进程（除了一些采用 socket 实现进程间通信的原生进程外）都会去打开一个指向 /dev/binder 的句柄。

  Binder 是一种远程过程调用(RPC)机制。**它能够让应用间以程序调用的方式进行通信，而无须关心消息到底是如何发送和接收的。从应用程序（无论是客户端还是服务器）的角度来看，客户端程序要做的就是调用一个方法、服务端要做的就是提供一个方法**。当客户端调用某个方法时，服务端程序中的对应方法就会调用，所有底层细节由Binder实现。这些底层工作包括：

**找到服务端进程** : 在大多数情况下，客户端和服务端分别是两个不同的进程（除非是system_server中的各个服务相互调用）。Binder 需要为客户端找出服务端进程，然后才能向它投递消息。这个“找到服务端”（也就是众所周知的“端点映射”（endpoint mapping））的工作，从技术上讲是由servicernanager来完成的。但是servicemanager只负责维护一个服务目录，把一个接口名（interface name）映射成一个Binder句柄（handle)而己。而这个“句柄”却是Binder交给servicernanager的，它是一个谁也看不懂得标识符(identifier），只有Binder才知道它的“ 真正”含义，其中记录了要找的服务端进程的PID。

**传递消息** : 如前文所述，生成获取被调用方法的参数，并将其序列化（serialize)（即把它们顺序打包到内存里的一个结构体中去）或是解序列化（deserialize)（即把结构体中各个参数逐个还原出来）的代码的任务是由AIDL来完成的。但是从一个进程向另一个进程传递序列化了的结构体的工作，则是由Binder亲自完成的。客户端进程会用BINDER_WRITE_READ参数调用ioctl(2）。这将会通过Binder发送消息，并且阻塞掉客户端进程，直到服务端进程返回结果为止（因此，代码是先写，后读） 。

**传递对象** : Binder也可以用来传递对象。如前文所述，service处理的是一种类型的对象，这些对象也包括“ 文件描述符”（比如 Unix Domain Socket）。传递文件描述符是一个非常重要的特性。因为这使得可信进程（比如 system_server）可以用原生代码为一个不可信进程（比如 用户安装的App）打开某个设备或socket。当然，这里我们假设这个不可信进程是拥有相应权限的（即在App的manifest文件中声明了该种权限）

**支持安全认证** : 进程间通信的安全性，自然是极为重要的，消息的接受者应该能够验证消息的发送者的身份，以免落入圈套，进而殃及整个系统的安全性。Binder可以获取到它的使用者的安全证书（PID和UID）并把它们安全地嵌入到消息中去。这样，服务端进程就能按合理的安全级的要求做出相应的安全认证操作。

![在这里插入图片描述](https://s2.loli.net/2023/08/30/d5z6I8PQev3DVN2.png)

Binder的实现需要以下几个部分：

|              名称               |                           **备注**                           |
| :-----------------------------: | :----------------------------------------------------------: |
|       **1	Binder驱动**       | **Binder机制中，Server和Client都运行在应用层，要完成IPC就必需内核层建立通道才能通信，Binder驱动的作用就在此，它帮助Server和Client进行数据交互（执行数据R/W操作）** |
| **2	servicemanager守护进程** | **守护进程servicemanager是Android系统中所有服务的管理器。每个Server都需要在servicemanager中进行注册，Client在servicemanager查询服务并获取服务的信息。** |
|    **3	Server	服务端**    | **服务端（也是Android的系统服务），本质是响应Client的请求，负责在servicemanager中注册服务。同时，Server还包括监昕请求、处理请求、应答Client的过程。** |
|    **4	Client	客户端**    |  **客户端，一般指Android系统的应用程序，是服务的请求方。**   |
|         **5	Proxy**          | **Proxy即服务代理对象，是在Client建立一个Server的“引用”，使得该代理对象具有Server的功能。从应用程序的角度看，代理对象和本地对象一样，可以调用其方法并返回相应的结果。** |

  Android系统中，每个Activity和Service都是进程，两者之间的通信看起来就好像是一个进程进人另一个进程中执行代码，然后带着执行的结果返回。**这种跨进程的通信方式依赖于 Binder 的用户空间为每个进程维护一个可用的线程池，IPC 以及进程的本地消息等交由线程池处理和执行**。
  在Android源码中，主要的 Binder 库由本地原生代码实现。应用程序通过Binder接口调用 Binder 原生库。Binder的系统架构如下图所示：

Binder 驱动用于实现Binder的设备驱动，主要负责组织Binder 的服务节点，调用Binder 相关的处理线程，完成实际的Binder传输等。它位于Binder结构的最底层（Linux内核层）。
Binder Adapter 层是对Binder 驱动的封装，实现对Binder 驱动的操作，实现包括 IPCThreadState.cpp 和 ProcessState.cpp，以及 Parcel.cpp 中的部分内容。
Binder 核心库是Binder 构架的核心实现，主要包括IBinder、Binder（服务器端）和 BPBinder（客户端）。
最上面两层的Binder 构架和具体的Server端/Client端都有 Java 和 C++ 两种实现方案，主要供应用程序使用，都是通过调用 Binder 的核心库来实现的。

## 2 Binder驱动

  Binder驱动的实现遵循Linux设备驱动模型，通过 binder_ioctl() 函数与用户空间的进程交换数据。使用BINDER_WRITE_READ 进行数据的读写操作，binder_thread_write() 函数发送请求或返回结果，binder_thread_read() 函数用于读取结果。实现代码主要涉及以下文件：kernel/drivers/staging/binder.h 和 kernel/drivers/staging/binder.c 。
  其实，很多Android系统相关的书籍里面，很喜欢长篇大论分析Binder驱动代码，作为一个Android系统Native/HAL层-Framework层的开发者来说确实没必要研究Binder驱动相关的原理，大致了解即可，把重点应该放在如何利用好Binder机制。

## 3 Binder驱动的封装Binder Adapter

  Binder Adapter 层的作用是完成对 Binder 驱动的封装，实现IPCThreadState.cpp 和 ProcessState.cpp，以及Parcel.cpp 中的部分内容。

ProcessState 属于 singleton 类型，其作用是维护当前进程中的所有 Proxy。一个 Client 通常需要多个 Service 的服务，这样就需要创建多个 Service 代理（Proxy），客户端进程中的Process State 对象负责维护 Proxy。
IPCThreadState 对象主要负责调用 Binder 设备的相关操作，完成数据读取、写入和请求处理框架。主要实现 talkWithDriver() 读取/写入函数，executeCommand() 请求处理函数，joinThreadPool() 设备轮询函数。IPCThreadState 具体实现了所有进程(包括客户端和服务端的进程)与 Binder 设备的通信操作。

  **每个进程只有一个 Process State 对象，每一个线程中都会有一个 IPCThreadState 对象**，其源代码位于：frameworks/native/inculde/binder 和 frameworks/native/libs/binder 两个文件夹中。

 Binder框架定义了四个角色：Server，Client，ServiceManager（以后简称SM）以及Binder驱动。其中Server，Client，SM运行于用户空间，驱动运行于内核空间。





## 4 **ServiceManager与实名Binder**

<u>**Client获得实名Binder的引用**</u>
    <u>Server向SM注册了Binder实体及其名字后，Client就可以通过名字获得该Binder的引用了。</u>

<u>**匿名Binder**</u>
    <u>并不是所有Binder都需要注册给SM广而告之的。Server端可以通过已经建立的Binder连接将创建的Binder实体传给Client，当然这条已经建立的Binder连接必须是通过实名Binder实现。由于这个Binder没有向SM注册名字，所以是个匿名Binder。Client将会收到这个匿名Binder的引用，通过这个引用向位于Server中的实体发送请求。匿名Binder为通信双方建立一条私密通道，只要Server没有把匿名Binder发给别的进程，别的进程就无法通过穷举或猜测等任何方式获得该Binder的引用，向该Binder发送请求。</u>



————————————————
版权声明：本文为CSDN博主「小馬佩德罗」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Xiaoma_Pedro/article/details/103919527