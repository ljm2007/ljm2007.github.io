---
title: Java / Kotlin 反射学习总结（Android 逆向方向）
date: 2026-10-24 13:19:32
top_img: /img/post-covers/Java-Kotlin反射学习总结.png
cover: /img/post-covers/Java-Kotlin反射学习总结.png
tags:
  - Java
  - Kotlin
  - 反射
  - Android
categories: 安全研究
---
> 目标：理解反射是什么、常用 API 怎么用，以及它与 Android ART / `ArtMethod` 的关系。

## 1. 什么是反射？

**反射（Reflection）** 是程序在运行时检查类的信息，并动态访问字段、调用方法或创建对象的能力。

普通调用：

```java
Dog dog = new Dog();
dog.bark("hello");
```

反射调用：

```kotlin
val clazz = Dog::class.java
val dog = clazz.getDeclaredConstructor().newInstance()

val method = clazz.getDeclaredMethod("bark", String::class.java)
method.isAccessible = true
method.invoke(dog, "hello")
```

两种方式都能调用方法。区别是：反射可以在运行时根据类名、方法名和参数类型查找目标，不必在编写代码时直接写死调用表达式。

## 2. 先分清几个核心概念

| 名称 | 含义 |
|---|---|
| `Class` 类 | Java 反射 API 中表示“类信息”的类型 |
| `Class` 对象 | 代表某个已加载类型的对象，例如 `Dog::class.java` |
| `ClassLoader` | 负责查找并加载类 |
| `Method` 类 | 表示方法信息的反射类型 |
| `Method` 对象 | 代表某个具体方法，例如变量 `method` |
| `Field` 类 | 表示字段信息的反射类型 |
| `Field` 对象 | 代表某个具体字段 |
| `Constructor` 类 | 表示构造方法的反射类型 |
| 实例对象 | 某个类创建出来的对象，例如 `Dog()` |
| `ArtMethod` | Android ART 运行时内部的方法结构，不是普通 Java 反射 API |

注意：`Class` 对象不是实例对象。`Dog::class.java` 代表 `Dog` 这个类型；`Dog()` 才是一个 `Dog` 实例。

## 3. 类加载与反射的关系

```text
DEX / APK 中的类定义
        |
        v
ClassLoader（类加载器）
        |
        v
Class 对象（代表已加载的类）
        |
        +---- getDeclaredMethod() ----> Method 对象
        |
        +---- getDeclaredField() -----> Field 对象
        |
        +---- getDeclaredConstructor() -> Constructor 对象
```

`ClassLoader` 负责加载类；反射 API 通过 `Class` 对象查询类的成员。通常，先得到 `Class` 对象，再使用反射查找方法或字段。

## 4. 获取 `Class` 对象的常见方式

```kotlin
// 方式一：编译时已知类
val c1 = Dog::class.java

// 方式二：已有实例对象
val dog = Dog()
val c2 = dog.javaClass

// 方式三：通过类名加载（使用当前调用环境的类加载器）
val c3 = Class.forName("com.example.Dog")

// 方式四：指定 ClassLoader
val loader = context.classLoader
val c4 = loader.loadClass("com.example.Dog")
```

Android 逆向时，如果目标类由应用的特定 `ClassLoader` 加载，使用正确的加载器很重要。同名类如果由不同的类加载器分别定义，通常属于不同的运行时类型。

## 5. 四类常用反射对象

### 5.1 `Class`：查询类

| API | 作用 |
|---|---|
| `getName()` | 获取类的全限定名 |
| `getSuperclass()` | 获取父类 |
| `getInterfaces()` | 获取实现的接口 |
| `getDeclaredMethods()` | 获取当前类声明的方法 |
| `getMethods()` | 获取可见的公共方法，包含继承的方法 |
| `getDeclaredFields()` | 获取当前类声明的字段 |
| `getFields()` | 获取公共字段，包含继承的公共字段 |
| `getDeclaredConstructors()` | 获取当前类声明的构造方法 |
| `getDeclaredClasses()` | 获取当前类声明的嵌套类 |

### 5.2 `Method`：查询或调用方法

```kotlin
val clazz = Target::class.java

// 按方法名和参数类型查找
val method = clazz.getDeclaredMethod(
    "add",
    Int::class.javaPrimitiveType!!,
    Int::class.javaPrimitiveType!!
)

// 查看信息
println(method.name)
println(method.parameterTypes.contentToString())
println(method.returnType)

// 调用实例方法：第一个参数是接收调用的对象
val obj = Target()
val result = method.invoke(obj, 10, 20)
```

如果目标方法是 `static`，调用时第一个参数通常传 `null`：

```kotlin
val result = method.invoke(null, 10, 20)
```

`getDeclaredMethod()` 只查找该类自己声明的方法（包括 `private` 方法），不会自动沿父类查找。`getMethod()` 查找公共方法，也可能找到继承来的公共方法。

### 5.3 `Field`：读取或修改字段

```kotlin
val clazz = Dog::class.java
val field = clazz.getDeclaredField("age")

field.isAccessible = true

val dog = Dog()
val age = field.getInt(dog)   // 读取 int 字段
field.setInt(dog, 5)          // 修改 int 字段
```

通用 API：

```kotlin
val value = field.get(dog)
field.set(dog, value)
```

读取字段时传入的对象必须是字段所属类的实例或兼容实例。对于 `static` 字段，可以传 `null`。

`Field` 是描述字段的反射对象；`getLong(obj)` 是读取对象中该字段的 `long` 值，并不是读取任意内存地址。

### 5.4 `Constructor`：创建对象

```kotlin
val clazz = Dog::class.java

// 无参构造方法
val constructor = clazz.getDeclaredConstructor()
constructor.isAccessible = true
val dog = constructor.newInstance()
```

带参数构造方法需要传入对应的参数类型，并在 `newInstance()` 时传入实际参数。

## 6. `vararg` 与 `*` 展开运算符

```kotlin
private fun dumpOne(name: String, vararg paramTypes: Class<*>) {
    val method = Target::class.java.getDeclaredMethod(name, *paramTypes)
}
```

- `vararg`：允许函数接收数量不固定的参数；函数内部将这些参数作为数组使用。
- `*paramTypes`：把数组展开为独立参数传给另一个函数。

例如：

```kotlin
dumpOne("test")
dumpOne("test", String::class.java)
dumpOne("test", String::class.java, Int::class.javaPrimitiveType!!)
```

## 7. Java `int` 与 `Integer` 的区别

Java 中，基本类型 `int` 与包装类型 `Integer` 是不同的类型。反射查找方法时，参数类型必须匹配。

```kotlin
// 对应 Java: void test(int value)
Int::class.javaPrimitiveType!!

// 对应 Java: void test(Integer value)
Integer::class.java
```

例如，目标签名是 `test(int)`，就不能用 `Integer::class.java` 去查找。

## 8. `isAccessible` 与访问权限

```kotlin
val field = clazz.getDeclaredField("age")
field.isAccessible = true
```

这表示请求反射忽略常规 Java 语言访问检查。它不代表任何环境下都必然成功。

Android 版本、隐藏 API 限制、运行时实现和目标字段的访问规则都可能影响结果。对系统内部字段尤其如此。

## 9. 反射与 Android ART / `ArtMethod`

这是 Android 逆向中容易混淆的部分。

```text
Class 对象
    |
    | getDeclaredMethod(...)
    v
Method 对象
    |
    | JNI: FromReflectedMethod(...)
    v
jmethodID
    |
    | 是否直接对应 ArtMethod*？
    v
取决于具体 ART 实现，不能仅凭数值判断
```

### 必须区分的概念

| 名称 | 含义 |
|---|---|
| `Method` 对象 | Java 反射层的对象，描述一个方法 |
| `jmethodID` | JNI 用来标识方法的句柄/标识 |
| `ArtMethod*` | ART 内部方法结构的指针 |
| 机器码入口地址 | 方法执行代码的入口，可能随解释执行、JIT/AOT 等状态而变化 |

JNI 标准 API `FromReflectedMethod()` 可以将反射方法转换为 `jmethodID`，但 JNI 并未承诺 `jmethodID` 在所有 Android 版本上都等于 `ArtMethod*`。

如果调试器显示 `Method` 对象有 `artMethod` 字段，也只能说明当前实现中看到了该字段。要将其当作有效的内部地址，需要核对目标 Android 分支的 ART 源码和实际运行结果。

**不要把 `ArtMethod*` 结构体地址直接当成机器码地址或函数指针调用。** 如果只是执行 Java 方法，优先使用 `Method.invoke()`；如果研究 ART 内部结构，则需要匹配系统版本、CPU 架构和源码布局。

## 10. 一个完整小例子

Java 类：

```java
package com.example;

public class Calculator {
    private int base = 10;

    private int add(int a, int b) {
        return base + a + b;
    }
}
```

Kotlin 反射：

```kotlin
val clazz = Class.forName("com.example.Calculator")

// 创建实例
val constructor = clazz.getDeclaredConstructor()
constructor.isAccessible = true
val obj = constructor.newInstance()

// 读取私有字段
val field = clazz.getDeclaredField("base")
field.isAccessible = true
println(field.getInt(obj)) // 10

// 查找并调用私有方法
val method = clazz.getDeclaredMethod(
    "add",
    Int::class.javaPrimitiveType!!,
    Int::class.javaPrimitiveType!!
)
method.isAccessible = true

val result = method.invoke(obj, 2, 3)
println(result) // 15
```

## 11. 常见错误排查

| 问题 | 常见原因 |
|---|---|
| `NoSuchMethodException` | 方法名、参数类型不匹配，或方法在父类中 |
| `NoSuchFieldException` | 字段不存在于当前类；可能在父类，或当前版本实现不同 |
| `IllegalAccessException` | 访问权限检查未通过 |
| `InvocationTargetException` | 被反射调用的方法内部抛出了异常 |
| `ClassNotFoundException` | 类名错误或使用了不合适的类加载器 |
| `ClassCastException` | 同名类由不同类加载器定义，或类型不兼容 |

## 12. 学习顺序建议

1. 理解 `Class` 对象与普通实例对象的区别。
2. 熟练使用 `getDeclaredMethod()`、`getDeclaredField()`。
3. 理解 `Method.invoke()`、`Field.get()` 和 `Field.set()`。
4. 学会用正确的 `ClassLoader` 查找 Android 应用类。
5. 最后研究 JNI 的 `jmethodID` 与 ART 内部 `ArtMethod` 的关系。

**一句话总结：反射是运行时查询和操作类成员的机制；`Method`、`Field`、`Constructor` 是描述类成员的反射对象；`ArtMethod` 则属于 Android ART 内部实现。**
