---
layout: default
title: "🐳 Apptainer 编译镜像时 home 空间不足怎么办"
date: 2026-05-07 00:00:00 +0800
categories: [Linux, HPC]
tags: [apptainer, singularity, hpc, linux, container]
description: "通过环境变量将 Apptainer 的缓存和临时目录重定向到空间充足的挂载盘，解决 home 分区不足的问题。"
---

在 HPC 集群或个人服务器上编译 Apptainer 镜像时，经常会遇到 home 分区空间不足的问题。原因在于 Apptainer 默认将缓存（cache）和临时文件（tmp）都写到 `$HOME/.apptainer/` 下，稍大一点的镜像就会把 home 配额撑爆。

解决思路很简单：**把这两个目录重定向到附加磁盘**。

---

## 🔍 先确认挂载盘路径

```bash
df -h
```

找到空间充足的那行对应的挂载点，例如 `/data` 或 `/mnt/extra`，后续用它替换示例中的 `/path/to/bigdisk`。

---

## ✅ 方法一：临时设置环境变量（当次终端生效）

```bash
export APPTAINER_CACHEDIR=/path/to/bigdisk/.apptainer/cache
export APPTAINER_TMPDIR=/path/to/bigdisk/.apptainer/tmp

mkdir -p $APPTAINER_CACHEDIR $APPTAINER_TMPDIR

apptainer build myimage.sif mydef.def
```

适合偶尔使用，不污染长期配置。

---

## 🔧 方法二：写入 `~/.bashrc`（永久生效）

```bash
echo 'export APPTAINER_CACHEDIR=/path/to/bigdisk/.apptainer/cache' >> ~/.bashrc
echo 'export APPTAINER_TMPDIR=/path/to/bigdisk/.apptainer/tmp'  >> ~/.bashrc
source ~/.bashrc
```

之后所有 `apptainer build` 都会自动使用新路径，无需每次手动 export。

---

## 📦 方法三：把输出 `.sif` 也放到大盘

编译产出的镜像文件本身可能也很大，直接指定输出路径：

```bash
apptainer build /path/to/bigdisk/myimage.sif mydef.def
```

---

## 🧹 顺手清理旧缓存，释放 home 空间

如果 home 下已经积累了大量旧缓存，可以先清一下：

```bash
# 官方命令（推荐）
apptainer cache clean

# 或手动删除
rm -rf ~/.apptainer/cache
```

---

## 📝 小结

| 环境变量 | 作用 |
|---|---|
| `APPTAINER_CACHEDIR` | 镜像层缓存目录（拉取远程镜像时产生） |
| `APPTAINER_TMPDIR` | 编译过程中的临时文件目录 |

两个变量配合使用，基本能解决 home 空间不足的问题。如果集群有专用的 scratch 目录（如 `/scratch/$USER`），优先用那里，通常不计入 home 配额且 I/O 更快。

---

**完**
