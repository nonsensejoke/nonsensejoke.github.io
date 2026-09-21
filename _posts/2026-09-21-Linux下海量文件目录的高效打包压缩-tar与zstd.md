---
layout: default
title: "Linux 下海量文件目录的高效打包压缩：tar + zstd"
date: 2026-09-21 15:00:00 +0900
categories: [Linux]
tags: [Linux, tar, zstd, compression, filesystem]
description: "Linux 下包含几十万甚至上百万文件的目录如何高效打包压缩，以及 tar -I zstd 与 tar | zstd 两种写法的区别。"
---

在 Linux 上，如果一个目录里有**几十万甚至上百万个文件**，压缩时需要考虑的不只是压缩算法本身，还包括文件系统元数据访问、inode 遍历、磁盘 I/O，以及大量小文件带来的额外开销。

对于这类场景，一个非常实用的方案是：

> **先用 `tar` 将大量文件串成一个连续的数据流，再使用 `zstd` 压缩。**

相比直接对几十万个文件逐个压缩，这种方式通常更加适合 Linux 文件树的归档和备份。

<!--more-->

## 推荐方案：tar + zstd

最常用的写法之一：

```bash
tar -I 'zstd -T0 -3' -cf backup.tar.zst /path/to/folder
```

也可以将 `tar` 和 `zstd` 通过管道明确拆开：

```bash
tar -cf - folder | zstd -T0 -1 -o folder.tar.zst
```

两者的核心流程其实完全一样：

```text
大量文件 -> tar 打包 -> zstd 压缩 -> .tar.zst 文件
```

`tar` 负责处理目录树、权限、软链接等 Unix 文件系统信息；`zstd` 负责压缩最终产生的连续数据流。

---

## 为什么推荐 zstd

`zstd`（Zstandard）比较适合这种场景，主要是因为：

- 压缩速度快
- 解压速度非常快
- 支持多线程压缩
- 压缩率和速度之间容易调整
- 很适合配合 `tar` 做流式归档

例如：

```bash
zstd -T0 -3
```

其中：

- `-T0`：自动使用可用 CPU 核心进行多线程压缩
- `-3`：压缩等级 3，速度与压缩率比较均衡

如果更重视速度，可以使用：

```bash
zstd -T0 -1
```

如果更重视压缩率，可以提高等级，例如：

```bash
zstd -T0 -10
```

不过，对于几十万个小文件来说，真正的瓶颈很多时候并不是压缩算法，而是文件系统本身，因此盲目提高压缩等级未必划算。

---

## 两种 tar + zstd 写法有什么区别

我们重点比较下面两条命令。

### 写法一：由 tar 调用 zstd

```bash
tar -I 'zstd -T0 -3' -cf backup.tar.zst /path/to/folder
```

这里的：

```bash
-I 'zstd -T0 -3'
```

等价于告诉 `tar`：

> 使用指定的外部压缩程序处理 tar 产生的数据。

流程可以理解为：

```text
tar
 └── zstd
      └── backup.tar.zst
```

这种方式最大的优点是命令简洁，适合日常手工使用。

---

### 写法二：使用 Shell 管道

```bash
tar -cf - folder | zstd -T0 -1 -o folder.tar.zst
```

这里：

```bash
tar -cf - folder
```

最后的 `-` 表示：

> 不让 `tar` 自己创建归档文件，而是把 tar 数据输出到标准输出 stdout。

然后通过：

```bash
|
```

把数据交给 `zstd`：

```text
tar -> stdout -> pipe -> zstd -> folder.tar.zst
```

这种方式稍微长一些，但更加直观、灵活。

例如可以很方便地插入 `pv` 查看吞吐速度：

```bash
tar -cf - folder | pv | zstd -T0 -3 -o folder.tar.zst
```

---

## 两种写法的性能差别大吗

如果使用完全相同的 `zstd` 参数：

```bash
tar -I 'zstd -T0 -3' -cf backup.tar.zst folder
```

和：

```bash
tar -cf - folder | zstd -T0 -3 -o backup.tar.zst
```

实际性能通常非常接近。

因为二者最终做的事情都是：

```text
tar 产生数据流 -> zstd 压缩数据流
```

因此，真正造成明显差异的通常不是 `-I` 和 `|`，而是压缩级别。

例如最开始比较的两条命令分别使用了：

```text
第一条：zstd -3
第二条：zstd -1
```

所以第二条通常会更快一些，但最终文件可能稍大。

简单对比如下：

| 项目 | `tar -I 'zstd ...'` | `tar \| zstd` |
| :--- | :--- | :--- |
| 命令长度 | 更短 | 稍长 |
| 可读性 | 简洁 | 数据流更直观 |
| 性能 | 基本相同 | 基本相同 |
| 自定义 zstd 参数 | 支持 | 非常方便 |
| 插入 `pv` 等工具 | 不够直观 | 很方便 |
| Shell 脚本错误处理 | 相对简单 | 建议配合 `pipefail` |

---

## 管道写法要注意 pipefail

如果在 Shell 脚本中使用：

```bash
tar -cf - folder | zstd -T0 -3 -o backup.tar.zst
```

建议启用：

```bash
set -o pipefail
```

完整写法：

```bash
set -o pipefail

tar -cf - folder | zstd -T0 -3 -o backup.tar.zst
```

原因是 Shell 管道默认主要关注最后一个命令的退出状态。

假设 `tar` 因为权限错误、磁盘 I/O 错误等原因中途失败，而 `zstd` 对已经收到的数据正常完成压缩，如果没有 `pipefail`，脚本有可能无法正确反映整个管道已经失败。

启用 `pipefail` 后，只要管道中的命令失败，就更容易在自动化脚本中正确检测到异常。

---

## 几十万个文件时，真正的瓶颈可能不是压缩

如果目录里有大量小文件，例如：

```text
500,000 个文件
每个文件几 KB 到几十 KB
```

那么系统需要执行大量类似操作：

```text
目录遍历
  -> stat inode
  -> open 文件
  -> read 文件
  -> close 文件
```

因此性能瓶颈可能来自：

- inode / metadata 访问
- 大量 `open()` / `stat()` 系统调用
- HDD 随机 I/O
- SSD 的文件系统元数据访问
- NFS / NAS 网络文件系统的 metadata 延迟

这种情况下，把 zstd 从 `-3` 调到 `-10`，往往只会增加 CPU 时间，对整体速度帮助不大。

如果目标是**尽快完成归档**，可以优先考虑：

```bash
tar -cf - folder | zstd -T0 -1 -o folder.tar.zst
```

如果希望速度和压缩率比较均衡：

```bash
tar -cf - folder | zstd -T0 -3 -o folder.tar.zst
```

---

## 为什么不优先使用 zip -r

当然也可以使用：

```bash
zip -r backup.zip folder/
```

但如果主要目标是 Linux 服务器上的大规模文件归档，`tar` 通常更自然。

主要原因包括：

- `tar` 本身就是 Unix/Linux 文件树归档工具
- 对权限、用户组、软链接等 Unix 元数据处理更加自然
- 可以先将海量小文件转换成连续数据流，再整体交给现代压缩器
- 可以灵活搭配 `zstd`、`gzip`、`xz` 等压缩工具

ZIP 的一个优点是单个文件可以相对独立地访问，因此如果主要考虑跨平台兼容性、Windows 用户使用，ZIP 仍然有它的价值。

但对于 Linux 服务器备份、迁移、大量小文件归档，通常更推荐：

```text
tar + zstd
```

---

## 不要直接用 shell 通配符处理几十万个文件

例如不建议这样：

```bash
gzip folder/*
```

当文件数量特别多时，Shell 会先把 `*` 展开成大量参数，然后再启动命令。

有可能遇到：

```text
Argument list too long
```

也就是参数列表超过系统允许的上限。

`tar` 直接遍历目录就不存在这种通配符一次性展开几十万个文件名的问题。

---

## 如果文件本身已经压缩过怎么办

如果目录中的主要内容已经是压缩格式，例如：

```text
.jpg
.jpeg
.png
.mp4
.mkv
.zip
.gz
.zst
.parquet
```

那么再次使用高等级压缩，收益通常不会太大。

这种情况下甚至可以只使用 `tar` 打包：

```bash
tar -cf backup.tar folder/
```

这样可以避免额外的 CPU 压缩开销。

如果仍然希望最终文件具有压缩封装，也可以使用较低等级：

```bash
tar -cf - folder | zstd -T0 -1 -o folder.tar.zst
```

---

## 关于绝对路径

如果直接执行：

```bash
tar -cf backup.tar /path/to/folder
```

GNU tar 通常会提示类似：

```text
tar: Removing leading `/' from member names
```

这是正常的安全行为，它会去掉归档成员名前面的 `/`，避免解压时直接覆盖系统的绝对路径。

如果希望归档内部路径更加干净，可以使用 `-C`：

```bash
tar -C /path/to -cf - folder | zstd -T0 -3 -o backup.tar.zst
```

这样归档中的顶层目录就是：

```text
folder/
```

而不是：

```text
path/to/folder/
```

对于备份文件来说，这种写法通常更加整洁。

---

## 解压 .tar.zst

如果 GNU tar 支持 zstd，可以直接：

```bash
tar -I zstd -xf folder.tar.zst
```

也可以明确写成数据流：

```bash
zstd -dc folder.tar.zst | tar -xf -
```

如果想解压到指定目录：

```bash
mkdir restore
zstd -dc folder.tar.zst | tar -xf - -C restore
```

---

## 实际推荐命令

### 场景一：追求最快归档速度

```bash
tar -C /path/to -cf - folder | zstd -T0 -1 -o folder.tar.zst
```

适合：

- 几十万个小文件
- SSD / HDD / NAS 上快速归档
- CPU 压缩率不是最重要

---

### 场景二：速度和压缩率平衡

```bash
tar -C /path/to -cf - folder | zstd -T0 -3 -o folder.tar.zst
```

这是比较通用的一档。

---

### 场景三：喜欢简洁命令

```bash
tar -C /path/to -I 'zstd -T0 -3' -cf folder.tar.zst folder
```

效果和管道方式基本一致，但命令更紧凑。

---

### 场景四：数据本身已经压缩

```bash
tar -C /path/to -cf folder.tar folder
```

如果大部分内容都是视频、图片、压缩包等，可能没必要继续花大量 CPU 再压一次。

---

## 总结

对于 Linux 下包含几十万甚至上百万文件的目录，比较推荐的思路是：

```text
大量小文件
    ↓
tar 顺序归档
    ↓
zstd 多线程压缩
    ↓
单个 .tar.zst 文件
```

最实用的两条命令是：

```bash
# 更快
tar -C /path/to -cf - folder | zstd -T0 -1 -o folder.tar.zst

# 更均衡
tar -C /path/to -cf - folder | zstd -T0 -3 -o folder.tar.zst
```

而：

```bash
tar -I 'zstd -T0 -3' ...
```

与：

```bash
tar ... | zstd -T0 -3 ...
```

本质上没有明显性能差别，主要区别只是调用方式和使用灵活性。

对于海量小文件场景，真正应该重点关注的往往是**文件系统 metadata / inode / I/O 性能**，而不是纠结 `tar -I` 和 Shell 管道之间那一点点差异。

---

**完**
