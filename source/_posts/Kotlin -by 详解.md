---
title: Kotlin 中 by 关键字
top_img: https://s2.loli.net/2023/05/20/nCWOkMglcT86x7p.png
cover: https://s2.loli.net/2023/05/20/nCWOkMglcT86x7p.png
date: 2024-02-18 23:43:06
tags: Kotlin
categories: Kotlin
permalink: 2024/02/18/Kotlin -by 详解/
---
---

Kotlin 中 by 关键字用来简化实现代理 (委托) 模式，不仅可以类代理，还可以代理类属性, 监听属性变化，下面我们来介绍by的几种主要使用场景:

- **类的代理 class**

- **属性延迟加载 lazy**

- **可观察属性 Delegates.observable ( 扩展 Delegates.vetoable )**

- **自定义监听属性变化 ReadWriteProperty**

- **属性非空强校验 Delegates.notNull()**

- **Map值 映射到类属性 map**

## 类的代理(代理/委托模式)

```kotlin
class ByTest {
// 定义一个接口,和一个方法 show()
interface Base {

    fun show()
}

// 定义类实现 Base 接口, 并实现 show 方法
open class BaseImpl : Base {

    override fun show() {

        AbLogUtil.e("BaseImpl::show()")
    }
}

// 定义代理类实现 Base 接口, 构造函数参数是一个 Base 对象
// by 后跟 Base 对象, 不需要再实现 show()
class BaseProxy(base: Base) : Base by base {

    fun showOther() {

        AbLogUtil.e("BaseImpl::showOther()")
    }

}

// main 方法
fun mainGo() {

    val base = BaseImpl()
    BaseProxy(base).show()
    BaseProxy(base).showOther()
}
    }
```

```kotlin
输出结果
 BaseImpl::show()
 BaseImpl::showOther()
```

转成 Java 代码

```kotlin
public interface Base {

   void show();
}

// BaseImpl.java

public class BaseImpl implements Base {

   public void show() {

      String var1 = "BaseImpl::show()";
      System.out.print(var1);

   }
}
```

```kotlin
// BaseProxy.java

public final class BaseProxy implements Base {

   // $FF: synthetic field
   private final Base $$delegate_0;

   public BaseProxy(@NotNull Base base) {

      Intrinsics.checkParameterIsNotNull(base, "base");
      super();
      this.$$delegate_0 = base;

   }

   public void show() {

      this.$$delegate_0.show();

   }
}
// NormalKt.java

public final class NormalKt {

   public static final
```

## **属性延迟加载 lazy：**

- `by` 也用于属性代理，其中属性的 get 和 set 操作被委托给另一个类的实现。这样可以在不改变类结构的情况下扩展属性的行为。

- 例如，使用

```plaintext
by lazy
```

 实现懒加载：

```plaintext
kotlinCopy codeval myValue: String by lazy {
    // 这部分代码只有在第一次访问 myValue 属性时才执行
    "Hello, Lazy World!"
}
```

这只是 `by` 关键字的两个主要用法之一。在其他上下文中，`by` 也可能用于不同的目的，因此具体用法可能取决于上下文和使用场景。

## **可观察属性 Delegates.observable ( 扩展 Delegates.vetoable )**

在 Kotlin 中，`Delegates.observable` 是一种用于创建可观察属性（Observable Properties）的属性代理。它允许你在属性值发生变化时得到通知。除了 `Delegates.observable`，还有一个类似的属性代理叫做 `Delegates.vetoable`，它允许你在属性值发生变化之前进行拦截。

下面是这两个属性代理的简单用法示例：

- **`Delegates.observable`：**

```kotlin
kotlinCopy codeimport kotlin.properties.Delegates

class Example {
    var propertyWithObserver: String by Delegates.observable("Initial Value") { _, oldValue, newValue ->
        println("Property changed from $oldValue to $newValue")
    }
}

fun main() {
    val example = Example()
    example.propertyWithObserver = "New Value"
}
```

在上面的例子中，当 `propertyWithObserver` 的值发生变化时，提供的 lambda 表达式将被调用，打印出属性的旧值和新值。

- **`Delegates.vetoable`：**

```kotlin
kotlinCopy codeimport kotlin.properties.Delegates

class Example {
    var propertyWithVeto: Int by Delegates.vetoable(0) { _, oldValue, newValue ->
        newValue > oldValue
    }
}

fun main() {
    val example = Example()
    println(example.propertyWithVeto) // 输出 0

    example.propertyWithVeto = 42
    println(example.propertyWithVeto) // 输出 42

    example.propertyWithVeto = 5
    println(example.propertyWithVeto) // 由于 vetoable 拦截条件不满足，输出 42
}
```

在上面的例子中，`propertyWithVeto` 的值只有在新值大于旧值时才会被接受，否则会被拦截。

这两种属性代理都提供了一种便捷的方式来处理属性值的变化，并在变化时执行自定义的操作。

## **自定义监听属性变化 ReadWriteProperty**

在 Kotlin 中，你可以通过实现 `ReadWriteProperty` 接口来创建自定义的属性代理，从而实现自定义监听属性变化的功能。`ReadWriteProperty` 接口有两个方法：`getValue` 用于获取属性值，`setValue` 用于设置属性值。你可以在 `setValue` 方法中添加属性变化的监听逻辑。

下面是一个简单的例子，演示如何使用 `ReadWriteProperty` 自定义属性代理来监听属性变化：

```kotlin
kotlinCopy codeimport kotlin.properties.ReadWriteProperty
import kotlin.reflect.KProperty

// 自定义属性代理，实现了 ReadWriteProperty 接口
class ObservableProperty<T>(initialValue: T, private val onChange: (newValue: T) -> Unit) : ReadWriteProperty<Any?, T> {

    private var value: T = initialValue

    // getValue 方法用于获取属性值
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value
    }

    // setValue 方法用于设置属性值，并在值变化时执行 onChange 回调
    override fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        if (value != newValue) {
            value = newValue
            onChange.invoke(newValue)
        }
    }
}

// 使用自定义属性代理
class Example {
    // 使用 ObservableProperty 作为属性代理，提供初始值和变化时的回调
    var observableValue: String by ObservableProperty("Initial Value") { newValue ->
        println("Property changed to: $newValue")
    }
}

fun main() {
    val example = Example()
    example.observableValue = "New Value"
}
```

在这个例子中，`ObservableProperty` 类是一个实现了 `ReadWriteProperty` 接口的自定义属性代理。它接受初始值和一个 `onChange` 回调函数，当属性值发生变化时，会调用这个回调函数。在 `Example` 类中，我们使用了 `ObservableProperty` 作为属性 `observableValue` 的代理，提供了初始值和一个打印变化的回调。

你可以根据需要扩展和修改 `ObservableProperty` 类，以满足不同的监听属性变化的场景。

## 属性非空强校验 Delegates.notNull()

在 Kotlin 中，你可以使用 `Delegates.notNull()` 来创建一个非空属性的代理，该代理会在属性被访问的时候检查其是否为 `null`，如果为 `null`，则抛出 `IllegalStateException` 异常。

下面是一个简单的示例：

```kotlin
kotlinCopy codeimport kotlin.properties.Delegates

class Example {
    // 使用 Delegates.notNull() 来创建非空属性代理
    var nonNullValue: String by Delegates.notNull()

    // 初始化非空属性
    fun initializeNonNullValue() {
        nonNullValue = "Initialized Value"
    }
}

fun main() {
    val example = Example()

    try {
        // 尝试访问未初始化的非空属性，会抛出 IllegalStateException 异常
        println(example.nonNullValue)
    } catch (e: IllegalStateException) {
        println("Exception: ${e.message}")
    }

    // 初始化非空属性
    example.initializeNonNullValue()

    // 访问已初始化的非空属性，不会抛出异常
    println(example.nonNullValue)
}
```

在这个例子中，`nonNullValue` 是一个使用 `Delegates.notNull()` 创建的非空属性代理。在尝试访问未初始化的属性时，会抛出 `IllegalStateException` 异常。为了避免异常，我们通过调用 `initializeNonNullValue` 方法来初始化属性，然后再访问它。

请注意，使用 `Delegates.notNull()` 时要确保在访问属性之前进行初始化，否则会抛出异常。

## Map值 映射到类属性 map

## 把属性储存在映射中

一个常见的用例是在一个映射（map）里存储属性的值。 这经常出现在像解析 JSON 或者做其他”动态”事情的应用中。 在这种情况下，你可以使用映射实例自身作为委托来实现委托属性。

```kotlin
class Site(val map: Map<String, Any?>) {
    val name: String by map
    val url: String  by map
}

fun main(args: Array<String>) {
    // 构造函数接受一个映射参数
    val site = Site(mapOf(
        "name" to "菜鸟教程",
        "url"  to "www.runoob.com"
    ))

    // 读取映射值
    println(site.name)
    println(site.url)
}
```

执行输出结果：

```kotlin
菜鸟教程
www.runoob.com
```

如果使用 var 属性，需要把 Map 换成 MutableMap：

```kotlin
class Site(val map: MutableMap<String, Any?>) {
    val name: String by map
    val url: String by map
}

fun main(args: Array<String>) {

    var map:MutableMap<String, Any?> = mutableMapOf(
            "name" to "菜鸟教程",
            "url" to "www.runoob.com"
    )

    val site = Site(map)

    println(site.name)
    println(site.url)

    println("--------------")
    map.put("name", "Google")
    map.put("url", "www.google.com")

    println(site.name)
    println(site.url)

}
```

执行输出结果：

```kotlin
菜鸟教程
www.runoob.com
--------------
Google
www.google.com
```
