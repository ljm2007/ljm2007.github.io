---
title: AndroidBinder
date: 2023-09-25 17:41:40
tags: 讲义
categories: 讲义
top_img: https://s2.loli.net/2023/02/13/JoBMthknSd29bID.png
cover: https://s2.loli.net/2023/02/13/JoBMthknSd29bID.png
---

## 一.什么是Binder

Android Binder是Android操作系统中的一种IPC（进程间通信）机制，用于在不同的Android应用程序组件（例如，Activity、Service、BroadcastReceiver等）之间进行通信。它是Android系统中非常重要的一部分，用于实现各种功能，包括跨进程通信和系统服务的访问。

Binder机制是Android进行IPC（进程间通信）的主要方式

Binder跨进程通信机制：基于C/S架构，由Client、Server、ServerManager和Binder驱动组成。 进程空间分为用户空间和内核空间。用户空间不可以进行数据交互；内核空间可以进行数据交互，所有进程共用 一个内核空间 Client、Server、ServiceManager均在用户空间中实现，而Binder驱动程序则是在内核空间中实现的；

1、效率：传输效率主要影响因素是内存拷贝的次数，拷贝次数越少，传输速率越高。从Android进程架构角度 分析：对于消息队列、Socket和管道来说，数据先从发送方的缓存区拷贝到内核开辟的缓存区中，再从内核缓存区拷 贝到接收方的缓存区，一共两次拷贝。

一次数据传递需要经历：用户空间 –> 内核缓存区 –> 用户空间，需要2次数据拷贝，这样效率不高。 

而对于Binder来说，数据从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同 一块物理地址的，节省了一次数据拷贝的过程 ： 共享内存不需要拷贝，Binder的性能仅次于共享内存。 

Android Binder是Android操作系统中的一种IPC（进程间通信）机制，用于在不同的Android应用程序组件（例如，Activity、Service、BroadcastReceiver等）之间进行通信。它是Android系统中非常重要的一部分，用于实现各种功能，包括跨进程通信和系统服务的访问。

2、稳定性：上面说到共享内存的性能优于Binder，那为什么不采用共享内存呢，因为共享内存需要处理并发同 步问题，容易出现死锁和资源竞争，稳定性较差。 Binder基于C/S架构 ，Server端与Client端相对独立，稳定性较 好。

3、安全性：传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Binder机制 为每个进程分配了UID/PID，且在Binder通信时会根据UID/PID进行有效性检测。

Android Binder的主要特点和功能包括：

1. 跨进程通信：Android Binder允许不同进程中的组件相互通信，这对于实现应用程序之间的协作和共享数据非常有用。例如，一个应用程序可以通过Binder向另一个应用程序发送数据或请求服务。

2. 服务访问：Android Binder还允许应用程序访问系统服务，如通知管理器、传感器服务、电池管理器等。这些系统服务通常在不同的进程中运行，Binder允许应用程序与它们进行通信。

3. 进程间通信的安全性：Android Binder提供了一种安全的IPC机制，可以防止恶意应用程序访问其他应用程序的数据或服务。每个应用程序都有自己的安全上下文，可以限制对其资源的访问。

4. 轻量级和高效性：Android Binder被设计为一种轻量级的IPC机制，它具有较低的开销，能够在Android设备上高效运行。

Binder的实现是基于Linux内核的一部分，并且在Android中有一个Binder驱动程序，它负责处理进程间通信请求。Android框架也提供了Java层次的Binder API，使开发人员能够更容易地使用Binder进行通信。

总的来说，Android Binder是Android系统中实现进程间通信和访问系统服务的关键机制，它有助于实现Android应用程序的各种功能和互操作性。