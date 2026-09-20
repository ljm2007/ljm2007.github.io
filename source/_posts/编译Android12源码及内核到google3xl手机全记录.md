layout: aosp
title: 编译Android12源码及内核到Google Pixel 3 XL手机全记录
date: 2022-10-31 11:30:56
tags: 修改源码
categories: 修改源码
top_img: https://bit.ly/3WkSvF9
cover: https://bit.ly/3WkSvF9	

------



###   **编译源码基本上都是在linux环境下编译，所以我们在window系统中使用虚拟机安装ubuntu系统来编译**

## 一.虚拟机安装Ubuntu18.04LTS

  1.下载Ubuntu系统镜像 [官方下载](https://releases.ubuntu.com/18.04.6/) 拉到下面就是

![](https://s2.loli.net/2022/10/31/wDPUjI8Noy7YcTH.png)

  2.虚拟机选择[VMware安装Ubuntu](https://blog.csdn.net/weixin_41805734/article/details/120698714)

## 二.下载AOSP源码

###  1.对Ubuntu更新操作,这样会在编译代码时 少走一些弯路

```
sudo apt-get update
```

###  2.安装所需的软件包 (Ubuntu 18.04)

​    您需要 64 位版本的 Ubuntu。

```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```

###  3.下载 repo工具

  Android源码包含数百个git库，光是下载这么多的git库就是一项繁重的任务，所以Google开发了         repo，它是用于管理Android版本库的一个工具，使用了Python对git进行了一定的封装，简化了对多 个Git版本库的管理。安装 Git，在Ubuntu 终端输入如下命令：

```csharp
sudo apt-get install git
```

设置git身份，添加自己的邮箱和姓名：

```csharp
git config --global user.email "xxxx@qq.com"
git config --global user.name "xxxx"
```

创建bin，并加入到PATH中:

```bash
mkdir ~/bin
PATH=~/bin:$PATH
```

安装curl库：

```csharp
sudo apt-get install curl
```

下载repo并设置权限：

```jsx
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo
```

安装python3，repo初始化时会用到：

```csharp
sudo apt-get install python
```

### 4.下载源码

####     4.1 创建工作目录

```
mkdir WORKING_DIRECTORY
cd WORKING_DIRECTORY
```

​          由于外网慢这里还是用清华源的镜像来下载,

```
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
open ~/.bash_profile
export REPO_URL='https://aosp.tuna.tsinghua.edu.cn/git-repo'
source ~/bash_profile
```

####   4.2.直接下载

```
repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest
## 如果提示无法连接到 gerrit.googlesource.com，可以编辑 ~/bin/repo，把 REPO_URL 一行替换成下面的：
## REPO_URL = 'https://gerrit-googlesource.proxy.ustclug.org/git-repo'
```

这里需要注意，默认的 repo 使用的地址是 REPO_URL = '[gerrit.googlesource.com/git-repo](https://link.juejin.cn/?target=https%3A%2F%2Fgerrit.googlesource.com%2Fgit-repo)' ，这里我们需要修改 REPO_URL，否则会出现无法下载的情况

1. 修改方法 1：在你的 rc 文件里面，加入一条配置即可：REPO_URL="[gerrit-googlesource.proxy.ustclug.org/git-repo](https://link.juejin.cn/?target=https%3A%2F%2Fgerrit-googlesource.proxy.ustclug.org%2Fgit-repo)"
2. 修改方法 2：直接打开 ～/bin/repo, 把 REPO_URL 一行替换成下面的： REPO_URL = '[gerrit-googlesource.proxy.ustclug.org/git-repo](https://link.juejin.cn/?target=https%3A%2F%2Fgerrit-googlesource.proxy.ustclug.org%2Fgit-repo)'

下载好 .repo 之后会有下面的信息

```
➜  Android12 repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest
Downloading Repo source from https://gerrit-googlesource.proxy.ustclug.org/git-repo

... A new version of repo (2.17) is available.
... You should upgrade soon:
    cp /home/gracker/Code/Android12/.repo/repo/repo /home/gracker/bin/repo

Downloading manifest from git://mirrors.ustc.edu.cn/aosp/platform/manifest
remote: Enumerating objects: 91965, done.
remote: Total 91965 (delta 0), reused 0 (delta 0)

Your identity is: Gracker <dreamtale.jg@gmail.com>
If you want to change this, please re-run 'repo init' with --config-name

repo has been initialized in /home/gracker/Code/Android12
```

 如果选择了直接下载，那么就不需要看 4.2 了

#### 4.3 下载特定的 Tag

这种方法指的是只下载单个 Tag 所对应的代码，这里的 Tag 可以 [查看这里 https://source.android.google.cn/setup/start/build-numbers](https://source.android.google.cn/docs/setup/about/build-numbers)，比如我的开发机是 Google Pixel 3 XL，我在 Tag 列表查看对应的机型都有哪些 TAG，目前 Android 12 只发布了两个，如下

![](https://s2.loli.net/2022/11/01/fvOreqPx3sV85oE.png)

对应的 Tag 分别是 android-12.0.0_r3 和 android-12.0.0_r1 ，所以下载的时候我可以制定对应的 TAG, 这样的好处是下载的代码比较少，下载速度会快一些；不方便的点是更新不方便，Google 会定期发邮件告诉你哪些新的 Tag 发布了，你可以根据这个来更新代码

```
repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b  android-12.0.0_r3
```

#### 4.4同步代码

上面步骤三只是下载了 .repo 文件，具体的代码还需要执行 repo sync 来进行下载。由于镜像站的限制和下载过程中可能会遇到的问题，建议大家用 -j4 来下载

```bash
repo sync -j4
```

然后就开始了漫长的下载，由于下载过程中可能会出现失败的情况，你可以搞一个 sh 脚步来循环下载，一觉醒来就下载好了

```bash
#!/bin/bash
repo sync -j4
while [ $? -ne 0 ]
do
echo "======sync failed ,re-sync again======"
sleep 3
repo sync -j4
done
复制代码
```

具体方法

```bash
touch repo.sh  # 1. 创建 repo.sh 文件
vim repo.sh # 2. 复制上面的脚本内容到 repo.sh 里面，这里你可以使用你自己喜欢的方法打开并修改文件，比如 vscode
chmod a+x repo.sh #3. 修改权限
./repo.sh # 4. 运行脚本，万事大吉
```

## 三. 驱动下载

代码下载完成之后，我们先不着急编译，如果要想在真机上跑，需要下载一些厂商闭源的驱动文件，这样后续编译的代码才可以跑到真机上，[此处对应的 官方文档 https://source.android.google.cn/setup/build/downloading#obtaining-proprietary-binaries](https://link.juejin.cn/?target=https%3A%2F%2Fsource.android.google.cn%2Fsetup%2Fbuild%2Fdownloading%23obtaining-proprietary-binaries)

**上面下载代码的时候，我们提到了两种方式，直接下载和下载特定 Tag，不同的下载方式对应的驱动也不一样**

### 3.1 下载特定 Tag 的代码所对应的驱动

如果下载的时候加了 -b ，那么就需要查看对应的 tag 所对应的驱动，[地址如下：https://developers.google.cn/android/drivers](https://developers.google.cn/android/drivers)

以我的 pixel 3 XL 为例，下载的 TAG 为 **android-12.0.0_r3** (repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-12.0.0_r3)

那么我们需要找到下面的部分，这里的 **SP1A.210812.016.A1** 跟上面 4.2 节是对应的，即 Tag **android-12.0.0_r3** 对应的 Build ID 是 **SP1A.210812.016.A1**。大家可以根据自己下载的 TAG 找到对应的 Build ID，然后根据 Build ID 寻找对应的驱动即可 [developers.google.cn/android/dri…](https://developers.google.cn/android/drivers)

![](https://s2.loli.net/2022/11/01/gGQLPBeDntHCNX4.png)

跟4.3 节下载的 Tag 是对应的：

![](https://s2.loli.net/2022/11/01/fvOreqPx3sV85oE.png)

### 2.3 驱动提取

下载的内容解压后，是两个 sh 文件，以我的 Pixel 3 XL 为例，在代码根目录执行，使用 D 来向下翻页，直到最后手动输入 I ACCEPT

```
# 解压缩 extract-google_devices-crosshatch.sh
./extract-google_devices-crosshatch.sh
```

![](https://s2.loli.net/2022/11/01/k2Mu76vtnOfJ1yF.png)



```
# 解压缩  ./extract-qcom-crosshatch.sh
 ./extract-qcom-crosshatch.sh
```

![](https://s2.loli.net/2022/11/01/SVKme2aybCJkZuB.png)

## 四.代码编译

代码和驱动都下载好之后，就可以开始代码的编译工作了，由于新版本不再支持 Mac 编译，所以建议大家还是使用 Linux 来进行编译，推荐使用 Ubuntu

![](https://s2.loli.net/2022/11/01/GUfDrhsTMi1Etwb.png)

不再支持Mac编译

### 4.1 设置代码编译环境

每次关闭 Shell 之后都需要重新执行下面这个脚本，相当于配置了一下编译环境

```bash
source build/envsetup.sh
复制代码
```

或者

```bash
. build/envsetup.sh
复制代码
```

### 4.2选择编译目标

```
lunch
复制代码
```

运行 lunch 之后，会有一堆设备出来让你选择，还是以我的 Pixel 3 XL 为例，其代号是 ，在[这里可以查看所有机型对应的代号：https://source.android.google.cn/setup/build/running#selecting-device-build](https://link.juejin.cn/?target=https%3A%2F%2Fsource.android.google.cn%2Fsetup%2Fbuild%2Frunning%23selecting-device-build) Pixel 3 XL 对应的代号是：**crosshatch**

![](https://s2.loli.net/2022/11/01/Ir7wJnShUti8Xv5.png)

所以我选择编译的是 aosp_crosshatch-userdebug ，这里可以输入编号也可以直接输入 aosp_crosshatch-userdebug

![](https://s2.loli.net/2022/11/01/BjWVrhMtwNcA7Dp.png)

lunch 选项

然后脚本会进行一系列的配置，输出下面的内容

![](https://s2.loli.net/2022/11/01/lhZT3EjaUOHCB2e.png)

### 4.3 全部编译

使用 m 构建所有内容。m 可以使用 -jN 参数处理并行任务。如果您没有提供 -j 参数，构建系统会自动选择您认为最适合您系统的并行任务计数。

```
m
复制代码
```

如上所述，您可以通过在 m 命令行中列出相应名称来构建特定模块，而不是构建完整的设备映像。此外，m 还针对各种特殊目的提供了一些伪目标。以下是一些示例：

1. droid - m droid 是正常 build。此目标在此处，因为默认目标需要名称。
2. all - m all 会构建 m droid 构建的所有内容，加上不包含 droid 标记的所有内容。构建服务器会运行此命令，以确保包含在树中且包含 Android.mk 文件的所有元素都会构建。
3. m - 从树的顶部运行构建系统。这很有用，因为您可以在子目录中运行 make。如果您设置了 TOP 环境变量，它便会使用此变量。如果您未设置此变量，它便会从当前目录中查找相应的树，以尝试找到树的顶层。您可以通过运行不包含参数的 m 来构建整个源代码树，也可以通过指定相应名称来构建特定目标。
4. mma - 构建当前目录中的所有模块及其依赖项。
5. mmma - 构建提供的目录中的所有模块及其依赖项。
6. croot - cd 到树顶部。
7. clean - m clean 会删除此配置的所有输出和中间文件。此内容与 rm -rf out/ 相同。

运行 m help 即可查看 m 提供的其他命令

输入 m 之后开始第一次全部编译，漫长的等待，编译时间取决于你的电脑配置... 主要是 cpu 和内存，建议内存 32G 走起，cpu 也别太烂

![img](https://s2.loli.net/2022/11/01/2lD1NM7VGfWFRzc.png)

编译成功之后，会有下面的输出

![](https://s2.loli.net/2022/11/01/2mOdJlyqTZ7FoGH.png)

## 五.刷入手机

自己编译的 UserDebug 固件用来 Debug 是非常方便的，不管是用来 Debug Framework 还是 App

编译好之后下面开始刷机，以我的测试机器 Pixel 3 XL 为例，依次执行下面的命令

```bash
adb reboot fastboot

# 等待手机进入 fastboot 界面之后
fastboot flashall -w

# 刷机完成之后，执行 fastboot reboot 长期系统即可
fastboot reboot
```

![](https://s2.loli.net/2022/11/01/MY39t4usCX8FnkQ.png)

之后手机会自动重启，然后进入主界面，至此，我们的代码下载 - 编译 - 刷机的这部分就结束了

## 六.下载内核源码

###         6.1查看手机内核版本

```csharp
#先连接手机
1.先刷入官方rom
2.查看内核版本
adb shell
crosshatch:/ $ cat /proc/version
Linux version 4.9.270-g862f51bac900_audio-g11cdada77cf3 (nickli@earth) (Android (7284624, based on r416183b) clang version 12.0.5 (https://android.googlesource.com/toolchain/llvm-project c935d99d7cf2016289302412d708641d52d2f7ee)) #1 repo:android-msm-crosshatch-4.9-android12 SMP PREEMPT Sun Dec
通过git commit 可以看到代码所属版本，然后下载对应版本源码
```

​           先查看手机内核版本 [官方网址](https://android.googlesource.com/kernel/msm.git/+refs)

​         ![](https://s2.loli.net/2022/11/01/qvQURWzHJInoGZs.png)

### 6.2下载内核 源码如下

```
 cd ~/work/
 mkdir android-kernel && cd android-kernel
 export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
 repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/kernel/manifest -b android-msm-crosshatch-4.9-android12
 repo sync
```

###   6.3然后用这个构建内核

```bash
build/build.sh
```

​    [构建启动映像](https://source.android.com/docs/setup/build/building-kernels#id-bootimage):

```
 cp out/android-msm-pixel-4.9/dist/Image.lz4 ~/work/aosp/device/google/crosshatch-kernel/
cp out/android-msm-pixel-4.9/dist/*.ko ~/work/aosp/device/google/crosshatch-kernel/
 cp out/android-msm-pixel-4.9/dist/*.img ~/work/aosp/device/google/crosshatch-kernel/
 cp out/android-msm-pixel-4.9/dist/*.dtb ~/work/aosp/device/google/crosshatch-kernel/
 cd ~/work/aosp/
 . build/envsetup.sh
 lunch aosp_crosshatch-userdebug
 adb reboot bootloader
 make bootimage
```

###   6.4刷入内核

```text
 adb reboot bootloader
 fastboot flash boot
 fastboot reboot
```

 注: ~/work/aosp/是AOSP所在目录.

## 七.刷机过程中错误修复

 遇到1:

```
/home/nickli/work/android-kernel/private/msm-google/scripts/extract-cert.c:21:10: fatal error: 'openssl/bio.h' file not found
#include <openssl/bio.h>
         ^~~~~~~~~~~~~~~
解决办法:
$ sudo apt install libssl-dev
```

 遇到2:

```
Downloading Repo source from https://aosp.tuna.tsinghua.edu.cn/git-repo
fatal: Cannot get https://aosp.tuna.tsinghua.edu.cn/git-repo/clone.bundle
fatal: error no host given
fatal: double check your --repo-rev setting.
fatal: cloning the git-repo repository failed, will remove '.repo/repo'
解决办法:
unset  http_proxy && unset https_proxy
```

遇到3:

```
File "/root/....../.repo/repo/main.py", line 79
    file=sys.stderr)  
SyntaxError: invalid syntax
解决办法：
由于repo 要求的python版本不匹配导致的，可以通过升级Python版本解决，如果还是出现这种异常，考虑是否是repo和Python版本不一致导致；解决过程如下
1、mkdir ~/bin
2、PATH=~/bin:$PATH
3、git clone https://gerrit-googlesource.lug.ustc.edu.cn/git-repo
4、cd git-repo/
5、cp repo ~/bin/
6、chmod a+x ~/bin/repo
7、mkdir workspace
8、cd workspace/
9、尝试重新repo init

```

遇到4: 

```
内存不够时,,如果能条件就是上32G内存,没条件如下命令:
解决办法:
1.dd if=/dev/zero of=/opt/swapfile bs=1M count=1000 (创建一个1G的文件作为交换分区使用) 

2.mkswap /opt/swapfile (格式化成swap分区) 

3.swapon /opt/swapfile (打开swap分区) 

4.vim /etc/fstab (在fstab中增加一条记录如下) /opt/swapfile swap swap defaults 0 0 

5.mount -a
```

## 八.安卓刷三方系统后WIFI标志打叉的解决办法

### 1、应用范围

适用于安卓刷LineageOS等三方系统后，预置的Captive Portal 和NTP服务器不能连接，表现出WIFI标志打叉，网络不稳定的情况。按照如下修改即可。本文以LineageOS为例，其他系统是否有内置终端需要自行判断。

### 2、已获得ROOT权限的：

打开开发者模式中本地终端的选项，在终端中获得root权限并输入以下命令：

settings put global ntp_server ntp1.aliyun.com

settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204

settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204

之后关闭WIFI再重新连接即可。

### 3、无ROOT权限的情况：

由ADB连接手机，在ADB环境下输入前述命令。

每条命令之前需加上adb shell。

如：settings put global ntp_server ntp1.aliyun.com

在ADB环境下就变成：

adb shell settings put global ntp_server ntp1.aliyun.com

adb shell settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204

adb shell settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204
