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


结果日志Log如下。

```java
01-12 19:17:00.282 31564-31564/? D/log: 构造器是否报错：com.example.javafanshe.Student@42055ca0
01-12 19:17:00.282 31564-31564/? D/log: 年龄为:22
01-12 19:17:00.282 31564-31564/? D/log: 名字为:大头
01-12 19:17:00.282 31564-31564/? D/log: 调用静态方法:静态反射
```

#### 6.总结


     写到这里，关于反射的介绍和使用都讲的差不多了，算是适合初学者比较轻松的认识反射吧，当然反射的使用远没有上面案例那么简单，实际使用还得需要多规范的组织一下，不能只为了实现功能，不顾代码的可读性和可维护性。另外，最好多用public来修饰，以防兼容版本风险，还有第三方开源的项目也要注意，不知道哪里突然维护修理了源码，也会出现兼容问题。

