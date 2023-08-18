---
title: Kotlin中的lambda
date: 2023-08-18 10:18:40
tags: lambda
categories: lambda
top_img: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
cover: https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png
---

------



![聊聊Kotlin中的lambda](https://s2.loli.net/2023/08/18/qGUY8y2d4CQXnxS.png)

#  

## 1. lambda 表达式基本语法



### 1.1 lambda 表达式分类

在 Kotlin 实际上可以把 Lambda 表达式分为两个大类，

***一个是普通的 lambda 表达式，另一个则是带接收者的 lambda 表达式。***

这两种 lambda 在使用和使用场景也是有很大的不同。先看下以下两种 lambda表达式的类型声明：

![image-20230818113654648](https://s2.loli.net/2023/08/18/Q6BdZvYoTAOafjl.png)

针对带接收者的 Lambda 表达式在 Kotlin 中标准库函数中也是非常常见的比如 with,apply 标准函数的声明。

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}

@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

看到以上的 lambda 表达式的分类，是不是想到之前的扩展函数了。实际上普通的 Lambda 表达式类似对应普通函数的声明，而带接收者的 lambda 表达式则类似对应扩展函数。扩展函数就是这种声明接收者类型，然后使用接收者对象调用直接类似成员函数调用，实际内部是通过这个接收者对象实例直接访问它的方法和属性。



### 1.2 lambda基本语法

lambda的**标准形式**基本声明满足三个条件：

- **含有实际参数**；
- **含有函数体(尽管函数体为空，也得声明出来)**；
- **以上内部必须被包含在花括号内部**。

![image-20230818110657799](https://s2.loli.net/2023/08/18/14t6ZB2RvV8E57w.png)

以上是 lambda 表达式最标准的形式，可能这种标准形式在以后的开发中可能见到比较少，更多是更加的简化形式，下面就是会介绍 Lambda 表达式简化规则。



### 1.3 lambda 语法简化转换

以后开发中我们更多的是使用简化版本的 lambda 表达式，因为看到标准的 lambda 表达式形式还是有些啰嗦，比如实参类型就可以省略，因为 Kotlin 这门语言支持根据上下文环境智能推导出类型，所以可以省略，摒弃啰嗦的语法，下面是 lambda 简化规则。

![image-20230818111950908](https://s2.loli.net/2023/08/18/cZzMihDgLJynwC7.png)

**注意**：语法简化是把双刃剑，简化固然不错，使用简单方便，但是不能滥用，也需要考虑到代码的可读性。

上图中 Lambda 简化成的最简单形式用 it 这种，一般在多个 Lambda 嵌套的时候不建议使用，严重造成代码可读性，到最后估计连开发者都不知道 it 指代什么了。比如以下代码：

这是 Kotlin 库中的 joinToString 扩展函数，最后一个参数是一个**接收一个集合元素类型T的参数返回一个CharSequence类型**的 lambda 表达式。

```kotlin
//joinToString内部声明
public fun <T> Iterable<T>.joinToString(separator: CharSequence = ", ", prefix: CharSequence = "", postfix: CharSequence = "", limit: Int = -1, truncated: CharSequence = "...", transform: ((T) -> CharSequence)? = null): String {
    return joinTo(StringBuilder(), separator, prefix, postfix, limit, truncated, transform).toString()
}


fun main(args: Array<String>) {
    val num = listOf(1, 2, 3)
    println(num.joinToString(separator = ",", prefix = "<", postfix = ">") {
        return@joinToString "index$it"
    })
}

```

我们可以看到 joinToString 的调用地方是使用了 lambda 表达式作为参数的简化形式，将它从圆括号中提出来了。这个确实给调用带来一点小疑惑，因为并没有显示表明 lambda 表达式应用到哪里，所以不熟悉内部实现的开发者很难理解。对于这种问题，Kotlin 实际上给我们提供解决办法，也就是我们之前博客提到过的命名参数。使用命名参数后的代码：

```kotlin
//joinToString内部声明
public fun <T> Iterable<T>.joinToString(separator: CharSequence = ", ", prefix: CharSequence = "", postfix: CharSequence = "", limit: Int = -1, truncated: CharSequence = "...", transform: ((T) -> CharSequence)? = null): String {
    return joinTo(StringBuilder(), separator, prefix, postfix, limit, truncated, transform).toString()
}
fun main(args: Array<String>) {
    val num = listOf(1, 2, 3)
    println(num.joinToString(separator = ",", prefix = "<", postfix = ">", transform = { "index$it" }))
}

```

### 1.4 lambda表达式的返回值

**lambda表达式返回值总是返回函数体内部最后一行表达式的值**：

```kotlin
package com.imooc.kotlin.lambda

fun main(args: Array<String>) {

    val isOddNumber = { number: Int ->
        println("number is $number")
        number % 2 == 1
    }

    println(isOddNumber.invoke(100))
}
```

![image-20230818112326611](https://s2.loli.net/2023/08/18/tIXnwfAKdiPSaTD.png)

将函数体内的两个表达式互换位置后：

```kotlin
package com.imooc.kotlin.lambda

fun main(args: Array<String>) {

    val isOddNumber = { number: Int ->
        number % 2 == 1
        println("number is $number")
    }

    println(isOddNumber.invoke(100))
}
```

![](https://s2.loli.net/2023/08/18/jDbXThEt56Iwf2g.png)

通过上面例子可以看出 lambda 表达式是返回函数体内最后一行表达式的值，由于 println 函数没有返回值，所以默认打印出来的是 Unit 类型，那它内部原理是什么呢？实际上是通过最后一行表达式返回值类型作为了 invoke 函数的返回值的类型，我们可以对比上述两种写法的反编译成 java 的代码：

```java
//互换位置之前的反编译代码
package com.imooc.kotlin.lambda;

import kotlin.jvm.internal.Lambda;

@kotlin.Metadata(mv = {1, 1, 10}, bv = {1, 0, 2}, k = 3, d1 = {"\000\016\n\000\n\002\020\013\n\000\n\002\020\b\n\000\020\000\032\0020\0012\006\020\002\032\0020\003H\n¢\006\002\b\004"}, d2 = {"<anonymous>", "", "number", "", "invoke"})
final class LambdaReturnValueKt$main$isOddNumber$ extends Lambda implements kotlin.jvm.functions.Function1<Integer, Boolean> {
    public final boolean invoke(int number) {//此时invoke函数返回值的类型是boolean，对应了Kotlin中的Boolean
        String str = "number is " + number;
        System.out.println(str);
        return number % 2 == 1;
    }

    public static final 1INSTANCE =new 1();

    LambdaReturnValueKt$main$isOddNumber$1() {
        super(1);
    }
}


//互换位置之后的反编译代码
package com.imooc.kotlin.lambda;

import kotlin.jvm.internal.Lambda;

@kotlin.Metadata(mv = {1, 1, 10}, bv = {1, 0, 2}, k = 3, d1 = {"\000\016\n\000\n\002\020\002\n\000\n\002\020\b\n\000\020\000\032\0020\0012\006\020\002\032\0020\003H\n¢\006\002\b\004"}, d2 = {"<anonymous>", "", "number", "", "invoke"})
final class LambdaReturnValueKt$main$isOddNumber$1 extends Lambda implements kotlin.jvm.functions.Function1<Integer, kotlin.Unit> {
    public final void invoke(int number) {//此时invoke函数返回值的类型是void，对应了Kotlin中的Unit
        if (number % 2 != 1) {
        }
        String str = "number is " + number;
        System.out.println(str);
    }

    public static final 1INSTANCE =new 1();

    LambdaReturnValueKt$main$isOddNumber$1() {
        super(1);
    }
}
```



### 1.5 lambda表达式类型

Kotlin 中提供了简洁的语法去定义函数的类型：

```kotlin
() -> Unit//表示无参数无返回值的Lambda表达式类型

(T) -> Unit//表示接收一个T类型参数，无返回值的Lambda表达式类型

(T) -> R//表示接收一个T类型参数，返回一个R类型值的Lambda表达式类型

(T, P) -> R//表示接收一个T类型和P类型的参数，返回一个R类型值的Lambda表达式类型

(T, (P,Q) -> S) -> R//表示接收一个T类型参数和一个接收P、Q类型两个参数并返回一个S类型的值的Lambda表达式类型参数，返回一个R类型值的Lambda表达式类型
```

上面几种类型前面几种应该好理解，估计有点难度是最后一种，最后一种实际上已经属于高阶函数的范畴。不过这里说下个人看这种类型的一个方法有点像剥洋葱一层一层往内层拆分，就是由外往里看，然后做拆分，对于本身是一个 Lambda 表达式类型的，先暂时看做一个整体，这样就可以确定最外层的Lambda 类型，然后再用类似方法往内部拆分。

![image-20230818112429512](/Users/lijiaming/myblog/source/_posts/Kotlin 中的 lambda/image-20230818112429512.png)

### 1.6 调用作为参数的函数

```kotlin
fun twoAndThree(operation: (Int, Int) -> Int){
    //调用函数类型的参数
    val result = operation(2, 3)
    println("The result is $result")
}

fun main(arg: Array<String>) {
    twoAndThree{ a, b -> a + b}
    twoAndThree{ a, b -> a * b}
}

这个和C++嗯的回调差不多，类似。
```

### 1.7 返回函数的函数

```kotlin
enum class Delivery { STANDARD, EXPEDITED }

class Order(val itemCount: Int)

fun getShippingCostCalculator(
        delivery: Delivery): (Order) -> Double {
    if (delivery == Delivery.EXPEDITED) {
        return { order -> 6 + 2.1 * order.itemCount }
    }
    return { order -> 1.2 * order.itemCount }
}
```

### 

## 2. lambda 表达式常用的场景

**场景一**： lambda 表达式与集合一起使用，是最常见的场景，可以各种筛选、映射、变换操作符和对集合数据进行各种操作，非常灵活，相信使用过 RxJava 中的开发者已经体会到这种快感，没错 Kotlin 在语言层面，无需增加额外库，就给你提供了支持函数式编程 API：

```kotlin
package com.imooc.kotlin.lambda

fun main(args: Array<String>) {
    val nameList = listOf("Kotlin", "Java", "Python", "JavaScript", "Scala", "C", "C++", "Go", "Swift")
    nameList.filter {
        it.startsWith("K")
    }.map {
        "$it is a very good language"
    }.forEach {
        println(it)
    }
}
```

**场景二**：替代原有匿名内部类，但是需要注意一点就是只能替代含有单抽象方法的类：

```java
	findViewById(R.id.submit).setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(View v) {
				...
			}
		});
```

用 kotlin lambda 实现：

```kotlin
findViewById(R.id.submit).setOnClickListener{
    ...
}

```

**场景三**：定义Kotlin扩展函数或者说需要把某个操作或函数当做值传入的某个函数的时候：

```kotlin
fun Context.showDialog(content: String = "", negativeText: String = "取消", positiveText: String = "确定", isCancelable: Boolean = false, negativeAction: (() -> Unit)? = null, positiveAction: (() -> Unit)? = null) {
	AlertDialog.build(this)
			.setMessage(content)
			.setNegativeButton(negativeText) { _, _ ->
				negativeAction?.invoke()
			}
			.setPositiveButton(positiveText) { _, _ ->
				positiveAction?.invoke()
			}
			.setCancelable(isCancelable)
			.create()
			.show()
}

fun Context.toggleSpFalse(key: String, func: () -> Unit) {
	if (!getSpBoolean(key)) {
		saveSpBoolean(key, true)
		func()
	}
}

fun <T : Any> Observable<T>.subscribeKt(success: ((successData: T) -> Unit)? = null, failure: ((failureError: RespException?) -> Unit)? = null): Subscription? {
	return transformThread()
			.subscribe(object : SBRespHandler<T>() {
				override fun onSuccess(data: T) {
					success?.invoke(data)
				}

				override fun onFailure(e: RespException?) {
					failure?.invoke(e)
				}
			})
}
```



## 3. lambda 表达式的作用域中访问变量和变量捕获



### 3.1 Kotlin 和 Java 内部类或 lambda 访问局部变量的区别

在 Java 中在函数内部定义一个匿名内部类或者 lambda，内部类访问的函数局部变量必须需要 final 修饰，也就意味着在内部类内部或者 lambda 表达式的内部是无法去修改函数局部变量的值。可以看一个很简单的 Android 事件点击的例子：

```kotlin
public class DemoActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);
        final int count = 0;//需要使用final修饰
        findViewById(R.id.btn_click).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                System.out.println(count);//在匿名OnClickListener类内部访问count必须要是final修饰
            }
        });
    }
}
```

在 Kotlin 中在函数内部定义 lambda 或者内部类，既可以访问final修饰的变量，也可以访问非 final 修饰的变量，也就意味着在 Lambda 的内部是可以直接修改函数局部变量的值。以上例子 Kotlin 实现：

访问 final 修饰的变量：

```kotlin
class Demo2Activity : AppCompatActivity() {

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_demo2)
		val count = 0//声明final
		btn_click.setOnClickListener {
			println(count)//访问final修饰的变量这个是和Java是保持一致的。
		}
	}
}
```

访问非 final 修饰的变量，并修改它的值：

```kotlin
class Demo2Activity : AppCompatActivity() {

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_demo2)
		var count = 0//声明非final类型
		btn_click.setOnClickListener {
			println(count++)//直接访问和修改非final类型的变量
		}
	}
}
```

通过以上对比会发现 Kotlin 中使用 lambda 会比 Java 中使用 lambda 更灵活，访问受到限制更少，这也就回答本博客最开始说的一句话，Kotlin 中的 lambda 表达式是真正意义上的支持闭包，而 Java中 的lambda 则不是。Kotlin 中的 lambda 表达式是怎么做到这一点的呢？



### 3.2 lambda 表达式的变量捕获及其原理

**什么是变量捕获?**

通过上述例子，我们知道在 Kotlin 中既能访问 final 的变量也能访问或修改非 final 的变量。原理是怎样的呢？在此之前先抛出一个高大上的概念叫做**lambdab表达式的变量捕获**。实际上就是 lambda 表达式在其函数体内可以访问外部的变量，我们就称这些外部变量被 lambda 表达式给捕获了。有了这个概念我们可以把上面的结论变得高大上一些：

第一在Java中lambda表达式只能捕获final修饰的变量

第二在Kotlin中lambda表达式既能捕获final修饰的变量也能访问和修改非final的变量

**变量捕获实现的原理**

函数的局部变量生命周期是属于这个函数的，当函数执行完毕，局部变量也就是销毁了，但是如果这个局部变量被 lambda 捕获了，那么使用这个局部变量的代码将会被存储起来等待稍后再次执行，也就是被捕获的局部变量是可以延迟生命周期的，**针对 lambda 表达式捕获 final 修饰的局部变量原理是局部变量的值和使用这个值的 lambda 代码会被一起存储起来；而针对于捕获非 final 修饰的局部变量原理是非 final 局部变量会被一个特殊包装器类包装起来，这样就可以通过包装器类实例去修改这个非 final 的变量，那么这个包装器类实例引用是 final 的会和 lambda 代码一起存储**。

以上第二条结论在 Kotlin 的语法层面来说是正确的，但是从真正的原理上来说是错误的，只不过是 Kotlin 在语法层面把这个屏蔽了而已，实质的原理 lambda 表达式还是只能捕获 final 修饰变量，而为什么 Kotlin 却能做到修改非 final 的变量的值，**实际上 kotlin 在语法层面做了一个桥接包装，它把所谓的非 final 的变量用一个 Ref 包装类包装起来，然后外部保留着 Ref 包装器的引用是 final 的，然后lambda 会和这个 final 包装器的引用一起存储，随后在 lambda 内部修改变量的值实际上是通过这个final 的包装器引用去修改的。**

![image-20230818112501891](/Users/lijiaming/myblog/source/_posts/Kotlin 中的 lambda/image-20230818112501891.png)

最后通过查看 Kotlin 修改非 final 局部变量的反编译成的 Java 代码就是一目了然了：

```kotlin
class Demo2Activity : AppCompatActivity() {

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_demo2)
		var count = 0//声明非final类型
		btn_click.setOnClickListener {
			println(count++)//直接访问和修改非final类型的变量
		}
	}
}
public final class Demo2Activity extends AppCompatActivity {
   private HashMap _$_findViewCache;

   protected void onCreate(@Nullable Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      this.setContentView(2131361820);
      final IntRef count = new IntRef();//IntRef特殊的包装器类的类型，final修饰的IntRef的count引用
      count.element = 0;//包装器内部的非final变量element
      ((Button)this._$_findCachedViewById(id.btn_click)).setOnClickListener((OnClickListener)(new OnClickListener() {
         public final void onClick(View it) {
            int var2 = count.element++;//直接是通过IntRef的引用直接修改内部的非final变量的值，来达到语法层面的lambda直接修改非final局部变量的值
            System.out.println(var2);
         }
      }));
   }

   public View _$_findCachedViewById(int var1) {
      if(this._$_findViewCache == null) {
         this._$_findViewCache = new HashMap();
      }

      View var2 = (View)this._$_findViewCache.get(Integer.valueOf(var1));
      if(var2 == null) {
         var2 = this.findViewById(var1);
         this._$_findViewCache.put(Integer.valueOf(var1), var2);
      }

      return var2;
   }

   public void _$_clearFindViewByIdCache() {
      if(this._$_findViewCache != null) {
         this._$_findViewCache.clear();
      }

   }
}
```



### 3.3 lambda表达式变量捕获注意问题

**注意**：对于Lambda表达式内部修改局部变量的值，只会在这个 Lambda 表达式被执行的时候触发。



## 4. lambda表达式的成员引用



### 4.1 为什么要使用成员引用

在 Lambda 表达式可以直接把一个代码块作为一个参数传递给函数，但是有没有遇到过这样一个场景就是我要传递过去的代码块，已经是作为了一个命名函数存在了，此时你还需要重复写一个代码块传递过去吗？肯定不是，Kotlin 拒绝啰嗦重复的代码。所以只需要成员引用替代即可。

```kotlin
fun main(args: Array<String>) {
    val persons = listOf(Person(name = "imooc1", age = 18), Person(name = "imooc2", age = 20), Person(name = "imooc3", age = 16))
    println(persons.maxBy({ p: Person -> p.age }))
}
```

可以替代为：

```kotlin
fun main(args: Array<String>) {
    val persons = listOf(Person(name = "imooc1", age = 18), Person(name = "imooc2", age = 20), Person(name = "imooc3", age = 16))
    println(persons.maxBy(Person::age))//成员引用的类型和maxBy传入的lambda表达式类型一致
}
```



### 4.2 成员引用的基本语法

成员引用由类、双冒号、成员三个部分组成：

![image-20230818112558105](/Users/lijiaming/myblog/source/_posts/Kotlin 中的 lambda/image-20230818112558105.png)



### 4.3 成员引用的使用场景

成员引用最常见的使用方式就是类名+双冒号+成员(属性或函数)：

```kotlin
fun main(args: Array<String>) {
    val persons = listOf(Person(name = "imooc1", age = 18), Person(name = "imooc2", age = 20), Person(name = "imooc3", age = 16))
    println(persons.maxBy(Person::age))//成员引用的类型和maxBy传入的lambda表达式类型一致
}
```

省略类名直接引用顶层函数：

```kotlin
package com.imooc.lambda

fun salute() = print("imooc")

fun main(args: Array<String>) {
    run(::salute)
}
```

成员引用用于扩展函数：

```kotlin
fun Person.isChild() = age < 18

fun main(args: Array<String>){
    val isChild = Person::isChild
    println(isChild)
}
```

