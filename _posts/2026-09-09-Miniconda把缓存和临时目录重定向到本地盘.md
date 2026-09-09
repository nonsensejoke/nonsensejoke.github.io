---
layout: default
title: "🐍 Miniconda 把缓存和临时目录挪到本地盘：CONDA_PKGS_DIRS、PIP_CACHE_DIR 与 TMPDIR"
date: 2026-09-09 10:00:00 +0800
categories: [Linux, HPC]
tags: [conda, miniconda, pip, hpc, nfs, tmpdir, cache, hardlink]
description: "在网络盘很慢或 home 有配额的集群上，用三个环境变量把 conda/pip 的缓存和临时文件重定向到本地 /tmp。顺带说清硬链接失效这个容易被忽略的坑。"
---

在 HPC 集群上用 Miniconda，经常会看到这样一行：

```bash
export CONDA_PKGS_DIRS=/tmp/$USER-conda-pkgs \
       PIP_CACHE_DIR=/tmp/$USER-pip-cache \
       TMPDIR=/tmp/$USER-tmp
```

它做的事情很单一：**把 conda 和 pip 的缓存、临时文件从家目录挪到本地磁盘**。

这个做法本身是对的，但收益的大小取决于几个前提条件。本文先说清三个变量各管什么，再说一个容易被忽略的坑：**硬链接跨文件系统会失效**。

---

## 🔍 三个变量各管什么

| 环境变量 | 默认位置 | 存放内容 |
|---|---|---|
| `CONDA_PKGS_DIRS` | `<conda根>/pkgs`、`~/.conda/pkgs` | conda 下载的包及解压后的内容 |
| `PIP_CACHE_DIR` | `~/.cache/pip` | pip 的 wheel 缓存与 HTTP 缓存 |
| `TMPDIR` | `/tmp` | 通用临时目录，pip 源码构建、conda 解包都会用 |

**`CONDA_PKGS_DIRS`** 是包缓存。conda 下载的每个包都会在这里留一份压缩包和一份解压后的文件树，装几个环境轻松涨到几个 GB。

**`PIP_CACHE_DIR`** 是 pip 自己的缓存。注意它和 `TMPDIR` 是两码事：前者存构建好的 wheel 以便复用，后者存构建过程中的中间产物。只设一个是常见的半吊子做法。

**`TMPDIR`** 默认就是 `/tmp`，显式设置它有两种情况：一是集群把它改到了家目录下的某个路径，这里是改回去；二是想加 `$USER` 前缀做隔离。pip 从源码构建包时的解压、编译（`.o`、build 目录）全在这里发生。

加 `$USER` 前缀是因为 `/tmp` 是全机器共享的，登录节点上多人同名目录会撞权限，而且报错信息通常很难懂。

---

## ✅ 两种动机，收益不一样

这个做法通常出于两个原因之一，值得分开看。

### 动机一：home 有配额

家目录被限制在比如 20 GB，装几个环境就 `DiskQuotaExceeded`。把缓存挪出去确实能让配额不再被缓存吃掉。

### 动机二：网络盘读写慢

家目录挂在 NFS 上，装包慢得让人怀疑人生。这时候把缓存挪到本地盘 —— **但收益比想象的小**，原因见下一节。

---

## ⚠️ 关键的坑：硬链接会失效

conda 装环境时的默认策略是把 `pkgs` 里的文件**硬链接**到环境目录，所以同一个包被多个环境引用也只占一份磁盘空间。

而硬链接不能跨文件系统。

对比两种情况：

| 布局 | 解压 | 装进环境 | NFS 总写入 |
|---|---|---|---|
| `pkgs` 和 env 都在 NFS | 往 NFS 写一份（慢） | 硬链接，只动元数据（几乎免费） | ≈ 一份 |
| `pkgs` 在 `/tmp`，env 在 NFS | 写本地（快） | 只能整份**复制**到 NFS（慢） | ≈ 还是一份 |

也就是说，**如果环境本身还在网络盘上，单个环境的安装速度基本是打平的**。装多个共用包的环境时，前者还能靠硬链接去重，反而更快。

真正的瓶颈是「往 NFS 写几万个小文件」这一步，只搬缓存并不能绕过它。

按这个逻辑再看三个变量：

- `TMPDIR` —— **纯赚**。编译中间产物是一次性垃圾，从来不需要落到网络盘，这部分小文件 IO 量很大。
- `PIP_CACHE_DIR` —— **基本纯赚**。wheel 和 HTTP 缓存没理由放网络盘。
- `CONDA_PKGS_DIRS` —— **看情况**，取决于环境放在哪。

---

## 🚀 想真的快，环境也得放本地

```bash
conda create -p /tmp/$USER/envs/myenv python=3.11 numpy
conda activate /tmp/$USER/envs/myenv
```

环境和 `pkgs` 在同一个文件系统，硬链接恢复，跨文件系统复制消失。

更重要的是：**每次 `import` 要读几千个 `.py` / `.so` 文件**。这部分只有环境本身在本地盘才能省掉，而它对体感的影响往往比安装快慢更明显 —— 尤其是 import numpy、torch 这类大包的时候。

代价是环境随 `/tmp` 清理一起消失。如果是跑一次性任务，这个代价通常可以接受。

---

## 🧪 动手前先确认两件事

### `/tmp` 是真磁盘还是 tmpfs

```bash
findmnt /tmp
df -h /tmp
```

如果 `FSTYPE` 显示 `tmpfs`，那它是**内存盘**。解压 conda 包动辄几个 GB，会直接吃掉作业的内存配额，甚至 OOM —— 这时候千万别往这里放 `CONDA_PKGS_DIRS`。

### `/tmp` 是不是节点本地的

本地 `/tmp` 通常每个计算节点各自一份，重启必清，很多系统还有 `systemd-tmpfiles` 定期删旧文件。

后果是：**换节点缓存就没了，每次重新下载**。如果工作流是「反复提交作业、每次装一遍环境」，省下的 NFS 写入时间可能又被网络下载吃回去。

这种情况下比较稳的组合是：

- 环境建在 NFS 或 `$SCRATCH` 上，一次建好长期复用；
- 只把 `TMPDIR` 和 `PIP_CACHE_DIR` 丢到本地 `/tmp`（反正是一次性垃圾，不存在复用问题）。

如果集群提供了 `$SCRATCH` 这类大容量持久空间，指到那里通常比 `/tmp` 更合适。

### 顺手补一个 `mkdir -p`

conda 和 pip 一般会自建目录，但 `TMPDIR` 指向不存在的路径时，编译链里的某些第三方脚本会直接报一个莫名其妙的错：

```bash
mkdir -p /tmp/$USER-conda-pkgs /tmp/$USER-pip-cache /tmp/$USER-tmp
```

---

## 🐌 如果慢在「Solving environment」

这一点值得单独提：如果卡住的地方是 `Solving environment: \` 转了好几分钟，那**跟磁盘完全无关**，是依赖求解的问题，改路径一点用都没有。

conda 23.10 之后 libmamba 已经是默认求解器。老版本需要手动启用：

```bash
conda install -n base conda-libmamba-solver
conda config --set solver libmamba
```

或者直接换 `mamba` / `micromamba`，效果比调路径明显得多。

---

## 📝 小结

```bash
# 建目录
mkdir -p /tmp/$USER-{conda-pkgs,pip-cache,tmp}

# 重定向
export CONDA_PKGS_DIRS=/tmp/$USER-conda-pkgs \
       PIP_CACHE_DIR=/tmp/$USER-pip-cache \
       TMPDIR=/tmp/$USER-tmp
```

要点回顾：

- 三个变量分别管 **conda 包缓存**、**pip wheel 缓存**、**通用临时目录**，三个一起设才算覆盖完整；
- `TMPDIR` 和 `PIP_CACHE_DIR` 挪到本地盘是纯赚；
- `CONDA_PKGS_DIRS` 的收益取决于环境放在哪 —— 环境在 NFS 的话，硬链接失效会抵消掉大部分好处；
- **想真正提速，把环境本身也建到本地盘**，`import` 的收益比安装更明显；
- 动手前用 `findmnt /tmp` 确认它不是 tmpfs；
- 卡在 solving environment 的话，问题不在磁盘，换 libmamba。

最后，`export` 只对当前 shell 及其子进程生效。想长期使用得写进 `~/.bashrc`，或者放进作业脚本模板里。

---

**完**
