---
title: grep 速查笔记（安卓逆向实战版）
date: 2026-09-23 10:54:22
top_img: /img/post-covers/grep速查笔记.png
cover: /img/post-covers/grep速查笔记.png
tags:
  - grep
  - Android
  - 逆向
  - 日志分析
categories: 安全研究
---
> 来源：2026-09-23 学习笔记整理
> 适用场景：Android logcat 日志过滤、崩溃定位、代码搜索
> 优先级：**-E、-A、-B、-C、-v、管道 | 是 Android Native 调试最常用的**

---

## 1. 基本语法

```bash
grep [选项] "搜索内容" 文件
```

例如：

```bash
grep "ERROR" log.txt
```

意思：

```
log.txt
  ↓
逐行搜索
  ↓
包含 ERROR 的行
  ↓
输出
```

---

## 2. 最常用参数

| 参数 | 含义 | 示例 |
|---|---|---|
| `-i` | 忽略大小写 | `grep -i error log.txt` |
| `-v` | 排除匹配行 | `grep -v DEBUG log.txt` |
| `-n` | 显示行号 | `grep -n ERROR log.txt` |
| `-c` | 统计匹配行数 | `grep -c ERROR log.txt` |
| `-r` | 递归搜索目录 | `grep -r ERROR ./src` |
| `-R` | 递归搜索，跟随符号链接 | `grep -R ERROR ./src` |
| `-w` | 匹配完整单词 | `grep -w pid log.txt` |
| `-x` | 整行完全匹配 | `grep -x "hello" log.txt` |
| `-F` | 普通字符串匹配，不使用正则 | `grep -F "a+b" log.txt` |
| `-E` | 扩展正则表达式 | `grep -E "SIGSEGV\|SIGABRT" log.txt` |
| `-A N` | 匹配行之后 N 行 | `grep -A 10 ERROR log.txt` |
| `-B N` | 匹配行之前 N 行 | `grep -B 10 ERROR log.txt` |
| `-C N` | 匹配行前后各 N 行 | `grep -C 10 ERROR log.txt` |
| `-o` | 只输出匹配部分 | `grep -o "pid=[0-9]*" log.txt` |
| `-l` | 只显示匹配的文件名 | `grep -l ERROR *.log` |
| `-L` | 显示没有匹配的文件 | `grep -L ERROR *.log` |
| `-q` | 静默，不输出结果 | `grep -q ERROR log.txt` |
| `-s` | 忽略错误信息 | `grep -s ERROR *.log` |

---

## 3. -A / -B / -C 很重要

### -A（After = 后面）

```bash
grep -A 10 "ERROR" log.txt
```

```
ERROR
  ↓
后面10行
```

### -B（Before = 前面）

```bash
grep -B 10 "ERROR" log.txt
```

```
前面10行
  ↓
ERROR
```

### -C（Context = 前后）

```bash
grep -C 10 "ERROR" log.txt
```

```
前面10行
  ↓
ERROR
  ↓
后面10行
```

**记忆口诀**：

```
-A = 后面
-B = 前面
-C = 前后
```

> 这个你调 Android 崩溃日志会经常用。

---

## 4. -i 忽略大小写

```bash
grep -i "error" log.txt
```

下面这些都能匹配：

```
ERROR
Error
error
eRrOr
```

---

## 5. -v 反向过滤

```bash
grep -v "DEBUG" log.txt
```

意思：**不要**包含 DEBUG 的行。

输入：

```
DEBUG xxx
INFO xxx
ERROR xxx
DEBUG xxx
```

结果：

```
INFO xxx
ERROR xxx
```

> 这个在 logcat 非常好用。

---

## 6. -n 显示行号

```bash
grep -n "ERROR" log.txt
```

结果：

```
123:ERROR something
256:ERROR something
```

表示第 123 行、第 256 行。

---

## 7. -c 统计数量

```bash
grep -c "ERROR" log.txt
```

例如输出 `17`，表示找到 17 行。

---

## 8. -w 完整单词

```bash
grep -w "pid" log.txt
```

会匹配：`pid`

不会匹配：`rapid`、`pid123`、`mypid`

---

## 9. -E 多条件搜索（非常重要）

```bash
grep -E "SIGSEGV|SIGABRT|SIGBUS" log.txt
```

意思：匹配 SIGSEGV **或** SIGABRT **或** SIGBUS。

Android Native 崩溃排查：

```bash
adb logcat -d | grep -E "SIGSEGV|SIGABRT|SIGBUS|F DEBUG"
```

---

## 10. 多个关键词

```bash
grep -E "Inject|ptrace|zygote|SIGSEGV"
```

相当于：Inject **或** ptrace **或** zygote **或** SIGSEGV。

---

## 11. -F：不要把内容当正则

你想搜索字面上的 `[ERROR]`：

```bash
# ✗ 错误：[] 会被当成正则表达式
grep "[ERROR]" log.txt

# ✓ 正确：-F = 纯字符串搜索
grep -F "[ERROR]" log.txt
```

---

## 12. -r：搜索整个目录

```bash
grep -r "ptrace" ./src
```

会搜索：

```
./src
├── a.c
├── b.c
├── include/
│   ├── x.h
│   └── y.h
└── test/
    └── test.c
```

所有文件。

---

## 13. -l：只显示文件名

```bash
grep -rl "ptrace" ./src
```

结果：

```
./src/inject.c
./src/trace.c
./src/include/trace.h
```

不会把代码内容打印出来。

---

## 14. -o：只显示匹配内容

```bash
grep -o "SIG[A-Z]*" log.txt
```

可能得到：

```
SIGSEGV
SIGABRT
SIGBUS
```

---

## 15. grep + 管道 |（最应该掌握）

```bash
命令1 | grep "关键词"
```

例如：

```bash
adb logcat | grep "HERMES_HOOK"
```

意思：

```
adb logcat
  ↓ 产生大量日志
|
  ↓ 管道
grep
  ↓ 只留下 HERMES_HOOK
```

---

## 16. grep 可以连续使用

```bash
adb logcat | grep "HERMES_HOOK" | grep "ERROR"
```

相当于：

```
全部 Logcat
  ↓
HERMES_HOOK
  ↓
ERROR
```

最终只留下**同时满足**条件的内容。

---

## 17. grep + grep -v

```bash
adb logcat | grep "HERMES_HOOK" | grep -v "DEBUG"
```

意思：

```
Logcat
  ↓ 只要 HERMES_HOOK
  ↓ 排除 DEBUG
```

---

## 18. grep + head

```bash
grep "ERROR" log.txt | head        # 默认前 10 行
grep "ERROR" log.txt | head -20    # 前 20 行
```

---

## 19. grep + tail

```bash
grep "ERROR" log.txt | tail        # 默认最后 10 行
grep "ERROR" log.txt | tail -20    # 最后 20 行
```

---

## 20. Android Logcat 实战

```bash
# 只看自己的 TAG
adb logcat | grep "HERMES_HOOK"

# 忽略大小写
adb logcat | grep -i "hermes"

# 找 Native 崩溃
adb logcat -d | grep -E "SIGSEGV|SIGABRT|SIGBUS|F DEBUG"

# 找崩溃并看后面30行
adb logcat -d | grep -A30 "F DEBUG"

# 找崩溃并看前后30行
adb logcat -d | grep -C30 "F DEBUG"

# 找 ptrace 相关
adb logcat -d | grep -i "ptrace"

# 多关键词（-E + -i 组合）
adb logcat -d | grep -Ei "ptrace|inject|zygote|usap|sigsegv"

# 排除无关日志
adb logcat -d | grep -v "SystemUI"
```

---

## 21. 正则表达式最基础的几个（grep -E 配合）

| 写法 | 含义 |
|---|---|
| `A\|B` | A 或 B |
| `.` | 任意一个字符 |
| `*` | 前面的字符 0 次或多次 |
| `+` | 前面的字符 1 次或多次 |
| `?` | 前面的字符 0 或 1 次 |
| `[0-9]` | 一个数字 |
| `[a-z]` | 一个小写字母 |
| `[A-Z]` | 一个大写字母 |
| `^` | 行开头 |
| `$` | 行结尾 |

例如：

```bash
grep -E "pid=[0-9]+"
```

匹配：`pid=1234`、`pid=56789`。

---

## 22. 你这条命令完整拆解

```bash
adb shell "logcat -d | grep -A10 'F DEBUG' | tail -20"
```

记成：

```
adb shell
│   └── 在 Android 执行
logcat -d
│   └── 把现有日志倒出来
grep -A10 'F DEBUG'
│   └── 找 F DEBUG + 后10行
tail -20
    └── 最后20行
```

---

---

## 23. 实战：实时监控 Android 崩溃 + Inject 日志

这条命令就是一个实时监控 Android 崩溃 + Inject 日志的过滤器：

```bash
adb logcat -v threadtime | grep -E "Inject|HERMES_HOOK|Fatal signal|SIGSEGV|SIGABRT|backtrace|F DEBUG"
```

拆开看：

```text
adb logcat
↓
实时读取 Logcat
↓
-v threadtime
↓
使用时间/线程格式
↓
|
↓
grep -E
↓
使用扩展正则
↓
只保留指定关键词
```

**① adb logcat**

```bash
adb logcat
```

实时读取日志。

只要 Android 又产生新的日志，就会继续显示。

**② -v threadtime**

```bash
-v threadtime
```

`-v` = 指定 Logcat 输出格式。

threadtime 会显示类似：

```text
09-23 10:35:21.123 1234 5678 I HERMES_HOOK: attach success
```

大概是：

```text
日期 时间 PID TID 等级 TAG
↓     ↓     ↓   ↓   ↓   ↓
09-23 10:35:21.123 1234 5678 I HERMES_HOOK
```

其中：

```text
PID = 进程 ID
TID = 线程 ID
```

对于你现在分析 ptrace / 多线程 / Inject，这个格式比较有用。

**③ |**

```text
|
```

叫管道。

意思：

```text
左边命令的输出
↓
交给
↓
右边命令
```

所以：

```bash
adb logcat | grep ...
```

就是：

```text
Logcat 大量日志
↓
grep
↓
只留下需要的
```

**④ grep -E**

```bash
grep -E
```

`-E` = 使用扩展正则表达式。

这里最重要的是：

```text
A|B|C
```

表示：

```text
A 或 B 或 C
```

所以：

```bash
grep -E "Inject|HERMES_HOOK|SIGSEGV"
```

就是：

```text
包含 Inject
或
包含 HERMES_HOOK
或
包含 SIGSEGV
```

**⑤ 这些关键词分别干什么**

```text
Inject
```

找你的 Inject 相关日志。

```text
HERMES_HOOK
```

找你代码自己的 TAG。

```text
Fatal signal
```

Native 程序发生致命信号。

```text
SIGSEGV
```

常见的非法内存访问。

```text
SIGABRT
```

程序主动 abort()，或者某些运行时错误导致 abort。

```text
backtrace
```

崩溃调用栈。

```text
F DEBUG
```

Android debuggerd/tombstone 产生的崩溃信息。

### 最终效果

比如 Android 出现：

```text
09-23 10:40:01.100 ... I HERMES_HOOK: ptrace attach
09-23 10:40:01.200 ... I HERMES_HOOK: waitpid
09-23 10:40:01.300 ... F DEBUG: *** *** *** *** ***
09-23 10:40:01.301 ... F DEBUG: signal 11 (SIGSEGV)
09-23 10:40:01.302 ... F DEBUG: backtrace:
09-23 10:40:01.303 ... F DEBUG: #00 pc 000000...
```

这些会被保留下来，而大量：

```text
SystemUI
SurfaceFlinger
Audio
Wifi
Bluetooth
其他 App
```

之类的日志会被过滤掉。

### 你现在调 Inject，我建议直接用

```bash
adb logcat -c
adb logcat -v threadtime | grep -E "Inject|HERMES_HOOK|Fatal signal|SIGSEGV|SIGABRT|backtrace|F DEBUG"
```

然后另开一个终端启动 Inject。

这样可以做到：

```text
清空旧日志
↓
启动实时监控
↓
启动 Inject
↓
Inject 正常日志 ──→ HERMES_HOOK
↓
如果崩溃
↓
SIGSEGV / SIGABRT
↓
backtrace
↓
F DEBUG
```

如果你下一步要分析 SIGSEGV + backtrace 中的 pc 000000... 怎么对应到你 Inject 源码具体哪一行，那就是 llvm-addr2line / ndk-stack 的用法。

---

## 24. 优先记这 10 个（Android 逆向够用）

```bash
grep "xxx"              # 基本搜索
grep -i "xxx"           # 忽略大小写
grep -v "xxx"           # 排除
grep -n "xxx"           # 带行号
grep -c "xxx"           # 统计
grep -E "A|B"           # 多条件
grep -A 10 "xxx"        # 匹配行后10行
grep -B 10 "xxx"        # 匹配行前10行
grep -C 10 "xxx"        # 匹配行前后10行
grep -r "xxx" ./目录    # 递归搜目录
```

其中 **-E、-A、-B、-C、-v、管道 |** 是目前 Android Native 调试最常用的。
