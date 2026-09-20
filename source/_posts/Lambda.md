---
title: Kotlin中的lambda
top_img: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
cover: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
date: 2024-02-16 10:18:40
tags: lambda
categories: lambda
permalink: 2024/02/16/Lambda/
---
# 关于lambda回调函数还是有一些不懂

## Lambda 就是回调函数简写

## 最外层定义回调函数 最里层调用函数

```kotlin
fun LambdaFunTest(param: (x: Int, y: Int) -> Int) {
    val result = param(1, 2)
    println("Result: $result")
}

val ParamTest = { x: Int, y: Int -> x + y }

fun main() {
    LambdaFunTest(ParamTest)
}
```
