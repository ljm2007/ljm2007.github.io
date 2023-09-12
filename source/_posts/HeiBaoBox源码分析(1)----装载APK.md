---
title: HeiBaoBox源码分析(1)----装载APK
date: 2023-09-02 11:40:23
tags: HeiBaoBox
categories: HeiBaoBox
top_img: https://s2.loli.net/2023/09/02/JmQp1xFdUXbIZvH.jpg
cover: https://s2.loli.net/2023/09/02/JmQp1xFdUXbIZvH.jpg

---



## 一.装载APK

装载apk 并不是真的装载到androd系统里,而是装到HeiBaoBox里,所以真的系统中是找不到这个app存在的,

<img src="https://s2.loli.net/2023/09/02/JmQp1xFdUXbIZvH.jpg" alt="img" style="zoom: 25%;" />



BPackageManagerService 简写是PMS他里面有一个函数installPackageAsUserLocked是装载APK的方法,

CreateUserExecutor implements Executor 创建各种文件夹在虚拟空间里 到此装载apk 大致完成了.后期等全部分析完成在细分析其中的东西
