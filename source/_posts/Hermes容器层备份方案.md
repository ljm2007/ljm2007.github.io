---
title: Hermes 容器层备份方案
date: 2026-09-20 20:00:00
top_img: /img/post-covers/Hermes容器层备份方案.png
cover: /img/post-covers/Hermes容器层备份方案.png
tags:
  - Hermes
  - 备份
categories: 运维
---
> 日期：2026-09-20
> 环境：UGREEN NAS（DXP4800PLUS-158）内 Docker 容器运行的 Hermes Agent

---

## 一、为什么要区分「持久层」和「容器层」

Hermes 的数据实际分布在三个位置，容器的挂载情况实测如下：

| 位置 | 真实归属 | 容器删了重建 | 整机/硬盘坏了 |
|---|---|---|---|
| `/opt/data` | NAS 存储卷 `ug_..._pool3-volume1`（独立挂载） | ✅ 还在 | ❌ 没了 |
| `/volume3/hermes/data` | 同一存储卷，另一个挂载点 | ✅ 还在 | ❌ 没了 |
| `/root`、`/usr/local/bin`、`/etc`、`/opt/hermes` | 容器 overlay upper 层 | ❌ **一起没** | ❌ 没了 |

`mount` 输出可确认前两者是独立挂载的设备：

```
/dev/mapper/ug_8F2769_1783484762_pool3-volume1 on /opt/data            type ext4
/dev/mapper/ug_8F2769_1783484762_pool3-volume1 on /volume3/hermes/data type ext4
```

**结论**：
- 前两处 = 持久层 → **用户已用其他方式自行备份，本方案不覆盖**
- 第三处 = 容器层 → 只有本方案负责，这是「重装 Hermes 后懒得重做」的部分

---

## 二、容器层里到底有什么（实测）

| 内容 | 体积 | 丢了要紧吗 |
|---|---|---|
| `/opt/hermes` 程序源码（4534 个文件） | 压缩后 12M | 要紧：里面有本地改动（如 `run_agent.py`、`.bak` 原件） |
| `/root/.ssh`（known_hosts 等）、`.gitconfig`、`.npmrc`、`.profile`、`.bashrc` | 几十 KB | 要紧，但可以手工补 |
| `/usr/local/bin` 自定义脚本（`macssh`、`mihomo` 软链、`ensure-tailscale-userspace`…） | 24K | 要紧 |
| `/etc` 关键文件（sshd_config、hosts、localtime、profile.d、apt 源） | 几十 KB | 要紧 |
| `/usr/share/fonts`（DejaVu 等，做中文封面图要用） | 3.3M | 要重装，麻烦 |
| 已装软件包清单（apt 841 个 / pip 12 个 / node、hexo 版本） | 68K | 靠清单重装 |

**刻意不备**（重装会自动有，占大头）：

| 目录 | 体积 | 原因 |
|---|---|---|
| `/opt/hermes/.venv` | 732M | 安装程序自带 |
| `/opt/hermes/.playwright` | 262M | 浏览器二进制，可重下 |
| `/opt/hermes/node_modules` | 176M | `npm install` 即可 |
| `/root/.cache`、`/root/.local/lib` | 40M | pip / 构建缓存 |

---

## 三、脚本

**路径**：`/opt/data/scripts/hermes_container_backup.sh`（权限 700）

### 用法

```bash
bash /opt/data/scripts/hermes_container_backup.sh                  # 完整备份（含程序源码，14M）
bash /opt/data/scripts/hermes_container_backup.sh --no-hermes-src  # 精简（仅配置+字体+清单，1.8M）
bash /opt/data/scripts/hermes_container_backup.sh --keep 14        # 保留最近 14 份
bash /opt/data/scripts/hermes_container_backup.sh --out /path      # 换输出目录
```

默认输出：`/volume3/hermes/data/backups/container/hermes_container_<时间戳>.tar.gz`，自动保留最近 7 份，权限 600。

### 实测结果

| 模式 | 体积 | 耗时 |
|---|---|---|
| 完整（含源码） | **14 MB** | 3.6 秒 |
| 精简（不含源码） | **1.8 MB** | 约 3 秒 |

### 包内结构

```
hermes-container-layer/
├── MANIFEST.txt              各部分体积、包数量、明确列出「不含什么」
├── RESTORE.txt               分步恢复说明
├── opt-hermes-src/
│   ├── hermes-src.tar.gz     程序源码（排除 .venv / .playwright / node_modules）
│   ├── hermes-src.md5        4534 个文件的校验和 → 升级后 diff 出「我改过哪些文件」
│   └── local-patches/        带 .bak/.orig 后缀的原件（手改前的版本）
├── container-config/
│   ├── root-ssh/             /root/.ssh
│   ├── root-local-bin/       /root/.local/bin（只留自己写的脚本，不含 pip 库）
│   ├── usr-local-bin/        /usr/local/bin
│   ├── etc/                  hosts、environment、localtime、resolv.conf
│   ├── etc-profile.d/        /etc/profile.d
│   ├── sshd_config           /etc/ssh/sshd_config
│   ├── apt-sources.list.d/   apt 源
│   ├── .gitconfig / .npmrc / .profile / .bashrc
│   └── system-crontab.txt    系统级 crontab
├── fonts/                    /usr/share/fonts + font-list.txt
└── package-lists/
    ├── apt-packages.txt      841 个包（名 + 版本）
    ├── apt-selections.txt
    ├── pip-freeze.txt        12 个
    ├── npm-global.txt
    └── runtime-versions.txt  node / npm / python / uv / hexo 版本
```

---

## 四、恢复流程

```bash
# 1) 先把两处 NAS 持久文件夹挂回容器（用用户自己的备份方式）
# 2) 解包
tar xzf hermes_container_*.tar.gz -C /tmp
cd /tmp/hermes-container-layer

# 3) 程序源码（可选，重装 Hermes 通常已带）
#    先看 diff，避免把新版程序盖回旧版：
md5sum -c opt-hermes-src/hermes-src.md5 2>/dev/null | grep -v OK
tar xzf opt-hermes-src/hermes-src.tar.gz -C /opt/hermes

# 4) 配置拷回
cp -a container-config/root-ssh/.       /root/.ssh/
cp -a container-config/root-local-bin/. /root/.local/bin/
cp -a container-config/usr-local-bin/.  /usr/local/bin/
cp -a container-config/.gitconfig       /root/
cp -a container-config/.npmrc           /root/
cp -a container-config/etc-profile.d/.  /etc/profile.d/
cp -a container-config/sshd_config      /etc/ssh/sshd_config

# 5) 字体
cp -a fonts/. /usr/share/fonts/ && fc-cache -fv

# 6) 软件包
awk '{print $1}' package-lists/apt-packages.txt | xargs apt-get install -y --no-install-recommends
pip install -r package-lists/pip-freeze.txt
```

**恢复后检查**：git 凭据可用（`.gitconfig` 指向 `/opt/data/secrets/git-credentials`）、中文封面图能渲染（字体）、`macssh`/`mihomo` 等脚本能跑。

---

## 五、关键设计点

1. **只备"增量"**：程序本体、虚拟环境、node_modules 全部排除，体积从 1.3G 降到 14M
2. **`md5` 清单**：没有原始 git 仓库可比对，用校验和记录"当前状态"，以后升级 Hermes 后能一条命令找出本地改动过哪些文件
3. **保留本地补丁原件**：把 `.bak` / `.orig` 后缀的文件单独收集，明确标记"这是手改前的版本"
4. **`--no-hermes-src` 降级模式**：只想留配置和包清单时体积压到 1.8M，适合高频跑
5. **恢复说明写在包里**：不在容器外的文档里，避免备份和文档走散
6. **包内明确声明"不含什么"**：`MANIFEST.txt` 直写「`/opt/data`、`/volume3/hermes/data` 请用你自己的备份」，防止以后误以为全都在里面

---

## 六、自动执行（已配置）

**关键认知**：乙包**不是自动更新的** —— 它是被脚本「生成」出来的。NAS 上的同步任务只负责**搬运**，必须有人先**生产**。

### Hermes 定时任务

| 项 | 值 |
|---|---|
| 任务名 | `hermes-container-backup-weekly` |
| Job ID | `79c482b9d972` |
| 时间 | `0 3 * * 1` —— **每周一凌晨 03:00** |
| 预运行脚本 | `/opt/data/scripts/container_backup_cron.py` |
| 投递 | 回到当前飞书会话（一到两句话汇报） |
| 状态 | 已启用，实测通过 |

`container_backup_cron.py` 做三件事：
1. 调用 `hermes_container_backup.sh --keep 7`
2. 校验新包完整性（`tar tzf | wc -l`，条目 > 10 才算 OK），输出 `ACTION=OK / WARN / ERROR`
3. 列出备份目录现状（份数、体积、时间），并提示"同步任务需排在本任务之后"

### 完整闭环

```
每周一 03:00  Hermes 定时任务 → 生成新乙包（14M，留最近 7 份）
        ↓
之后任意时间  NAS 上的同步任务 → 把 backups/container/ 搬到异地
```

⚠️ **顺序要求**：NAS 同步任务必须在 03:00 **之后**跑，否则搬走的是上周的旧包。

如果 NAS 同步任务也是周一凌晨跑，把它的时间往后挪（例如 04:00），或告诉我时间，我把本任务提到它前面。

---

## 七、遗留事项

- 旧的整机包 `backups/hermes/hermes_slim_*.tar.gz`（154M + 178M）已被本方案取代，**已删除**（腾出 332M）
- 旧的 `hermes_backup.sh` / `hermes_restore.sh` 已无用（它们备的是甲抽屉），可删可留
- ⚠️ **甲抽屉（NAS 持久文件夹）现在没有任何备份保护**，用户声明自行负责，建议尽快在 NAS 上确认备份任务真的在跑
