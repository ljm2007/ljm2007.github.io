---
title: HeiBaoBox源码分析(2)----启动APK
date: 2023-09-07 10:33:23
tags: HeiBaoBox
categories: HeiBaoBox
top_img: https://s2.loli.net/2023/09/07/Q3kjRwDr8dlMZbC.webp
cover: https://s2.loli.net/2023/09/07/Q3kjRwDr8dlMZbC.webp
---



## 一.创建应用进程

`Android`应用进程的启动是**被动式**的，在桌面点击图标启动一个应用的组件如`Activity`时，如果`Activity`所在的进程不存在，就会创建并启动进程。**`Android`系统中一般应用进程的创建都是统一由`zygote`进程`fork`创建的，`AMS`在需要创建应用进程时，会通过`socket`连接并通知到到`zygote`进程在开机阶段就创建好的`socket`服务端，然后由`zygote`进程`fork`创建出应用进程。**整体架构如下图所示：

<img src="https://s2.loli.net/2023/09/07/Q3kjRwDr8dlMZbC.webp" alt="img" style="zoom: 50%;" />

从`AMS#startProcessAsync`创建进程函数入手，继续看一下应用进程创建相关简化流程代码：

### 1.AMS 发送socket请求

```java
/*frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*/  
   @GuardedBy("this")
    final ProcessRecord startProcessLocked(...) {
        return mProcessList.startProcessLocked(...);
   }
   
   /*frameworks/base/services/core/java/com/android/server/am/ProcessList.java*/
   private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
            int mountExternal, String seInfo, String requiredAbi, String instructionSet,
            String invokeWith, long startTime) {
        try {
            // 原生标识应用进程创建所加的systrace tag
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            ...
            // 调用Process的start方法创建进程
            startResult = Process.start(...);
            ...
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
    
    /*frameworks/base/core/java/android/os/Process.java*/
    public static ProcessStartResult start(...) {
        // 调用ZygoteProcess的start函数
        return ZYGOTE_PROCESS.start(...);
    }
    
    /*frameworks/base/core/java/android/os/ZygoteProcess.java*/
    public final Process.ProcessStartResult start(...){
        try {
            return startViaZygote(...);
        } catch (ZygoteStartFailedEx ex) {
           ...
        }
    }
    
    private Process.ProcessStartResult startViaZygote(...){
        ArrayList<String> argsForZygote = new ArrayList<String>();
        ...
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
```

在`ZygoteProcess#startViaZygote`中，最后创建应用进程的逻辑：

1. **`openZygoteSocketIfNeeded`函数中打开本地`socket`客户端连接到`zygote`进程的`socket`服务端**；
2. **`zygoteSendArgsAndGetResult`发送`socket`请求参数，带上了创建的应用进程参数信息**；
3. **`return`返回的数据结构`ProcessStartResult`中会有新创建的进程的`pid`字段**。

### 2 Zygote 处理socket请求

其实早在系统开机阶段，`zygote`进程创建时，就会在`ZygoteInit#main`入口函数中创建服务端`socket`，**并预加载系统资源和框架类（加速应用进程启动速度）**，代码如下：

```java
/*frameworks/base/core/java/com/android/internal/os/ZygoteInit.java*/
 public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;
         ...
        try {
            ...
            // 1.preload提前加载框架通用类和系统资源到进程，加速进程启动
            preload(bootTimingsTraceLog);
            ...
            // 2.创建zygote进程的socket server服务端对象
            zygoteServer = new ZygoteServer(isPrimaryZygote);
            ...
            // 3.进入死循环，等待AMS发请求过来
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            ...
        } finally {
            ...
        }
        ...
    }
```

继续往下看`ZygoteServer#runSelectLoop`如何监听并处理AMS客户端的请求：

```java
/*frameworks/base/core/java/com/android/internal/os/ZygoteServer.java*/
 Runnable runSelectLoop(String abiList) {
     // 进入死循环监听
     while (true) {
        while (--pollIndex >= 0) {
           if (pollIndex == 0) {
             ...
           } else if (pollIndex < usapPoolEventFDIndex) {
             // Session socket accepted from the Zygote server socket
             // 得到一个请求连接封装对象ZygoteConnection
             ZygoteConnection connection = peers.get(pollIndex);
             // processCommand函数中处理AMS客户端请求
             final Runnable command = connection.processCommand(this, multipleForksOK);
           }
        }
     }
 }
 
 Runnable processCommand(ZygoteServer zygoteServer, boolean multipleOK) {
         ...
         // 1.fork创建应用子进程
         pid = Zygote.forkAndSpecialize(...);
         try {
             if (pid == 0) {
                 ...
                 // 2.pid为0，当前处于新创建的子应用进程中，处理请求参数
                 return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
             } else {
                 ...
                 handleParentProc(pid, serverPipeFd);
             }
          } finally {
             ...
          }
 }
 
  private Runnable handleChildProc(ZygoteArguments parsedArgs,
            FileDescriptor pipeFd, boolean isZygote) {
        ...
        // 关闭从父进程zygote继承过来的ZygoteServer服务端地址
        closeSocket();
        ...
        if (parsedArgs.mInvokeWith != null) {
           ...
        } else {
            if (!isZygote) {
                // 继续进入ZygoteInit#zygoteInit继续完成子应用进程的相关初始化工作
                return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mDisabledCompatChanges,
                        parsedArgs.mRemainingArgs, null /* classLoader */);
            } else {
                ...
            }
        }
    }
```

以上过程从systrace上看如下图所示：

![img](https://s2.loli.net/2023/09/07/yD2RmodZ3841Alk.webp)

### 3 应用进程初始化

接上一节中的分析，`zygote`进程监听接收`AMS`的请求，`fork`创建子应用进程，然后`pid`为0时进入子进程空间，然后在 `ZygoteInit#zygoteInit`中完成进程的初始化动作，相关简化代码如下：

```java
/*frameworks/base/core/java/com/android/internal/os/ZygoteInit.java*/
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        ...
        // 原生添加名为“ZygoteInit ”的systrace tag以标识进程初始化流程
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();
        // 1.RuntimeInit#commonInit中设置应用进程默认的java异常处理机制
        RuntimeInit.commonInit();
        // 2.ZygoteInit#nativeZygoteInit函数中JNI调用启动进程的binder线程池
        ZygoteInit.nativeZygoteInit();
        // 3.RuntimeInit#applicationInit中反射创建ActivityThread对象并调用其“main”入口方法
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
                classLoader);
 }
```

应用进程启动后，初始化过程中主要依次完成以下几件事情：

1. **应用进程默认的`java`异常处理机制（可以实现监听、拦截应用进程所有的`Java crash`的逻辑）；**
2. **`JNI`调用启动进程的`binder`线程池（注意应用进程的`binder`线程池资源是自己创建的并非从`zygote`父进程继承的）；**
3. **通过反射创建`ActivityThread`对象并调用其“`main`”入口方法。**

我们继续看`RuntimeInit#applicationInit`简化的代码流程：

```java
/*frameworks/base/core/java/com/android/internal/os/RuntimeInit.java*/
 protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        ...
        // 结束“ZygoteInit ”的systrace tag
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        // Remaining arguments are passed to the start class's static main
        return findStaticMain(args.startClass, args.startArgs, classLoader);
  }
  
  protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;
        try {
            // 1.反射加载创建ActivityThread类对象
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            ...
        }
        Method m;
        try {
            // 2.反射调用其main方法
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            ...
        } catch (SecurityException ex) {
            ...
        }
        ...
        // 3.触发执行以上逻辑
        return new MethodAndArgsCaller(m, argv);
    }
```

我们继续往下看`ActivityThread`的`main`函数中又干了什么：

```java
/*frameworks/base/core/java/android/app/ActivityThread.java*/
public static void main(String[] args) {
     // 原生添加的标识进程ActivityThread初始化过程的systrace tag，名为“ActivityThreadMain”
     Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
     ...
     // 1.创建并启动主线程的loop消息循环
     Looper.prepareMainLooper();
     ...
     // 2.attachApplication注册到系统AMS中
     ActivityThread thread = new ActivityThread();
     thread.attach(false, startSeq);
     ...
     Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
     Looper.loop();
     ...
}

private void attach(boolean system, long startSeq) {
    ...
    if (!system) {
       ...
       final IActivityManager mgr = ActivityManager.getService();
       try {
          // 通过binder调用AMS的attachApplication接口将自己注册到AMS中
          mgr.attachApplication(mAppThread, startSeq);
       } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
       }
    }
}
```

可以看到进程`ActivityThread#main`函数初始化的主要逻辑是：

1. **创建并启动主线程的`loop`消息循环；**
2. **通过`binder`调用`AMS`的`attachApplication`接口将自己`attach`注册到`AMS`中。**

以上初始化过程。从systrace上看如下图所示：

![img](https://s2.loli.net/2023/09/07/IxByElTXdM3g5J9.webp)

### 4. 应用主线程消息循环机制建立

接上一节的分析，我们知道应用进程创建后会通过反射创建`ActivityThread`对象并执行其`main`函数，进行主线程的初始化工作：

```java
/*frameworks/base/core/java/android/app/ActivityThread.java*/
public static void main(String[] args) {
     ...
     // 1.创建Looper、MessageQueue
     Looper.prepareMainLooper();
     ...
     // 2.启动loop消息循环，开始准备接收消息
     Looper.loop();
     ...
}

// 3.创建主线程Handler对象
final H mH = new H();

class H extends Handler {
  ...
}

/*frameworks/base/core/java/android/os/Looper.java*/
public static void prepareMainLooper() {
     // 准备主线程的Looper
     prepare(false);
     synchronized (Looper.class) {
          if (sMainLooper != null) {
              throw new IllegalStateException("The main Looper has already been prepared.");
          }
          sMainLooper = myLooper();
     }
}

private static void prepare(boolean quitAllowed) {
      if (sThreadLocal.get() != null) {
          throw new RuntimeException("Only one Looper may be created per thread");
      }
      // 创建主线程的Looper对象，并通过ThreadLocal机制实现与主线程的一对一绑定
      sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
      // 创建MessageQueue消息队列
      mQueue = new MessageQueue(quitAllowed);
      mThread = Thread.currentThread();
}
```

主线程初始化完成后，**主线程就有了完整的** **`Looper`、`MessageQueue`、`Handler`，此时** **`ActivityThread`** **的** **`Handler`** **就可以开始处理** **`Message`，包括** **`Application`、`Activity`、`ContentProvider`、`Service`、`Broadcast`** **等组件的生命周期函数，都会以** **`Message`** **的形式，在主线程按照顺序处理**，这就是 `App` 主线程的初始化和运行原理，部分处理的 `Message` 如下

```java
/*frameworks/base/core/java/android/app/ActivityThread.java*/
class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        @UnsupportedAppUsage
        public static final int RECEIVER                = 113;
        @UnsupportedAppUsage
        public static final int CREATE_SERVICE          = 114;
        @UnsupportedAppUsage
        public static final int BIND_SERVICE            = 121;
        
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    ...
            }
         }
         ...
}
```

主线程初始化完成后，主线程就进入阻塞状态，等待 `Message`，一旦有 `Message` 发过来，主线程就会被唤醒，处理 `Message`，处理完成之后，如果没有其他的 `Message` 需要处理，那么主线程就会进入休眠阻塞状态继续等待。可以说`Android`系统的运行是受消息机制驱动的，而整个消息机制是由上面所说的四个关键角色相互配合实现的（`**Handler**`、`**Looper**`、`**MessageQueue**`、`**Message**`），其运行原理如下图所示：

![img](https://s2.loli.net/2023/09/07/O1EZxNah2YyPG8b.webp)

主要是用来处理 `Message`，应用可以在任何线程创建 `Handler`，只要在创建的时候指定对应的 `Looper` 即可，如果不指定，默认是在当前 `Thread` 对应的 `Looper`。

1. **`Looper`****:**`Looper` 可以看成是一个循环器，**其****`loop`****方法开启后，不断地从****`MessageQueue`****中获取****`Message`**，对 `Message` 进行 `Delivery` 和 `Dispatch`，最终发给对应的 `Handler` 去处理。
2. `**MessageQueue**：MessageQueue` 就是一个 `Message` 管理器，队列中是 `Message`，在没有 `Message` 的时候，`**MessageQueue**`**借助**`**Linux**`**的****`ePoll`机制，阻塞休眠等待，直到有**`**Message**`**进入队列将其唤醒**。
3. `**Message**：Message` 是传递消息的对象，其内部包含了要传递的内容，最常用的包括 `what`、`arg`、`callback` 等。

## 二.应用Application和Activity组件创建与初始化

### 1 Application的创建与初始化

从前面结中的分析我们知道，**应用进程启动初始化执行`ActivityThread#main`函数过程中，在开启主线程`loop`消息循环之前，会通过`Binder`调用系统核心服务`AMS`的`attachApplication`接口将自己注册到`AMS`中**。下面我们接着这个流程往下看，我们先从systrace上看看`AMS`服务的`attachApplication`接口是如何处理应用进程的attach注册请求的：

![img](https://pic4.zhimg.com/80/v2-9fe5564a08917c9a5110ac46a4d98f03_1440w.webp)

我们继续来看相关代码的简化流程：

```java
/*frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*/
@GuardedBy("this")
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
     ...
     if (app.isolatedEntryPoint != null) {
           ...
     } else if (instr2 != null) {
           // 1.通过oneway异步类型的binder调用应用进程ActivityThread#IApplicationThread#bindApplication接口
           thread.bindApplication(...);
     } else {
           thread.bindApplication(...);
     }
     ...
     // See if the top visible activity is waiting to run in this process...
     if (normalMode) {
          try {
            // 2.继续执行启动应用Activity的流程
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
          } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
          }
      }
}

/*frameworks/base/core/java/android/app/ActivityThread.java*/
private class ApplicationThread extends IApplicationThread.Stub {
      @Override
      public final void bindApplication(...) {
            ...
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            ...
            // 向应用进程主线程Handler发送BIND_APPLICATION消息，触发在应用主线程执行handleBindApplication初始化动作
            sendMessage(H.BIND_APPLICATION, data);
      }
      ...
}

class H extends Handler {
      ...
      public void handleMessage(Message msg) {
           switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    // 在应用主线程执行handleBindApplication初始化动作
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    ...
           }
      }
      ...
}

@UnsupportedAppUsage
private void handleBindApplication(AppBindData data) {
    ...
}
```

从上面的代码流程可以看出：**`AMS`服务在执行应用的`attachApplication`注册请求过程中，会通过`oneway`类型的`binder`调用应用进程`ActivityThread#IApplicationThread`的`bindApplication`接口，而`bindApplication`接口函数实现中又会通过往应用主线程消息队列post** **`BIND_APPLICATION`消息触发执行`handleBindApplication`初始化函数**，从systrace看如下图所示：

![img](https://pic3.zhimg.com/80/v2-2891d7a66084e2a6bba2f8e399c54a6a_1440w.webp)

我们继续结合代码看看handleBindApplication的简化关键流程：

```java
/*frameworks/base/core/java/android/app/ActivityThread.java*/
@UnsupportedAppUsage
private void handleBindApplication(AppBindData data) {
    ...
    // 1.创建应用的LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    // 2.创建应用Application的Context、触发Art虚拟机加载应用APK的Dex文件到内存中，并加载应用APK的Resource资源
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    ...
    // 3.调用LoadedApk的makeApplication函数，实现创建应用的Application对象
    app = data.info.makeApplication(data.restrictedBackupMode, null);
    ...
    // 4.执行应用Application#onCreate生命周期函数
    mInstrumentation.onCreate(data.instrumentationArgs);
    ...
}
```

在`ActivityThread#**handleBindApplication`初始化过程中在应用主线程中主要完成如下几件事件**：

1. 根据框架传入的`ApplicationInfo`信息创建应用`APK`对应的`LoadedApk`对象;
2. 创建应用`Application`的`Context`对象；
3. **创建类加载器`ClassLoader`对象并触发`Art`虚拟机执行`OpenDexFilesFromOat`动作加载应用`APK`的`Dex`文件**；
4. **通过`LoadedApk`加载应用`APK`的`Resource`资源**；
5. 调用`LoadedApk`的`makeApplication`函数，创建应用的`Application`对象;
6. **执行应用`Application#onCreate`生命周期函数**（`APP`应用开发者能控制的第一行代码）;

下面我们结合代码重点看看`APK Dex`文件的加载和`Resource`资源的加载流程。

### 2.应用APK的Dex文件加载

```java
/*frameworks/base/core/java/android/app/ContextImpl.java*/
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
            String opPackageName) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    // 1.创建应用Application的Context对象
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, null,
                0, null, opPackageName);
    // 2.触发加载APK的DEX文件和Resource资源
    context.setResources(packageInfo.getResources());
    context.mIsSystemOrSystemUiContext = isSystemOrSystemUI(context);
    return context;
}

/*frameworks/base/core/java/android/app/LoadedApk.java*/
@UnsupportedAppUsage
public Resources getResources() {
     if (mResources == null) {
         ...
         // 加载APK的Resource资源
         mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                    splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                    Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                    getClassLoader()/*触发加载APK的DEX文件*/, null);
      }
      return mResources;
}

@UnsupportedAppUsage
public ClassLoader getClassLoader() {
     synchronized (this) {
         if (mClassLoader == null) {
             createOrUpdateClassLoaderLocked(null /*addedPaths*/);
          }
          return mClassLoader;
     }
}

private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {
     ...
     if (mDefaultClassLoader == null) {
          ...
          // 创建默认的mDefaultClassLoader对象，触发art虚拟机加载dex文件
          mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
                    zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
                    libraryPermittedPath, mBaseClassLoader,
                    mApplicationInfo.classLoaderName, sharedLibraries);
          ...
     }
     ...
     if (mClassLoader == null) {
         // 赋值给mClassLoader对象
         mClassLoader = mAppComponentFactory.instantiateClassLoader(mDefaultClassLoader,
                    new ApplicationInfo(mApplicationInfo));
     }
}

/*frameworks/base/core/java/android/app/ApplicationLoaders.java*/
ClassLoader getClassLoaderWithSharedLibraries(...) {
    // For normal usage the cache key used is the same as the zip path.
    return getClassLoader(zip, targetSdkVersion, isBundled, librarySearchPath,
                              libraryPermittedPath, parent, zip, classLoaderName, sharedLibraries);
}

private ClassLoader getClassLoader(String zip, ...) {
        ...
        synchronized (mLoaders) {
            ...
            if (parent == baseParent) {
                ...
                // 1.创建BootClassLoader加载系统框架类，并增加相应的systrace tag
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
                ClassLoader classloader = ClassLoaderFactory.createClassLoader(
                        zip,  librarySearchPath, libraryPermittedPath, parent,
                        targetSdkVersion, isBundled, classLoaderName, sharedLibraries);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                ...
                return classloader;
            }
            // 2.创建PathClassLoader加载应用APK的Dex类，并增加相应的systrace tag
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
            ClassLoader loader = ClassLoaderFactory.createClassLoader(
                    zip, null, parent, classLoaderName, sharedLibraries);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            return loader;
        }
}

/*frameworks/base/core/java/com/android/internal/os/ClassLoaderFactory.java*/
public static ClassLoader createClassLoader(...) {
        // 通过new的方式创建ClassLoader对象，最终会触发art虚拟机加载APK的dex文件
        ClassLoader[] arrayOfSharedLibraries = (sharedLibraries == null)
                ? null
                : sharedLibraries.toArray(new ClassLoader[sharedLibraries.size()]);
        if (isPathClassLoaderName(classloaderName)) {
            return new PathClassLoader(dexPath, librarySearchPath, parent, arrayOfSharedLibraries);
        }
        ...
}
```

从以上代码可以看出：在创建`Application`的`Context`对象后会立马尝试去加载`APK`的`Resource`资源，而在这之前需要通过`LoadedApk`去创建类加载器`ClassLoader`对象，而这个过程最终就会触发`Art`虚拟机加载应用`APK`的`dex`文件，从systrace上看如下图所示：

![img](https://s2.loli.net/2023/09/08/aUTrylHmL1N7vDW.webp)

具体art虚拟机加载dex文件的流程由于篇幅所限这里就不展开讲了，这边画了一张流程图可以参考一下，感兴趣的读者可以对照追一下源码流程：

![img](https://s2.loli.net/2023/09/08/T1yPn8VH4Yd5FCv.webp)

### 3 应用APK的Resource资源加载

```java
/*frameworks/base/core/java/android/app/LoadedApk.java*/
@UnsupportedAppUsage
public Resources getResources() {
     if (mResources == null) {
         ...
         // 加载APK的Resource资源
         mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                    splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                    Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                    getClassLoader()/*触发加载APK的DEX文件*/, null);
      }
      return mResources;
}

/*frameworks/base/core/java/android/app/ResourcesManager.java*/
public @Nullable Resources getResources(...) {
      try {
          // 原生Resource资源加载的systrace tag
          Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "ResourcesManager#getResources");
          ...
          return createResources(activityToken, key, classLoader, assetsSupplier);
      } finally {
          Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
      }
}

private @Nullable Resources createResources(...) {
      synchronized (this) {
            ...
            // 执行创建Resources资源对象
            ResourcesImpl resourcesImpl = findOrCreateResourcesImplForKeyLocked(key, apkSupplier);
            if (resourcesImpl == null) {
                return null;
            }
            ...
     }
}

private @Nullable ResourcesImpl findOrCreateResourcesImplForKeyLocked(
            @NonNull ResourcesKey key, @Nullable ApkAssetsSupplier apkSupplier) {
      ...
      impl = createResourcesImpl(key, apkSupplier);
      ...
}

private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key,
            @Nullable ApkAssetsSupplier apkSupplier) {
        ...
        // 创建AssetManager对象，真正实现的APK文件加载解析动作
        final AssetManager assets = createAssetManager(key, apkSupplier);
        ...
}

private @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key,
            @Nullable ApkAssetsSupplier apkSupplier) {
        ...
        for (int i = 0, n = apkKeys.size(); i < n; i++) {
            final ApkKey apkKey = apkKeys.get(i);
            try {
                // 通过loadApkAssets实现应用APK文件的加载
                builder.addApkAssets(
                        (apkSupplier != null) ? apkSupplier.load(apkKey) : loadApkAssets(apkKey));
            } catch (IOException e) {
                ...
            }
        }
        ...   
}

private @NonNull ApkAssets loadApkAssets(@NonNull final ApkKey key) throws IOException {
        ...
        if (key.overlay) {
            ...
        } else {
            // 通过ApkAssets从APK文件所在的路径去加载
            apkAssets = ApkAssets.loadFromPath(key.path,
                    key.sharedLib ? ApkAssets.PROPERTY_DYNAMIC : 0);
        }
        ...
    }

/*frameworks/base/core/java/android/content/res/ApkAssets.java*/
public static @NonNull ApkAssets loadFromPath(@NonNull String path, @PropertyFlags int flags)
            throws IOException {
        return new ApkAssets(FORMAT_APK, path, flags, null /* assets */);
}

private ApkAssets(@FormatType int format, @NonNull String path, @PropertyFlags int flags,
            @Nullable AssetsProvider assets) throws IOException {
        ...
        // 通过JNI调用Native层的系统system/lib/libandroidfw.so库中的相关C函数实现对APK文件压缩包的解析与加载
        mNativePtr = nativeLoad(format, path, flags, assets);
        ...
}
```

从以上代码可以看出：**系统对于应用`APK`文件资源的加载过程其实就是创建应用进程中的`Resources`资源对象的过程，其中真正实现`APK`资源文件的`I/O`解析作，最终是借助于`AssetManager`中通过JNI调用系统`Native`层的相关`C`函数实现。**整个过程从systrace上看如下图所示：

![img](https://pic1.zhimg.com/80/v2-895384d78c96f31a031138a0b7007b50_1440w.webp)

## 三.Activity的创建与初始化

我们回到6.1小结中，看看`AMS`在收到应用进程的`attachApplication`注册请求后，先通过oneway类型的binder调用应用及进程的`IApplicationThread`#`bindApplication`接口，触发应用进程在主线程执行`handleBindeApplication`初始化操作，然后继续执行启动应用`Activity`的操作，下面我们来看看系统是如何启动创建应用`Activity`的，简化代码流程如下：

```java
/*frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*/
@GuardedBy("this")
private boolean attachApplicationLocked(...) {
     ...
     if (app.isolatedEntryPoint != null) {
           ...
     } else if (instr2 != null) {
           // 1.通过oneway异步类型的binder调用应用进程ActivityThread#IApplicationThread#bindApplication接口
           thread.bindApplication(...);
     } else {
           thread.bindApplication(...);
     }
     ...
     // See if the top visible activity is waiting to run in this process...
     if (normalMode) {
          try {
            // 2.继续执行启动应用Activity的流程
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
          } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
          }
      }
}

/*frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java*/
public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
       synchronized (mGlobalLockWithoutBoost) {
            if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
                // 原生标识attachApplication过程的systrace tag
                Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "attachApplication:" + wpc.mName);
            }
            try {
                return mRootWindowContainer.attachApplication(wpc);
            } finally {
                Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            }
       }
}

/*frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java*/
boolean attachApplication(WindowProcessController app) throws RemoteException {
       ...
       final PooledFunction c = PooledLambda.obtainFunction(
                // startActivityForAttachedApplicationIfNeeded执行启动应用Activity流程
                RootWindowContainer::startActivityForAttachedApplicationIfNeeded, this,
                PooledLambda.__(ActivityRecord.class), app,
                rootTask.topRunningActivity());
       ...
}
 
private boolean startActivityForAttachedApplicationIfNeeded(ActivityRecord r,
            WindowProcessController app, ActivityRecord top) {
        ...
        try {
            // ActivityStackSupervisor的realStartActivityLocked真正实现启动应用Activity流程
            if (mStackSupervisor.realStartActivityLocked(r, app,
                    top == r && r.isFocusable() /*andResume*/, true /*checkConfig*/)) {
                ...
            }
        } catch (RemoteException e) {
            ..
        }
}

/*frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java*/
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
         ...
        // 1.先通过LaunchActivityItem封装Binder通知应用进程执行Launch Activity动作       
         clientTransaction.addCallback(LaunchActivityItem.obtain(...);
         // Set desired final state.
         final ActivityLifecycleItem lifecycleItem;
         if (andResume) {
                // 2.再通过ResumeActivityItem封装Binder通知应用进程执行Launch Resume动作        
                lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
         }
         ...
         clientTransaction.setLifecycleStateRequest(lifecycleItem);
         // 执行以上封装的Binder调用
         mService.getLifecycleManager().scheduleTransaction(clientTransaction);
         ...
}
```

从以上代码分析可以看到，框架`system_server`进程最终是通过`ActivityStackSupervisor`#`realStartActivityLocked`函数中，通过`LaunchActivityItem`和`ResumeActivityItem`两个类的封装，依次实现binder调用通知应用进程这边执行`Activity`的Launch和Resume动作的，我们继续往下看相关代码流程：

### 1 Activity Create

```java
/*frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java*/
@Override
public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
     // 原生标识Activity Launch的systrace tag
     Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
     ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
     // 调用到ActivityThread的handleLaunchActivity函数在主线程执行应用Activity的Launch创建动作
     client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
     Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}

/*frameworks/base/core/java/android/app/ActivityThread.java*/
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
     ...
     final Activity a = performLaunchActivity(r, customIntent);
     ...
}

/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        // 1.创建Activity的Context
        ContextImpl appContext = createBaseContextForActivity(r);
        try {
            //2.反射创建Activity对象
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        } catch (Exception e) {
            ...
        }
        try {
            ...
            if (activity != null) {
                ...
                // 3.执行Activity的attach动作
                activity.attach(...);
                ...
                // 4.执行应用Activity的onCreate生命周期函数,并在setContentView调用中创建DecorView对象
                mInstrumentation.callActivityOnCreate(activity, r.state);
                ...
            }
            ...
        } catch (SuperNotCalledException e) {
            ...
        }
}

/*frameworks/base/core/java/android/app/Activity.java*/
 @UnsupportedAppUsage
 final void attach(...) {
        ...
        // 1.创建表示应用窗口的PhoneWindow对象
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        ...
        // 2.为PhoneWindow配置WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        ...
}
```

从上面代码可以看出，应用进程这边在收到系统binder调用后，**在主线程中创建`Activiy`的流程主要步骤如下**：

1. 创建`Activity`的`Context`；
2. 通过反射创建`Activity`对象；
3. 执行`Activity`的`attach`动作，**其中会创建应用窗口的`PhoneWindow`对象并设置`WindowManage`**；
4. **执行应用`Activity`的`onCreate`生命周期函数，并在`setContentView`中创建窗口的`DecorView`对象**；

从systrace上看整个过程如下图所示：

![img](https://pic4.zhimg.com/80/v2-afc6216fbc4e4631ddc8a5c1ba5b348b_1440w.webp)

### 2 Activity Resume

```java
/*frameworks/base/core/java/android/app/servertransaction/ResumeActivityItem.java*/
@Override
public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
   // 原生标识Activity Resume的systrace tag
   Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
   client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
   Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}

/*frameworks/base/core/java/android/app/ActivityThread.java*/
 @Override
public void handleResumeActivity(...){
    ...
    // 1.执行performResumeActivity流程,执行应用Activity的onResume生命周期函数
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...
    if (r.window == null && !a.mFinished && willBeVisible) {
            ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    ...
                    // 2.执行WindowManager#addView动作开启视图绘制逻辑
                    wm.addView(decor, l);
                } else {
                  ...
                }
            }
     }
    ...
}

public ActivityClientRecord performResumeActivity(...) {
    ...
    // 执行应用Activity的onResume生命周期函数
    r.activity.performResume(r.startsNotResumed, reason);
    ...
}

/*frameworks/base/core/java/android/view/WindowManagerGlobal.java*/
public void addView(...) {
     // 创建ViewRootImpl对象
     root = new ViewRootImpl(view.getContext(), display);
     ...
     try {
         // 执行ViewRootImpl的setView函数
         root.setView(view, wparams, panelParentView, userId);
     } catch (RuntimeException e) {
         ...
     } 
}
```

从上面代码可以看出，应用进程这边在接收到系统Binder调用请求后，**在主线程中`Activiy`** **`Resume`的流程主要步骤如下**：

1. **执行应用`Activity`的`onResume`生命周期函数**;
2. 执行`WindowManager`的`addView`动作开启视图绘制逻辑;
3. 创建`Activity`的`ViewRootImpl`对象;
4. **执行`ViewRootImpl`的`setView`函数开启UI界面绘制动作**；

从systrace上看整个过程如下图所示：

![img](https://s2.loli.net/2023/09/08/sVKILqGeirl3Wty.webp)

## 四. 应用UI布局与绘制

接上一节的分析，应用主线程中在执行`Activity`的Resume流程的最后，会创建`ViewRootImpl`对象并调用其setView函数，从此并开启了应用界面UI布局与绘制的流程。在开始讲解这个过程之前，我们先来整理一下前面代码中讲到的这些概念，如`Activity`、`PhoneWindow`、`DecorView`、`ViewRootImpl`、`WindowManager`它们之间的关系与职责，因为这些核心类基本构成了Android系统的GUI显示系统在应用进程侧的核心架构，其整体架构如下图所示：

![img](https://s2.loli.net/2023/09/08/ugMrZlQSmj6hUeL.webp)

- `Window`是一个抽象类，**通过控制`DecorView`提供了一些标准的UI方案，比如`背景、标题、虚拟按键等`**，而`PhoneWindow`是`Window`的唯一实现类，在`Activity`创建后的attach流程中创建，应用启动显示的内容装载到其内部的`mDecor`（`DecorView`）；
- `DecorView`是整个界面布局View控件树的根节点，通过它可以遍历访问到整个View控件树上的任意节点；
- `WindowManager`是一个接口，继承自`ViewManager`接口，提供了`View`的基本操作方法；`WindowManagerImp`实现了`WindowManager`接口，内部通过`组合`方式持有`WindowManagerGlobal`，用来操作`View`；**`WindowManagerGlobal`是一个全局单例，内部可以通过`ViewRootImpl`将`View`添加至`窗口`中**；
- **`ViewRootImpl`是所有`View`的`Parent`，用来总体管理`View`的绘制以及与系统`WMS`窗口管理服务的IPC交互从而实现`窗口`的开辟**；`ViewRootImpl`是应用进程运转的发动机，可以看到`ViewRootImpl`内部包含`mView`（就是`DecorView`）、`mSurface`、`Choregrapher`，`mView`代表整个控件树，`mSurfacce`代表画布，应用的UI渲染会直接放到`mSurface`中，`Choregorapher`使得应用请求`vsync`信号，接收信号后开始渲染流程；
  我们从`ViewRootImpl`的setView流程继续结合代码往下看：

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
      synchronized (this) {
         if (mView == null) {
             mView = view;
         }
         ...
         // 开启绘制硬件加速，初始化RenderThread渲染线程运行环境
         enableHardwareAcceleration(attrs);
         ...
         // 1.触发绘制动作
         requestLayout();
         ...
         inputChannel = new InputChannel();
         ...
         // 2.Binder调用访问系统窗口管理服务WMS接口，实现addWindow添加注册应用窗口的操作,并传入inputChannel用于接收触控事件
         res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);
         ...
         // 3.创建WindowInputEventReceiver对象，实现应用窗口接收触控事件
         mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                            Looper.myLooper());
         ...
         // 4.设置DecorView的mParent为ViewRootImpl
         view.assignParent(this);
         ...
      }
}
```

从以上代码可以看出`ViewRootImpl`的setView内部关键流程如下：

1. **requestLayout()通过一系列调用触发界面绘制（measure、layout、draw）动作**，下文会详细展开分析；
2. **通过Binder调用访问系统窗口管理服务`WMS`的`addWindow`接口**，**实现添加、注册应用窗口的操作**，并传入本地创建inputChannel对象用于后续接收系统的触控事件，这一步执行完我们的`View`就可以显示到屏幕上了。关于`WMS`的内部实现流程也非常复杂，由于篇幅有限本文就不详细展开分析了。
3. 创建WindowInputEventReceiver对象，封装实现应用窗口接收系统触控事件的逻辑；
4. 执行view.assignParent(this)，设置`DecorView`的mParent为`ViewRootImpl`。所以，**虽然`ViewRootImpl`不是一个`View`,但它是所有`View`的顶层`Parent`。**

我们顺着`ViewRootImpl`的`requestLayout`动作继续往下看界面绘制的流程代码：

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
         // 检查当前UI绘制操作是否发生在主线程，如果发生在子线程则会抛出异常
         checkThread();
         mLayoutRequested = true;
         // 触发绘制操作
         scheduleTraversals();
    }
}

@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
         ...
         // 注意此处会往主线程的MessageQueue消息队列中添加同步栏删，因为系统绘制消息属于异步消息，需要更高优先级的处理
         mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
         // 通过Choreographer往主线程消息队列添加CALLBACK_TRAVERSAL绘制类型的待执行消息，用于触发后续UI线程真正实现绘制动作
         mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
         ...
     }
}
```

`Choreographer` 的引入，主要是配合系统`Vsync`垂直同步机制（Android“黄油计划”中引入的机制之一，协调APP生成UI数据和`SurfaceFlinger`合成图像，避免Tearing画面撕裂的现象），给上层 App 的渲染提供一个稳定的 `Message` 处理的时机，也就是 `Vsync` 到来的时候 ，系统通过对 `Vsync` 信号周期的调整，来控制每一帧绘制操作的时机。`**Choreographer**` **扮演 Android 渲染链路中承上启下的角色**：

1. **承上**：负责接收和处理 App 的各种更新消息和回调，等到 `Vsync` 到来的时候统一处理。比如集中处理 Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 `measure、layout、draw` 等操作) ，判断卡顿掉帧情况，记录 CallBack 耗时等；
2. **启下**：负责请求和接收 Vsync 信号。接收 Vsync 事件回调(通过 `FrameDisplayEventReceiver`.`onVsync` )，请求 `Vsync`(`FrameDisplayEventReceiver`.`scheduleVsync`) 。

`Choreographer`在收到`CALLBACK_TRAVERSAL`类型的绘制任务后，其内部的工作流程如下图所示：

![img](https://s2.loli.net/2023/09/08/1EDGx9qaoJrw4BX.webp)

从以上流程图可以看出：`ViewRootImpl`调用`Choreographer`的`postCallback`接口放入待执行的绘制消息后，`Choreographer`会先向系统申请`APP` 类型的`vsync`信号，然后等待系统`vsync`信号到来后，去回调到`ViewRootImpl`的`doTraversal`函数中执行真正的绘制动作（measure、layout、draw）。这个绘制过程从systrace上看如下图所示：

![img](https://s2.loli.net/2023/09/08/qm3Ml6I9kNE1JyT.webp)

我们接着`ViewRootImpl`的`doTraversal`函数的简化代码流程往下看：

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
void doTraversal() {
     if (mTraversalScheduled) {
         mTraversalScheduled = false;
         // 调用removeSyncBarrier及时移除主线程MessageQueue中的Barrier同步栏删，以避免主线程发生“假死”
         mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
         ...
         // 执行具体的绘制任务
         performTraversals();
         ...
    }
}

private void performTraversals() {
     ...
     // 1.从DecorView根节点出发，遍历整个View控件树，完成整个View控件树的measure测量操作
     windowSizeMayChange |= measureHierarchy(...);
     ...
     if (mFirst...) {
    // 2.第一次执行traversals绘制任务时，Binder调用访问系统窗口管理服务WMS的relayoutWindow接口，实现WMS计算应用窗口尺寸并向系统surfaceflinger正式申请Surface“画布”操作
         relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
     }
     ...
     // 3.从DecorView根节点出发，遍历整个View控件树，完成整个View控件树的layout测量操作
     performLayout(lp, mWidth, mHeight);
     ...
     // 4.从DecorView根节点出发，遍历整个View控件树，完成整个View控件树的draw测量操作
     performDraw();
     ...
}

private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
        ...
        // 通过Binder IPC访问系统WMS服务的relayout接口，申请Surface“画布”操作
        int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mTmpFrame, mTmpRect, mTmpRect, mTmpRect, mPendingBackDropFrame,
                mPendingDisplayCutout, mPendingMergedConfiguration, mSurfaceControl, mTempInsets,
                mTempControls, mSurfaceSize, mBlastSurfaceControl);
        if (mSurfaceControl.isValid()) {
            if (!useBLAST()) {
                // 本地Surface对象获取指向远端分配的Surface的引用
                mSurface.copyFrom(mSurfaceControl);
            } else {
               ...
            }
        }
        ...
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        ...
        // 原生标识View树的measure测量过程的trace tag
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            // 从mView指向的View控件树的根节点DecorView出发，遍历访问整个View树，并完成整个布局View树的测量工作
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
}

private void performDraw() {
     ...
     boolean canUseAsync = draw(fullRedrawNeeded);
     ...
}

private boolean draw(boolean fullRedrawNeeded) {
    ...
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ...
        // 如果开启并支持硬件绘制加速，则走硬件绘制的流程（从Android 4.+开始，默认情况下都是支持跟开启了硬件加速的）
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 否则走drawSoftware软件绘制的流程
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
         }
    }
}
```

从上面的代码流程可以看出，**`ViewRootImpl`中负责的整个应用界面绘制的主要流程如下**：

1. 从界面View控件树的根节点`DecorView`出发，递归遍历整个View控件树，完成对整个`View`控件树的`measure`测量操作，由于篇幅所限，本文就不展开分析这块的详细流程；
2. 界面第一次执行绘制任务时，会通过`Binder``IPC`访问系统窗口管理服务WMS的relayout接口，实现窗口尺寸的计算并向系统申请用于本地绘制渲染的Surface“画布”的操作（**具体由`SurfaceFlinger`负责创建应用界面对应的`BufferQueueLayer`对象，并通过内存共享的方式通过`Binder`将地址引用透过WMS回传给应用进程这边**），由于篇幅所限，本文就不展开分析这块的详细流程；
3. 从界面View控件树的根节点`DecorView`出发，递归遍历整个View控件树，完成对整个`View`控件树的`layout`测量操作；
4. 从界面View控件树的根节点`DecorView`出发，递归遍历整个`View`控件树，完成对整个`View`控件树的`draw`测量操作，**如果开启并支持硬件绘制加速（从Android 4.X开始谷歌已经默认开启硬件加速），则走`GPU`硬件绘制的流程，否则走`CPU`软件绘制的流程**；

以上绘制过程从systrace上看如下图所示：

![img](https://s2.loli.net/2023/09/08/BKxDrqPcsoIeAuj.webp)

![img](https://s2.loli.net/2023/09/08/TkZWj4pM3lXrRuQ.webp)

借用一张图来总结应用UI绘制的流程，如下所示：

![img](https://s2.loli.net/2023/09/08/gTRqKBiE3phCZSu.webp)

## 五. RenderThread渲染

截止到目前，在`ViewRootImpl`中完成了对界面的measure、layout和draw等绘制流程后，用户依然还是看不到屏幕上显示的应用界面内容，因为整个`Android`系统的显示流程除了前面讲到的UI线程的绘制外，界面还需要经过`RenderThread`线程的渲染处理，渲染完成后，还需要通过`Binder`调用“上帧”交给`surfaceflinger`进程中进行合成后送显才能最终显示到屏幕上。本小节中，我们将接上一节中`ViewRootImpl`中最后draw的流程继续往下分析开启硬件加速情况下，`RenderThread`渲染线程的工作流程。由于目前Android 4.X之后系统默认界面是开启硬件加速的，所以本文我们重点分析硬件加速条件下的界面渲染流程，我们先分析一下简化的代码流程：

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
private boolean draw(boolean fullRedrawNeeded) {
    ...
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ...
        // 硬件加速条件下的界面渲染流程
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        ...
    }
}

/*frameworks/base/core/java/android/view/ThreadedRenderer.java*/
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    ...
    // 1.从DecorView根节点出发，递归遍历View控件树，记录每个View节点的绘制操作命令，完成绘制操作命令树的构建
    updateRootDisplayList(view, callbacks);
    ...
    // 2.JNI调用同步Java层构建的绘制命令树到Native层的RenderThread渲染线程，并唤醒渲染线程利用OpenGL执行渲染任务；
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    ...
}
```

从上面的代码可以看出，**硬件加速绘制主要包括两个阶段**：

1. 从`DecorView`根节点出发，递归遍历`View`控件树，记录每个`View`节点的`drawOp`绘制操作命令，完成绘制操作命令树的构建；
2. `JNI`调用同步`Java`层构建的绘制命令树到`Native`层的`RenderThread`渲染线程，并唤醒渲染线程利用`OpenGL`执行渲染任务；

### 1 构建绘制命令树

我们先来看看第一阶段构建绘制命令树的代码简化流程：

```java
/*frameworks/base/core/java/android/view/ThreadedRenderer.java*/
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        // 原生标记构建View绘制操作命令树过程的systrace tag
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        // 递归子View的updateDisplayListIfDirty实现构建DisplayListOp
        updateViewTreeDisplayList(view);
        ...
        if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
            // 获取根View的SkiaRecordingCanvas
            RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
            try {
                ...
                // 利用canvas缓存DisplayListOp绘制命令
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                ...
            } finally {
                // 将所有DisplayListOp绘制命令填充到RootRenderNode中
                mRootNode.endRecording();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}

private void updateViewTreeDisplayList(View view) {
        ...
        // 从DecorView根节点出发，开始递归调用每个View树节点的updateDisplayListIfDirty函数
        view.updateDisplayListIfDirty();
        ...
}

/*frameworks/base/core/java/android/view/View.java*/
public RenderNode updateDisplayListIfDirty() {
     ...
     // 1.利用`View`对象构造时创建的`RenderNode`获取一个`SkiaRecordingCanvas`“画布”；
     final RecordingCanvas canvas = renderNode.beginRecording(width, height);
     try {
         ...
         if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
              // 如果仅仅是ViewGroup，并且自身不用绘制，直接递归子View
              dispatchDraw(canvas);
              ...
         } else {
              // 2.利用SkiaRecordingCanvas，在每个子View控件的onDraw绘制函数中调用drawLine、drawRect等绘制操作时，创建对应的DisplayListOp绘制命令，并缓存记录到其内部的SkiaDisplayList持有的DisplayListData中；
              draw(canvas);
         }
     } finally {
         // 3.将包含有`DisplayListOp`绘制命令缓存的`SkiaDisplayList`对象设置填充到`RenderNode`中；
         renderNode.endRecording();
         ...
     }
     ...
}

public void draw(Canvas canvas) {
    ...
    // draw the content(View自己实现的onDraw绘制，由应用开发者自己实现)
    onDraw(canvas);
    ...
    // draw the children
    dispatchDraw(canvas);
    ...
}

/*frameworks/base/graphics/java/android/graphics/RenderNode.java*/
public void endRecording() {
        ...
        // 从SkiaRecordingCanvas中获取SkiaDisplayList对象
        long displayList = canvas.finishRecording();
        // 将SkiaDisplayList对象填充到RenderNode中
        nSetDisplayList(mNativeRenderNode, displayList);
        canvas.recycle();
}
```

从以上代码可以看出，**构建绘制命令树的过程是从`View`控件树的根节点`DecorView`触发，递归调用每个子`View`节点的`updateDisplayListIfDirty`函数，最终完成绘制树的创建，简述流程如下**：

1. 利用`View`对象构造时创建的`RenderNode`获取一个`SkiaRecordingCanvas`“画布”；
2. 利用`SkiaRecordingCanvas`，**在每个子`View`控件的`onDraw`绘制函数中调用`drawLine`、`drawRect`等绘制操作时，创建对应的`DisplayListOp`绘制命令，并缓存记录到其内部的`SkiaDisplayList`持有的`DisplayListData`中**；
3. 将包含有`DisplayListOp`绘制命令缓存的`SkiaDisplayList`对象设置填充到`RenderNode`中；
4. 最后将根`View`的缓存`DisplayListOp`设置到`RootRenderNode`中，完成构建。

以上整个构建绘制命令树的过程可以用如下流程图表示：

![img](https://s2.loli.net/2023/09/08/HKJOz1F9eM25vnX.webp)

硬件加速下的整个界面的View树的结构如下图所示：

![img](https://s2.loli.net/2023/09/08/zEwruLJ1I6xXc7R.webp)

最后从systrace上看这个过程如下图所示：

![img](https://s2.loli.net/2023/09/08/SPpcveWVK1YDCqJ.webp)

### 2 执行渲染绘制任务

经过上一小节中的分析，应用在`UI`线程中从根节点`DecorView`出发，递归遍历每个子`View`节点，搜集其`drawXXX`绘制动作并转换成`DisplayListOp`命令，将其记录到`DisplayListData`并填充到`RenderNode`中，最终完成整个`View`绘制命令树的构建。从此UI线程的绘制任务就完成了。下一步`UI`线程将唤醒`RenderThread`渲染线程，触发其利用`OpenGL`执行界面的渲染任务，本小节中我们将重点分析这个流程。我们还是先看看这块代码的简化流程：

```java
/*frameworks/base/graphics/java/android/graphics/HardwareRenderer.java*/
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
    // JNI调用native层的相关函数
    return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}

/*frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp*/
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    ...
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}

/*frameworks/base/libs/hwui/renderthread/RenderProxy.cpp*/
int RenderProxy::syncAndDrawFrame() {
    // 唤醒RenderThread渲染线程，执行DrawFrame绘制任务
    return mDrawFrameTask.drawFrame();
}

/*frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp*/
int DrawFrameTask::drawFrame() {
    ...
    postAndWait();
    ...
}

void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    // 向RenderThread渲染线程的MessageQueue消息队列放入一个待执行任务，以将其唤醒执行run函数
    mRenderThread->queue().post([this]() { run(); });
    // UI线程暂时进入wait等待状态
    mSignal.wait(mLock);
}

void DrawFrameTask::run() {
    // 原生标识一帧渲染绘制任务的systrace tag
    ATRACE_NAME("DrawFrame");
    ...
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        //1.将UI线程构建的DisplayListOp绘制命令树同步到RenderThread渲染线程
        canUnblockUiThread = syncFrameState(info);
        ...
    }
    ...
    // 同步完成后则可以唤醒UI线程
    if (canUnblockUiThread) {
        unblockUiThread();
    }
    ...
    if (CC_LIKELY(canDrawThisFrame)) {
        // 2.执行draw渲染绘制动作
        context->draw();
    } else {
        ...
    }
    ...
}

bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    ATRACE_CALL();
    ...
    // 调用CanvasContext的prepareTree函数实现绘制命令树同步的流程
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);
    ...
}

/*frameworks/base/libs/hwui/renderthread/CanvasContext.cpp*/
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued,
                                RenderNode* target) {
     ...
     for (const sp<RenderNode>& node : mRenderNodes) {
        ...
        // 递归调用各个子View对应的RenderNode执行prepareTree动作
        node->prepareTree(info);
        ...
    }
    ...
}

/*frameworks/base/libs/hwui/RenderNode.cpp*/
void RenderNode::prepareTree(TreeInfo& info) {
    ATRACE_CALL();
    ...
    prepareTreeImpl(observer, info, false);
    ...
}

void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
    ...
    if (info.mode == TreeInfo::MODE_FULL) {
        // 同步绘制命令树
        pushStagingDisplayListChanges(observer, info);
    }
    if (mDisplayList) {
        // 遍历调用各个子View对应的RenderNode的prepareTreeImpl
        bool isDirty = mDisplayList->prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                   bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                });
        ...
    }
    ...
}

void RenderNode::pushStagingDisplayListChanges(TreeObserver& observer, TreeInfo& info) {
    ...
    syncDisplayList(observer, &info);
    ...
}

void RenderNode::syncDisplayList(TreeObserver& observer, TreeInfo* info) {
    ...
    // 完成赋值同步DisplayList对象
    mDisplayList = mStagingDisplayList;
    mStagingDisplayList = nullptr;
    ...
}

void CanvasContext::draw() {
    ...
    // 1.调用OpenGL库使用GPU，按照构建好的绘制命令完成界面的渲染
    bool drew = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue,
                                      mContentDrawBounds, mOpaque, mLightInfo, mRenderNodes,
                                      &(profiler()));
    ...
    // 2.将前面已经绘制渲染好的图形缓冲区Binder上帧给SurfaceFlinger合成和显示
    bool didSwap =
            mRenderPipeline->swapBuffers(frame, drew, windowDirty, mCurrentFrameInfo, &requireSwap);
    ...
}
```

从以上代码可以看出：`UI`线程利用`RenderProxy`向`RenderThread`线程发送一个`DrawFrameTask`任务请求，**`RenderThread`被唤醒，开始渲染，大致流程如下**：

1. `syncFrameState`中遍历`View`树上每一个`RenderNode`，执行`prepareTreeImpl`函数，实现同步绘制命令树的操作；
2. 调用`OpenGL`库`API`使用`GPU`，按照构建好的绘制命令完成界面的渲染（具体过程，由于本文篇幅所限，暂不展开分析）；
3. 将前面已经绘制渲染好的图形缓冲区`Binder`上帧给`SurfaceFlinger`合成和显示；

整个过程可以用如下流程图表示：

![img](https://s2.loli.net/2023/09/08/3wcSklGu5XQR9Nz.webp)

从systrace上这个过程如下图所示：

![img](https://s2.loli.net/2023/09/08/wqmLNgtAnRIeHib.webp)
