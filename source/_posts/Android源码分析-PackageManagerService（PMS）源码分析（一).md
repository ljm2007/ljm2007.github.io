---
title: PackageManagerService
date: 2023-08-30 15:50:43
tags: android
categories: android
top_img: https://s2.loli.net/2023/08/29/LQAcvPmrTH1NwW6.jpg
cover: https://s2.loli.net/2023/08/29/LQAcvPmrTH1NwW6.jpg
---



## 一，PackageManagerService（PMS）服务简介：

PackageManagerService（简称PMS），顾名思义，用于进行Android包的管理。利用PackageManagerService服务，可以查询应用程序等信息，以及安装包（package）信息，以及该应用activity，service，权限等组件的信息。PackageManagerService是Android系统中的一个系统服务。

**Android系统中，提供了PackageManager类来对外（应用程序）提供API接口，对内（Android系统）与PMS进行IPC通信，最终通过PMS来实现真正的功能。**
![img](https://s2.loli.net/2023/08/29/IThmqFkZLDuKHls.png)

Android系统对PackageManager定义成了抽象类，而其真正的具体功能是在ApplicationPackageManager类中（ApplicationPackageManager是PackageManager的派生类）实现的。

## 二，PackageManagerService架构图：

PackageManagerService架构图（相关类图）：

![img](https://s2.loli.net/2023/08/29/LQAcvPmrTH1NwW6.jpg)

## 三，PackageManager的常用数据成员和常用方法：

列出了3个常用的方法：

```java
public abstract PackageInfo getPackageInfo(String packageName, int flags) throws NameNotFoundException; //获取包的信息

public abstract PermissionInfo getPermissionInfo(String name, int flags)
        throws NameNotFoundException; //获取权限信息
public abstract ActivityInfo getActivityInfo(ComponentName component,
        int flags) throws NameNotFoundException; //获取activity信息
```

更多的方法，如下图：

![img](https://s2.loli.net/2023/08/29/Zx2oQ9czPageCyK.jpg)

## 四，ApplicationPackageManager的常用数据成员和常用方法：

ApplicationPackageManager继承自PackageManager，PackageManager的很多abstract方法都是ApplicationPackageManager来实现的。

常用方法，如下图：

![img](https://s2.loli.net/2023/08/29/hlnrZLmj3oQ9BGt.jpg)

## 五，PackageManagerService的初始化代码分析：

### 5.1 PMS的初始化：

和其它Framework组件一样，PMS是在SystemServer中进行的初始化，代码如下：

```java
SystemServer.java中，PackageManagerService的初始化:

pm = PackageManagerService.main(context,
                    factoryTest != SystemServer.FACTORY_TEST_OFF,
                    onlyCore);
            boolean firstBoot = false;
            try {
                firstBoot = pm.isFirstBoot();
            } catch (RemoteException e) {
            }

。。。。。。


try {
            pm.performBootDexOpt();
        } catch (Throwable e) {
            reportWtf("performing boot dexopt", e);
        }

。。。。。。

try {
            pm.systemReady();
        } catch (Throwable e) {
            reportWtf("making Package Manager Service ready", e);
        }
```

在PackageManagerService.java中, 有：

```java
public static final IPackageManager main(Context context, boolean factoryTest,
            boolean onlyCore) {
        PackageManagerService m = new PackageManagerService(context, factoryTest, onlyCore);
        ServiceManager.addService("package", m);
        return m;
    }
```

说明：

返回的是IPackageManager类型，其真正的实例其实是：

PackageManagerService
将其加入ServerManager中，以便进行管理。

### 5.2 PackageManagerService的构造函数分析：

#### 5.2.1 创建Installer实例, 获取pi屏幕大小:

```java
    //创建Installer实例
    mInstaller = new Installer();
 
    //屏幕大小相关
    WindowManager wm = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
    Display d = wm.getDefaultDisplay();
    d.getMetrics(mMetrics);
```
#### 5.2.2  获取app安装相关目录：

```javascript
        //获取app安装相关目录
        File dataDir = Environment.getDataDirectory();
        mAppDataDir = new File(dataDir, "data");
        mAsecInternalPath = new File(dataDir, "app-asec").getPath();
        mUserAppDataDir = new File(dataDir, "user");
 
        //2. app安装目录： “/data/data/app-private”目录
        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
```
####  5.2.3  读取权限：

```java
        readPermissions();
```
####   5.2.4  获取framework目录和虚拟机缓存目录：

```java
        //framework目录
        mFrameworkDir = new File(Environment.getRootDirectory(), "framework");
        //虚拟机缓存目录
        mDalvikCacheDir = new File(dataDir, "dalvik-cache");
```
####   5.2.5  加载框架层资源：framework-res和核心包：

```java
        libFiles.add(mFrameworkDir.getPath() + "/framework-res.apk");
        
        String[] frameworkFiles = mFrameworkDir.list();
        if (frameworkFiles != null) {
            for (int i=0; i<frameworkFiles.length; i++) {
                File libPath = new File(mFrameworkDir, frameworkFiles[i]);
                String path = libPath.getPath();
                // Skip the file if we alrady did it.
                if (libFiles.contains(path)) {
                    continue;
                }
                // Skip the file if it is not a type we want to dexopt.
                if (!path.endsWith(".apk") && !path.endsWith(".jar")) {
                    continue;
                }
                try {
                    if (dalvik.system.DexFile.isDexOptNeeded(path)) {
                        mInstaller.dexopt(path, Process.SYSTEM_UID, true);
                        didDexOpt = true;
                    }
                } catch (FileNotFoundException e) {
                    Slog.w(TAG, "Jar not found: " + path);
                } catch (IOException e) {
                    Slog.w(TAG, "Exception reading jar: " + path, e);
                }
            }
        }
```

####   5.2.6  获取系统app，wendor app的包信息： 

```java
        // Collect all system packages.
        mSystemAppDir = new File(Environment.getRootDirectory(), "app");
        mSystemInstallObserver = new AppDirObserver(
            mSystemAppDir.getPath(), OBSERVER_EVENTS, true);
        mSystemInstallObserver.startWatching();
        scanDirLI(mSystemAppDir, PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);
 
        // Collect all vendor packages.
        mVendorAppDir = new File("/vendor/app");
        mVendorInstallObserver = new AppDirObserver(
            mVendorAppDir.getPath(), OBSERVER_EVENTS, true);
        mVendorInstallObserver.startWatching();
        scanDirLI(mVendorAppDir, PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);
```

#### 5.2.7 非核心app的扫描：

```java
        if (!mOnlyCore) {
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                    SystemClock.uptimeMillis());
            mAppInstallObserver = new AppDirObserver(
                mAppInstallDir.getPath(), OBSERVER_EVENTS, false);
            mAppInstallObserver.startWatching();
            //13.扫描安装目录，最重要的步骤
            scanDirLI(mAppInstallDir, 0, scanMode, 0);

            mDrmAppInstallObserver = new AppDirObserver(
                mDrmAppPrivateInstallDir.getPath(), OBSERVER_EVENTS, false);
            mDrmAppInstallObserver.startWatching();
 
            //13.扫描安装目录，最重要的步骤
            scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
                    scanMode, 0);
 
            ......
            }j
        } 
```

#### 5.2.8 更新权限： 

```java
        updatePermissionsLPw(null, null, UPDATE_PERMISSIONS_ALL
                | (regrantPermissions
                        ? (UPDATE_PERMISSIONS_REPLACE_PKG|UPDATE_PERMISSIONS_REPLACE_ALL)
                        : 0));
```
 其中，最主要是scanDirLI方法，即扫描目录下的apk文件。下面主要分析这个方法（5.3）。

### 5.3 scanDirLI方法代码分析：

（1）scanDirLI的定义：

```java
private void scanDirLI(File dir, int flags, int scanMode, long currentTime) {
...
//扫描dir目录下的文件，如果是apk文件，就继续执行“scanPackageLI”方法。
PackageParser.Package pkg = scanPackageLI(file,
        flags|PackageParser.PARSE_MUST_BE_APK, scanMode, currentTime);

}
```

（2）scanPackageLI（重点）：

```java
private PackageParser.Package scanPackageLI(File scanFile,
            int parseFlags, int scanMode, long currentTime) {
        mLastScanError = PackageManager.INSTALL_SUCCEEDED;
        String scanPath = scanFile.getPath();
        parseFlags |= mDefParseFlags;
      PackageParser pp = new PackageParser(scanPath);
    ......
//
//1.创建参数
createInstallArgs


//2.身份认证
collectCertificatesLI

//3.调用另外一个重载函数
scanPackageLI
```

（3）主要分析scanPackageLI这个重载函数：

```java
//初始化source和资源文件目录
File destCodeFile = new File(pkg.applicationInfo.sourceDir);
File destResourceFile = new File(pkg.applicationInfo.publicSourceDir);

......

//签名校验
verifySignaturesLP(pkgSetting, pkg)

//安装apk
mInstaller.install(pkgName, pkg.applicationInfo.uid,
                        pkg.applicationInfo.uid);

//创建用户目录
sUserManager.installPackageForAllUsers(pkgName, pkg.applicationInfo.uid);

//native 目录的处理

//providers的处理

//servuce的处理

//receivers的处理

//permissions（权限）的处理
```

## 六，总结：

```java
PMS中将各个app进行解析，并且保存在内存中，即PMS类的数据成员，典型的有：

PackageManagerService.java的数据成员:

final File mAppDataDir; //应用数据目录
final File mFrameworkDir; //framework目录
final File mSystemAppDir; //系统目录
final File mVendorAppDir; //vendor目录
final File mAppInstallDir; //应用安装目录
final File mDalvikCacheDir;//虚拟机缓存目录

final ActivityIntentResolver mActivities =
        new ActivityIntentResolver(); //有效activity
final ActivityIntentResolver mReceivers =
        new ActivityIntentResolver(); //有效receiver
final ServiceIntentResolver mServices = new ServiceIntentResolver();//有效service

final HashMap<ComponentName, PackageParser.Provider> mProvidersByComponent =
        new HashMap<ComponentName, PackageParser.Provider>();//有效ProvidersBy
当用户想查询apk的某些信息时（例如，查询包信息），就可以通过PMS查询到这些信息了。
```