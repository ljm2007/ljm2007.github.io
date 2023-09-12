---
title: 浅显易懂的startActivity原理分析
date: 2023-09-08 18:50:40
tags: HeiBaoBox
categories: HeiBaoBox
top_img: https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg
cover: https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg
---

<img src="https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg" style="zoom:150%;" />

## 前言

我们想一想几个问题，Android是如何启动一个Activity的呢？我们调用Context.startActivity()，一个Activity就展现了出来，这个Activity的onCreate()和onResume()等方法会被系统调用，其内部到底经历了怎样的过程呢？当前展示的Activity的onPause()和onStop()方法是怎么被调用的？

首先，要明白一件事，就是startActivity()是一个跨进程的操作，相当于给另一个进程发送一个消息，然后该方法就马上返回了，背后系统会进行一系列复杂的操作，之后这个Activity才会展示出来，我们不能认为startActivity之后这个Activity就启动展示出来了，包括Activity的finish()也一样，它们只负责给系统进程发送信息而已。

那么，我们马上会想到两个问题：

**1. startActivity()是发消息给谁呢？**
 答案是ActivityManagerService这个服务，而这个服务驻留在系统进程SystemServer中，之前的文章谈过，一个进程可以驻留多个服务，比如SystemServer这个进程包含了ActivityManagerService、PackageManagerService、WindowManagerService、PowerManagerService等很多个系统服务。因此，startActivity()是发消息给SystemServer进程，而进程间通信的方法正是安卓世界强大的Binder机制。

**2. startActivity()发送的是什么消息呢？**
 这里的消息是通过Intent参数携带的，比如客户端可以通过PackageManager的getLaunchIntentForPackage()方法获取特定包名下特定Activity的Intent，这个Intent便指定了要启动的Activity，AMS进程则通过解析这个Intent创建对应的ActivityRecord，继续后续启动Activity的过程。

### Activity在AMS中的数据表示 —— ActivityRecord

Android中的Activity是通过Task来组织的，系统只支持一个Task处于前台，即用户当前看到的Activity所属的Task，其余的Task均处于后台，这些后台Task内部的Activity保持顺序不变。用户可以一次将整个Task挪到后台或者置为前台。在一些安卓手机中，按Menu键，系统会弹出近期Task列表，用户能快速在多个Task间切换。与Task概念相关的还有Activity启动的模式，比如可以通过在Manifest指定launchMode为singleTask或在Intent中指定Flag为FLAG_ACTIVITY_NEW_TASK来控制Activity独占一个Task，或者通过taskAffinity属性来指定Activity所属的Task，这块不是本文的重点，详细可以看这里：[https://developer.android.google.cn/guide/components/activities/tasks-and-back-stack](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Factivities%2Ftasks-and-back-stack)。

**注意：Task并不是进程的意思，它只是一个虚拟的概念，由AMS中的TaskRecord表示。**

AMS内部是通过ActivityStack来组织Activity和Task，Activity用ActivityRecord来表示，Task用TaskRecord来表示，需要注意的是，ActivityStack并不是从Task的维度来保存ActivityRecord的，而是直接扁平化地使用一个数组来保存ActivityRecord，用mHistory变量指向这个数组，表示历史的Activity。由于系统只有一个ActivityStack实例，因此mMainStack一直为true。ActivityStack有一系列方法来搜索和操作ActivityRecord，其主要结构如图：

![img](https:////upload-images.jianshu.io/upload_images/3178450-59ab835d283eab22.png?imageMogr2/auto-orient/strip|imageView2/2/w/292/format/webp)

在ActivityStack中，startActivityXXX的一系列流程是为了创建ActivityRecord和TaskRecord，然后将TaskRecord设置到ActivityRecord中，进而通过该ActivityRecord来启动Activity。

### 启动Activity

ActivityRecord就绪之后，AMS会判断ActivityRecord中的ProcessRecord中pid指向的进程是否存在，如果存在则直接向该进程发送启动Activity的消息，否则会启动进程。与应用进程通信的方式同样是Binder，接口是IApplicationThread，AMS通过IApplicationThread将APK相关的信息传递给应用进程。

当需要启动应用进程时，AMS向zygote进程发送信息，使得zygote分裂出一个新的进程，AMS随即得到该进程的pid，并把该pid与ActivityRecord的ProcessRecord关联起来。zygote分裂出来的是一个通用的进程，它并不知道自己的使命，也不知道自己运行的是哪个apk包，只是个空白的进程。但该进程在启动Java虚拟机和进行必要的初始化后会进入Java世界，然后会执行ActivityThread的main()方法，在该main()方法中，会调用ActivityThread的attach方法，attach方法将会调用AMS服务的attachApplication方法，并传入自己的IApplicationThread对象，从而开始应用进程和AMS进程间的通信。

因此，应用进程在被创建之后会主动与AMS进程建立通信，而AMS在创建一个应用进程后，会设置一个超时时间（一般是10秒）。如果超过这个时间，应用进程还没有和AMS交互，则断定该进程创建失败。

建立完通信链路后，AMS内部便可以使用IApplicationThread接口，通过bindApplication方法，将APK相关的信息传给应用进程，此时应用进程也就活起来了。应用进程在收到bindApplication消息后，将会根据AndroidManifest中声明的Application标签创建一个Application。

而真正启动Activity是通过ActivityStack的realStartActivityLocked方法，其大体的流程是向应用进程发送信息，使得应用进程调用ActivityThread的handleLaunchActivity以及PerformLaunchActivity方法。

其实启动一个Activity的关键流程是：

1. startActivity所在的进程向AMS发送Intent
2. AMS根据Intent寻找ActivityRecord
3. 如果进程不存在，则：
    a. 通知Zygote分裂启动新的应用
    b. 应用进程与AMS建立Binder通信
    c. AMS告诉应用进程APK包的信息
4. AMS通知应用进程启动Activity
5. 应用进程会通过ActivityThread执行performLaunchActivity，callOnCreate等方法

这只是主要的流程，其实里边的细节还有很多，比如当下Activity的onPause和onStop等方法的调用，但原理其实很好理解，当AMS启动一个新的Activity以及该Activity的onCreate方法在被调用之前，AMS会发信息给当前展示的Activity的应用进程，通知其ActivityThread调用performPauseActivityIfNeeded方法，进而调用当前Activity的onPause方法。

至此，启动Activity的原理介绍完毕，AMS极其复杂，要完全理解需要花费很长时间，但我认为只需要把关键的节点理清楚就好，具体细节等需要用到再仔细研究。

完毕。

其它文章可以学习：
 理解Application创建过程：[http://gityuan.com/2017/04/02/android-application/](https://links.jianshu.com/go?to=http%3A%2F%2Fgityuan.com%2F2017%2F04%2F02%2Fandroid-application%2F)

