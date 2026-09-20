---
title: kotlin 的标准函数
top_img: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
cover: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
date: 2024-02-15 10:18:40
tags: kotlin
categories: kotlin
permalink: 2024/02/15/Kotlin的几个常用标准函数/
---
# kotlin 的标准函数指的是 Standard.kt 文件中定义的函数，任何 Kotlin 代码都可以自由的调用标准函数

## let

**let 配合 ?. 来使用进行判空辅助，表示如果目标变量不为空，则执行括号中的内容**

```kotlin
val str: String? = "abc"

str?.let{ -> it
	it.toString()
}
```

## with

**with 同时接收两个参数，第一个参数可以是任意类型的对象，第二个参数是一个 Lambda 表达式**

**with 函数会中 Lambda 表达式中提供第一个参数对象的上下文，并使用 Lambda 表达式中最后一行代码作为返回值**

```kotlin
val result = with(obj) {
	// obj 上下文
	"value" // 返回值
}
```

它可以在连续调用同一个对象的多个方法时让代码变得更加精简

```kotlin
// 遍历所有水果并打印
val list = listOf("apple", "banana", "orange", "grape")
val builder = StringBuilder()
builder.append("start.\n")
for (fruit in list) {
    builder.append(fruit).append("\n")
}
builder.append("end")
val result = builder.toString()
print(result)
```

上述代码我们用了很多次的 builder 对象的方法。这种时候就可以考虑使用 with 函数来让代码变得更简洁

```kotlin
val list = listOf("apple", "banana", "orange", "grape")
val result = with(StringBuilder()) {
    append("start.\n")
    for (fruit in list) {
        append(fruit).append("\n")
    }
    append("end")
    toString()
}
print(result)
给 with 参数的第一个参数传人了 StringBuilder() 对象，那么接下来的整个 Lambda 表达式的上下文就会是这个 StringBuilder() ，所以就不用频繁调用 builder. 而是可以直接使用 StringBuilder() 对象的方法
```

点评：有点鸡肋且有点违反 调用方法的习惯，注定不会很流行，但是前人这样写我们要能看得懂

## run

**run 函数的用法和使用场景和 with 十分相似，只是有稍微的语法改动。首先 run 不能直接调用，必须调用某个对象的 run 函数才行。其次 run 只接收一个 Lambda 参数，其他和 with 一样**

```kotlin
val relust = obj.run {
	// 上下文
	"value" // 返回值
}
```

打印出所有的水果（原始代码在 with 函数中）
我们使用 run 来修改这段代码

```kotlin
val list = listOf("apple", "banana", "orange", "grape")
val result = StringBuilder().run {
    append("start.\n")
    for (fruit in list) {
        append(fruit).append("\n")
    }
    append("end")
    toString()
}
print(result)
相比较 with 来说，变化很小
```

## apply

**apply 和 run 函数及其类似，都要在某个对象上调用，并且只接收一个 Lambda 参数**

**但是 apply 无法指定返回值，而会自动返回调用对象本身**

```kotlin
val result = obj.apply {
	// obj 上下文
}
// result = obj
```

打印出所有的水果（原始代码在 with 函数中）
使用 apply 改造

```kotlin
val list = listOf("apple", "banana", "orange", "grape")
val result = StringBuilder().apply {
    append("start.\n")
    for (fruit in list) {
        append(fruit).append("\n")
    }
    append("end")
}
print(result.toString())
```

持续完善中。。。
