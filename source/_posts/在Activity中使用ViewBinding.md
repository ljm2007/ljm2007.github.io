---
title: 在Activity中使用ViewBinding
top_img: https://s2.loli.net/2023/09/02/zEhOgs8dyqo6ZjM.webp
cover: https://s2.loli.net/2023/09/02/zEhOgs8dyqo6ZjM.webp
date: 2024-02-16 14:04:43
tags: android
categories: android
permalink: 2024/02/16/在Activity中使用ViewBinding/
---
## 在Activity中使用ViewBinding

一旦启动了ViewBinding功能之后，Android Studio会自动为我们所编写的每一个布局文件都生成一个对应的Binding类。

Binding类的命名规则是将布局文件按驼峰方式重命名后，再加上Binding作为结尾。

比如说，前面我们定义了一个activity_main.xml布局，那么与它对应的Binding类就是ActivityMainBinding。

当然，如果有些布局文件你不希望为它生成对应的Binding类，可以在该布局文件的根元素位置加入如下声明：

```xml
<LinearLayout
    xmlns:tools="http://schemas.android.com/tools"
    ...
    tools:viewBindingIgnore="true">
    ...
</LinearLayout>
```

## @JvmStatic        @JvmOverloads

`@JvmStatic` 和 `@JvmOverloads` 是 Kotlin 中的注解，它们用于与 Java 互操作时的一些特定行为。下面是它们的简要说明：

- `@JvmStatic`:

在 Kotlin 中，成员函数默认是非静态的。使用 `@JvmStatic` 注解可以将一个成员函数声明为静态方法，以便在 Java 代码中可以直接通过类名调用，而不需要通过实例来调用。

- 这个注解通常用于在 Kotlin 中定义的伴生对象的成员函数，以便在 Java 代码中更方便地访问它们。

示例：

```kotlin
kotlinCopy codeclass MyClass {
    companion object {
        @JvmStatic
        fun myStaticFunction() {
            // 静态方法的实现
        }
    }
}
```

- `@JvmOverloads`:

在 Kotlin 中，如果一个函数有默认参数值，那么生成的 Java 代码会包含多个重载版本，以支持不同的参数组合。使用 `@JvmOverloads` 注解可以生成更少的重载方法，提高 Java 代码的互操作性。

- 当使用 `@JvmOverloads` 注解时，编译器会生成一个包含所有参数的版本以及每个可能的参数组合的重载版本。

示例：

```kotlin
kotlinCopy code@JvmOverloads
fun myFunction(param1: Int, param2: String = "default") {
    // 函数实现
}
```

在这个示例中，`@JvmOverloads` 允许 Java 代码调用 `myFunction` 方法时只传递一个参数，而省略默认参数的部分。

`@JvmOverloads` 是一个 Kotlin 注解，用于在与 Java 互操作时，自动生成重载的方法，以支持 Kotlin 中的默认参数。让我更详细地解释一下。

在 Kotlin 中，你可以为函数参数设置默认值，例如：

```plaintext
kotlinCopy codefun exampleFunction(param1: Int, param2: String = "default") {
    // 函数实现
}
```

在这个例子中，`param2` 参数有一个默认值，因此你可以在调用函数时只传递 `param1`，而 `param2` 将使用默认值。在 Kotlin 中，这样的函数调用是有效的：

```plaintext
kotlinCopy code
exampleFunction(42)
```

现在，考虑在 Java 代码中调用这个函数。由于 Java 不支持默认参数，如果没有 `@JvmOverloads` 注解，Java 代码将需要提供所有参数的值。而添加了 `@JvmOverloads` 注解后，Kotlin 编译器会为每个带有默认值的参数生成重载的方法，使得在 Java 中可以省略掉某些参数。

使用 `@JvmOverloads` 的情况：

```kotlin
kotlinCopy code@JvmOverloads
fun exampleFunction(param1: Int, param2: String = "default") {
    // 函数实现
}
```

生成的 Java 代码：

```kotlin
javaCopy codepublic static void exampleFunction(int param1) {
    // 生成的函数实现
}

public static void exampleFunction(int param1, String param2) {
    // 生成的函数实现
}
```

这样，Java 代码可以选择性地传递参数，而不必提供所有参数的值。这有助于提高 Kotlin 与 Java 之间的互操作性。

## apply  什么意思 没懂

`apply` 是 Kotlin 中的一个作用域函数，它的作用是在对象上执行指定的代码块，并返回这个对象。它的语法结构如下：

```kotlin
kotlinCopy code
public inline fun <T> T.apply(block: T.() -> Unit): T
```

这个函数接收一个 lambda 表达式作为参数，而这个 lambda 表达式则成为了调用者对象的一个扩展函数，可以在 lambda 表达式中通过 `this` 引用到调用者对象。这种特性使得在 lambda 表达式中可以方便地访问对象的属性和方法。

在你提供的代码中，`apply` 函数被用于对一个对象进行配置，而不需要显式地引用该对象。这样可以使代码更加简洁和易读。

举个例子，假设有一个 `Person` 类：

```kotlin
kotlinCopy codeclass Person {
    var name: String = ""
    var age: Int = 0
kotlin
```

使用 `apply` 函数可以这样进行对象的初始化和配置：

```kotlin
kotlinCopy codeval person = Person().apply {
    name = "John"
    age = 30
}
```

这等价于以下代码，但更为简洁：

```kotlin
kotlinCopy codeval person = Person()
person.name = "John"
person.age = 30
```

总的来说，`apply` 函数是一种简化代码、初始化对象和进行对象配置的便捷方式。
