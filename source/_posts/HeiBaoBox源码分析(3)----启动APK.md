---
title: HeiBaoBox源码分析(3)----启动APK
date: 2023-09-07 10:33:23
tags: HeiBaoBox
categories: HeiBaoBox
top_img: https://s2.loli.net/2023/09/07/Q3kjRwDr8dlMZbC.webp
cover: https://s2.loli.net/2023/09/07/Q3kjRwDr8dlMZbC.webp
---

## 启动过程

AppsViewModel 这里类里面有LaunchApk方法 负责启动一个apk

```kotlin
//AppsViewModel类 lambda 语言 launchOnUI 最后一个参数是lambda 可以直接把参数写以外面 类似于c++ 函数指针
fun launchApk(packageName: String, userID: Int) {
    launchOnUI {
        repo.launchApk(packageName, userID, launchLiveData)
    }
}

//AppsRepository类
  fun launchApk(packageName: String, userId: Int, launchLiveData: MutableLiveData<Boolean>) {
        val result = BlackBoxCore.get().launchApk(packageName, userId)
        launchLiveData.postValue(result)
    }
//BlackBoxCore类
    public boolean launchApk(String packageName, int userId) {
        Intent launchIntentForPackage = getBPackageManager().getLaunchIntentForPackage(packageName, userId);
        if (launchIntentForPackage == null) {
            return false;
        }
        startActivity(launchIntentForPackage, userId);
        return true;
    }
//BlackBoxCore类
    public void startActivity(Intent intent, int userId) {
        if (mClientConfiguration.isEnableLauncherActivity()) {
            LauncherActivity.launch(intent, userId);
        } else {
            getBActivityManager().startActivity(intent, userId);
        }
    }
//LauncherActivity类
   public static void launch(Intent intent, int userId) {
        Intent splash = new Intent();
        splash.setClass(BlackBoxCore.getContext(), LauncherActivity.class);
        splash.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

        splash.putExtra(LauncherActivity.KEY_INTENT, intent);
        splash.putExtra(LauncherActivity.KEY_PKG, intent.getPackage());
        splash.putExtra(LauncherActivity.KEY_USER_ID, userId);
        BlackBoxCore.getContext().startActivity(splash);
    }
```

通过这几个方法 就可以启动一个APK 

## 那么如何运行原理是什么?

### 1.Android 使用Java的反射机

 **Java的反射机制是在运行的状态中，对于任意一个类，都能够这个类的所有属性和方法；对于任意一个对象，都能够调用任何方法和属性；像这种动态获取信息以及动态调用对象方法的功能成为Java的反射机制。**

### 2.组成使用

关系组成图：

<img src="https://s2.loli.net/2023/09/12/ciFzYsbNyB9mwju.png" alt="img" style="zoom:50%;" />

1. **java.lang.Class.java：   类对象；**
2. **java.lang.reflect.Constructor.java： 类构造器；（注意：反射的类必须有构造函数！！！）**
3. **java.lang.reflect.Method.java：   类方法；**
4. **java.lang.reflect.Field.java：   类属性；**

#### 1、反射获取类的实例化对象：（三种方法）

```java
Class<?> cls=Class.forName("com.fanshe.Person"); //forName(包名.类名)
Person p=(Person)cls.newInstance();
或
Person p = new Person();
Class<?> cls=p.getClass();
Person p2=(Person)cls.newInstance();
或
Class<?> cls=Person.Class();
Person p=(Person)cls.newInstance();
```

#### 2、反射获取类的构造器：

```java
Constructor constructors = cls.getDeclaredConstructor(String.class);//参数根据类的构造，这里假设包含String类型
Object newclass = constructors.newInstance("构造实例并赋值");
```

#### 3、反射获取类的方法和返回值：

```java
Method getMethod = cls.getDeclaredMethod(methodName);//传入方法名
Object invokeMethod =getMethod.invoke(newclass);//传入实例值
静态方法不用借助实例 
Method getMethod = cls.getDeclaredMethod(methodName);
Object invokeMethod =getMethod.invoke(null);
```

#### 4、反射获取类的属性的值：

```java
Field getField = cls.getDeclaredField(fieldName);//传入属性名
Object getValue =getField.get(newclass);//传入实例值
       静态属性不用借助实例 
Field getField = cls.getDeclaredField(fieldName);
Object getValue =getField.get(null);
```

#### 5.工作原理

我们知道，Java的每个.class文件里承载了类的所有信息内容，包含基佬父类、接口、属性、构造、方法等；这些class文件会在程序运行状态下通过ClassLoad类机制加载到虚拟机内存中并产生一个对象实例，然后这个对象就可以对这块内存地址进行引用；一般我们通过New一个对象创建就行，而利用反射我们通过JVM查找并指定加载的类名，就可以创建对象实例，而在java中，类的对象被实例化可以在内存中被引用时，就可以获取该类的所有信息，就也就是不成文的规矩吧。

看完了前面的讲解，我们再来看一个小案例来加深认识，一起实践一下。

 

```java
public class Student {
    private int age;
    private String name;
    private static String mStatic;//静态属性
    /**
     * 无参构造
     */
    public Student() {
        throw new IllegalAccessError("使用默认构造函数会报错");
    }
    /**
     * 有参构造
     * @param age 年龄
     * @param name 姓名
     */
    private Student(int age, String name) {
        this.age = age;
        this.name = name;
        mStatic = "静态反射";
    }
    private String getName() {
        return name;
    }
    /**
     * 静态带返回值方法
     * @return String
     */
    private static String getTest() {
        return mStatic;
    }
}
```


这个类用到了Private私有修饰，包含带参构造函数、普通属性和静态属性，以及静态方法。

 

```java
public void Student() throws Exception{
    Class<?> clazz = Class.forName("com.example.javafanshe.Student");
    Constructor constructors = clazz.getDeclaredConstructor(int.class, String.class);
    constructors.setAccessible(true);//可访问私有为true
    //利用构造器生成对象
    Object mStudent = constructors.newInstance(22, "大头");
    Log.d("log","构造器是否报错："+mStudent.toString());
    //获取隐藏的int属性
    Field mField = clazz.getDeclaredField("age");
    mField.setAccessible(true);
    int age = (int) mField.get(mStudent);
    Log.d("log","年龄为:"+age);
    //调用隐藏的方法
    Method getMethod = clazz.getDeclaredMethod("getName");
    getMethod.setAccessible(true);
    String newname = (String) getMethod.invoke(mStudent);
    Log.d("log","名字为:"+newname);
    //调用静态方法
    Method getStaticMethod = clazz.getDeclaredMethod("mStatic");
    getStaticMethod.setAccessible(true);
    String result = (String) getStaticMethod.invoke(null);
    Log.d("log","调用静态方法:"+result);
}
```

下面这个我是用Kotlin 做的反射

```kotlin
package com.example.testreflect

import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import java.lang.reflect.Constructor
import java.lang.reflect.Field
import java.lang.reflect.Method

class Student {
    private var age: Int = 0

    lateinit var name: String;

    /**
     * 无参构造
     */
     constructor() {
        throw IllegalAccessError("使用默认构造函数会报错");
    }

    /**
     * 有参构造
     * @param age 年龄
     * @param name 姓名
     */
     constructor(age: Int, name: String) {
        this.age = age;
        this.name = name;
        mStatic = "静态反射";
    }

    fun get_Name(): String {
        return name;
    }

    /**
     * 静态带返回值方法
     * @return String
     */
    companion object {
        lateinit var mStatic: String;//静态属性
        @JvmStatic //注意，在 Kotlin 中使用反射调用静态方法时需要添加 @JvmStatic 注解，否则会抛出异常。
        fun getTest(): String {
            return mStatic;
        }
    }

}

@Throws(Exception::class)
fun Student1() {
    val clazz = Class.forName("com.example.testreflect.Student")
    val constructors: Constructor<*> = clazz.getDeclaredConstructor(
        Int::class.javaPrimitiveType,
        String::class.java
    )
    constructors.setAccessible(true) //可访问私有为true
    //利用构造器生成对象
    val mStudent: Any = constructors.newInstance(22, "大头")
    Log.d("log", "构造器是否报错：$mStudent")
    //获取隐藏的int属性
    val mField: Field = clazz.getDeclaredField("age")
    mField.isAccessible=true
    val age = mField.get(mStudent) as Int
    Log.d("log", "年龄为:$age")
    //调用隐藏的方法
    val getMethod: Method = clazz.getDeclaredMethod("getName")
    getMethod.isAccessible=true
    val newname = getMethod.invoke(mStudent) as String
    Log.d("log", "名字为:$newname")
    //调用静态方法
    val getStaticMethod: Method = clazz.getMethod("getTest")

    getStaticMethod.isAccessible = true
    val result = getStaticMethod.invoke(null) as String
    Log.d("log", "调用静态方法:$result")



}

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Student1()
    }
}
```

那么如何在Kotlin 中使用反射调用静态方法时不需要添加 @JvmStatic 注解 方法如下

在build.gradle 添加

```java
 implementation "org.jetbrains.kotlin:kotlin-reflect:1.8.0"
```

```kotlin

@Throws(Exception::class)
fun Student1() {
    val clazz = Class.forName("com.example.testreflect.Student")
    val constructors: Constructor<*> = clazz.getDeclaredConstructor(
        Int::class.javaPrimitiveType,
        String::class.java
    )
    constructors.setAccessible(true) //可访问私有为true
    //利用构造器生成对象
    val mStudent: Any = constructors.newInstance(22, "大头")
    Log.d("log", "构造器是否报错：$mStudent")
    //获取隐藏的int属性
    val mField: Field = clazz.getDeclaredField("age")
    mField.isAccessible=true
    val age = mField.get(mStudent) as Int
    Log.d("log", "年龄为:$age")
    //调用隐藏的方法
    val getMethod: Method = clazz.getDeclaredMethod("getName")
    getMethod.isAccessible=true
    val newname = getMethod.invoke(mStudent) as String
    Log.d("log", "名字为:$newname")
    //调用静态方法 没有添加@JvmStatic
   val  result =  Student.Companion::getTest.call()
    Log.d("log", "调用静态方法:$result")



}
```

结果日志Log如下。

```java
01-12 19:17:00.282 31564-31564/? D/log: 构造器是否报错：com.example.javafanshe.Student@42055ca0
01-12 19:17:00.282 31564-31564/? D/log: 年龄为:22
01-12 19:17:00.282 31564-31564/? D/log: 名字为:大头
01-12 19:17:00.282 31564-31564/? D/log: 调用静态方法:静态反射
```

#### 6.总结


     写到这里，关于反射的介绍和使用都讲的差不多了，算是适合初学者比较轻松的认识反射吧，当然反射的使用远没有上面案例那么简单，实际使用还得需要多规范的组织一下，不能只为了实现功能，不顾代码的可读性和可维护性。另外，最好多用public来修饰，以防兼容版本风险，还有第三方开源的项目也要注意，不知道哪里突然维护修理了源码，也会出现兼容问题。

https://www.cnblogs.com/KillBugMe/p/13607470.html

### **Java 的动态代理**

![img](https://s2.loli.net/2023/09/13/Xb1TVLOhcMv56Wt.jpg)

#### InvocationHandler 接口和 Proxy 类

　　我们来分析一下动态代理模式中 ProxySubject 的生成步骤：

1. 获取 RealSubject 上的所有接口列表；
2. 确定要生成的代理类的类名，系统默认生成的名字为：com.sun.proxy.$ProxyXXXX ；
3. 根据需要实现的接口信息，在代码中动态创建该 ProxySubject 类的字节码；
4. 将对应的字节码转换为对应的 Class 对象；
5. 创建 InvocationHandler 的实例对象 h，用来处理 Proxy 角色的所有方法调用；
6. 以创建的 h 对象为参数，实例化一个 Proxy 角色对象。

具体的代码为：

**Subject.java**

```java
public interface Subject {
    String operation();
}
```

**RealSubject.java**

```java
public class RealSubject implements Subject{
    @Override
    public String operation() {
        return "operation by subject";
    }
}
```

**ProxySubject.java**

```java
public class ProxySubject implements InvocationHandler{
     protected Subject subject;
    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //do something before
        return method.invoke(subject, args);
    }

}
```

**测试代码**

```java
Subject subject = new RealSubject();
ProxySubject proxy = new ProxySubject(subject);
Subject sub = (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(),
        subject.getClass().getInterfaces(), proxy);
sub.operation();
```

以上就是动态代理模式的最简单实现代码，JDK 通过使用 java.lang.reflect.Proxy 包来支持动态代理，我们来看看这个类的表述：

```java
Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the 
superclass of all dynamic proxy classes created by those methods.
```

一般情况下，我们使用下面的

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[]interfaces,InvocationHandler h) throws IllegalArgumentException { 
    // 检查 h 不为空，否则抛异常
    if (h == null) { 
        throw new NullPointerException(); 
    } 

    // 获得与指定类装载器和一组接口相关的代理类类型对象
    Class cl = getProxyClass(loader, interfaces); 

    // 通过反射获取构造函数对象并生成代理类实例
    try { 
        Constructor cons = cl.getConstructor(constructorParams); 
        return (Object) cons.newInstance(new Object[] { h }); 
    } catch (NoSuchMethodException e) { throw new InternalError(e.toString()); 
    } catch (IllegalAccessException e) { throw new InternalError(e.toString()); 
    } catch (InstantiationException e) { throw new InternalError(e.toString()); 
    } catch (InvocationTargetException e) { throw new InternalError(e.toString()); 
    } 
}
```

### **Android 中利用动态代理实现 ServiceHook**

通过上面对 InvocationHandler 的介绍，我们对这个接口应该有了大体的了解，但是在运行时动态生成的代理类有什么作用呢，其实它的作用就是在调用真正业务之前或者之后插入一些额外的操作：

![这里写图片描述](https://s2.loli.net/2023/09/13/a2WjsxQAZYvkSgM.png)

所以简而言之，代理类的处理逻辑很简单，就是在调用某个方法前及方法后插入一些额外的业务。而我们在 Android 中的实践例子就是在真正调用系统的某个 Service 之前和之后选择性的做一些自己特殊的处理，这种思想在插件化框架上也是很重要的。那么我们具体怎么去实现 hook 系统的 Service ，在真正调用系统 Service 的时候附加上我们需要的业务呢，这就需要介绍 ServiceManager 这个类了。

#### **ServiceManager 介绍以及 hook 的步骤**

##### 第一步

　　关于 ServiceManager 的详细介绍在我的博客：android IPC通信（下）－AIDL 中已经介绍过了，这里就不赘述了，强烈建议大家去看一下那篇博客，我们这里就着重看一下 ServiceManager 的 getService(String name) 方法：



```java
public final class ServiceManager {
   private static final String TAG = "ServiceManager";
private static IServiceManager sServiceManager;
private static HashMap<String, IBinder> sCache = new HashMap<String, IBinder>();

private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}

/**
 * Returns a reference to a service with the given name.
 * 
 * @param name the name of the service to get
 * @return a reference to the service, or <code>null</code> if the service doesn't exist
 */
public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name);
        if (service != null) {
            return service;
        } else {
            return getIServiceManager().getService(name);
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}

public static void addService(String name, IBinder service) {
...
}
....
}
```
我们可以看到，getService 方法第一步会去 sCache 这个 map 中根据 Service 的名字获取这个 Service 的 IBinder 对象，如果获取到为空，则会通过 ServiceManagerNative 通过跨进程通信获取这个 Service 的 IBinder 对象，所以我们就以 sCache 这个 map 为切入点，反射该对象，然后修改该对象，由于系统的 android.os.ServiceManager 类是 @hide 的，所以只能使用反射，根据这个初步思路，写下第一步的代码：

```java
Class c_ServiceManager = Class.forName("android.os.ServiceManager");
if (c_ServiceManager == null) {
    return;
}

if (sCacheService == null) {
    try {
        Field sCache = c_ServiceManager.getDeclaredField("sCache");
        sCache.setAccessible(true);
        sCacheService = (Map<String, IBinder>) sCache.get(null);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
sCacheService.remove(serviceName);
sCacheService.put(serviceName, service);
```


源码下载地址：https://github.com/zhaozepeng/ServiceHook
