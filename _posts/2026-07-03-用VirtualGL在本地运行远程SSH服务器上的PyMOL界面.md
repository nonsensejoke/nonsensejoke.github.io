---
layout: default
title: "用 VirtualGL 在本地运行远程 SSH 服务器上的 PyMOL 界面（免 VNC）"
date: 2026-07-02 12:00:00 +0800
categories: [远程可视化, 工具]
tags: [virtualgl, pymol, egl, ssh, macos]
description: "不借助 VNC，用 VirtualGL 的 EGL 后端把远程 SSH 服务器上 GPU 渲染好的 PyMOL 界面直接投到本地 Mac，附 GPU 选择与带宽压缩调优。"
---

想在本地 Mac 上操作跑在远程 SSH 服务器上的 PyMOL，最常见的方案是 VNC，但要起虚拟桌面、连着一层桌面环境，略显笨重。其实 VirtualGL 自带一条**不依赖 VNC** 的路子：服务器用 GPU 把 PyMOL 的 OpenGL 画面渲染好，压缩后通过网络直接发回本地，PyMOL 窗口就像本地窗口一样出现在 Mac 屏幕上。本文记录一套走通的最小流程，用的是 VirtualGL 的 **EGL 后端**，省掉了传统 `vglserver_config` 那一整套 X server 配置。

---

## 0. 前提

- 远程服务器有 GPU 且驱动就绪（`nvidia-smi` 有正常输出）。EGL 后端直接通过 GPU 的 DRI 设备做离屏渲染，不需要服务器上跑真实的 `:0` X server。
- 本地 Mac 需要一个 X server（XQuartz）来承接画面。

---

## 1. 本地 Mac 安装

**VirtualGL（作为客户端）**：从官方 releases 页下载对应架构的安装包。

- 下载地址：<https://github.com/VirtualGL/virtualgl/releases/>

**XQuartz**：Mac 客户端本质是个 X11 程序，必须装 XQuartz 来提供本地 X server。

- 下载地址：<https://www.xquartz.org/>

装完 XQuartz 后建议注销重登一次，确保 `DISPLAY` 环境变量生效。

---

## 2. 远程 SSH 服务器安装

在服务器上同样从 releases 页下载安装 VirtualGL（作为服务端）：

- 下载地址：<https://github.com/VirtualGL/virtualgl/releases/>

用 EGL 后端时不需要运行 `vglserver_config`，装好即可。

---

## 3. 连接并运行

在**本地 Mac** 上打开终端（XQuartz 已在运行、`DISPLAY` 指向它），用 `vglconnect` 登录远程主机：

```bash
/opt/VirtualGL/bin/vglconnect user@remote_host
```

`vglconnect` 会把 `vglclient` 和 SSH 包在一起，登录的同时在本地起好接收画面的客户端。登录成功后，在这个 shell 里用 EGL 后端启动 PyMOL：

```bash
vglrun -d egl pymol
```

PyMOL 界面就会显示在本地 Mac 上，而计算和渲染都发生在远程服务器的 GPU 上。

---

## 4. 指定使用哪块 GPU

多卡机器可以用 `-d egl0`、`-d egl1`、`-d egl6` 这样的编号指定具体的渲染设备，把渲染分散到不同的卡：

```bash
vglrun -d egl0 pymol      # 用第 0 块卡
vglrun -d egl6 pymol      # 用第 6 块卡
```

编号可以对照服务器上 `nvidia-smi` 看到的卡序来选。

> 注意是 `egl` 不是 `elg`，别拼反了。

---

## 5. 带宽紧张时的压缩调优

网络不好、拖动卡顿时，可以让 VirtualGL 把回传的画面压得更狠：

```bash
vglrun -d egl -q 30 -samp 4x pymol
```

两个参数的含义：

| 参数 | 作用 | 取值 |
|------|------|------|
| `-q` | JPEG 质量，越低越省带宽、块状感越明显 | 1–100，默认 95；差网络可用 30，一般网络 50–80 |
| `-samp` | 色度子采样，控制颜色信息压缩程度 | `1x` 全彩最清晰 / `2x` 折中 / `4x` 最省带宽 |

对分子这类大面积平滑着色的画面，`-samp 4x` 通常看不太出差别；但要看**精细彩色标注、细文字**时可以临时切回 `1x` 保证色彩锐利。

运行中还能按 `Ctrl+Shift+F9` 调出面板实时改这些参数，不用重启 PyMOL——平时用激进压缩图流畅，要仔细看时临时拉高质量即可。

---

## 6. 两个可选增强

**防火墙 / NAT 环境连不上**：默认 `vglconnect` 只把 X11 走 SSH 隧道，画面流是直连回本地的。如果本地在 NAT 后面导致连不上，加 `-s` 把画面流也塞进 SSH 隧道：

```bash
/opt/VirtualGL/bin/vglconnect -s user@remote_host
```

代价是局域网下可能损失 20–40% 性能，公网基本无感。

**懒得每次敲 vglrun**：用 `vglconnect -s -v` 登录，之后 shell 里直接 `pymol` 就是加速的，省掉前缀。

---

## 7. 出正式图片建议走 ray

交互靠 VirtualGL，但要导出正式图片时，更推荐 PyMOL 自带的光线追踪：

```python
ray 1920, 1080
png /path/to/output.png, dpi=300
```

`ray` 是在服务器端完整渲染成文件的，完全不经过 VGL 的有损压缩，画质和本地跑没有区别。**交互靠 VGL、出图靠 ray**，两边各取所长。

---

## 小结

免 VNC 远程跑 PyMOL 的最小流程：

1. 本地装 VirtualGL 客户端 + XQuartz，服务器装 VirtualGL 服务端（EGL 后端免配置）
2. `vglconnect user@remote_host` 登录
3. `vglrun -d egl pymol` 启动，多卡用 `-d egl0`/`-d egl6` 指定
4. 卡就上 `-q 30 -samp 4x`，出图改用 `ray` + `png`

核心优势是省掉了 VNC 的虚拟桌面和 `vglserver_config` 配置，窗口无缝、上手快。

---

**完**
