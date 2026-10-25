---
title: Kotlin 反射笔记（最终定稿）
date: 2026-10-25 10:20:19
top_img: /img/post-covers/Kotlin反射笔记.png
cover: /img/post-covers/Kotlin反射笔记.png
tags:
  - Kotlin
  - 反射
  - Android
categories: 安全研究
---
> 本笔记使用 `Dog::class.java` / `dog.javaClass` 走 **Java 反射 API**，不是 `kotlin-reflect` 的 `KClass` 反射。Java 反射 API 在 Android 上更通用，也不需要额外依赖。
>
> Kotlin 里 `isAccessible = true` 等价于 Java 的 `setAccessible(true)`；Kotlin 没有受检异常，因此示例不再写 `throws Exception`，异常处理会在文中单独说明。

## 一、什么是反射？

**反射（Reflection）就是程序运行时，动态地查看类的信息，并操作类、方法、字段和构造方法。**

正常情况下，我们写代码时就知道要调用哪个方法：

```kotlin
val dog = Dog()
dog.bark()
```

使用反射时，可以在运行过程中再去查找这个方法，然后调用它：

```kotlin
val clazz = Dog::class.java
val method = clazz.getDeclaredMethod("bark")
method.isAccessible = true
method.invoke(dog)
```

上面这段代码的意思是：

1. `Dog::class.java`：拿到 Dog 类对应的 `Class` 对象。
2. `getDeclaredMethod("bark")`：找到 Dog 类自己声明的 `bark()` 方法。
3. `isAccessible = true`：尝试允许反射访问这个方法，即使它是 private 方法。
4. `invoke(dog)`：让这个方法在 `dog` 对象上执行。

注意：反射不是直接修改 DEX 文件，而是操作程序运行时的类和成员信息。

---

## 二、反射涉及的几个核心对象

| 对象 | 作用 | 可以理解为 |
|---|---|---|
| `Class` | 表示运行时的类 | 类的信息入口 |
| `Method` | 表示一个方法 | 方法的描述信息 |
| `Field` | 表示一个字段 | 字段的描述信息 |
| `Constructor` | 表示一个构造方法 | 创建对象的方式 |
| 普通对象 | 类创建出来的实例 | 真正的对象 |

例如：

```kotlin
class Dog {
    private var age: Int = 0

    private fun bark() {
        println("汪汪")
    }
}

val dog = Dog()
val clazz = dog.javaClass
```

这里有两个不同的东西：

- `dog`：Dog 类创建出来的普通对象。
- `clazz`：描述 Dog 这个类的 `Class` 对象。

**记住：类对象 `Class` 和普通实例对象不是同一个东西。**

可以这样理解：

- `Class` 像一张说明书。
- `Method` 像说明书中某个方法的介绍。
- `Field` 像说明书中某个字段的介绍。
- `dog` 才是真正的狗对象。

---

## 三、Class：获取类的信息

### 1. 获取 Class 对象

```kotlin
val clazz1 = Dog::class.java
val clazz2 = dog.javaClass
val clazz3 = Class.forName("com.example.Dog")
```

三种方式的区别：

| 写法 | 含义 |
|---|---|
| `Dog::class.java` | 通过类名获取 Class 对象 |
| `dog.javaClass` | 通过已有对象获取它实际所属的类 |
| `Class.forName("完整类名")` | 通过类名字符串查找并加载类 |

注意：

- `dog.javaClass` 要求 `dog` 不是 `null`。
- `Class.forName()` 默认会初始化目标类，可能触发静态初始化代码。
- `ClassLoader.loadClass()` 通常只加载类，不会立即初始化它。
- 同名类如果由不同的 ClassLoader 加载，通常会被视为不同的运行时类型。

如果想使用指定的 ClassLoader 加载类，并尽量避免立即初始化，可以使用：

```kotlin
val clazz = Class.forName("com.example.Dog", false, classLoader)
```

### 2. 获取类名

```kotlin
clazz.name        // 完整类名
clazz.simpleName  // 简单类名
clazz.superclass  // 直接父类
clazz.interfaces  // 直接实现的接口
```

`interfaces` 只返回这个类直接声明实现的接口，不会自动递归列出父类的所有接口。

### 3. 获取类的方法

```kotlin
val methods = clazz.declaredMethods
```

这会获取 Dog 类自己声明的方法，包括不同访问级别的方法，但不包括从父类继承的方法。

方法数组的排列顺序没有保证。

**补充七：字段和方法数组的顺序**

```kotlin
clazz.declaredMethods
clazz.declaredFields
```

这两个属性返回的数组，其排列顺序都没有保证。

如果需要查找某个特定字段或方法，应该根据名称、参数类型等信息查找，而不是假设某个成员永远位于数组的第几个位置。

---

## 四、getDeclaredXxx() 和 getXxx() 的区别

这是反射里非常重要的一点。

| API | 能获取什么 |
|---|---|
| `getDeclaredMethod()` | 当前类自己声明的方法，包含 private 等访问级别 |
| `getMethod()` | public 方法，包括符合条件的继承 public 方法 |
| `getDeclaredField()` | 当前类自己声明的字段，包含 private 等访问级别 |
| `getField()` | public 字段，包括符合条件的继承 public 字段 |
| `getDeclaredConstructor()` | 当前类自己声明的指定构造方法 |
| `getConstructor()` | 当前类的指定 public 构造方法 |

注意：

1. `Declared` 不代表只能获取 private，它可以获取当前类声明的各种访问级别的成员。
2. `Declared` 的重点是**只查当前类自己声明的成员，不查继承来的成员**。
3. 构造方法不会被继承，因此构造方法没有“获取父类继承构造方法”的问题。
4. 找不到对应成员时，通常会抛出 `NoSuchMethodException` 或 `NoSuchFieldException`。

例如：

```kotlin
val m1 = clazz.getDeclaredMethod("bark")
val m2 = clazz.getMethod("toString")
```

如果 `bark()` 是 Dog 自己声明的 private 方法，第一句可以找到它。
第二句可以找到从父类继承来的 public `toString()` 方法。

**补充三：如何查找父类的 private 字段**

`getDeclaredField()` 只查当前类自己声明的字段。

例如：

```kotlin
open class Animal {
    private var age: Int = 3
}

class Dog : Animal()
```

下面的代码找不到 `age`：

```kotlin
val field = Dog::class.java.getDeclaredField("age")
```

因为 `age` 声明在父类 `Animal` 中。

可以逐级向上查找：

```kotlin
var clazz: Class<*>? = Dog::class.java
var field: Field? = null

while (clazz != null) {
    try {
        field = clazz.getDeclaredField("age")
        break
    } catch (e: NoSuchFieldException) {
        clazz = clazz.superclass
    }
}

if (field != null) {
    field.isAccessible = true  // 尝试抑制访问检查，不保证成功
    println(field.getInt(Dog()))
}
```

理解这个循环的关键：

1. 先检查 Dog 自己有没有 `age`。
2. 没找到，就通过 `superclass` 获取父类。
3. 继续查父类，直到找到字段或到达继承链顶端。

注意：找到字段后，能否访问仍取决于运行环境的访问限制。

---

## 五、Method：查找和调用方法

### 1. 获取方法

```kotlin
val method = clazz.getDeclaredMethod("bark")
```

如果方法有参数，需要指定参数类型：

```kotlin
val method = clazz.getDeclaredMethod("eat", String::class.java)
```

假设目标方法是：

```kotlin
private fun eat(food: String) {
    println(food)
}
```

这里的 `String::class.java` 表示参数类型是 String，而不是传入一个 String 实例。

如果有两个参数：

```kotlin
private fun test(name: String, age: Int) {
}
```

可以这样获取：

```kotlin
val method = clazz.getDeclaredMethod(
    "test",
    String::class.java,
    Int::class.java
)
```

**补充一：查找方法的规则和调用方法的规则要分开理解**

`getDeclaredMethod()` 查找方法时，根据**方法名 + 方法声明中的参数类型**精确匹配。

这里的“参数类型”指的是方法签名里声明的类型（经过泛型擦除等处理后的运行时类型），
不是你想传入的实参类型，也不支持用子类/父类类型替代。

例如：

```kotlin
class Demo {
    private fun foo(obj: Any) {
    }
}
```

正确：

```kotlin
val m = Demo::class.java.getDeclaredMethod("foo", Any::class.java)
```

错误：

```kotlin
val m = Demo::class.java.getDeclaredMethod("foo", String::class.java)
```

虽然 `String` 是 `Any` 的子类，但方法声明的是 `Any`，
所以查找时必须写 `Any::class.java`，用 `String::class.java` 会抛 `NoSuchMethodException`。

注意几个容易踩坑的声明类型：

- **可变参数** `fun foo(vararg args: String)`：声明类型是 `Array<String>::class.java`（即 `String[].class`），不是 `String::class.java`。
- **泛型方法** `fun <T> foo(t: T)`：擦除后是类型变量 `T` 的上界（无上界时通常是 `Any::class.java`；如果写成 `<T : Number>`，擦除后通常是 `Number::class.java`）。
- **基本类型与包装类型在 Kotlin 里的表达**：
  - `Int::class.java` 对应 Java `int.class`（基本类型）
  - `Int::class.javaObjectType` 对应 Java `Integer.class`（包装类型）
  - `Long::class.java` 对应 `long.class`，`Long::class.javaObjectType` 对应 `Long.class`
  - 查找时 `Int::class.java` 和 `Int::class.javaObjectType` 是两个不同的条件。

**查找规则和调用规则不是一回事：**

| 阶段 | 规则 |
|---|---|
| `getDeclaredMethod(...)` | 按方法声明的参数类型精确匹配，不做子类替代、不做装箱/拆箱 |
| `Method.invoke(...)` | 按反射调用自身的参数转换规则处理，允许一定的装箱、拆箱和基本类型拓宽，但不完全等同于 Kotlin/Java 源码编译时的所有调用转换 |

需要特别注意：反射调用的转换规则并不完全等同于源码层面的调用规则。

例如，不能指望把 `Integer` 自动转换成 `Long` 后传给 `long` 参数；
也不要假设“源码里能自动转换的，反射里一定也能自动转换”。

最稳妥的做法是：**按目标方法声明的参数类型，传入类型匹配或明确可转换的实参。**

可以用下面三条对照帮助记忆（以 `Integer` 对象传给目标参数为例）：

| 传入值 | 目标参数 | 是否允许 |
|---|---|---|
| `Integer` | `int` | 允许，拆箱 |
| `Integer` | `long` | 反射调用可以通过拆箱后再进行基本类型拓宽 |
| `Integer` | `Long` | 不能依赖自动转换，包装类型之间不会自动互转 |

核心区别：

- `Integer → int`：拆箱。
- `Integer → long`：拆箱后再进行基本类型拓宽。
- `Integer → Long`：不能依赖自动转换。

下面用代码把 `Integer → long` 和 `Integer → Long` 两种情况对比清楚。

**可以：`Integer` 传给 `long` 参数**

```kotlin
class Demo {
    private fun test(value: Long) {
        println(value)
    }
}

val m = Demo::class.java.getDeclaredMethod("test", Long::class.java)
m.isAccessible = true  // 尝试抑制访问检查，不保证成功
// Integer 拆箱为 int，再拓宽为 long
m.invoke(Demo(), java.lang.Integer.valueOf(5))
```

预期输出：

```text
5
```

这段示例在常规环境下可以正常演示 `Integer → long` 的转换。

`m.isAccessible = true` 只是尝试抑制访问检查，并不保证一定成功；
如果运行环境限制访问，可能需要调整代码或运行环境。

**不可以：`Integer` 传给 `Long` 参数**

```kotlin
class Demo {
    private fun test(value: java.lang.Long) {
        println(value)
    }
}

val m = Demo::class.java.getDeclaredMethod("test", java.lang.Long::class.java)
m.isAccessible = true  // 尝试抑制访问检查，不保证成功
// 不要认为 Integer 会自动转换成 Long
m.invoke(Demo(), java.lang.Integer.valueOf(5))
```

因为 `Integer` 和 `Long` 是两种不同的包装类型，反射不会自动把前者转换成后者。

这类调用通常会因为参数类型不匹配而抛出 `IllegalArgumentException`；
具体表现可能因调用方式和运行时略有差异，但不要把“源码里能自动转换”直接套到这里。

但也不要因此把 `Integer → long` 理解成任意数值类型都能随意互转；
反射调用的转换仍受具体规则约束。

### 2. 调用方法

```kotlin
method.isAccessible = true  // 尝试抑制访问检查，不保证成功
method.invoke(dog)
```

`invoke()` 的第一个参数通常是**方法要在哪个对象上执行**。

```kotlin
method.invoke(dog)
```

表示在 `dog` 对象上调用这个方法。

如果是静态方法，则可以传 `null`：

```kotlin
method.invoke(null as Any?)
```

如果目标方法需要参数：

```kotlin
private fun eat(food: String) {
    println(food)
}
```

调用方式：

```kotlin
method.invoke(dog, "苹果")
```

如果目标方法抛出异常，反射调用通常会把目标方法的异常包装在 `InvocationTargetException` 中，需要进一步查看其原因。

**补充一（续）：invoke() 的装箱与拆箱**

```kotlin
class Demo {
    private fun test(age: Int) {
        println(age)
    }
}
```

查找时用声明类型 `Int::class.java`（即 `int.class`）：

```kotlin
val m = Demo::class.java.getDeclaredMethod("test", Int::class.java)
```

调用时可以直接传 `Int` 字面量：

```kotlin
m.isAccessible = true  // 尝试抑制访问检查，不保证成功
m.invoke(Demo(), 5)
```

`Method.invoke()` 的形参是 `Object...`（Kotlin 侧是 `vararg args: Any?`）。传入 `5` 时，Kotlin 先把它装箱为 `Integer`；
进入反射调用后，再依据目标方法声明的参数类型 `int`，执行允许的拆箱转换。

也就是说，`5 → Integer` 是语言层面的装箱，而 `Integer → int` 是反射调用参数匹配时允许的拆箱转换，两者属于不同的步骤。

但是，不能给基本类型参数传 `null`：

```kotlin
m.invoke(Demo(), null as Any?)
```

这通常会因为参数不匹配而抛出 `IllegalArgumentException`，
具体表现取决于调用方式和运行时，不必把它简单归结为某一种固定异常。

**补充五：查看方法或字段的修饰符**

可以使用 `modifiers` 获取修饰符信息，再通过 `Modifier` 判断：

```kotlin
import java.lang.reflect.Modifier

val method = clazz.getDeclaredMethod("bark")
val modifiers = method.modifiers
println(Modifier.isPublic(modifiers))
println(Modifier.isPrivate(modifiers))
println(Modifier.isStatic(modifiers))
```

字段也可以这样判断：

```kotlin
val field = clazz.getDeclaredField("age")
val modifiers = field.modifiers
println(Modifier.isPrivate(modifiers))
println(Modifier.isFinal(modifiers))
println(Modifier.isStatic(modifiers))
```

`modifiers` 返回的是修饰符编码，建议使用 `Modifier.isXxx()` 判断，不要自己猜测数字代表什么。

---

## 六、Field：读取和修改字段

假设类是：

```kotlin
class Dog {
    private var age: Int = 3
    private var name: String = "小黑"
}
```

### 1. 获取字段

```kotlin
val ageField = clazz.getDeclaredField("age")
val nameField = clazz.getDeclaredField("name")
```

`Field` 表示字段的描述信息。

注意：**获取到 Field，不代表已经拿到了某个对象中的字段值。**

**Kotlin 属性与 Java 字段的关系：**

Kotlin 的 `val` / `var` 属性如果带有 backing field，编译后会在 JVM 上生成同名字段，
`getDeclaredField("age")` 一般能找到。但以下情况**没有普通 backing field**，反射找不到对应字段：

- 计算属性（只有自定义 `get()`，没有 backing field）。
- 扩展属性。
- 委托属性（字段可能被委托对象持有，名称也可能不同）。

所以对 Kotlin 类做字段反射时，要先确认目标属性确实有 backing field。

### 2. 读取字段值

```kotlin
ageField.isAccessible = true  // 尝试抑制访问检查，不保证成功
val age = ageField.getInt(dog)
```

这里有两个重要部分：

- `ageField`：决定读取哪个字段。
- `dog`：决定读取哪个对象里的字段值。

如果还有一个对象：

```kotlin
val dog1 = Dog()
val dog2 = Dog()
```

那么：

```kotlin
ageField.getInt(dog1)
ageField.getInt(dog2)
```

读取的是两个不同实例各自的 `age` 字段。

对于普通实例字段，传入的对象必须是目标字段所属类的实例，或者它的兼容子类实例。

### 3. 不同类型的字段怎么读取？

| 字段类型 | 常用读取方式 |
|---|---|
| `Int` | `getInt(obj)` |
| `Long` | `getLong(obj)` |
| `Boolean` | `getBoolean(obj)` |
| `Double` | `getDouble(obj)` |
| `String` 等引用类型 | `get(obj)` |

特别注意：

- Java 标准反射 API 没有 `Field.getObject()` 这个方法。
- 读取普通对象引用，使用 `field.get(obj)`。
- `getLong(obj)` 适合读取 `long` 字段，不是读取任意类型的字段。
- 如果某个字段保存的是数字，即使这个数字看起来像内存地址，也不能因此认定它就是有效指针。

**补充二：Field.get() 的返回值**

```kotlin
val field = clazz.getDeclaredField("age")
field.isAccessible = true  // 尝试抑制访问检查，不保证成功
val value: Any? = field.get(dog)
```

`field.get(dog)` 返回的是 `Any?`（Java 侧的 `Object`）。

如果字段是基本类型，例如 `int`，读取时会自动装箱：

```text
int 字段
  |
  v
field.get(dog)
  |
  v
Integer 对象
```

如果字段是 `String` 等引用类型，返回的就是对应的对象引用。

因此：

- 已知字段是 `Int`，可以使用 `getInt(obj)`。
- 已知字段是 `Long`，可以使用 `getLong(obj)`。
- 想统一读取不同类型的字段，可以使用 `get(obj)`，再根据实际类型处理返回值。

### 4. 修改字段值

```kotlin
val ageField = clazz.getDeclaredField("age")
ageField.isAccessible = true  // 尝试抑制访问检查，不保证成功
ageField.setInt(dog, 5)
```

这表示把 `dog` 对象的 `age` 字段改成 5。

也可以用通用方法：

```kotlin
ageField.set(dog, 5)
```

如果是对象引用字段：

```kotlin
val nameField = clazz.getDeclaredField("name")
nameField.isAccessible = true  // 尝试抑制访问检查，不保证成功
nameField.set(dog, "小白")
```

如果字段是 `static` 静态字段，读取或修改时通常传 `null` 作为对象参数。

**补充四：final 字段不能随意修改**

例如：

```kotlin
private val age: Int = 3
```

即使获取到 `Field` 并设置 `isAccessible = true`，也不能保证修改成功或修改后的值能被程序正常观察到。

`final` 字段的反射修改受到字段类型、字段修饰符、运行时实现和平台版本等因素影响。对某些 `final` 字段，修改会被禁止；即使某种环境允许，也不应依赖这种行为。

**初学阶段可以先把反射的重点放在读取字段和调用方法上，不要把修改 `final` 字段当作常规做法。**

---

## 七、isAccessible 和 setAccessible 的区别

这部分容易被误解。

```kotlin
field.isAccessible = true
```

等价于 Java 的 `field.setAccessible(true)`，含义是：**尝试关闭这个反射对象的常规 Java 访问检查，让反射可以访问原本受访问权限限制的成员。**

它并不意味着所有访问一定成功。

例如，访问限制、运行时规则、系统隐藏 API 限制或其他安全机制，都可能导致访问失败。

还要注意：访问失败不一定表现为设置 `isAccessible = true` 时当场抛出异常。
在某些环境下，它可能只是无法真正抑制访问检查，后续的 `get`/`set`/`invoke` 等操作才会失败。

不同平台和版本的行为可能不同，不能一概而论。

旧式 API：

```kotlin
field.isAccessible  // 查询旧式访问标志
field.isAccessible = true
```

其中 `isAccessible` 表示这个反射对象是否设置了抑制访问检查的标志，**不是简单地查询“这个字段是否能够访问”**。

较新的 Java 版本提供了：

```kotlin
field.canAccess(obj)
field.trySetAccessible()
```

不过，这些 API 在 Android 上的可用性取决于 Android API 版本。学习 Android 反射时，应以实际运行的 Android 版本和 SDK 为准。

**Kotlin 写法小结：**

```kotlin
field.isAccessible = true
method.isAccessible = true
constructor.isAccessible = true
```

三者都等价于对应的 `setAccessible(true)`，同样只是尝试抑制访问检查，并不保证访问一定成功。

---

## 八、Constructor：通过构造方法创建对象

假设：

```kotlin
class Dog {
    private var name: String

    private constructor(name: String) {
        this.name = name
    }
}
```

获取构造方法：

```kotlin
val constructor = clazz.getDeclaredConstructor(String::class.java)
```

尝试允许访问：

```kotlin
constructor.isAccessible = true  // 尝试抑制访问检查，不保证成功
```

创建对象：

```kotlin
val dog: Any = constructor.newInstance("小黑")
```

这里的 `newInstance()` 才是真正调用构造方法、创建实例的步骤。

注意：

- `getDeclaredConstructor()` 只负责查找构造方法。
- `newInstance()` 才负责创建对象。
- 没有匹配的构造方法时，会抛出 `NoSuchMethodException`。
- 如果类是抽象类或接口，不能直接通过构造方法创建实例。
- 如果构造方法本身抛出异常，反射调用通常会通过 `InvocationTargetException` 抛出包装后的异常。

如果是无参构造方法：

```kotlin
val constructor = clazz.getDeclaredConstructor()
```

---

## 九、完整示例

下面用一个小例子，把获取类、获取字段、获取方法和调用方法连起来。

```kotlin
import java.lang.reflect.Field
import java.lang.reflect.Method

class Dog {
    private var age: Int = 3

    private fun bark(text: String) {
        println(text)
    }
}

fun main() {
    // 注意：Kotlin 没有受检异常，所以这里不需要 throws Exception。
    // 但 isAccessible = true 仍然不保证一定成功，
    // 实际代码应根据运行环境捕获可能的异常。

    // 1. 创建普通实例对象
    val dog = Dog()

    // 2. 获取运行时的 Class 对象
    val clazz = dog.javaClass

    // 3. 获取 age 字段
    val field = clazz.getDeclaredField("age")

    // 4. 尝试允许反射访问私有字段（可能受运行环境限制，不保证成功）
    field.isAccessible = true

    // 5. 读取 dog 对象的 age
    val age = field.getInt(dog)
    println("原来的年龄：$age")

    // 6. 修改 dog 对象的 age
    field.setInt(dog, 5)
    println("修改后的年龄：${field.getInt(dog)}")

    // 7. 获取 bark 方法
    val method = clazz.getDeclaredMethod("bark", String::class.java)

    // 8. 尝试允许反射访问私有方法（可能受运行环境限制，不保证成功）
    method.isAccessible = true

    // 9. 在 dog 对象上调用 bark 方法
    method.invoke(dog, "汪汪！")
}
```

关于异常处理：Kotlin 没有受检异常，编译器不会强制你声明 `throws`，
但这不代表反射调用一定成功。实际代码中应根据运行环境处理可能出现的异常，
而不是默认这些步骤一定成功。

不同步骤可能出现的异常并不相同：

- `isAccessible = true`（即 `setAccessible(true)`）：可能因访问限制抛出 `SecurityException`；在标准 Java API 的模块机制下，无法抑制访问检查时会抛出 `InaccessibleObjectException`。Android 上的行为需要结合具体版本和访问限制判断，不能直接套用 Java 模块机制的行为。另外，访问受限时不一定当场抛异常，有时它本身不报错，但后续访问仍然失败。
- `invoke()`：可能因访问权限问题抛出 `IllegalAccessException`。
- 目标方法自己抛出的异常：通常会被包装在 `InvocationTargetException` 中，需要查看其 `cause` 才能拿到原始异常。

所以不要把这几类异常混在一起笼统处理，而要按步骤理解它们各自的来源，
同时注意平台差异和“不抛异常但访问失败”的情况。

运行结果：

```text
原来的年龄：3
修改后的年龄：5
汪汪！
```

整个过程可以理解为：

```text
普通实例对象 dog
  |
  v
获取 Class 对象
  |
  +------> 获取 Field
  |          |
  |          v
  |        读取或修改 dog 的字段
  |
  +------> 获取 Method
             |
             v
           在 dog 上调用方法
```

---

## 十、Android ART 与反射的关系

在 Android 上，可以把相关概念理解为四个层次：

| 名称 | 主要作用 |
|---|---|
| DEX | 保存类、方法、字段等信息以及 Dalvik 字节码 |
| ClassLoader | 负责查找和加载类 |
| ART | 管理运行时类和对象，并执行或编译代码 |
| Java 反射 API | 让程序在运行时查询和操作类、方法、字段等信息 |

简单来说：

```text
DEX 文件
  |
  v
ClassLoader 加载类
  |
  v
ART 管理运行时类与对象
  |
  v
Java 反射 API 查询或操作成员
```

这只是帮助理解的简化图，并不代表 ART 内部只有这几步。

### 1. Method 和 ArtMethod 不是同一个东西

```kotlin
val method = clazz.getDeclaredMethod("bark")
```

这里的 `method` 是 Java 反射 API 提供的 `Method` 对象。

`ArtMethod` 则是 ART 内部实现方法信息时涉及的内部结构或概念。

两者相关，但**不能直接画等号**。

### 2. 不要假设 Method 一定有 artMethod 字段

例如：

```kotlin
val field = method.javaClass.getDeclaredField("artMethod")
```

这段代码并不保证成功。

原因是：

- Android 不同版本的内部实现可能不同。
- `Method` 的内部字段布局并不是稳定的公开 API。
- `artMethod` 这个字段不一定存在。
- 即使找到了字段，也必须先确认它的真实类型，不能直接假设它是 `long`。

如果字段确实是 `long` 类型，才可以考虑使用：

```kotlin
field.getLong(method)
```

但这只是在读取这个 Java 字段中保存的 `long` 值，不能自动证明这个值就是可直接使用的内存地址。

**补充六：Android 隐藏 API 限制**

Android 9（API 28）开始，对部分非 SDK 接口引入了隐藏 API 限制。

需要区分两种情况：

- **应用自己的 private 方法或字段：** 反射访问通常可以尝试使用 `isAccessible = true`。
- **Android 系统内部的非 SDK 接口：** 可能受到隐藏 API 限制，访问结果取决于接口类别、Android 版本和运行环境。

所以，不能简单地认为 `isAccessible = true` 可以访问 Android 系统内部的一切内容。

在研究 ART 内部实现时，还要注意内部字段和结构并不是稳定的公开 API，不同 Android 版本可能发生变化。

### 3. 反射不等于 Hook

反射可以帮助你找到一个方法并调用它：

```kotlin
method.invoke(dog)
```

但这不等于修改了方法的执行入口，也不等于安装了 Hook。

可以简单理解为：

- **反射：** 查找、查看、调用或修改类的成员。
- **Hook：** 在目标方法执行过程中插入自己的处理逻辑，或改变原本的调用行为。

在 Android 逆向分析中，两者可以配合使用，但作用不同。

---

## 十一、最后记住这几句话

1. `Class` 表示运行时的类，不是普通实例对象；Kotlin 里用 `Dog::class.java` / `dog.javaClass` 获取。
2. `Method` 描述一个方法，`Field` 描述一个字段，`Constructor` 描述一个构造方法。
3. `getDeclaredXxx()` 查当前类自己声明的成员，包含 private 等访问级别。
4. `getXxx()` 对方法和字段通常查 public 成员，并可能包含继承来的成员；构造方法不会被继承。
5. `Method.invoke(obj, args...)` 中，`obj` 通常是方法要在哪个对象上执行。
6. `Field.getInt(obj)` 中，`obj` 决定读取哪个实例的字段值。
7. `isAccessible = true`（等价 `setAccessible(true)`）是尝试抑制访问检查，不保证所有访问都成功；失败不一定当场抛异常，可能延迟到后续访问操作；实际代码不应默认它一定成功。
8. `getLong(obj)` 不是万能的内存读取函数，必须先确认字段类型。
9. `getDeclaredMethod()` 负责按声明类型查找方法，`invoke()` 负责按反射调用规则传入参数，两者是不同的步骤。`invoke()` 的转换规则不完全等同于源码编译时的调用规则。记住三条：`Integer → int` 是拆箱，`Integer → long` 是拆箱加基本类型拓宽，`Integer → Long` 不能依赖自动转换（通常会因参数类型不匹配抛 `IllegalArgumentException`）。
10. `Method` 和 ART 内部的 `ArtMethod` 不是同一个东西。
11. 反射操作的是运行时的类和成员信息，不是直接读取或修改 DEX 文件。
12. Kotlin 里 `Int::class.java` 是 `int.class`，`Int::class.javaObjectType` 是 `Integer.class`；查找方法时要按声明类型选对。

---

## 后续学习建议

后续学习 Android 反射和 ART 时，建议重点掌握三个问题：

1. `Class` 对象与普通实例对象的区别。
2. `Method` 如何找到目标方法，以及如何正确调用它。
3. Android ART 中 `Method` 与内部方法结构（如 `ArtMethod`）之间的关系。

把这三个问题弄清楚，再去看 ART 内部实现和 Hook 相关内容，会更容易建立清晰的概念边界。

**最值得先动手的练习：**

用 Kotlin 写一个 `Dog` 类，然后分别完成三件事：

1. 用 `Class` 找到这个类（`Dog::class.java` 或 `dog.javaClass`）。
2. 用 `Field` 读取它的一个私有字段。
3. 用 `Method` 调用它的一个私有方法。

每做完一步，停下来问自己：刚才操作的是**类的信息**，还是**某个实例本身**？

把这个问题想清楚，再进入 ART 和 Hook 的学习，概念会稳很多。

**额外建议：**

- 对比 `Int::class.java` 与 `Int::class.javaObjectType`，用 `getDeclaredMethod` 验证两者在查找 `fun foo(x: Int)` 和 `fun foo(x: Int?)` 时的区别。
- 用 Kotlin 属性（`private var age: Int`）和计算属性（`private val age get() = 3`）各试一次 `getDeclaredField("age")`，体会“Kotlin 属性不等于 Java 字段”的含义。
