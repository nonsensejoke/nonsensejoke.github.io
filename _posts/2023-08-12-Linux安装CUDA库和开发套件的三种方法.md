---
layout: default
title: "Linux安装CUDA库和开发套件的三种方法"
date: 2023-08-12 10:00:00 +0800
categories: [Linux]
tags: [CUDA, NVIDIA, Linux, Deep Learning]
---

CUDA是英伟达显卡适配的计算编程平台，在日常的应用和开发中经常使用，特别是在机器学习领域。

本文总结了三种在Linux系统下安装CUDA的方法。

<!--more-->

## 前提

在安装CUDA软件之前，请先确保：
1.  你的机器使用的是NVIDIA显卡（如果是AMD的显卡，无法使用CUDA）；
2.  显卡的驱动是否已经安装（可以通过 `nvidia-smi` 命令检查显卡状态和驱动版本号 Driver Version）。

假如以上两个条件都满足了，下面我们可以开始安装合适的CUDA版本了。

**需要注意的是：** CUDA的版本选择取决于驱动程序的版本（见下表）。也就是说太老版本的驱动运行不了太新的CUDA。

| CUDA版本 | 最低驱动版本 | GCC推荐版本 | Intel编译器推荐版本 |
| :--- | :--- | :--- | :--- |
| CUDA 12.1.x | >=530.30.02 | 12.2 | 2021.7 |
| CUDA 12.0.0 | >=525.85.05 | 12.1 | 2021.6 |
| CUDA 11.8.x | >=520.61.05 | 11 | 2021 |
| CUDA 11.7.x | >=515.48.07 | 11 | 2021 |
| CUDA 11.6.x | >=510.47.03 | 11 | 2021 |
| CUDA 11.5.x | >=495.29.05 | 11 | 2021 |
| CUDA 11.4.x | >=470.82.01 | 10.3 | 19.1 |
| CUDA 11.3.x | >=465.19.01 | 10.3 | 19.1 |
| CUDA 11.2.x | >=460.32.03 | 10.3 | 19.1 |
| CUDA 11.1 (11.1.0) | >=455.32 | 10.3 | 19.1 |
| CUDA 11.0 (11.0.3) | >=450.51.06 | 10.3 | 19.1 |
| CUDA 10 | >= 440.33 | 10.2 | 18.0 |
| CUDA 9 | >= 396.37 | 4.8.5 | 17.0 |
| CUDA 8 | >= 375.26 | 4.8.2 | 15, 16 |

## 安装CUDA

### 方法一：通过Linux发行版自带的软件包管理器

以Ubuntu为例，我们可以使用以下命令：

```bash
sudo apt update
sudo apt install nvidia-cuda-toolkit
```

这种方法比较简便，而且库函数和编译器可执行程序(如 `nvcc`) 的路径已经自动配置加载好了。我们可以使用 `nvcc -V` 来检查当前CUDA的版本。

**缺点**：一般情况下不能选择特定的CUDA版本（但也有例外，比如Pop OS是支持的）。

---

### 方法二：通过nvidia官网的可执行脚本安装（推荐）

*   官网的最新版CUDA地址：[https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads)
*   老版本CUDA的下载地址：[https://developer.nvidia.com/cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive)

**1. 下载**
CUDA的版本选择请参考上面的表格。
选择 `Linux` -> `合适的处理器架构` -> `发行版名称和版本` -> `runfile(local)`，下载得到一个 `.run` 文件。这个文件包含了驱动程序和CUDA，我们只需要其中的CUDA。

**2. 安装**
将这个文件（这边以 `cuda_11.6.0_510.39.01_linux.run` 为例）放到普通用户的目录下之后，我们运行：

```bash
# 注意：将USER替换成你的用户名
sh cuda_11.6.0_510.39.01_linux.run --toolkit --override --installpath=/home/USER/cuda/cuda11.6
```

其中 `installpath` 是cuda的目标安装目录。

*   询问是否接受条款，输入 `accept`。


*   在新页面中，用**空格键**将 `Driver` 前的 **[X]** 去掉（取消勾选驱动），`CUDA Toolkit` 前的 **[X]** 必须保留，其他的看情况去掉。

 
*   然后进入 `Options` -> `Toolkit Options`。

 

*   用空格键将下面这些选项的 **[X]** 去掉，然后选择 `Done` 两次（防止它们被安装到根目录）。

 

*   回到最开始的页面，选择 `Install`！

 
**3. 配置环境变量**
安装完成之后，会显示 Summary，并告诉我们安装路径（例如 `/home/USER/cuda/cuda11.6/`）。

假如我们需要使用这个CUDA的时候，可以依次运行下面三个命令：

```bash
export CUDATK=/home/USER/cuda/cuda11.6
export PATH=/home/USER/cuda/cuda11.6/bin:${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/home/USER/cuda/cuda11.6/lib64:${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

此时假如我们运行 `nvcc -V`：

 

显示cuda编译器的版本为11.6，说明已经安装成功了。

我们也可以将上面三个命令放入一个 bash 脚本，在需要用 CUDA 的时候 `source` 一下就可以加载了。这种安装方法可以在用户目录安装不同版本的 CUDA，非常灵活方便。

---

### 方法三：通过miniconda/conda安装CUDA（推荐）

conda 可以在用户目录下创建不同的 python 环境， 我们可以为不同 CUDA 版本创建不同的 conda 环境。

**需要注意的是：** 在 conda 下面我们需要区分 `cuda` 和 `cudatoolkit` 这两个包。

*   `cuda` 是 nvidia 频道的包：[https://anaconda.org/nvidia/cuda](https://anaconda.org/nvidia/cuda)
*   `cudatoolkit` 是 anaconda 频道的包：[https://anaconda.org/anaconda/cudatoolkit](https://anaconda.org/anaconda/cudatoolkit)

> 后者 `cudatoolkit` 是不包含 CUDA 编译器的（如 nvcc）。可以将后者理解为前者的子集。

我们以 CUDA 11.6 为例，运行：

```bash
conda create --name cuda-11.6
conda activate cuda-11.6
# 注意：版本号 11.6.2 可以在cuda的conda页面查到
conda install -y -c "nvidia/label/cuda-11.6.2" cuda
```

这样安装好之后，我们只需要加载适当的 conda 环境即可使用了。

## 总结

本文介绍了三种不同的安装CUDA的方法，其中第二和第三种可以在普通用户目录安装不同版本的CUDA，非常灵活方便。
