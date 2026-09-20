---
title: 获取android apk信息
top_img: https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409161654123-1599913937.png
cover: https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409161654123-1599913937.png
date: 2024-02-15 09:55:37
tags: android
categories: android
permalink: 2024/02/15/获取android apk信息(包名、启动Activity)/
---
---

## 方法一：（需要app的安装包，即XX.apk文件）

使用aapt获取android apk信息(包名、启动Activity、权限)

1、 配置android sdk中appt的路径至环境变量，一般在androidsdk的build-tools文件夹内

2、 打开cmd窗口，输入aapt，有对应信息输出则表示配置成功

3、 aapt 查看apk信息(需要apk文件)

cmd控制台输入 aapt dump badging XXX.apk，则可查看包名、启动的activity信息、权限等

获取apk包名：

![img](https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409161654123-1599913937.png)

cmd命令窗口往下拉，直到看到launchable-activity，后面就是启动的Activity

获取启动Activity：

![img](https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409162027128-574009015.png)

备注：因为数据比较多，所以建议把获取的文件输入到一个txt文件里

实际使用命令就是：aapt dump badging APK文件 > d:/apk.txt 你就可以去D盘下的apl.txt里去找相关信息了

实例：aapt dump badging D:\ceshi.apk > d:/apk.txt

​      在apk.txt文件中搜package，后面的name就是包名了，搜activity，可以获取到appActivity，其他的信息一样

![img](https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409164932294-20154185.png)

## 方法二：（不需要app的安装包）

使用 adb shell dumpsys activity | find “mFocusedActivity” 命令

获取真机或模拟器前台正在运行的apk包名、启动Activity

1.进行cmd命令窗口，执行adb devices 确认真机、模拟器与电脑连接上，如下图表示已连接

![img](https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409170456669-420123081.png)

2.cmd命令窗口，执行 adb shell dumpsys activity | find “mFocusedActivity”

![img](https://img2020.cnblogs.com/blog/1380949/202004/1380949-20200409173147411-200946431.png)
