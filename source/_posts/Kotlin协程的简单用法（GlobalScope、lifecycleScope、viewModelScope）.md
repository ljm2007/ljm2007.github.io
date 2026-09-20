---
title: Kotlin协程的简单用法（GlobalScope、lifecycleScope、viewModelScope）
top_img: https://s2.loli.net/2024/03/07/DNjiK3n4k7Q86mZ.png
cover: https://s2.loli.net/2024/03/07/DNjiK3n4k7Q86mZ.png
date: 2024-03-07 23:41:40
tags: Kotlin
permalink: 2024/03/07/Kotlin协程的简单用法（GlobalScope、lifecycleScope、viewModelScope）/
---
# Kotlin协程的简单用法（GlobalScope、lifecycleScope、viewModelScope）

![Kotlin: A Beginner's Guide and Tutorial](https://s2.loli.net/2024/03/07/DNjiK3n4k7Q86mZ.png)

## 协程（Coroutine）

协程就像非常轻量级的线程。线程是由系统调度的，线程切换或线程阻塞的开销都比较大。而协程依赖于线程，但是协程挂起时不需要阻塞线程，协程是由开发者控制的。所以协程也像用户态的线程，非常轻量级，一个线程中可以创建任意个协程。
协程就像轻量级的线程。线程由系统调度，协程由开发者控制。
kotlin协程本质上是对线程池的封装
协程通过将线程切换的复杂性封装入库来简化异步编程。程序的逻辑可以在协程中顺序地表达，而底层库会为我们解决其异步性。

## GlobalScope（不推荐）

GlobalScope.launch
使用的是DefaultDispatcher，会自动切换到后台线程，不能做UI操作

​

```kotlin
         GlobalScope.launch {
     	//GlobalScope开启协程：DefaultDispatcher-worker-1
      	Log.d(TAG, "GlobalScope开启协程：" + Thread.currentThread().name)
//子线程中此处不可以做UI操作
//Toast.makeText(this@MainActivity, "GlobalScope开启协程", Toast.LENGTH_SHORT).show()
}
```

可以在协程中切换线程

```kotlin
     GlobalScope.launch {
     	//GlobalScope开启协程：DefaultDispatcher-worker-1
      	Log.d(TAG, "GlobalScope开启协程：" + Thread.currentThread().name)
//子线程中此处不可以做UI操作
//Toast.makeText(this@MainActivity, "GlobalScope开启协程", Toast.LENGTH_SHORT).show()
withContext(Dispatchers.Main){
             Toast.makeText(this@MainActivity, "协程中切换线程", Toast.LENGTH_SHORT).show()
         }
     }
```

GlobalScope.launch(Dispatchers.Main)
通过Dispatchers.Main使协程依托于主线程中，此时可以更新UI等操作

```kotlin
GlobalScope.launch(Dispatchers.Main) {
  	//GlobalScope开启协程：main
      Log.d(TAG, "GlobalScope开启协程：" + Thread.currentThread().name)
      //可以做UI操作
      Toast.makeText(this@MainActivity, "GlobalScope开启协程", Toast.LENGTH_SHORT).show()
  }
```

## lifecycleScope、viewModelScope（推荐）

- 引入方式

```kotlin
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.2.0'//lifecycleScope
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0'//viewModelScope
```

**GlobalScope是生命周期是process级别的，即使Activity或Fragment已经被销毁，协程仍然在执行。所以需要绑定生命周期。**
**lifecycleScope只能在Activity、Fragment中使用，会绑定Activity和Fragment的生命周期**
**viewModelScope只能在ViewModel中使用，绑定ViewModel的生命周期**

## 协程的执行顺序

```kotlin
private fun test() {
     Log.d(TAG, "test: 方法开始")
     lifecycleScope.launch {
         delay(1000)
         Log.d(TAG, "test: " + Thread.currentThread().name)
         Log.d(TAG, "test: 协程结束")
         Toast.makeText(this@MainActivity, "协程结束", Toast.LENGTH_SHORT).show()
     }
     Log.d(TAG, "test: 方法结束")
 }
```

```plaintext
D/MainActivity: test: 方法开始
D/MainActivity: test: 方法结束
D/MainActivity: test: main
D/MainActivity: test: 协程结束
```

协程内的阻塞不会影响协程外
由打印结果可以看出协程体是异步执行的，但是可以在其中做UI操作。线程也是异步的，但是不能更新UI，线程需要先切换到主线程。
协程中多个耗时任务的串行

## 默认情况下协程中的内容是串行的

```kotlin
   private fun test2() {
        lifecycleScope.launch {
            val startTime = System.currentTimeMillis()
            val a = getDataA()
            val b = getDataB()
            val sum = a + b
            //D/MainActivity: test2: sum = 3，耗时：3008
            Log.d(TAG, "test2: sum = $sum，耗时：${System.currentTimeMillis() - startTime}")
        }
    }
    private suspend fun getDataA(): Int {
    delay(1000)
    return 1
}

private suspend fun getDataB(): Int {
    delay(2000)
    return 2
}
```

```plaintext
D/MainActivity: test2: sum = 3，耗时：3008
```

## 协程中多个耗时任务的并行

如果需要并行，例如请求多个接口拿到数据后才能进行操作

```kotlin
private fun test3(){
      lifecycleScope.launch {
          val startTime = System.currentTimeMillis()
          val a = lifecycleScope.async { getDataA() }
          val b = lifecycleScope.async { getDataB() }
          val sum = a.await() + b.await()
          //D/MainActivity: test3: sum = 3，耗时：2009
          Log.d(TAG, "test3: sum = $sum，耗时：${System.currentTimeMillis() - startTime}")
      }
  }
```

​

```kotlin
private suspend fun getDataA(): Int {
    delay(1000)
    return 1
}

private suspend fun getDataB(): Int {
    delay(2000)
    return 2
}
```

```plaintext
D/MainActivity: test3: sum = 3，耗时：2009
```

## 协程的停止

手动停止的情况 job?.cancel()

```kotlin
    private var job: Job? = null
private fun test4() {
    job = lifecycleScope.launch {
        ...
    }
    job?.cancel()
}
```

lifecycleScope和viewModelScope会绑定调用者的生命周期，因此通常情况下不需要手动去停止

## suspend协程挂起原理

在编译期，将suspend标记的方法转化成接口回调的方式，本质上还是基于回调实现的。
