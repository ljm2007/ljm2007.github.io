---
title: 某音正式版 TLS Bypass —— cronet 崩溃排雷全记录
date: 2026-09-24 20:00:00
top_img: /img/post-covers/某音正式版TLSBypass——cronet崩溃排雷全记录.png
cover: /img/post-covers/某音正式版TLSBypass——cronet崩溃排雷全记录.png
tags:
  - cronet
  - Android逆向
  - ShadowHook
  - IDA
  - BoringSSL
categories: 安全研究
---
> 日期：2026-09-24
> 目标 App：某音正式版 `com.ss.android.ugc.aweme`
> 设备：Pixel 3 XL（crosshatch, Android 12, root, USB `94NY19J44`）
> 分析对象：`libsscronet.so`（5.1 MB, ELF ARM64, stripped, BuildId `60008d56c237cb8281c067b6b0d744eca846b5ea`）
> 工具链：CLion（CMakePresets 编译）+ ShadowHook（inline hook）+ IDA Pro 9.4 + ida-pro-mcp + adb/logcat

---

## 0. 一句话结论

**「伪造证书让 boringssl 放行」和「cronet 内部状态机的一致性」之间的矛盾，是这次崩溃的根因。**
先后尝试「改返回值（裁决官）」「堵证据袋（状态断言）」都只能挪走崩点；最终用
**「拆引线（NOP 掉跳进自爆区的跳转）+ 铺平雷区（NOP 掉 BRK/HLT）」** 物理封死自爆路径，
存活时间从 **4~5 秒 → 8 分钟以上**（无崩溃）。

---

## 1. 背景：为什么要 hook cronet

某音的 TLS 库结构（本日再次确认）：

```
libsscronet.so   (cronet 主体，NEEDED libttboringssl.so)
   ├── libttboringssl.so   ← BoringSSL（TLS 实现，SSL_set_verify / custom_verify 在这里）
   ├── libttcrypto.so      ← 密码学（X509_digest 在这里）
   ├── libnpth.so          ← 字节自研线程库（信号处理器，会接管致命信号并 tgkill）
   └── libjato.so          ← JIT/加速库
```

之前的 7 颗钩全部打在 `libttboringssl.so` / `libttcrypto.so` 上（标准 SSL verify 链）：

| # | 目标 | 作用 |
|---|---|---|
| 1 | `SSL_set_verify` | 强制 `SSL_VERIFY_NONE` |
| 2 | `SSL_CTX_set_verify` | 同上 |
| 3 | `SSL_set_custom_verify` | 回调换空 |
| 4 | `SSL_CTX_set_custom_verify` | 回调换空 / 返回值强制 0 |
| 5 | `SSL_CTX_set_reverify_on_resume` | 关闭会话复用时的重验 |
| 6 | `SSL_get_verify_result` | 伪装 `X509_V_OK` |
| 7 | `X509_digest`（libttcrypto） | 覆写真实证书指纹 32 字节 |

**问题**：这 7 颗钩在极速版（`com.ss.android.ugc.aweme.lite`）上稳跑，在**正式版**上 4~5 秒必崩：

```
Fatal signal 5 (SIGTRAP), code 1 (TRAP_BRKPT)
  #00 pc 00000000002c8db8  libsscronet.so
```

---

## 2. 环境与命令模板（照抄可用）

### 2.1 SSH 到 Mac（走 Tailscale + SOCKS5 代理）

```bash
export PATH=/opt/data/bin:$PATH
sshpass -p *** ssh -o StrictHostKeyChecking=no -o ConnectTimeout=20 \
  -o ProxyCommand="/opt/data/bin/socks5-nc --proxy 127.0.0.1:1055 --proxy-type socks5 %h %p" \
  ljm2007@*** '<远程命令>'
```

> ⚠️ Mac 合盖会睡，睡后报 `general SOCKS server failure` / `No route to host`，唤醒后重试即可。
> ⚠️ 非交互 SSH 的 PATH 不含 Android SDK，adb 必须先 export（见下）。

### 2.2 adb（Mac 上）

```bash
export PATH=$HOME/Library/Android/sdk/platform-tools:$PATH
adb devices                       # 需 USB 连接，序列号 94NY19J44
adb shell date                    # 手机时间（对齐崩溃时间戳用）
```

### 2.3 编译 + 推送（项目约定：只用 CLion 线）

```bash
bash ~/Documents/androidsrc/AndroidInject-master/app/src/main/cpp/scripts/build-inject.sh push
```

产物：
- `clion-build/android-arm64/InlineHook/libinlinehook.so`
- `clion-build/android-arm64/Inject/Inject`（独立 ELF）

### 2.4 起注入器（后台盯梢模式）

```bash
adb shell 'exec </dev/null >/dev/null 2>&1; cd /data/local/tmp; \
  setsid ./Inject -w -f -n com.ss.android.ugc.aweme -so /data/local/tmp/libinlinehook.so \
  > /data/local/tmp/watch.log 2>&1 &'
sleep 3
adb shell pidof Inject            # ★ 必须有数字，否则后面全是假实验
```

---

## 3. 今日完整时间线（9 轮实验）

| 轮 | 时间 | pid | 崩溃点（so 内偏移） | 信号 | 当轮代码状态 |
|---|---|---|---|---|---|
| R1 | 11:04:31 | 22611 | `0x2C8DB8` | SIGTRAP / TRAP_BRKPT | 7 颗符号钩 |
| R2 | 11:06:12 | 31214 | `0x2C8DB8` | TRAP_BRKPT | + 裁决官 2C9DCC 钩（地址钩首秀）|
| R3 | 11:21:23 | 2479 | `0x2C8DB8` | TRAP_BRKPT | **⚠️ 没注入**（注入器已退出）——无效实验 |
| R4 | 11:26:53 | 22044 | `0x2C8DB8` | TRAP_BRKPT | + 2CB4F8 钩 |
| R5 | 11:38:12 | 27254 | `0x2C8DB4` | SIGTRAP / **SI_TKILL** | 崩点首次挪位（第二颗雷 HLT）|
| R6 | 11:47:30 | 32141 | `0x2C8DB8` | TRAP_BRKPT | + 2C926C 钩（**未生效**）|
| R7 | 11:56:38 | 29683 | `0x2C8DBC` | **SIGILL** / ILL_ILLOPC | NOP 两颗 BRK 后滑到 HLT |
| R8 | 12:01:36 | 6649 | `0x280C1C` | SIGTRAP / SI_TKILL | 另一函数 assert 块（存活 143s）|
| **R9** | **12:02:05 起** | **23291** | **无崩溃 ✅** | — | **引线 + 雷区全 NOP（存活 8min+）** |

---

## 4. 排雷七轮详解

### R1–R2：确认崩点、装第一颗地址钩

**取证命令**

```bash
adb shell "logcat -d | grep -A3 'F DEBUG' | tail -10"
adb shell "logcat -d | grep 'Fatal signal' | tail -3"
adb shell "logcat -d | grep -E 'HERMES_HOOK.*(裁决|安装成功)' | tail -4"
```

**关键判读**

```
09-24 11:04:33  F DEBUG : backtrace:
                #00 pc 00000000002c8db8 libsscronet.so
                lr  000000767e3c1b3c   pc 000000767e3c1db8
```

- `pc − base = asm_off`：`0x767e3c1db8 − 0x767e3f??? ...` 换算得 **so 内偏移 `0x2C8DB8`**
- 信号 `SIGTRAP + TRAP_BRKPT` ⇒ **CPU 真的执行了一条 `BRK` 指令**（内核填的 `si_code`，骗不了人）

> 💡 日志里 `lr`/`pc` 是**运行时绝对地址**，而 IDA 打开的是**文件偏移（= so 内偏移）**。
> 两者关系：`运行时地址 − 加载基址 = so 内偏移`。
> 加载基址可从 `logcat` 里我们自己的钩子日志拿到（见下）。

**本项目的取基址小技巧**：日志里直接打印

```
HERMES_HOOK: [tls] 裁决官 target = 0x76853bddcc (base=0x76850f4000 + 0x2C9DCC)
```

⇒ 这条日志里的 `base` 就是本次进程里 `libsscronet.so` 的加载基址，**用它减崩溃 pc 立刻得到偏移**。

---

### R3：一次无效实验（重要教训）

用户复现后仍然崩在 `0x2C8DB8`，但日志里**一条 `HERMES_HOOK` 都没有**。

```bash
adb shell pidof Inject          # 输出为空 → 注入器早就退出了
adb shell pidof com.ss.android.ugc.aweme
```

**教训**：`Inject -w` 盯梢模式**不是永生的**——目标 App 多次崩溃、手机重启后它会退出。
**每次实验前必须先验 `pidof Inject`**，否则拿到的崩溃日志跟本轮改动毫无关系，全是浪费。

---

### R4：装「裁决官」钩 —— 崩点不动

从 IDA 拿到的调用链（详见第 5 节），`sub_2C9DCC` 是「裁决官」：
把校验结果转成枚举返回（0 = OK，非 0 = 失败）。策略：**hook 它，永远 `return 0`**。

代码（按地址 hook，`libsscronet.so` 无符号）：

```c
// ① 求基址：读自己的 /proc/self/maps，只认 r-xp 段
static void *get_so_base(const char *soname) {
    FILE *fp = fopen("/proc/self/maps", "r");
    if (!fp) return NULL;
    char line[512]; void *base = NULL;
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, soname) && strstr(line, "r-xp")) {
            base = (void *)(uintptr_t)strtoull(line, NULL, 16);
            break;
        }
    }
    fclose(fp);
    return base;
}

// ② 声明 + proxy（保留副作用：先调原函数再改返回值）
static int (*orig_sub_2C9DCC)(void *ctx);
static int proxy_sub_2C9DCC(void *ctx) {
    int r = orig_sub_2C9DCC(ctx);
    if (r != 0) { LOGI("[tls] sub_2C9DCC 裁决=%d → 强制 0", r); return 0; }
    return r;
}

// ③ 安装（按地址）
    do {
        void *base = get_so_base("libsscronet.so");
        if (!base) { LOGE("[tls] 拿不到 libsscronet.so 基址"); break; }
        void *target = (char *)base + 0x2C9DCC;
        LOGI("[tls] 裁决官 target = %p (base=%p + 0x2C9DCC)", target, base);
        void *stub = shadowhook_hook_func_addr_2(
                target, (void *)proxy_sub_2C9DCC, (void **)&orig_sub_2C9DCC,
                SHADOWHOOK_HOOK_WITH_SHARED_MODE | SHADOWHOOK_HOOK_RECORD, NULL, NULL);
        if (stub) LOGI("[tls] hook libsscronet.so+0x2C9DCC 安装成功 stub=%p", stub);
        else      LOGE("[tls] hook 失败 errno=%d", shadowhook_get_errno());
    } while (0);
```

`shadowhook.h` 真实签名（实测确认）：

```c
void *shadowhook_hook_func_addr_2(void *func_addr, void *new_addr, void **orig_addr,
                                  uint32_t flags, ... /* record_lib_name, record_sym_name */);
```

**结果**：钩子报「安装成功」，日志也出现 `sub_2C9DCC 裁决=2 → 强制 0`，**但仍然崩在 `0x2C8DB8`**。

> 🔑 `裁决=2` 这个值本身是线索：**2 = 校验异步进行中**，不是简单的通过/失败。

---

### R5：装「证据袋」钩 —— 崩点首次挪位

用 IDA MCP 反编译 `sub_2C8850`（崩点所在函数），看到：

```c
sub_2F51C0(&v50);                              // 初始化局部结构
if ( (sub_2CB4F8(a1, &v50) & 1) == 0 )         // ← 检查「SSL 会话信息」是否可用
    __break(0);                                 // → 0x2C8DB8 BRK！
```

`sub_2CB4F8` 的伪代码尾部：

```c
v4 = *(_QWORD *)(a1 + 528);
return v4 != 0;          // 只关心 a1+528 是否为 NULL
```

**结论修正（推翻之前的猜测）**：`sub_2CB4F8` **不是证书校验**，而是**状态机断言辅助**——
判断「SSL 会话信息指针 `a1+528` 是否为空」。

**因果链**：假的证书在 boringssl 内部更深层真实失败 → `a1+528` 被置空 →
cronet 以为握手成功继续走收尾 → 断言读到空指针 → `BRK` 自爆。

**对策**：hook `sub_2CB4F8`，无条件 `return 1`（“袋子里有货”）。

**结果**：崩点变成 `0x2C8DB4`，信号从 `TRAP_BRKPT` 变成 `SI_TKILL`。
读 IDA 反汇编才明白——`0x2C8DB0` 是 `BRK`，紧跟着 `0x2C8DB4` 是 **`HLT #0`**：

```asm
2c8db0  BRK #0    ← 第一颗雷（+1175 标志位断言）
2c8db4  HLT #0
2c8db8  BRK #0    ← 第二颗雷（证据袋断言）
2c8dbc  HLT #0
2c8dc0  BL __stack_chk_fail
```

**两颗雷都是「BRK + HLT」成对出现**（双保险：BRK 被处理掉还有 HLT）。

**同时定位到死亡标志的写入点**（IDA MCP 反编译 `sub_2C926C`）：

```c
*(_BYTE *)(v4 + 1175) = 1;      // ← 0x2C9334：写「死亡标志」
v18 = 错误码;                    // 失败路径
LABEL_11:
    return 1;                    // ← 直接 return 1，【不经过】裁决官！
```

而 `sub_2C8850` 里有：

```c
if ( *(_BYTE *)(a1 + 1175) != 0 )   // 读到死亡标志
    __break(0);                      // → 0x2C8DB0 BRK
```

> 💡 `+1175` 十进制 = `+0x497`，与之前反汇编里看到的 `LDRB W8,[X19,#0x497]` 完全对上。

---

### R6：装「校验回调」钩 —— 本日最大的意外

理论上最干净的方案：hook `sub_2C926C`（boingssl 注册的 `custom_verify` 回调），
**proxy 里直接 `return 0`、根本不调用原函数** ⇒ 失败路径永不执行 ⇒ `+1175` 永不被写。

```c
static int (*orig_sub_2C926C)(void *ssl, unsigned char *out_alert);
static int proxy_sub_2C926C(void *ssl, unsigned char *out_alert) {
    LOGI("[tls] 校验回调 sub_2C926C 被调 → 直接判合格（不执行原函数）");
    return 0;
}
```

**结果**：钩子报「安装成功」，但——

```
11:47:23.479  hook libsscronet.so+0x2C926C(校验回调) 安装成功
11:47:25.377  sub_2C9DCC 裁决=2 → 强制 0        ← 原函数居然还是执行了！
11:47:27.875  崩 0x2C8DB8
```

**逻辑死结**：`sub_2C9DCC` 的唯一调用者就是 `sub_2C926C`（IDA `xrefs_to` 实锤只有 1 条）。
裁决官被执行 ⇒ 原回调执行了 ⇒ **我们的 2C926C 钩没拦住**（日志里也没有「被调」那行）。

**差异分析**：
- `2C9DCC`：cronet 内部 `BL` 指令**直接调用** → inline hook 生效 ✅
- `2C926C`：boringssl **拿着注册时缓存的函数指针跨 so 调用** → 同样手法的 inline hook 实测未拦到 ⚠️

（具体机制未查明；考虑到目标是「抓到包」而不是「长期稳定」，不继续深追。）

---

### R7：NOP 掉两颗 BRK —— 又踩到 HLT

```c
// ARM64 NOP = 0xD503201F
static void patch_brk_to_nop(void *base, long off, const char *tag) {
    void *addr = (char *)base + off;
    long page = (long)addr & ~0xFFFL;
    if (mprotect((void *)page, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC) != 0) {
        LOGE("[tls] mprotect %s 失败: %s", tag, strerror(errno));
        return;
    }
    uint32_t nop = 0xD503201F;
    memcpy(addr, &nop, 4);
    LOGI("[tls] patch %s @ %p → NOP 完成", tag, addr);
    mprotect((void *)page, 0x1000, PROT_READ | PROT_EXEC);
}
```

只 NOP 了 `0x2C8DB0` 和 `0x2C8DB8` 两个 `BRK`，结果是：

```
11:56:40  Fatal signal 4 (SIGILL), code 1 (ILL_ILLOPC), fault addr 0x76b0521dbc
          #00 pc 00000000002c8dbc  libsscronet.so
```

**`BRK` 变 `NOP` 后 CPU 不炸了，但继续顺序执行，一头撞上紧跟的 `HLT #0`**——
用户态执行 `HLT` = 非法指令 = `SIGILL`。

> 🔑 **教训：雷不是单颗，是「BRK+HLT」成对埋设的。只拆雷管不拆弹壳没用。**

---

### R8：改思路 —— 不排雷，**拆引线**

看 `sub_2C8850` 的完整反汇编，雷区（`0x2C8DB0~0x2C8DBC`）后面紧跟着 `BL __stack_chk_fail`，
所以把雷区全 NOP 会让异常流滑进 stack-chk（同样死）。

**正解：让执行流根本到不了雷区** —— 把**跳进雷区的跳转指令** NOP 掉：

```asm
0x2C8A68  CBNZ W8, loc_2C8DB0    ← 引线1（+1175 死亡标志）→ NOP
0x2C8B3C  TBZ W0,#0, loc_2C8DB8  ← 引线2（2CB4F8 返回失败）→ NOP
```

（`sub_2C8850` 里跳进雷区的跳转只有这两条，IDA 全函数扫描确认。）

**最终 patch 清单**（`hook_entry.c` 第 419–426 行）：

```c
    do {
        void *base = get_so_base("libsscronet.so");
        if (!base) break;
        patch_brk_to_nop(base, 0x2C8A68, "引线1(CBNZ→BRK-1)");
        patch_brk_to_nop(base, 0x2C8B3C, "引线2(TBZ→BRK-2)");
        patch_brk_to_nop(base, 0x2C8DB0, "BRK-1");
        patch_brk_to_nop(base, 0x2C8DB4, "HLT-1");
        patch_brk_to_nop(base, 0x2C8DB8, "BRK-2");
        patch_brk_to_nop(base, 0x2C8DBC, "HLT-2");
    } while (0);
```

**结果**（12:02:05 起，pid 23291）：**存活 8 分钟以上，无任何崩溃** ✅
（对比：之前 4~5 秒必崩）

> 12:01:36 那次崩在 `0x280C1C` 的是**上一版（无引线）**的进程，
> 崩点位于**另一个函数** `sub_27D148` 的 C++ assert 雷区，属于独立的、低概率的第三颗雷（暂未处理）。

---

## 5. IDA 侧的完整地图（本次分析成果）

### 5.1 校验链（自上而下）

```
sub_2C7C1C   SSL_CTX 初始化
   └─ SSL_CTX_set_custom_verify(ctx, mode=1, 回调 = sub_2C926C)
        │   ↳ 注册点 0x2C7CC8（IDA xrefs 唯一）
        │
sub_2C926C   证书校验回调（本函数 0x414 字节）
   ├─ SSL_get0_peer_certificates 拿证书链
   ├─ #define 56 字节任务结构（含 Cronet_FrontierParams_Destroy 函数指针）
   ├─ 0x2C94AC 处 vtable 虚调用 → 真校验异步跑，结果写 a1+712
   ├─ 失败路径：*(v4 + 1175) = 1  @ 0x2C9334   ← ★死亡标志★
   │            return 1（LABEL_11，绕过裁决官！）
   └─ 尾声：sub_2C9DCC(a1) 把结果转成枚举
        │
sub_2C9DCC   裁决官：结果 == 0 或白名单(-150 等) → return 0 ✅ / 否则 return 1 ❌
        │
sub_2C8850   握手收尾机（3 个自杀出口）
   ├─ sub_2CB4F8(a1,&v50) 为假 → BRK @ 0x2C8DB8   （读 a1+528 会话信息）
   ├─ *(a1+1175) 非零        → BRK @ 0x2C8DB0   （死亡标志）
   └─ 其它                  → BL __stack_chk_fail @ 0x2C8DC0
```

### 5.2 地址速查表

| 偏移 | 名称 | 说明 |
|---|---|---|
| `0x195DFC` | helper | 填 `a1+528` 用 |
| `0x27D148` | sub_27D148 | 另一函数，含 `0x280C00~0x280C2C` BRK/HLT 雷列（C++ assert）|
| `0x27D6E8` | `CMP X20,X21; B.EQ loc_280C00` | 上述雷列的引线（空区间 assert）|
| `0x280C1C` | `HLT #0` | R8 轮残雷崩点 |
| `0x2C7C1C` | sub_2C7C1C | SSL_CTX 初始化 / 注册 custom_verify |
| `0x2C7D58` | sub_2C7D58 | `return SSL_get_ex_data(a2, a1);` 包装 |
| `0x2C8850` | sub_2C8850 | 握手收尾机（342 条指令，雷区所在）|
| `0x2C8A68` | `CBNZ W8, loc_2C8DB0` | **引线1**（+1175）|
| `0x2C8B3C` | `TBZ W0,#0, loc_2C8DB8` | **引线2**（2CB4F8 失败）|
| `0x2C8DB0/4` | `BRK` + `HLT` | **雷1**（+1175 死亡标志）|
| `0x2C8DB8/C` | `BRK` + `HLT` | **雷2**（a1+528 为空）|
| `0x2C8DC0` | `BL __stack_chk_fail` | 雷区后的第三出口 |
| `0x2C926C` | sub_2C926C | 证书校验回调（写 +1175 @ 0x2C9334）|
| `0x2C94AC` | vtable 调用点 | 真校验异步执行 |
| `0x2C9DCC` | sub_2C9DCC | 裁决官 |
| `0x2CB4F8` | sub_2CB4F8 | 证据袋检查（读 `a1+528`）|

---

## 6. IDA MCP 用法（本次分析的主力工具）

### 6.1 链路结构

```
手机 ──USB── Mac(IDA Pro 9.4 GUI) ──ida-pro-mcp(端口 13337)──SSH隧道── Hermes(NAS)
```

- Mac 侧：IDA 载入 `libsscronet.so` → `Edit ▸ Plugins ▸ MCP`（⌘⌥M）起服务
- NAS 侧：SSH 隧道把 `13337` 映射过来，然后用 Python 直接调 JSON-RPC

### 6.2 Python 调用模板（照抄可用）

```python
import json, urllib.request

URL = "http://127.0.0.1:13337/mcp"
HDRS = {"Content-Type": "application/json", "Accept": "application/json, text/event-stream"}
_sid = None; _id = 0

def rpc(method, params=None, notify=False):
    global _id, _sid
    _id += 1
    body = {"jsonrpc": "2.0", "method": method}
    if params is not None: body["params"] = params
    if not notify: body["id"] = _id
    req = urllib.request.Request(URL, data=json.dumps(body).encode(), headers=HDRS, method="POST")
    if _sid: req.add_header("Mcp-Session-Id", _sid)
    r = urllib.request.urlopen(req, timeout=180)
    if r.headers.get("Mcp-Session-Id"): _sid = r.headers.get("Mcp-Session-Id")
    raw = r.read().decode()
    if raw.startswith("data:") or "\ndata:" in raw:
        for line in raw.splitlines():
            if line.startswith("data:"): raw = line[5:].strip(); break
    return json.loads(raw) if raw.strip() else None

def tool(name, args=None):
    r = rpc("tools/call", {"name": name, "arguments": args or {}})
    return "\n".join(c.get("text", "") for c in r.get("result", {}).get("content", []) if c.get("type") == "text")

# 初始化（每次新会话必须先握手）
rpc("initialize", {"protocolVersion": "2025-03-26", "capabilities": {},
                   "clientInfo": {"name": "hermes", "version": "1.0"}})
rpc("notifications/initialized", {}, notify=True)
```

### 6.3 本次最常用的 5 个 IDA 调用

```python
# 1) 反编译函数（拿伪代码，找逻辑/断言）
tool("decompile", {"addr": "0x2C926C"})

# 2) 反汇编（看指令级细节，尤其 BRK/HLT/跳转）
tool("disasm", {"addr": "0x280C00", "max_instructions": 60})

# 3) 查交叉引用（谁调用了它 / 谁引用了这个数据）
tool("xrefs_to", {"addrs": ["0x2C9DCC"], "limit": 20})

# 4) 批量扫描指令（例：找出所有跳进某片雷区的跳转）
tool("py_eval", {"code": '''
import idautils, idc
res = []
for ea in idautils.FuncItems(0x27D148):
    m = idc.print_insn_mnem(ea)
    if m.startswith("B") or m.startswith("CB") or m.startswith("TB"):
        tgt = idc.get_operand_value(ea, 0)
        if 0x280BF0 <= tgt <= 0x280C70:
            res.append((hex(ea), idc.GetDisasm(ea)))
print(res)
'''})

# 5) 写注释（把分析结论固化在 IDA 里，避免重复分析）
tool("set_comments", {"items": [{"addr": "0x2C8DB0", "comment": "HERMES: BRK自爆点..."}]})
```

> 💡 `py_eval` 是**本次的杀手锏**：找“所有跳进雷区的跳转”这种事，
> 用汇编窗口一条条翻要几十分钟，脚本 3 秒出结果。

---

## 7. 命令速查（按使用频率）

```bash
# ── Mac 侧 ──
export PATH=$HOME/Library/Android/sdk/platform-tools:$PATH

adb devices                                       # 确认手机在线
adb shell date                                    # 手机时间
adb shell pidof Inject                            # ★ 注入器在岗？（实验前必查）
adb shell pidof com.ss.android.ugc.aweme          # App 在跑？
adb shell "grep -c libinlinehook /proc/<pid>/maps"  # 注入成功的铁证（>0）

# 崩溃取证
adb shell "logcat -d | grep -A3 'F DEBUG' | tail -30"
adb shell "logcat -d | grep 'Fatal signal' | tail -3"

# 钩子日志
adb shell "logcat -d | grep -E 'HERMES_HOOK.*(裁决|安装成功|patch)' | tail -6"

# ── 起注入器（盯梢+自动拉起）──
adb shell 'exec </dev/null >/dev/null 2>&1; cd /data/local/tmp; \
  setsid ./Inject -w -f -n com.ss.android.ugc.aweme -so /data/local/tmp/libinlinehook.so \
  > /data/local/tmp/watch.log 2>&1 &'

# ── 编译 + 推送 ──
bash ~/Documents/androidsrc/AndroidInject-master/app/src/main/cpp/scripts/build-inject.sh push
```

---

## 8. 踩坑与教训

| # | 坑 | 教训 |
|---|---|---|
| 1 | **没注入就当实验结论** | `Inject -w` 会退出；实验前必查 `pidof Inject`，否则崩溃日志是上一版残留 |
| 2 | **logcat 缓冲区会滚** | 崩完要尽快取证；事后 `grep HERMES_HOOK` 可能一条都没有（被某音日志淹没）|
| 3 | **BRK 是成对埋的** | `BRK` 后面紧跟 `HLT #0`，只 NOP BRK 会滑到 HLT → SIGILL |
| 4 | **`loc_xxx` 不参与文本搜索** | IDA 里要找 `loc_2C8DB0` 得按 **G** 跳地址，不能文本搜 |
| 5 | **`TRAP_BRKPT` vs `SI_TKILL`** | 前者 = CPU 执行到 `BRK` 指令（内核填的 si_code）；后者 = 有人 `tgkill` 主动杀（npth 崩溃收集器接管）|
| 6 | **跨 so 的函数指针调用** | 同样手法的按地址 inline hook，`BL` 直调的生效、**经注册指针跨 so 调的实测没拦到** |
| 7 | **改返回值 ≠ 改状态** | hook 裁决官只是“改判结果”，内部状态（a1+528 / +1175）没同步 → 下游断言照样炸 |
| 8 | **别打地鼠，拆引线** | 一颗颗排雷成本高；**把跳向雷区的跳转 NOP 掉**，雷区变孤岛，一次解决 |
| 9 | **`mprotect` 改代码段** | root 注入环境下可行：`mprotect(页, 4K, RWX)` → 写 4 字节 → 恢复 `R|X` |

---

## 9. 当前代码与产物状态

**源文件**：`app/src/main/cpp/InlineHook/src/hook_entry.c`（540 行）

**钩子清单**

| 类型 | 数量 | 内容 |
|---|---|---|
| 符号钩 | 7 | `libttboringssl.so`：`SSL_set_verify` / `SSL_CTX_set_verify` / `SSL_set_custom_verify` / `SSL_CTX_set_custom_verify` / `SSL_CTX_set_reverify_on_resume` / `SSL_get_verify_result`；`libttcrypto.so`：`X509_digest` |
| 地址钩 | 3 | `libsscronet.so` + `0x2C9DCC`（裁决官）/ `+0x2CB4F8`（证据袋）/ `+0x2C926C`（校验回调，未生效）|
| 内存 patch | 6 | `0x2C8A68`、`0x2C8B3C`（引线）；`0x2C8DB0/4/8/C`（雷区）→ 全部 NOP |

**产物**：
- `clion-build/android-arm64/InlineHook/libinlinehook.so`
- 手机 `/data/local/tmp/libinlinehook.so`（MD5 与 Mac 产物一致：`de265f202e43a0506bc0463c47f7afa8`）

**实测结果**（12:02:05 版）：正式版某音**存活 8 分钟以上无崩溃** ✅

---

## 10. 遗留问题与下一步

| 优先级 | 事项 |
|---|---|
| 🔴 高 | **挂 Charles 抓正式版明文包**（进程已稳定，正是收网时机）|
| 🟡 中 | `sub_2C926C` 地址钩为何没拦住（跨 so 函数指针调用）——不影响抓包，可后置 |
| 🟡 中 | 残雷 `0x280C1C`（`sub_27D148` 的 C++ assert 区，引线 `0x27D6E8`）——复现再处理 |
| 🟢 低 | 日志文案 `8 钩全部安装完毕（ttboringssl 7 + conscrypt 4）` 数字不自洽，建议改为 `ttboringssl 7 + cronet 3` |

---

## 附：本次形成的方法论（可复用到其他 App）

1. **先取证，再动手**：`si_code` + backtrace 的第一行 pc 是唯一可信的起点
2. **`pc − base = 偏移`**：自己日志里打印 base，换算崩溃偏移，直接丢给 IDA
3. **IDA 读逻辑，不要猜**：`decompile` 看断言条件，`xrefs_to` 找唯一入口，`py_eval` 扫跳转
4. **改状态 > 改结果**：改返回值只是掩耳盗铃，状态一致性才是关键
5. **拆引线 > 排雷**：跳转来源是有限的、可枚举的；雷区可能是无限的
6. **每个结论都要有日志/反汇编证据**，推断要标明是推断
