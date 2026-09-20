---
title: 又玩一遍Apply
top_img: https://s2.loli.net/2024/03/20/YqThUrp4GBbARHl.png
cover: https://s2.loli.net/2024/03/20/YqThUrp4GBbARHl.png
date: 2024-03-20 14:04:43
tags: Kotlin
categories: Kotlin
permalink: 2024/03/20/又玩了一遍apply/
---
```kotlin
state = state.copy(
     radioBtnselected = state.radioBtnselected.apply {
         state.radioBtnselected.indices.forEach {
             this[it] = it == index
         }
     }
 )
```

![Kotlin development](https://s2.loli.net/2024/03/20/YqThUrp4GBbARHl.png)

在这段代码中，您使用了 `apply` 函数来修改 `radioBtnselected` 列表的元素，并将修改后的状态赋值给 `state` 变量。

让我解释一下这段代码的作用：

- `state.copy()` 会创建一个新的 `ControlStatus` 对象，其属性与原始 `state` 对象相同。

- 在 `apply` 函数中，我们可以访问 `state.radioBtnselected` 列表，并对其进行修改。`this` 关键字在这里引用的就是 `state.radioBtnselected` 列表对象。

- 在 `apply` 的 lambda 表达式中，使用 `state.radioBtnselected.indices.forEach` 遍历了 `radioBtnselected` 列表的所有索引。然后，对于每个索引，将 `radioBtnselected` 列表中对应位置的元素设置为 `it == index`，即将当前索引对应的元素设置为 `true`，其余的元素设置为 `false`。

- 最后，通过 `state.copy()` 复制了原始的状态对象，并将修改后的 `radioBtnselected` 列表赋值给新的状态对象的相应属性。

总的来说，这段代码的作用是创建一个新的 `ControlStatus` 对象，其中 `radioBtnselected` 列表的元素根据 `index` 的值进行更新，并将更新后的状态赋值给 `state` 变量。

## 没有 toMutableList 就不更新

我理解了，您希望确保在更新 `state` 变量时不修改原始列表。在这种情况下，您可以使用 `toMutableList()` 来创建原始列表的副本，然后对副本进行修改，最后将修改后的副本赋值给新的 `state` 对象。

下面是使用 `toMutableList()` 的修改后的代码：

```kotlin
kotlinCopy codestate = state.copy(
    radioBtnselected = state.radioBtnselected.toMutableList().apply {
        state.radioBtnselected.indices.forEach {
            this[it] = it == index
        }
    }
)
```

在这里，`state.radioBtnselected.toMutableList()` 创建了 `state.radioBtnselected` 列表的副本，然后我们对副本进行修改。这样做不会修改原始列表，因为我们在副本上进行操作。最后，我们将修改后的副本赋值给新的 `state` 对象。

这样做可以确保您修改的只是副本，而不会影响到原始列表，同时更新了 `state` 变量以反映新的状态。

## 是 this 付给了

对于 `apply` 函数来说，`this` 关键字指的是调用 `apply` 函数的对象，也就是在这个场景下是 `state.radioBtnselected` 列表。因此，`this` 在这里引用的就是 `state.radioBtnselected` 列表对象，而不是新创建的副本。

所以，您的代码中的 `this` 指的是 `state.radioBtnselected` 列表对象，而不是副本。因此，在这个代码中，修改的是原始列表，而不是副本。

如果您希望确保在更新 `state` 变量时不修改原始列表，您可以按照前面提供的示例，在 `apply` 函数中创建副本，并在副本上进行修改，然后将修改后的副本赋值给新的 `state` 对象。

## state.radioBtnselected.indices 这个也是副本？

在这个上下文中，`state.radioBtnselected.indices` 是对 `radioBtnselected` 列表索引的引用，而不是副本。`.indices` 是一个属性，它返回一个表示列表索引范围的 `IntRange` 对象，而不是列表的副本。

因此，`state.radioBtnselected.indices` 表示的是 `radioBtnselected` 列表的索引范围，可以用于遍历该列表的所有索引，而不修改列表本身。

在 `forEach` 循环中，我们对 `state.radioBtnselected.indices` 进行遍历，然后对列表进行操作，但是并没有修改列表的索引范围。所以，`state.radioBtnselected.indices` 并不是副本，而是对索引的引用。

## 谁是副本呀

在您的代码中，`state.radioBtnselected` 代表了列表的原始引用，而不是副本。当您调用 `toMutableList()` 函数时，它创建了 `radioBtnselected` 列表的一个副本，然后您对副本进行了修改。这样做是为了确保在修改列表时不会影响到原始的 `state` 对象。

下面是您的代码中涉及到的副本：

```kotlin
kotlinCopy codestate = state.copy(
    radioBtnselected = state.radioBtnselected.toMutableList().apply {
        // 在这里创建了 radioBtnselected 列表的副本，并对副本进行了修改
        state.radioBtnselected.indices.forEach {
            this[it] = it == index
        }
    }
)
```

在这段代码中，`state.radioBtnselected.toMutableList()` 创建了 `radioBtnselected` 列表的副本。在 `apply` 函数中，我们对副本进行了修改，而不是直接修改原始列表。最后，将修改后的副本赋值给了新的 `state` 对象。

## this[it]  副本？

在这个特定的情况下，`this[it]` 不是副本，而是对副本的引用。在 `apply` 函数中，`this` 引用的是 `state.radioBtnselected` 列表的副本，因为 `apply` 函数会将闭包内的代码块应用到调用它的对象上。

所以，当您在 `apply` 中使用 `this[it]` 时，您实际上是在引用副本列表中的索引 `it` 处的元素，并修改该副本列表的元素值。

这是因为 `this` 在 `apply` 中指的是 `state.radioBtnselected` 的副本，而不是原始列表。因此，您可以放心地在 `apply` 中修改副本列表的元素值，而不会影响到原始列表。

## 也没return 呀

您是对的，在上述的代码片段中，`apply` 函数并没有返回值。`apply` 函数是一个 Kotlin 标准库中的扩展函数，它是一个作用域函数，可以在对象上执行代码块，然后返回这个对象本身。

在这个特定的代码片段中，`apply` 函数的主要目的是允许您在 `state.radioBtnselected` 列表的副本上执行操作，并且不需要显式地返回任何值。由于 `apply` 函数返回的是调用它的对象本身，所以最终返回的值是 `state` 对象，而不是 `apply` 函数内部的任何操作结果。

因此，即使在 `apply` 内部没有显式地使用 `return` 语句，`state` 对象仍然会被修改，并且最终的 `state` 对象将被用于赋值操作。
