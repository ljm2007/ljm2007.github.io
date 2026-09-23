---
title: 从手机拖 so 到 Mac —— 完整操作手册（以抖音为例）
date: 2026-09-23 11:26:18
top_img: /img/post-covers/从手机拖so到Mac——完整操作手册.png
cover: /img/post-covers/从手机拖so到Mac——完整操作手册.png
tags:
  - Android
  - 逆向
  - adb
  - IDA
categories: 安全研究
---
> 场景：逆向分析时，需要把 App 的 native 库（.so）从手机里拷到 Mac 上，用 IDA / llvm 工具分析。
> 环境：Mac + root 安卓真机（USB 连接）+ adb
> 日期：2026-09-23

---

## 完整流程（5 步）

```bash
# ① 确认手机连上了（要看到 device，不是 unauthorized / 空）
adb devices
# 期望输出：
# List of devices attached
# 94NY19J44	device

# ② 找到 App 的安装路径（得到 /data/app/~~xxx==/包名-yyy==/base.apk）
adb shell pm path com.ss.android.ugc.aweme
# 期望输出：
# package:/data/app/~~3h8ceKVYeibGYb7x-yu3XQ==/com.ss.android.ugc.aweme-02imtyubQ4hf4F2xW3RNOA==/base.apk
#           └────── 记下这一段：/data/app/~~3h8ceKVYeibGYb7x-yu3XQ==/com.ss.android.ugc.aweme-02imtyubQ4hf4F2xW3RNOA==

# ③ 用 root 把 so 拷出来（su -c cat 重定向到本地文件）
#    把 <上面那段路径> 替换进去，后面接 /lib/arm64/<要拖的so名>
adb shell "su -c cat /data/app/~~3h8ceKVYeibGYb7x-yu3XQ==/com.ss.android.ugc.aweme-02imtyubQ4hf4F2xW3RNOA==/lib/arm64/libsscronet.so" > /tmp/libsscronet.so

# ④ 校验文件（两件事：大小不是0 + 确实是ARM64的so）
ls -la /tmp/libsscronet.so
file /tmp/libsscronet.so
# 期望输出：
# -rw-r--r--  1 ljm2007  wheel  5137648  /tmp/libsscronet.so        ← 大小 5MB 左右，不是 0
# /tmp/libsscronet.so: ELF 64-bit LSB shared object, ARM aarch64, dynamically linked, stripped

# ⑤ 挪到顺手的地方（比如桌面目录），然后拖进 IDA
mkdir -p ~/Desktop/ida_targets
cp /tmp/libsscronet.so ~/Desktop/ida_targets/
open ~/Desktop/ida_targets
```

---

## 通用模板（换 App / 换 so 时照抄）

```bash
# ② 的模板：换包名
adb shell pm path <包名>

# ③ 的模板：换路径 + 换 so 文件名
adb shell "su -c cat <步骤②得到的路径>/lib/arm64/<so文件名>" > /tmp/<so文件名>
```

### 常用 App 的 so（本项目相关）

| App | 包名 | 常分析 so |
|---|---|---|
| 抖音正式版 | `com.ss.android.ugc.aweme` | libsscronet.so、libttboringssl.so、libttcrypto.so |
| 抖音极速版 | `com.ss.android.ugc.aweme.lite` | 同上 |

极速版实测路径（DHCP/升级后路径会变，**每次都要用步骤②现查**）：

```bash
adb shell "su -c cat /data/app/~~tvv0UtfXYKTHnFGsLGHZMQ==/com.ss.android.ugc.aweme.lite-Uj_d1zOzm-kB2UVblkZSUQ==/lib/arm64/libsscronet.so" > /tmp/libsscronet.so
```

---

## 常见坑

| 现象 | 原因 | 解法 |
|---|---|---|
| `adb devices` 列表为空 | 没插 USB / 手机没开调试 | 插线 + 手机上允许 USB 调试 |
| `error: no devices/emulators found` | adb 没认到设备 | 重插，或 `adb kill-server && adb start-server` |
| 拖出来是 **0 字节** | 直接 `adb pull` 被 /data/app 权限拒了 | 必须用 `su -c cat` + 重定向（本文方法） |
| `su: not found` | 手机没 root | 这台 crosshatch 已 root（Magisk），不会遇到 |
| `file` 显示 x86-64 | 拖错了文件（拖了 Mac 本地同名文件） | 用 `file` 确认是 `ARM aarch64` |
| so 路径报错 `No such file` | App 升级后 `~~xxx==` 路径变了 | **别用旧路径，每次跑步骤②现查** |
| `/tmp` 里文件消失 | Mac 重启会清空 /tmp | 长期要用的放 `~/Desktop/ida_targets/` 或项目目录 |

---

## 拖到 Mac 之后能干什么

| 工具 | 用途 | 命令/操作 |
|---|---|---|
| **IDA Pro (GUI)** | 反汇编/反编译，看崩溃地址的代码 | 拖进去 → G → 输地址（如 `0x2C8DB8`） |
| `llvm-nm -D` | 列导出/导入符号 | `llvm-nm -D --undefined-only /tmp/libsscronet.so` |
| `llvm-readelf -d` | 看动态依赖（NEEDED） | `llvm-readelf -d /tmp/libsscronet.so` |
| `llvm-addr2line` | 地址→函数名（需有符号，stripped 会得 `??`） | `llvm-addr2line -e /tmp/libsscronet.so -f -C 0x2C8DB8` |
| `llvm-objdump -d` | 反汇编指定区间 | `llvm-objdump -d --start-address=0x2C8D80 --stop-address=0x2C8DF0 /tmp/libsscronet.so` |

> llvm 工具位置：`~/Library/Android/sdk/ndk/29.0.14206865/toolchains/llvm/prebuilt/darwin-x86_64/bin/`
> 先 `export PATH` 加进 PATH，或写全路径。

---

## 本次案例的完整真实命令（2026-09-23 实测跑通）

```bash
adb devices
adb shell pm path com.ss.android.ugc.aweme
adb shell "su -c cat /data/app/~~3h8ceKVYeibGYb7x-yu3XQ==/com.ss.android.ugc.aweme-02imtyubQ4hf4F2xW3RNOA==/lib/arm64/libsscronet.so" > /tmp/libsscronet.so
ls -la /tmp/libsscronet.so
file /tmp/libsscronet.so
mkdir -p ~/Desktop/ida_targets && cp /tmp/libsscronet.so ~/Desktop/ida_targets/ && open ~/Desktop/ida_targets
```

结果：5137648 字节，`ELF 64-bit LSB shared object, ARM aarch64 ... stripped`，BuildID `60008d56...` 与崩溃日志一致 ✅
