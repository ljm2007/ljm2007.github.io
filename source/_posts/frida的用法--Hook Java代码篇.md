---
title: frida的用法–Hook Java代码篇
date: 2023-02-12 13:47:04
tags: frida
categories: frida
top_img: https://bit.ly/3WkSvF9
cover: https://bit.ly/3WkSvF9
---

------

# [frida的用法--Hook Java代码篇](https://www.cnblogs.com/luoyesiqiu/p/10718997.html)

frida是一款方便并且易用的跨平台Hook工具，使用它不仅可以Hook Java写的应用程序，而且还可以Hook原生的应用程序。

# 1. 准备

frida分客户端环境和服务端环境。在客户端我们可以编写Python代码，用于连接远程设备，提交要注入的代码到远程，接受服务端的发来的消息等。在服务端，我们需要用Javascript代码注入到目标进程，操作内存数据，给客户端发送消息等操作。我们也可以把客户端理解成控制端，服务端理解成被控端。
假如我们要用PC来对Android设备上的某个进程进行操作，那么PC就是客户端，而Android设备就是服务端。

## 1.1 准备frida服务端环境[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#11-准备frida服务端环境)

本文，服务端在Android平台测试。服务端环境准备步骤如下：

1. 根据自己的平台下载frida服务端并解压
   https://github.com/frida/frida/releases
   [![frida_server](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_frida_release.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_frida_release.png)
2. 执行以下命令将服务端推到手机的/data/local/tmp目录
   `adb push frida-server /data/local/tmp/frida-server`
3. 执行以下命令修改frida-server文件权限
   `adb shell chmod 777 /data/local/tmp/frida-server`

> 注：Windows系统执行命令可以在CMD中进行；Linux和MacOS执行命令可以在终端中进行。adb是Android一个调试工具，具体安装方法不是本文的重点。

## 1.2 准备客户端环境[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#12-准备客户端环境)

在PC上安装Python的运行环境，安装完成后执行下面的命令安装frida

```
pip install frida-tools
```

## 1.3 客户端命令参数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#13-客户端命令参数)

下面是frida客户端命令行的参数帮助

```vhdl
Usage: frida [options] target

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -D ID, --device=ID    connect to device with the given ID
  -U, --usb             connect to USB device
  -R, --remote          connect to remote frida-server
  -H HOST, --host=HOST  connect to remote frida-server on HOST
  -f FILE, --file=FILE  spawn FILE
  -n NAME, --attach-name=NAME
                        attach to NAME
  -p PID, --attach-pid=PID
                        attach to PID
  --debug               enable the Node.js compatible script debugger
  --enable-jit          enable JIT
  -l SCRIPT, --load=SCRIPT
                        load SCRIPT
  -c CODESHARE_URI, --codeshare=CODESHARE_URI
                        load CODESHARE_URI
  -e CODE, --eval=CODE  evaluate CODE
  -q                    quiet mode (no prompt) and quit after -l and -e
  --no-pause            automatically start main thread after startup
  -o LOGFILE, --output=LOGFILE
                        output to log file
```

### 1.3.1 将一个脚本注入到Android目标进程[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#131-将一个脚本注入到android目标进程)

```
frida -U -l myhook.js com.xxx.xxxx
```

参数解释：

- -U 指定对USB设备操作
- -l 指定加载一个Javascript脚本
- 最后指定一个进程名，如果想指定进程pid,用`-p`选项。正在运行的进程可以用`frida-ps -U`命令查看

### 1.3.2 重启一个Android进程并注入脚本[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#132-重启一个android进程并注入脚本)

```
frida -U -l myhook.js -f com.xxx.xxxx --no-pause
```

参数解释：

- -f 指定一个进程，重启它并注入脚本
- --no-pause 自动运行程序

这种注入脚本的方法，常用于hook在App就启动期就执行的函数。

> frida运行过程中，执行`%resume`重新注入，执行`%reload`来重新加载脚本；执行`exit`结束脚本注入

# 2. Hook Java方法

## 2.1 载入类[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#21-载入类)

Java.use方法用于加载一个Java类，相当于Java中的`Class.forName()`。比如要加载一个String类：

```
var StringClass = Java.use("java.lang.String");
```

加载内部类：

```
var MyClass_InnerClass = Java.use("com.luoyesiqiu.MyClass$InnerClass");
```

其中InnerClass是MyClass的内部类

## 2.2 修改函数的实现[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#22-修改函数的实现)

修改一个函数的实现是逆向调试中相当有用的。修改一个函数的实现后，如果这个函数被调用，我们的Javascript代码里的函数实现也会被调用。

### 2.2.1 函数参数类型表示[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#221-函数参数类型表示)

不同的参数类型都有自己的表示方法

1. 对于基本类型，直接用它在Java中的表示方法就可以了，不用改变，例如：

- int
- short
- char
- byte
- boolean
- float
- double
- long

1. 基本类型数组，用左中括号接上基本类型的缩写

基本类型缩写表示表：

| 基本类型 | 缩写 |
| -------- | ---- |
| boolean  | Z    |
| byte     | B    |
| char     | C    |
| double   | D    |
| float    | F    |
| int      | I    |
| long     | J    |
| short    | S    |

例如：`int[]`类型，在重载时要写成`[I`

1. 任意类，直接写完整类名即可

例如：`java.lang.String`

1. 对象数组，用左中括号接上完整类名再接上分号

例如：`[java.lang.String;`

### 2.2.2 带参数的构造函数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#222-带参数的构造函数)

修改参数为byte[]类型的构造函数的实现

```php
ClassName.$init.overload('[B').implementation=function(param){
    //do something
}
```

> 注：ClassName是使用Java.use定义的类;param是可以在函数体中访问的参数

修改多参数的构造函数的实现

```php
ClassName.$init.overload('[B','int','int').implementation=function(param1,param2,param3){
    //do something
}
```

### 2.2.3 无参数构造函数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#223-无参数构造函数)

```php
ClassName.$init.overload().implementation=function(){
    //do something
}
```

调用原构造函数

```kotlin
ClassName.$init.overload().implementation=function(){
    //do something
    this.$init();
    //do something
}
```

> 注意：当构造函数(函数)有多种重载形式，比如一个类中有两个形式的func：`void func()`和`void func(int)`，要加上overload来对函数进行重载，否则可以省略overload

### 2.2.4 一般函数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#224-一般函数)

修改函数名为func，参数为byte[]类型的函数的实现

```javascript
ClassName.func.overload('[B').implementation=function(param){
    //do something
    //return ...
}
```

### 2.2.5 无参数的函数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#225-无参数的函数)

```delphi
ClassName.func.overload().implementation=function(){
    //do something
}
```

> 注： 在修改函数实现时，如果原函数有返回值，那么我们在实现时也要返回合适的值

```javascript
ClassName.func.overload().implementation=function(){
    //do something
    return this.func();
}
```

# 3. 调用函数

和Java一样，创建类实例就是调用构造函数，而在这里用`$new`表示一个构造函数。

```php
var ClassName=Java.use("com.luoye.test.ClassName");
var instance = ClassName.$new();
```

实例化以后调用其他函数

```go
var ClassName=Java.use("com.luoye.test.ClassName");
var instance = ClassName.$new();
instance.func();
```

# 4. 字段操作

字段赋值和读取要在字段名后加`.value`，假设有这样的一个类：

```java
package com.luoyesiqiu.app;
public class Person{
    private String name;
    private int age;
}
```

写个脚本操作Person类的name字段和age字段：

```php
var person_class = Java.use("com.luoyesiqiu.app.Person");
//实例化Person类
var person_class_instance = person_class.$new();
//给name字段赋值
person_class_instance.name.value = "luoyesiqiu";
//给age字段赋值
person_class_instance.age.value = 18;
//输出name字段和age字段的值
console.log("name = ",person_class_instance.name.value, "," ,"age = " ,person_class_instance.age.value);
```

输出：

```ini
name =  luoyesiqiu , age =  18
```

# 5. 类型转换

用`Java.cast`方法来对一个对象进行类型转换，如将`variable`转换成`java.lang.String`：

```php
var StringClass=Java.use("java.lang.String");
var NewTypeClass=Java.cast(variable,StringClass);
```

# 6. Java.available字段

这个字段标记Java虚拟机（例如： Dalvik 或者 ART）是否已加载, 操作Java任何东西之前，要确认这个值是否为true

# 7. Java.perform方法

Java.perform(fn)在Javascript代码成功被附加到目标进程时调用，我们核心的代码要在里面写。格式：

```javascript
Java.perform(function(){
//do something...
});
```

# 8. 实例讲解

有了以上的基础知识，我们就可以进行编写代码了

## 8.1 修改返回值[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#81-修改返回值)

### 8.1.1 场景[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#811-场景)

假设有以下的程序，给isExcellent方法传入两个值，通过计算，返回一个布尔值，表示是否优秀。默认情况下，它是只会显示`是否优秀：false`的，因为我们默认传入的数很小:

[![exp1_before](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_before_hook2.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_before_hook2.png)

```scala
public class MainActivity extends AppCompatActivity {
    private  String TAG="Crackme";
    private TextView textView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView =findViewById(R.id.tv);
        textView.setText("是否优秀："+isExcellent(46,54));
    }

    private  boolean isExcellent(int chinese, int math){
        if( chinese + math >=180){
            return true;
        }
        else{
            return false;
        }
    }

}
```

我们编写一个脚本来Hook isExcellent函数，使它返回true，显示为`是否优秀：true`

对于这种简单的场景，直接修改返回值就可以了，因为只有结果是重要的。

### 8.1.2 代码[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#812-代码)

想直接返回结果很简单，直接在匿名方法里return即可。

```php
if(Java.available){
    Java.perform(function(){
        var MainActivity = Java.use("com.luoyesiqiu.crackme.MainActivity");
        MainActivity.isExcellent.implementation=function(){
            return true;        
        }
    });

}
```

- 将上面的代码保存为：`exp1.js`
- 执行`adb shell 'su -c /data/local/tmp/frida-server'`启动服务端
- 运行目标App
- 执行`frida -U -l exp1.js com.luoyesiqiu.crackme`注入代码
- 按返回键返回桌面，再重新打开App,发现达到预期
- 在命令行输入`exit`，回车，停止注入代码

[![exp1_after](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_after_hook2.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_after_hook2.png)

> 注：这里为什么要打开两次App？第一打开是为了让frida能够找到进程，第二次打开是为了验证结果，即使Hook成功了，界面是有缓存的，并不能实时显示Hook结果，所以需要重新打开App

## 8.2 修改参数[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#82-修改参数)

### 8.2.1 场景[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#821-场景)

假设有以下场景，isExcellent除了返回是否优秀以外，方法的内部还把分数打印出来。

[![exp2_before](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_before_hook.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_before_hook.png)

```scala
public class MainActivity extends AppCompatActivity {
    private  String TAG="Crackme";
    private TextView textView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView =findViewById(R.id.tv);
        textView.append("是否优秀："+isExcellent(46,54)+"\n");
    }

    private  boolean isExcellent(int chinese, int math){
        textView.append("语文+数学总分："+(chinese+math)+"\n");
        if( chinese + math >=180){
            return true;
        }
        else{
            return false;
        }
    }
}
```

这种情况下我们不可能只返回是否优秀吧，显示的总分很低，但是却返回优秀，是很尴尬的...所以我们要修改isExcellent方法的参数，使其通过计算打印和返回合理的值。

### 8.2.2 代码[#](https://www.cnblogs.com/luoyesiqiu/p/10718997.html#822-代码)

```javascript
if(Java.available){
    Java.perform(function(){
        var MainActivity = Java.use("com.luoyesiqiu.crackme.MainActivity");
        MainActivity.isExcellent.overload("int","int").implementation=function(chinese,math){
            return this.isExcellent(95,96);      
        }
    });

}
```

上面的代码，通过overload方法重载参数，修改isExcellent方法实现，并在实现函数里调用原来的方法，得到新的返回值

- 将上面的代码保存为：`exp2.js`
- 执行`adb shell 'su -c /data/local/tmp/frida-server'`启动服务端（如果上面启动的服务端还开着可省略这一步）
- 运行目标App
- 执行`frida -U -l exp2.js com.luoyesiqiu.crackme`注入代码
- 按返回键，再重新打开App,发现达到预期
- 在命令行输入`exit`，回车，停止注入代码

[![exp2_after](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_after_hook.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1445698/o_after_hook.png)

# 9. 配合Python脚本注入

在本文刚开始的时候说到，我们可以编写Python代码来配合Javascript代码注入。下面我们来看看，怎么使用，先看一段代码：

```python
# -*- coding: UTF-8 -*-

import frida, sys

jscode = """
if(Java.available){
    Java.perform(function(){
        var MainActivity = Java.use("com.luoyesiqiu.crackme.MainActivity");
        MainActivity.isExcellent.overload("int","int").implementation=function(chinese,math){
            console.log("[javascript] isExcellent be called.");
            send("isExcellent be called.");
            return this.isExcellent(95,96);      
        }
    });

}
"""

def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)
pass

# 查找USB设备并附加到目标进程
session = frida.get_usb_device().attach('com.luoyesiqiu.crackme')
# 在目标进程里创建脚本
script = session.create_script(jscode)
# 注册消息回调
script.on('message', on_message)
print('[*] Start attach')
# 加载创建好的javascript脚本
script.load()
# 读取系统输入
sys.stdin.read()
```

- 将上面的代码，保存为`exp3.py`
- 执行`adb shell 'su -c /data/local/tmp/frida-server'`启动服务端（如果上面启动的服务端还开着可省略这一步）
- 运行目标App
- 执行`python exp3.py`注入代码
- 按返回键，再重新打开App,发现达到预期
- 按`Ctrl+C`停止脚本和停止注入代码

上面是一段Python代码，我们来分析它的步骤：

1. 通过调用`frida.get_usb_device()`方法来得到一个连接中的USB设备（Device类）实例
2. 调用Device类的`attach()`方法来附加到目标进程并得到一个会话（Session类）实例，该方法有一个参数，参数是需要注入的进程名或者进程pid。如果需要Hook的代码在App的启动期执行，那么在调用attach方法前需要先调用Device类的`spawn()`方法，这个方法也有一个参数，参数是进程名，该方法调用后会重启对应的进程，并返回新的进程pid。得到新的进程pid后，我们可以将这个进程pid传递给`attach()`方法来实现附加。
3. 接着调用Session类的`create_script()`方法创建一个脚本，传入需要注入的javascript代码并得到Script类实例
4. 调用Script类的`on()`方法添加一个消息回调，第一个参数是信号名，乖乖传入`message`就行，第二个是回调函数
5. 最后调用Script类的`load()`方法来加载刚才创建的脚本。

> 注：如果想在javascript输出日志，可以调用`console.log()`方法。如果想给客户端发送消息，可以在javascript代码里调用`send()`方法，并在客户端Python代码里注册一个消息回调来接收服务端发来的消息。

可以看到，结合python代码，使注入更加的灵活了。如果想看Python端frida模块的代码，可以访问：https://github.com/frida/frida-python/blob/master/frida/core.py

# 10. 参考

- https://www.frida.re/docs/home/
- https://github.com/esofar/cnblogs-theme-silence)

   