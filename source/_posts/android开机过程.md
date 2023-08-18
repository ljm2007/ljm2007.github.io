---
title: android开机过程
date: 2022-11-02 10:28:43
tags: android
categories: android
top_img: https://s2.loli.net/2022/11/02/sa9YJcMAbWQ2Umz.png
cover: https://s2.loli.net/2022/11/02/sa9YJcMAbWQ2Umz.png
---

![](https://s2.loli.net/2022/11/02/Fc1EHnIVAWMTsbo.png)

# 一.开机过程

1. ### 手机长按开机键,引导芯片从固定的Rom读取预设代码到Ram中 启动Android系统引导程序 检测硬件参数等
2. ### 到达内核层的流程后，这里初始化一些**进程管理、内存管理、加载各种Driver**等相关操作，如**Camera Driver、Binder Driver** 等。下一步就是内核线程，如软中断线程、内核守护线程。下面一层就是Native层，这里额外提一点知识，层于层之间是不可以直接通信的，所以需要一种中间状态来通信。Native层和Kernel层之间通信用的是syscall，Native层和Java层之间的通信是JNI。
3. ### 在Native层会初始化init进程，也就是用户组进程的祖先进程。init中加载配置文件init.rc，init.rc中孵化出ueventd、logd、healthd、installd、lmkd等用户守护进程。开机动画启动等操作。核心的一步是孵化出Zygote进程，此进程是所有APP的父进程，这也是Xposed注入的核心，同时也是Android的**第一个Java进程（虚拟机进程）**。
4. ### 进入框架层后，加载zygote init类，注册zygote socket套接字，通过此套接字来做进程通信，并加载虚拟机、类、系统资源等。zygote第一个孵化的进程是system_server进程，负责启动和管理整个Java Framework，包含ActivityManager、PowerManager等服务。
5. ### 应用层的所有APP都是从zygote孵化而来

![](https://s2.loli.net/2022/11/02/sa9YJcMAbWQ2Umz.png)









