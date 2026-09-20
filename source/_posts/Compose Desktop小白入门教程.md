---
title: ComposeDesktop小白入门教程从零到打包
date: 2024-02-26 16:50:56
tags: Kotlin
categories: Kotlin
permalink: 2024/02/26/Compose Desktop小白入门教程/
---
## 前言

作为一名安卓小白，大部分时间都在忙着学安卓，电脑软件的开发可谓是一点没接触过，在学习了Compose之后，发现谷歌近年新推出的Compose Desktop技术可以轻松在电脑上实现好看的图形界面，那么下面我们就来写个小程序练练手！

## 一、创建Compose Multipalform项目

新建一个项目，命名为CoolBookstore（酷欧书店），下面的single和multiple代表是否要构建多平台包，如果选multiple就会额外生成一个Android平台包，这里我们只需要构建一个单平台包即可，Platform默认桌面端。组随便填就行，工件默认和项目名一致。我一直用的是JDK17，听说15有bug，总之先这么选吧！

![img](https://s2.loli.net/2024/02/26/yrfz2hPsEZKI9l3.png)

## 二、运行你的第一个ComposeDesktop程序

### 1.主程序入口

![img](https://s2.loli.net/2024/02/26/RVgZd9c8C7OB2is.png)

等待环境下载好之后，这个就是默认生成的项目模板了，代码很简单，在main程序入口中创建了一个窗口，再看App()函数，你会发现里面有一个MaterialTheme函数，是不是和ComponentActivity中的setContent{ xxxTheme }很像？里面放了一个按钮，它的语法可以说和安卓一模一样！

### 2.运行

点击右上角小三角，运行一下吧！

![img](https://s2.loli.net/2024/02/26/pFNixMST7CKbLze.png)

这样你就运行了你的第一个Compose Desktop程序！现在只有简单的一个按钮，很快我们就会给它添加充足的数据！那么接下来，先从这个窗口开始下手吧！

```kotlin
fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "酷欧书店",
        state = WindowState(width = 400.dp, height = 700.dp)
    ) {
        App()
    }
}
```

这里简单加了两个参数，把窗口的标题改成了我们程序的名称，然后调整的窗口的高宽，再次运行一下吧！

![img](https://s2.loli.net/2024/02/26/1GW7OZmg9RY8tzK.png)

熟悉的感觉是不是来了，哈哈哈！

### 3.添加运行配置

为了让项目更加可读，我们可能会创建很多.kt文件，现在右上角的运行配置还是“当前文件”，创建一个运行配置，那样就可以在别的类中随时调试程序了！右键点击main.kt，找到创建运行配置

![img](https://s2.loli.net/2024/02/26/VhoWE4Y9CfKT6cJ.png)

这样就弄好了！现在正式开始编写页面！

## 三、编写界面

### 1.实现界面的切换

由于Compose Desktop尚不成熟，有些安卓中提供的库还没发布，比如导航，直接依赖过来会提示添加失败，见下图

![img](https://s2.loli.net/2024/02/26/7l1yHwdvLDQWVTm.png)

Compose强调分离解耦，我们不能像Activity那样随时跳转，往后页面越多越复杂，所以拥有一个导航器是非常重要的。既然导航用不了的话，我们就自己实现一个吧，其实很简单，见下方代码：

```kotlin
var currentPage = mutableStateOf("FirstBookListPage")

@Composable
fun App() {
    MaterialTheme {
        when(currentPage.value){
            "FirstBookListPage" -> FirstBookListPage()
            "SecondBookInfoPage" -> SecondBookInfoPage()
        }
    }
}
```

使用mutableStateOf()我们可以定义顶级变量保存一个状态，通过改变这个状态的值引发Compose的重组，这样就实现了界面的切换。

### 2.首页书籍列表页面的编写

为了充分发挥Compose组合的优势，我建议大家可以多新建几个.kt文件（Composable函数属于顶层函数，不需要建类！）存放Composable函数，方便阅读与维护！

FirstBookListPage是首页书籍列表

SecondBookInfoPage是点进去后的书籍详细信息

![img](https://s2.loli.net/2024/02/26/wqLY21GQzuDTBSb.png)

文件创好了，那么话不多说，上代码！

Book.kt ，FirstBookListPage.kt

```kotlin
data class Book(
    val title : String = "",
    val content: String = "",
    val price: String = "",
    val image: String="",
)
```

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun FirstBookListPage(){
    val bookList = mutableListOf(
        Book("【官方正版】第一行代码-Android",
            "【旗舰店正版】第一行代码 Android 第3版 郭霖著 android10 开发入门到精通 android studio",
            "￥49.5",
            "https://pic4.zhimg.com/80/v2-75c79dfc1bef40f0737d0d340fc09af7_720w.webp"
        ),
    )
    val content = StringBuffer()
    repeat((1..5).random()){
        content.append("其他书籍")
    }
    repeat(9){
        bookList.add(
            Book("其他书籍",
                content.toString(),
                "￥${(30..80).random()}.${(0..10).random()}",
            ),
        )
    }
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(title = {
                Text(text = "酷欧书店")
            })
        },
    ){
        LazyVerticalGrid(cells = GridCells.Fixed(2), modifier = Modifier.background(Color.LightGray).fillMaxSize()) {
            bookList.forEach {
                item{
                    BookItem(it)
                }
            }
        }
    }
}
```

我们先创建了一个Book数据类，存放了书籍的信息（标题，内容，价格，图片）。

在FirstBookListPage函数中，构建了一个书籍列表，并放了10本书进去作为数据集。使用Scaffold脚手架我们可以轻松实现MD风格的界面，就像在安卓开发中一样，放入Topbar与内容。

内容我选择用表格布局（实验性的组件，需要添加注解）构建一个列表，将这个列表设置为两列，注意Lazy相关的布局中只能存放item函数，把你想构建的卡片布局都放在item函数中就可以了。

下面我们编写卡片布局！

```kotlin
@Composable
fun BookItem(book:Book){
    Card(modifier = Modifier.height(300.dp).padding(5.dp).clickable {
        currentPage.value = "SecondBookInfoPage"
        bookInfo.value = book
    }) {  Column(Modifier.fillMaxSize().padding(5.dp)){
        if(book.image.isNotEmpty()){
            ImageAsyncImageUrl(book.image,
                imageCallback = ImageCallback {
                    Image(modifier= Modifier.fillMaxWidth().weight(2F).padding(5.dp),
                        painter = it, contentDescription = null)
                })
        }else{
            Image(modifier= Modifier.fillMaxWidth().weight(2F).padding(5.dp),
                painter = painterResource("android.jpg"), contentDescription = null)
        }
        Column(modifier = Modifier.fillMaxWidth().weight(1F).padding(5.dp)
        ) {
            Text(book.title)
            Text(book.price,
                modifier = Modifier.padding(top = 10.dp),
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold,
                color = Color.Red
            )
        }
    } }
}
```

很好理解，一个卡片布局，上面图片下面文字，点击卡片就把Book值传给书籍详细信息页面的变量中，书籍详细信息页面的变量和一开始定义导航的那个变量差不多，稍后就会说到！

图片分为网络加载和本地加载，本地加载只需要把图片放在kotlin同目录下的resource目录中即可

![img](https://s2.loli.net/2024/02/26/iQ5I1XNhV2TSDJt.png)

刷文章时候刷到了这个第三方网络加载库，非常方便，需要在build.gradle.kts中的sourceSets中引用：

```kotlin
implementation("io.github.succlz123:compose-imageloader-desktop:0.0.2")
```

完整代码：

```kotlin
sourceSets {
        val jvmMain by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
//下面这个就是依赖库
                implementation("io.github.succlz123:compose-imageloader-desktop:0.0.2")
            }
        }
        val jvmTest by getting
    }
```

下面是这个网络图片加载库的使用方法，只需要将Image函数添加在ImageAsyncImageUrl的imageCallback参数中即可：

```plaintext
ImageAsyncImageUrl(要加载的图片URL,
                imageCallback = ImageCallback {
                    Image(painter = it, contentDescription = null)
                })
```

这样我们就编写完书籍列表的页面了，运行后就是这个样子：

![img](https://s2.loli.net/2024/02/26/AN4cBLq6ZCHRhlJ.gif)

这里用了随机数生成价格和内容，避免数据重复

### 3.书籍详细信息页面的编写

SecondBookInfoPage.kt

```plaintext
var bookInfo = mutableStateOf(Book())
```

创建一个bookInfo变量，用来存储书籍列表传过来的信息数据，就像第一行代码中水果列表的那个demo一样，哈哈哈

```kotlin
@Composable
fun SecondBookInfoPage() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(title = {
                Text(text = "书本详情页面")
            }, navigationIcon = {
                Icon(
                    imageVector = Icons.Default.KeyboardArrowLeft,
                    contentDescription = null,
                    modifier = Modifier.fillMaxHeight().width(50.dp).clickable {
                        currentPage.value = "FirstBookListPage"
                    }
                )
            })
        },
        bottomBar = {
            BottomAppBar(backgroundColor = Color.White, modifier = Modifier.height(50.dp)) {
                Box(modifier = Modifier.fillMaxSize()) {
                    Button(modifier = Modifier.align(Alignment.BottomEnd).padding(5.dp), onClick = {}) {
                        Text("下单")
                    }
                }
            }
        }
    ) {
        BookInfo()
    }
}
```

这个没啥可说的，依然是脚手架，标题栏底部栏和内容，唯一要注意的是，底部栏会挡住内容，所以这里设置为50dp，方便一会给内容留出下方的空间。接下来该写书籍的详细信息了：

```kotlin
@Composable
fun BookInfo() {
    Column(
        Modifier.fillMaxSize()
            .padding(bottom = 50.dp)
            .verticalScroll(rememberScrollState())
            .background(Color.LightGray)
    ) {
        if (bookInfo.value.image.isNotEmpty()) {
            ImageAsyncImageUrl(bookInfo.value.image,
                imageCallback = ImageCallback {
                    Image(
                        modifier = Modifier.fillMaxWidth(),
                        painter = it, contentDescription = null
                    )
                })
        } else {
            Image(
                modifier = Modifier.fillMaxWidth(),
                painter = painterResource("android.jpg"), contentDescription = null
            )
        }
        Card(modifier = Modifier.fillMaxWidth().padding(10.dp), elevation = 0.dp, shape = RoundedCornerShape(10.dp)) {
            Text(
                bookInfo.value.content,
                fontSize = 24.sp, fontWeight = FontWeight.Bold, modifier = Modifier.padding(15.dp),
                maxLines = 2,
                overflow = TextOverflow.Ellipsis
            )
        }
        if (bookInfo.value.title.contains("第一行代码")) {
            Card(
                modifier = Modifier.fillMaxWidth().padding(10.dp),
                elevation = 0.dp,
                shape = RoundedCornerShape(10.dp)
            ) {
                Column(Modifier.fillMaxSize().padding(5.dp)) {
                    for (i in 1..100) {
                        Text("用户$i", fontSize = 14.sp, modifier = Modifier.padding(10.dp))
                        Text(
                            "${i}人血书第一行代码第四版！（$i / 100)",
                            fontSize = 16.sp, modifier = Modifier.padding(10.dp)
                        )
                        if (i != 100) Spacer(
                            modifier = Modifier.fillMaxWidth().height(1.dp).background(Color.LightGray)
                        )
                    }
                }
            }
        }
    }
}
```

这次不是列表，而是布局，只需要将lazy布局换成普通的Column，用modifier的.verticalScroll(rememberScrollState())方法设置可垂直滑动即可！设置Text函数的maxline属性将文本显示为最多两行，默认是截取，设置overflow属性改为省略号。后面的代码就不说了，哈哈哈哈！看效果吧！

![img](https://s2.loli.net/2024/02/26/2CsSaAuML5TUWo4.gif)

## 四、项目的打包

点击右上角Gradle，这里有很多打包的方案

![img](https://s2.loli.net/2024/02/26/plkMhHFU9uC6LOi.png)

![img](https://s2.loli.net/2024/02/26/6a9YMCxbrOVoJRc.png)

这个是打包jar的，打包完后会打印输出路径，我的是：

E:\CoolBookstore\CoolBookstore\build\compose\jars

![img](https://s2.loli.net/2024/02/26/8DEmGwfQexbU9Bl.png)

这个是打包Msi可执行程序的，运行后会弹出安装界面，Msi和exe都是可以在windows上执行的程序。

![img](https://s2.loli.net/2024/02/26/ZpibJ7nv4s8TcGV.png)

在build.gradle.kts中把上面那个Msi改成Exe就可以打包exe程序了

macOS（ .dmg 、 .pkg ）Windows （ .exe 、 .msi ）Linux （ .deb 、 .rpm ）

![img](https://s2.loli.net/2024/02/26/6dIwmGBPD7AgiYF.png)

这个是我最喜欢的，会生成一个文件夹，可以直接运行。

![img](https://s2.loli.net/2024/02/26/4gkVLRSEJn28XvK.png)

如果你的项目用上了jar包，必须要在build.gradle.kts的依赖配置中添加这行，否则会爆ClassNotFound的异常，上周因为作业用到sql库，被这个问题卡了一个多小时..

```plaintext
// 配置lib包路径
                implementation(files("lib/你的jar包"))
```

## 总结

第一次写文章，写的不是很好，代码也写得很简单qwq，本意就是分享一下这个有趣的开发过程，compose desktop属于刚起步阶段，还有很大优化空间，但是作为一个懒人能像开发安卓端一样去开发电脑端，简直不要太舒服，希望后续它能带来更多惊喜！
