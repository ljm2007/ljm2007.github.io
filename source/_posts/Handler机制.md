---
title: Android Handler：图文解析 Handler通信机制 的工作原理
top_img: https://s2.loli.net/2024/02/24/qYeL3VQMEbXmwJd.webp
cover: https://s2.loli.net/2024/02/24/qYeL3VQMEbXmwJd.webp
date: 2024-02-17 10:18:40
tags: android
categories: android
permalink: 2024/02/17/Handler机制/
---
# Android Handler：图文解析 Handler通信机制 的工作原理

[Carson带你学Android](https://juejin.cn/user/2524134385917293/posts)

2018-03-072,244阅读2分钟

原文链接： [www.jianshu.com](https://link.juejin.cn/?target=https://www.jianshu.com/p/f0b23ee5a922)

![img](https://s2.loli.net/2024/02/24/qYeL3VQMEbXmwJd.webp)

# 前言

- 在`Android`开发的**多线程应用场景**中，`Handler`机制十分常用

- 今天，我将图文详解 `Handler`机制 的工作原理，希望你们会喜欢

---

# 目录

![img](https://s2.loli.net/2024/02/24/3JlWQu1XGvmIPgN.webp) *示意图*

---

# 1. 定义

一套 `Android` 消息传递机制

---

# 2. 作用

在多线程的应用场景中，**将工作线程中需更新`UI`的操作信息 传递到 `UI`主线程**，从而实现 工作线程对`UI`的更新处理，最终实现异步消息的处理

![img](https://s2.loli.net/2024/02/24/FqxBXH4AULEKz3M.webp)*示意图*

---

# 3. 为什么要用 `Handler`消息传递机制

- 答：**多个线程并发更新UI的同时 保证线程安全**

- 具体描述如下

![img](https://s2.loli.net/2024/02/24/iGrkYlPhXKTUIHv.webp) *示意图*

---

# 4. 相关概念

关于 `Handler`机制中的相关概念如下：

> 在下面的讲解中，我将直接使用英文名讲解，即 `Handler`、`Message`、`Message Queue`、`Looper`，希望大家先熟悉相关概念

![img](https://s2.loli.net/2024/02/24/sqmfMF6vRla1W9x.webp) *示意图*

---

# 5. 工作原理 解析

下面，我将定性地讲解`Handler`机制的工作流程

### 5.1 工作流程解析

`Handler`机制的工作流程主要包括4个步骤：

- 异步通信准备

- 消息发送

- 消息循环

- 消息处理

具体如下图：

![img](https://s2.loli.net/2024/02/24/tZ6IgTcWujC7H1Q.webp) *示意图*

### 5.2 工作流程图

![img](https://s2.loli.net/2024/02/24/ytfq3nRgaYQCNPm.webp) *示意图*

### 5.3 示意图

![img](https://s2.loli.net/2024/02/24/MKvsOAQ7B9rZzbh.webp) *示意图*

### 5.4 特别注意

线程`（Thread）`、循环器`（Looper）`、处理者`（Handler）`之间的对应关系如下：

- 1个线程`（Thread）`只能绑定 1个循环器`（Looper）`，但可以有多个处理者`（Handler）`

- 1个循环器`（Looper）` 可绑定多个处理者`（Handler）`

- 1个处理者`（Handler）` 只能绑定1个1个循环器`（Looper）`

![img](https://s2.loli.net/2024/02/24/5xDG94nYCOSog3Q.webp) *示意图*

至此，关于`Handler`的异步消息传递机制的工作原理 讲解完毕。

---

# 6. 总结

- 本文对`Handler`机制的工作原理进行了全面讲解

- 下面我将继续深入讲解 `Android`中的`Handler`异步通信传递机制的相关知识，如 使用教程、源码解析等，有兴趣可以继续关注[Carson_Ho的安卓开发笔记](https://link.juejin.cn/?target=https://www.jianshu.com/users/383970bef0a0/latest_articles)

---

# 请点赞！因为你的鼓励是我写作的最大动力！

> **相关文章阅读**
> [Android开发：最全面、最易懂的Android屏幕适配解决方案](https://link.juejin.cn/?target=https://www.jianshu.com/p/ec5a1a30694b)
> [Android事件分发机制详解：史上最全面、最易懂](https://link.juejin.cn/?target=https://www.jianshu.com/p/38015afcdb58)
> [Android开发：史上最全的Android消息推送解决方案](https://link.juejin.cn/?target=https://www.jianshu.com/p/b61a49e0279f)
> [Android开发：最全面、最易懂的Webview详解](https://link.juejin.cn/?target=https://www.jianshu.com/p/3c94ae673e2a)
> [Android开发：JSON简介及最全面解析方法!](https://link.juejin.cn/?target=https://www.jianshu.com/p/b87fee2f7a23)
> [Android四大组件：Service服务史上最全面解析](https://link.juejin.cn/?target=https://www.jianshu.com/p/d963c55c3ab9)
> [Android四大组件：BroadcastReceiver史上最全面解析](https://link.juejin.cn/?target=https://www.jianshu.com/p/ca3d87a4cdf3)
