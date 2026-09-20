---
title: KotlinrunBlockinglaunchjoinasyncdelay 原理
date: 2024-02-16 10:18:40
tags: Kotlin
categories: Kotlin
permalink: 2024/02/16/Kotlin协程函数/
---
# 来，跟我一起撸Kotlin runBlocking/launch/join/async/delay 原理&使用

前言
协程系列文章：

一个小故事讲明白进程、线程、Kotlin 协程到底啥关系？
少年，你可知 Kotlin 协程最初的样子？
讲真，Kotlin 协程的挂起/恢复没那么神秘(故事篇)
讲真，Kotlin 协程的挂起/恢复没那么神秘(原理篇)
Kotlin 协程调度切换线程是时候解开真相了
Kotlin 协程之线程池探索之旅(与Java线程池PK)
Kotlin 协程之取消与异常处理探索之旅(上)
Kotlin 协程之取消与异常处理探索之旅(下)
来，跟我一起撸Kotlin runBlocking/launch/join/async/delay 原理&使用
继续来，同我一起撸Kotlin Channel 深水区
Kotlin 协程 Select：看我如何多路复用
Kotlin Sequence 是时候派上用场了
Kotlin Flow 背压和线程切换竟然如此相似
Kotlin Flow啊，你将流向何方？
Kotlin SharedFlow&StateFlow 热流到底有多热？
之前一些列的文章重点在于分析协程本质原理，了解了协程的内核再来看其它衍生的知识就比较容易了。
接下来这边文章着重分析协程框架提供的一些重要的函数原理，通过本篇文章，你将了解到：

runBlocking 使用与原理
launch 使用与原理
join 使用与原理
async/await 使用与原理
delay 使用与原理

- runBlocking 使用与原理

- 默认分发器的runBlocking
使用

- 老规矩，先上Demo：

```kotlin

    fun testBlock() {
        println("before runBlocking thread:${Thread.currentThread()}")
        //①
        runBlocking {
            println("I'm runBlocking start thread:${Thread.currentThread()}")
            Thread.sleep(2000)
            println("I'm runBlocking end")
        }
        //②
        println("after runBlocking:${Thread.currentThread()}")
 }
runBlocking 开启了一个新的协程，它的特点是：
```

协程执行结束后才会执行runBlocking 后的代码。

也就是① 执行结束后 ② 才会执行。

可以看出，协程运行在当前线程，因此若是在协程里执行了耗时函数，那么协程之后的代码只能等待，基于这个特性，runBlocking 经常用于一些测试的场景。

runBlocking 可以定义返回值，比如返回一个字符串：

```kotlin
fun testBlock2() {
    var name = runBlocking {
        "fish"
    }
    println("name $name")
}
```

原理

```kotlin
#Builders.kt
    public fun <T> runBlocking(context: CoroutineContext = EmptyCoroutineContext, block: suspend CoroutineScope.() -> T): T {
        //当前线程
        val currentThread = Thread.currentThread()
        //先看有没有拦截器
        val contextInterceptor = context[ContinuationInterceptor]
        val eventLoop: EventLoop?
        val newContext: CoroutineContext
        //----------①
        if (contextInterceptor == null) {
            //不特别指定的话没有拦截器，使用loop构建Context
            eventLoop = ThreadLocalEventLoop.eventLoop
            newContext = GlobalScope.newCoroutineContext(context + eventLoop)
        } else {
            eventLoop = (contextInterceptor as? EventLoop)?.takeIf { it.shouldBeProcessedFromContext() }
                ?: ThreadLocalEventLoop.currentOrNull()
            newContext = GlobalScope.newCoroutineContext(context)
        }
        //BlockingCoroutine 顾名思义，阻塞的协程
        val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
        //开启
        coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
        //等待协程执行完成----------②
        return coroutine.joinBlocking()
    }
```

重点看①②。

先说①，因为我们没有指定分发器，因此会使用loop，实际创建的是BlockingEventLoop，它继承自EventLoopImplBase，最终继承自CoroutineDispatcher（注意此处是个重点）。
根据我们之前分析的协程知识可知，协程启动后会构造DispatchedContinuation，然后依靠dispatcher将runnable 分发执行，而这个dispatcher 即是BlockingEventLoop。

```kotlin
#EventLoop.common.kt
//重写dispatch函数
public final override fun dispatch(context: CoroutineContext, block: Runnable) = enqueue(block)

public fun enqueue(task: Runnable) {
    //将task 加入队列，task = DispatchedContinuation
    if (enqueueImpl(task)) {
        unpark()
    } else {
        DefaultExecutor.enqueue(task)
    }
}
BlockingEventLoop 的父类EventLoopImplBase 里有个成员变量：_queue，它是个队列，用来存储提交的任务。
```

再看②：
协程任务已经提交到队列里，就看啥时候取出来执行了。

```kotlin
#Builders.kt
    fun joinBlocking(): T {
        try {
            try {
                while (true) {
                    //当前线程已经中断了，直接退出
                    if (Thread.interrupted()) throw InterruptedException().also { cancelCoroutine(it) }
                    //如果eventLoop！= null，则从队列里取出task并执行
                    val parkNanos = eventLoop?.processNextEvent() ?: Long.MAX_VALUE
                    //协程执行结束，跳出循环
                    if (isCompleted) break
                    //挂起线程，parkNanos 指的是挂起时间
                    parkNanos(this, parkNanos)
                    //当线程被唤醒后，继续while循环
                }
            } finally { // paranoia
            }
        }
        //返回结果
        return state as T
    }

#EventLoop.common.kt
    override fun processNextEvent(): Long {
        //延迟队列
        val delayed = _delayed.value
        //延迟队列处理，这里在分析delay时再解释
        //从队列里取出task
        val task = dequeue()
        if (task != null) {
            //执行task
            task.run()
            return 0
        }
        return nextTime
    }
```

上面代码的任务有两个：

尝试从队列里取出Task。
若是没有则挂起线程。
结合①②两点，再来过一下场景：

先创建协程，包装为DispatchedContinuation，作为task。
分发task，将task加入到队列里。
从队列里取出task执行，实际执行的即是协程体。
当3执行完毕后，runBlocking()函数也就退出了。

其中虚线箭头表示执行先后顺序。
由此可见，runBlocking()函数需要等待协程执行完毕后才退出。

指定分发器的runBlocking
上个Demo在使用runBlocking 时没有指定其分发器，若是指定了又是怎么样的流程呢？

```kotlin
fun testBlock3() {
    println("before runBlocking thread:${Thread.currentThread()}")
    //①
    runBlocking(Dispatchers.IO) {
        println("I'm runBlocking start thread:${Thread.currentThread()}")
        Thread.sleep(2000)
        println("I'm runBlocking end")
    }
    //②
    println("after runBlocking:${Thread.currentThread()}")
}
指定在子线程里进行分发。
此处与默认分发器最大的差别在于：
```

默认分发器加入队列、取出队列都是同一个线程，而指定分发器后task不会加入到队列里，task的调度执行完全由指定的分发器完成。

也就是说，coroutine.joinBlocking()后，当前线程一定会被挂起。等到协程执行完毕后再唤醒当前被挂起的线程。
唤醒之处在于：

```kotlin
#Builders.kt
    override fun afterCompletion(state: Any?) {
        // wake up blocked thread
        if (Thread.currentThread() != blockedThread)
            //blockedThread 即为调用coroutine.joinBlocking()后阻塞的线程
            //Thread.currentThread() 为线程池的线程
            //唤醒线程
            unpark(blockedThread)
    }
```

红色部分比紫色部分先执行，因此红色部分执行的线程会阻塞，等待紫色部分执行完毕后将它唤醒，最后runBlocking()函数执行结束了。

不管是否指定分发器，runBlocking() 都会阻塞等待协程执行完毕。

- launch 使用与原理
想必大家刚接触协程的时候使用最多的还是launch启动协程吧。
看个Demo：

```kotlin
    fun testLaunch() {
        var job = GlobalScope.launch {
            println("hello job1 start")//①
            Thread.sleep(2000)
            println("hello job1 end")//②
        }
        println("continue...")//③
 }
```kotlin

非常简单，启动一个线程，打印结果如下：

③一定比①②先打印，同时也说明launch()函数并不阻塞当前线程。
关于协程原理，在之前的文章都有深入分析，此处不再赘述，以图示之：

3. join 使用与原理
虽然launch()函数不阻塞线程，但是我们就想要知道协程执行完毕没，进而根据结果确定是否继续往下执行，这时候该Job.join()出场了。
先看该函数的定义：

#Job.kt
public suspend fun join()
是个suspend 修饰的函数，suspend 是咱们的老朋友了，说明协程执行到该函数会挂起(当前线程不阻塞，另有他用)。
继续看其实现：

```kotlin
#JobSupport.kt
public final override suspend fun join() {
    //快速判断状态，不耗时
    if (!joinInternal()) { // fast-path no wait
        coroutineContext.ensureActive()
        return // do not suspend
    }
    //挂起的地方
    return joinSuspend() // slow-path wait
}

//suspendCancellableCoroutine 典型的挂起操作
//cont 是封装后的协程
private suspend fun joinSuspend() = suspendCancellableCoroutine<Unit> { cont ->
    //执行完这就挂起
    //disposeOnCancellation 是将cont 记录在当前协程的state里，构造为node
    cont.disposeOnCancellation(invokeOnCompletion(handler = ResumeOnCompletion(cont).asHandler))
}
```

其中suspendCancellableCoroutine 是挂起的核心所在，关于挂起的详细分析请移步：讲真，Kotlin 协程的挂起没那么神秘(原理篇)

joinSuspend()函数有2个作用：

将当前协程体存储到Job的state里（作为node)。
将当前协程挂起。
什么时候恢复呢？当然是协程执行完成后。

```kotlin
#JobSupport.kt
private class ResumeOnCompletion(
    private val continuation: Continuation<Unit>
) : JobNode() {
    //continuation 为协程的包装体，它里面有我们真正的协程体
    //之后重新进行分发
    override fun invoke(cause: Throwable?) = continuation.resume(Unit)
    }
```

当协程执行完毕，会例行检查当前的state是否有挂着需要执行的node，刚好我们在joinSuspend()里放了node，于是找到该node，进而找到之前的协程体再次进行分发。根据协程状态机的知识可知，这是第二次执行协程体，因此肯定会执行job.join()之后的代码，于是乎看起来的效果就是：

job.join() 等待协程执行完毕后才会往下执行。

语言比较苍白，来个图：

注：此处省略了协程挂起等相关知识，如果对此有疑惑请阅读之前的文章。

- async/await 使用与原理
launch 有2点不足之处：协程执行没有返回值。
这点我们从它的定义很容易获悉：

然而，在有些场景我们需要返回值，此时轮到async/await 出场了。

```kotlin
fun testAsync() {
    runBlocking {
        //启动协程
        var job = GlobalScope.async {
            println("job1 start")
            Thread.sleep(10000)
            //返回值
            "fish"
        }
        //等待协程执行结束，并返回协程结果
        var result = job.await()
        println("result:$result")
    }
}
```

运行结果：

接着来看实现原理。

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    //构造DeferredCoroutine
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    //coroutine == DeferredCoroutinek
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

与launch 启动方式不同的是，async 的协程定义了返回值，是个泛型。并且async里使用的是DeferredCoroutine，顾名思义：延迟给结果的协程。
后面的流程都是一样的，不再细说。

再来看Job.await()，它与Job.join()类似：

先判断是否需要挂起，若是协程已经结束/被取消，当然就无需等待直接返回。
先将当前协程体包装到state里作为node存放，然后挂起协程。
等待async里的协程执行完毕，再重新调度执行await()之后的代码。
此时协程的值已经返回。
这里需要重点关注一下返回值是怎么传递过来的。

将testAsync()反编译：

```kotlin
public final Object invokeSuspend(@NotNull Object $result) {
    //result 为协程执行结果
    Object var6 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
    Object var10000;
    switch(this.label) {
        case 0:
            //第一次执行这
            ResultKt.throwOnFailure($result);
            Deferred job = BuildersKt.async$default((CoroutineScope) GlobalScope.INSTANCE, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
                int label;
                @Nullable
                public final Object invokeSuspend(@NotNull Object var1) {
                    Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                    switch(this.label) {
                        case 0:
                            ResultKt.throwOnFailure(var1);
                            String var2 = "job1 start";
                            boolean var3 = false;
                            System.out.println(var2);
                            Thread.sleep(10000L);
                            return "fish";
                        default:
                            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                    }
                }
            }), 3, (Object)null);
            this.label = 1;
            //挂起
            var10000 = job.await(this);
            if (var10000 == var6) {
                return var6;
            }
            break;
        case 1:
            //第二次执行这
            ResultKt.throwOnFailure($result);
            //result 就是demo里的"fish"
            var10000 = $result;
            break;
        default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    String result = (String)var10000;
    String var4 = "result:" + result;
    boolean var5 = false;
    System.out.println(var4);
    return Unit.INSTANCE;
}
```

很明显，外层的协程(runBlocking)体会执行2次。
第1次：调用invokeSuspend(xx)，此时参数xx=Unit，后遇到await 被挂起。
第2次：子协程执行结束并返回结果”fish”，恢复外部协程时再次调用invokeSuspend(xx)，此时参数xx=“fish”，并将参数保存下来，因此result 就有了值。

值得注意的是：
async 方式启动的协程，若是协程发生了异常，不会像launch 那样直接抛出，而是需要等待调用await()时抛出。

- delay 使用与原理
线程可以被阻塞，协程可以被挂起，挂起后的协程等待时机成熟可以被恢复。

```kotlin
    fun testDelay() {
        GlobalScope.launch {
            println("before getName")
            var name = getUserName()
            println("after getName name:$name")
        }
    }
    suspend fun getUserName():String {
        return withContext(Dispatchers.IO) {
            //模拟网络获取
            Thread.sleep(2000)
            "fish"
        }
 }

```kotlin

获取用户名字是在子线程获取的，它是个挂起函数，当协程执行到此时挂起，等待获取名字之后再恢复运行。

有时候我们仅仅只是想要协程挂起一段时间，并不需要去做其它操作，这个时候我们可以选择使用delay(xx)函数：

```kotlin
fun testDelay2() {
    GlobalScope.launch {
        println("before delay")
        //协程挂起5s
        delay(5000)
        println("after delay")
    }
}
```

再来看看其原理。

```kotlin
#Delay.kt
    public suspend fun delay(timeMillis: Long) {
        //没必要延时
        if (timeMillis <= 0) return // don't delay
        return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
            //封装协程为cont，便于之后恢复
            if (timeMillis < Long.MAX_VALUE) {
                //核心实现
                cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
            }
        }
    }
```

主要看context.delay 实现：

```kotlin
#DefaultExecutor.kt
internal actual val DefaultDelay: Delay = kotlinx.coroutines.DefaultExecutor

//单例
internal actual object DefaultExecutor : EventLoopImplBase(), Runnable {
    const val THREAD_NAME = "kotlinx.coroutines.DefaultExecutor"
    //...
    private fun createThreadSync(): Thread {
        return DefaultExecutor._thread ?: Thread(this, DefaultExecutor.THREAD_NAME).apply {
            DefaultExecutor._thread = this
            isDaemon = true
            start()
        }
    }
    //...
    override fun run() {
        //循环检测队列是否有内容需要处理
        //决定是否要挂起线程
    }
    //...
}
```

DefaultExecutor 是个单例，它里边开启了线程，并且检测队列里任务的情况来决定是否需要挂起线程等待。

先看队列的出入队情况。

放入队列
我们注意到DefaultExecutor 继承自EventLoopImplBase()，在最开始分析runBlocking()时有提到过它里面有成员变量_queue 存储队列元素，实际上它还有另一个成员变量_delayed：

```kotlin
#EventLoop.common.kt
internal abstract class EventLoopImplBase: EventLoopImplPlatform(), Delay {
    //存放正常task
    private val _queue = atomic<Any?>(null)
    //存放延迟task
    private val _delayed = atomic<EventLoopImplBase.DelayedTaskQueue?>(null)
}

private inner class DelayedResumeTask(
    nanoTime: Long,
    private val cont: CancellableContinuation<Unit>
) : EventLoopImplBase.DelayedTask(nanoTime) {
    //协程恢复
    override fun run() { with(cont) { resumeUndispatched(Unit) } }
    override fun toString(): String = super.toString() + cont.toString()
}
```

delay.scheduleResumeAfterDelay 本质是创建task：DelayedResumeTask，并将该task加入到延迟队列_delayed里。

从队列取出
DefaultExecutor 一开始就会调用processNextEvent()函数检测队列是否有数据，如果没有则将线程挂起一段时间(由processNextEvent()返回值确定)。
那么重点转移到processNextEvent()上。

```kotlin
##EventLoop.common.kt
    override fun processNextEvent(): Long {
        if (processUnconfinedEvent()) return 0
        val delayed = _delayed.value
        if (delayed != null && !delayed.isEmpty) {
            //调用delay 后会放入
            //查看延迟队列是否有任务
            val now = nanoTime()
            while (true) {
                //一直取任务，直到取不到(时间未到)
                delayed.removeFirstIf {
                    //延迟任务时间是否已经到了
                    if (it.timeToExecute(now)) {
                        //将延迟任务从延迟队列取出，并加入到正常队列里
                        enqueueImpl(it)
                    } else
                        false
                } ?: break // quit loop when nothing more to remove or enqueueImpl returns false on "isComplete"
            }
        }
               // 从正常队列里取出
        val task = dequeue()
        if (task != null) {
            //执行
            task.run()
            return 0
        }
        //返回线程需要挂起的时间
        return nextTime
    }
```

而执行任务最终就是执行DelayedResumeTask.run()函数，该函数里会对协程进行恢复。

至此，delay 流程就比较清晰了：

构造task 加入到延迟队列里，此时协程挂起。
有个单独的线程会检测是否需要取出task并执行，没到时间的话就要挂起等待。
时间到了从延迟队列里取出并放入正常的队列，并从正常队列里取出执行。
task 执行的过程就是协程恢复的过程。
老规矩，上图：

图上虚线紫色框部分表明delay 执行到此就结束了，协程挂起(不阻塞当前线程），剩下的就交给单例的DefaultExecutor 调度，等待延迟的时间结束后通知协程恢复即可。

关于协程一些常用的函数分析到此就结束了，下篇开始我们一起探索协程通信(Channel/Flow 等)相关知识。
由于篇幅原因，省略了一些源码的分析，若你对此有疑惑，可评论或私信小鱼人。

本文基于Kotlin 1.5.3，文中完整Demo请点击

您若喜欢，请点赞、关注、收藏，您的鼓励是我前进的动力
持续更新中，和我一起步步为营系统、深入学习Android/Kotlin
1、Android各种Context的前世今生
2、Android DecorView 必知必会
3、Window/WindowManager 不可不知之事
4、View Measure/Layout/Draw 真明白了
5、Android事件分发全套服务
6、Android invalidate/postInvalidate/requestLayout 彻底厘清
7、Android Window 如何确定大小/onMeasure()多次执行原因
8、Android事件驱动Handler-Message-Looper解析
9、Android 键盘一招搞定
10、Android 各种坐标彻底明了
11、Android Activity/Window/View 的background
12、Android Activity创建到View的显示过
13、Android IPC 系列
14、Android 存储系列
15、Java 并发系列不再疑惑
16、Java 线程池系列
17、Android Jetpack 前置基础系列
18、Android Jetpack 易学易懂系列
19、Kotlin 轻松入门系列
20、Kotlin 协程系列全面解读

文章知识点与官方知识档案匹配，可进一步学习相关知识

[]: [https://blog.csdn.net/wekajava/article/details/126207606](https://blog.csdn.net/wekajava/article/details/126207606)
