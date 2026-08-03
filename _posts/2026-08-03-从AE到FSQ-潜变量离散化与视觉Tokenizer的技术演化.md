---
layout: default
title: "从 AE、VAE 到 VQ-VAE、FSQ：潜变量离散化与视觉 Tokenizer 的技术演化"
date: 2026-08-01 09:30:00 +0800
categories: [机器学习, 生成模型]
tags: [autoencoder, vae, vq-vae, fsq, lfq, bsq, visual-tokenizer, quantization]
use_math: true
description: "AE、VAE、VQ-VAE 与 FSQ 之间究竟是什么关系？本文从潜空间约束、离散量化、码本坍塌与量化几何出发，梳理 LFQ、BSQ、RFSQ、SimVQ、SoftVQ 和格点量化等后续路线。"
---

如果只看名字，很容易把下面这串缩写理解成一条不断升级的版本链：

```text
AE → VAE → VQ-VAE → FSQ → ?
```

但这条线实际上混合了三件不同的事：

1. **自编码器如何组织潜空间**；
2. **潜变量是连续的还是离散的**；
3. **离散化时使用什么量化器**。

因此，FSQ 后面并不存在一个公认的、唯一的“下一代”。更准确的理解是：从 VQ-VAE 开始，技术路线逐渐分叉成了**可学习码本、固定隐式码本、球面量化、残差量化、格点量化，以及重新连续化**等多个方向。

> **先说结论：**
>
> - 从量化几何看，可以粗略记成  
>   **VQ-VAE → FSQ / LFQ → BSQ → 更一般的球面格点量化**；
> - 从工程能力看，FSQ 后面可以接  
>   **RFSQ 等残差式 FSQ**；
> - 从整个视觉 Tokenizer 领域看，  
>   **SimVQ、SoftVQ、语义对齐 Tokenizer** 都是同时发生的平行路线，而不是 FSQ 的“后代”。

---

## 一张图看懂这条技术脉络

```text
Autoencoder
│
├── 连续潜空间
│   │
│   ├── AE
│   │    └── 确定性压缩与重建
│   │
│   └── VAE
│        ├── 概率潜变量
│        ├── KL 正则与可采样潜空间
│        └── 连续 Tokenizer / SoftVQ
│
└── 离散潜空间
    │
    └── VQ-VAE
         ├── VQGAN：感知损失与对抗训练
         ├── RQ-VAE / RVQ：多级残差量化
         ├── SimVQ：保留可学习码本，但改善码本优化
         │
         └── 无显式可学习码本
              ├── FSQ：多值标量量化
              │    └── RFSQ：多级残差 FSQ
              │
              ├── LFQ：逐维二值量化
              │
              └── BSQ：球面上的二值量化
                   ├── QLIP：加入语言语义对齐
                   └── Spherical Lattice Quantization
                        └── Leech lattice 等更优格点
```

这里最重要的认识是：

> **AE、VAE 描述的是潜变量模型；VQ、FSQ、LFQ、BSQ 描述的主要是量化方式。**

FSQ 并不是把整个 VQ-VAE 架构推倒重来，而是把其中最麻烦的 **Vector Quantization 模块**换成了更简单的标量量化器。

---

## 1. AE：先学会压缩，再学会重建

最基本的 Autoencoder 由编码器和解码器组成：

$$
z = E_\phi(x), \qquad \hat{x} = D_\theta(z)
$$

训练目标通常只是最小化重建误差：

$$
\mathcal{L}_{AE} = d(x, \hat{x})
$$

其中，$d$ 可以是均方误差、交叉熵、感知损失，或者多种重建损失的组合。

AE 的优点很直接：

- 可以把高维数据压缩到低维表示；
- 能够学习任务相关的特征；
- 训练目标简单、稳定。

但普通 AE 的潜空间通常没有明确的几何约束。两个样本在潜空间中相邻，并不保证它们之间的插值仍然对应合理数据；从某个随机位置采样，也不一定能解码出有效样本。

因此，AE 更像一个**非线性压缩器**，还不是一个结构良好的生成模型。

---

## 2. VAE：把潜空间变成可采样的概率空间

Variational Autoencoder 不再让编码器只输出一个确定向量，而是输出近似后验分布的参数：

$$
q_\phi(z \mid x)
=
\mathcal{N}
\left(
\mu_\phi(x),
\operatorname{diag}\left(\sigma_\phi^2(x)\right)
\right)
$$

通过重参数化技巧进行采样：

$$
z = \mu_\phi(x) + \sigma_\phi(x) \odot \epsilon,
\qquad
\epsilon \sim \mathcal{N}(0, I)
$$

其典型目标可以写成：

$$
\mathcal{L}_{VAE}
=
\underbrace{
\mathbb{E}_{q_\phi(z \mid x)}
[-\log p_\theta(x \mid z)]
}_{\text{重建项}}
+
\beta
\underbrace{
D_{\mathrm{KL}}
\left(
q_\phi(z \mid x)
\Vert
p(z)
\right)
}_{\text{潜空间正则}}
$$

VAE 的关键贡献不是“重建得更清楚”，而是让潜空间具有更规则的概率结构：

- 可以从先验分布直接采样；
- 样本之间可以平滑插值；
- 潜空间更适合生成建模。

代价是，分布正则会与精确重建发生冲突。KL 权重过强时，模型可能忽略潜变量，出现 posterior collapse；在像素级生成中，结果也常比普通 AE 更平滑、更模糊。[^vae]

---

## 3. VQ-VAE：把连续潜变量变成离散 Token

VQ-VAE 把连续潜表示映射到一个可学习码本：

$$
\mathcal{E}
=
\left\{
e_1, e_2, \ldots, e_K
\right\},
\qquad
e_j \in \mathbb{R}^{d}
$$

编码器得到连续表示 $z_e(x)$ 后，在码本中寻找最近的向量：

$$
k^*
=
\arg\min_j
\left\|
z_e(x) - e_j
\right\|_2^2
$$

量化结果为：

$$
z_q(x) = e_{k^*}
$$

这样，每个空间位置不再对应任意实数向量，而是对应一个整数编号 $k^*$。图像、音频或视频因此可以被转换成一串离散 Token，再交给 Transformer、PixelCNN 或其他先验模型学习。[^vqvae]

这一变化非常重要：

```text
连续特征图
    ↓
离散 Token 序列
    ↓
像语言模型一样建模
```

VQ-VAE 也绕开了传统 VAE 中常见的 posterior collapse：潜变量必须通过离散码传递，强大的解码器不能轻易把它完全忽略。

### VQ-VAE 的代价

VQ-VAE 的核心问题都集中在可学习码本上：

- **码本坍塌**：只有少量 code 被频繁使用；
- **死码**：部分 code 长期没有样本命中；
- **优化不对称**：一次前向传播只更新少数被选中的向量；
- **额外损失**：通常需要 codebook loss 和 commitment loss；
- **大码本检索成本**：最近邻搜索随码本增大而变重；
- **训练技巧较多**：EMA 更新、重置死码、熵正则、低维投影等。

后来出现的 VQGAN、RQ-VAE 等方法，并没有抛弃 VQ，而是分别增强了感知质量和多级量化能力。[^vqgan] [^rqvae]

---

## 4. FSQ：不是学习一个码本，而是定义一个坐标网格

Finite Scalar Quantization 的思路非常朴素：

1. 先把编码器输出投影到较低维度；
2. 每个维度独立量化；
3. 每个维度只能取有限个固定值；
4. 所有维度取值的笛卡尔积，就是隐式码本。

假设第 $i$ 个维度有 $L_i$ 个可选值，那么总码本大小为：

$$
|\mathcal{C}|
=
\prod_{i=1}^{d} L_i
$$

例如：

```text
维度数 d = 3
每维取值数 = [5, 5, 4]

隐式码本大小 = 5 × 5 × 4 = 100
```

FSQ 并不存储一个 $100 \times d$ 的可学习码本矩阵。它只定义三个标量轴上的有限取值，然后把三维组合编码成一个 Token ID。

可以把它理解成：

```text
VQ：从一张可学习的词典中找最近单词
FSQ：把坐标分别四舍五入，再组合成单词编号
```

### FSQ 为什么有效

FSQ 用一个结构化、固定的隐式码本，直接避开了传统 VQ 最麻烦的部分：

- 不需要最近邻码本检索；
- 不需要单独训练 code vector；
- 不需要 commitment loss；
- 不存在“某个 code 从未被更新”的经典死码问题；
- 码本大小可通过各维级数灵活组合。

FSQ 论文在图像生成、深度估计、着色和全景分割等任务中展示了与 VQ 方案相当的性能，同时显著简化了训练流程。[^fsq]

但 FSQ 也不是没有代价。它的量化点位于轴对齐的笛卡尔网格上，几何结构由各坐标轴独立决定。当真实潜变量分布具有强相关性、球面结构或更复杂的局部几何时，这种规则网格未必是最有效的空间划分。

---

## 5. AE → VAE → VQ-VAE → FSQ，究竟每一步改变了什么？

| 方法 | 潜变量 | 核心机制 | 主要收益 | 主要问题 |
|---|---|---|---|---|
| **AE** | 连续、确定性 | 编码后直接重建 | 简单、重建稳定 | 潜空间不规则，难以采样 |
| **VAE** | 连续、概率性 | 近似后验 + KL 正则 | 潜空间可采样、可插值 | 重建与正则冲突，可能 posterior collapse |
| **VQ-VAE** | 离散 | 可学习向量码本 + 最近邻 | 得到可供语言模型建模的 Token | 码本坍塌、死码、训练复杂 |
| **FSQ** | 离散 | 逐维有限标量量化 | 无显式码本、结构简单、训练稳定 | 轴对齐网格的表达效率有限 |

因此，这条路径不是单纯的“模型越来越强”，而是研究重点不断变化：

```text
AE：怎样压缩？
VAE：怎样让潜空间可采样？
VQ-VAE：怎样得到离散 Token？
FSQ：怎样不用麻烦的可学习码本，也能得到离散 Token？
```

---

## 6. FSQ 下面是什么？

### 6.1 LFQ：与 FSQ 同期出现的二值路线

Lookup-Free Quantization（LFQ）在 MAGVIT-v2 中被系统化使用。它同样不维护传统的显式码本，而是把每个潜变量维度量化为两个状态。若有 $d$ 个二值维度，隐式词表大小就是：

$$
|\mathcal{C}| = 2^d
$$

LFQ 与 FSQ 的共同点是：

- 都不依赖传统的最近邻码本查找；
- 都能通过低成本结构构造大词表；
- 都试图绕开 VQ 的码本利用率问题。

差别在于：

| 方法 | 每维状态数 | 典型结构 |
|---|---:|---|
| **FSQ** | 多值 | 小维度、多级标量 |
| **LFQ** | 二值 | 多个 bit 组合成大词表 |

时间上，FSQ 在 2023 年 9 月公开，采用 LFQ 的 MAGVIT-v2 在 2023 年 10 月公开。二者应当视为**几乎同期的平行探索**，而不是严格的前后代关系。MAGVIT-v2 还通过熵相关的正则项促进 Token 使用，并展示了统一图像与视频 Tokenizer 的能力。[^magvit2]

---

### 6.2 BSQ：从轴对齐量化走向球面几何

Binary Spherical Quantization（BSQ）先把低维潜表示投影或归一化到超球面，再进行二值量化。一个直观写法是：

$$
q(z)
=
\frac{\operatorname{sign}(z)}{\sqrt{d}}
$$

量化后的点不再只是普通超立方体顶点，而是落在单位超球面上的二值方向。

与 FSQ 相比，BSQ 的变化不只是“每维改成二值”，更重要的是它显式引入了**球面几何**：

```text
FSQ：轴对齐的多值网格
LFQ：超立方体上的二值顶点
BSQ：归一化到超球面的二值码
```

BSQ 仍然没有显式可学习码本，同时支持很大的隐式词表。原论文将其用于统一的图像与视频 Tokenizer，并报告了良好的重建、生成和压缩性能。[^bsq]

如果一定要给“FSQ 下一步”找一个最直观的名字，BSQ 是很合理的候选：

> **FSQ 解决码本学习问题；BSQ 进一步追问固定量化点应该具有怎样的几何结构。**

不过它仍然不是 FSQ 的官方继任者，两者更像是在不同结构假设下设计的非参数量化器。

---

### 6.3 RFSQ：把 FSQ 扩展到多级残差量化

单层量化器必须一次性完成全部近似。为了提高码率—失真表现，可以像 Residual Vector Quantization 一样逐层量化残差：

$$
r_0 = z
$$

$$
q_t = Q_t(r_{t-1}),
\qquad
r_t = r_{t-1} - q_t
$$

最终：

$$
\hat{z}
=
\sum_{t=1}^{T} q_t
$$

但直接堆叠 FSQ 会遇到一个问题：随着层数增加，残差幅度越来越小，后续量化层无法充分使用自己的离散范围。

Robust Residual Finite Scalar Quantization（RFSQ）通过可学习缩放或可逆归一化，缓解 residual magnitude decay，让多阶段 FSQ 真正发挥作用。[^rfsq]

因此，在神经压缩、音频 Codec 或可变码率场景中，一条更贴近工程需求的路线是：

```text
VQ-VAE
  → RVQ / RQ-VAE
  → FSQ
  → Residual FSQ / RFSQ
```

这条线关注的不是量化点几何，而是**如何用多个离散层逐步逼近连续信号**。

---

### 6.4 球面格点量化：从二值超立方体走向更优空间填充

BSQ 的码点本质上仍来自二值超立方体，只是经过球面归一化。进一步的问题是：

> 二值顶点是否是球面上最均匀、最高效的码点排列？

Spherical Leech Quantization 从 lattice coding 的角度统一解释非参数量化，并尝试使用更高效的格点结构。其核心候选之一是 24 维 Leech lattice：

$$
\Lambda_{24}
$$

Leech lattice 以高对称性和高密度球堆积著名。相关工作报告称，基于该格点的球面量化在图像重建、压缩和自回归生成中优于 BSQ，同时可以简化部分训练设计。[^leech]

从几何角度，可以把这条路线概括成：

```text
FSQ
轴对齐矩形格点
    ↓
LFQ / BSQ
二值超立方体与球面码
    ↓
Spherical Lattice Quantization
寻找更均匀、更高效的球面格点
```

这可能是 FSQ、LFQ、BSQ 之后最值得关注的理论方向之一：研究重点不再只是“是否需要学习码本”，而是转向**固定码本应当具有什么几何结构**。

需要注意，RFSQ 和 Spherical Leech Quantization 在本文写作时仍属于较新的研究工作，不能简单等同于已经形成共识的行业标准。

---

## 7. 另一条路线：不放弃可学习码本

FSQ 的基本判断是：既然可学习码本带来了坍塌和训练复杂度，不如直接去掉它。

但另一派工作认为，问题并不在“码本可学习”本身，而在于传统 VQ 的优化方式。

### SimVQ：用共享线性空间更新整个码本

普通 VQ 中，一次前向传播只会直接更新被选中的少数 code。SimVQ 通过一个共享线性变换重新参数化码本，让所有 code 共享同一个可学习潜在基底。

这样，即使某个 code 当前没有被选中，它所在的整体线性空间仍能随着共享变换得到更新。SimVQ 在图像和音频实验中展示了改善表示坍塌与码本利用率的效果。[^simvq]

这条路线可以写成：

```text
VQ-VAE
  → 更稳定的码本更新
  → SimVQ / 共享参数化 / 动态码本
```

它与 FSQ 不是上下代，而是两种不同哲学：

```text
FSQ：去掉码本，消灭问题来源
SimVQ：保留码本，修复优化机制
```

---

## 8. 离散化也不一定是终点：SoftVQ 与连续 Tokenizer

视觉生成模型使用离散 Token 的一个重要原因，是可以直接借用语言模型的自回归建模能力。

但 Diffusion Transformer、Flow Matching 等模型并不要求输入一定是整数 Token。对于这些生成器，连续、平滑、完全可微的潜变量可能更合适。

SoftVQ-VAE 不再强制选择唯一 code，而是使用软类别后验，对多个 codeword 进行加权组合。得到的 Token 保留码本结构，但本身仍是连续向量。该方法强调以较少的一维连续 Token 表示图像，并提升基于去噪模型的训练与推理效率。[^softvq]

这形成了一个很有意思的“钟摆”：

```text
VAE
连续潜变量
    ↓
VQ-VAE
硬离散 Token
    ↓
FSQ / LFQ / BSQ
结构化硬离散 Token
    ↓
SoftVQ
带码本结构的连续 Token
```

这不是回到原点，而是因为下游生成器发生了变化：

| 下游模型 | 更自然的潜变量 |
|---|---|
| 自回归 Transformer / LLM | 离散 Token |
| Masked Token Model | 离散 Token |
| Diffusion Transformer | 连续潜变量通常更自然 |
| Flow Matching | 连续、平滑潜空间 |
| 神经压缩 / Codec | 离散或分层离散码 |

---

## 9. 前沿正在从“量化器”转向“Tokenizer 系统设计”

到 2025—2026 年，仅仅问“VQ、FSQ 还是 BSQ”已经不够。一个优秀 Tokenizer 还需要同时回答下面几个问题。

### 9.1 压缩率与重建质量

Token 数量越少，后续 Transformer 的计算成本越低；但压缩过度会损失纹理与局部结构。

这是典型的 rate–distortion 权衡：

$$
\min
\quad
D(x, \hat{x}) + \lambda R
$$

其中 $D$ 表示失真，$R$ 表示码率或 Token 成本。

### 9.2 Token 是否具有语义

只优化像素重建，得到的 Token 可能很擅长保存纹理，却不一定适合理解、检索和推理。

QLIP 在基于 BSQ 的自编码器上同时加入重建目标与语言—图像对齐目标，使同一套视觉 Token 同时服务于多模态理解和图像生成。[^qlip]

这反映了 Tokenizer 目标的变化：

```text
过去：把图像还原出来就够了
现在：Token 还要能被多模态模型理解
```

### 9.3 图像、视频和音频是否能共享接口

统一 Tokenizer 希望用相近的结构处理不同长度、不同维度的信号。MAGVIT-v2 和 BSQ 等工作都把图像与视频统一建模视为重要目标。[^magvit2] [^bsq]

### 9.4 Tokenizer 是否适配下游生成器

一个在重建指标上最好的 Tokenizer，不一定能带来最好的生成结果。

真正需要联合考虑的是：

```text
编码器
+ 量化器
+ 解码器
+ 先验模型
+ 训练目标
+ 采样方式
```

因此，未来的“下一代”很可能不是某个孤立的新量化公式，而是一套针对特定生成范式联合优化的 Tokenizer 系统。

---

## 10. 实际项目应该怎么选？

| 场景 | 更值得优先尝试的路线 | 原因 |
|---|---|---|
| 想快速替换传统 VQ，减少训练技巧 | **FSQ** | 简单、无显式码本、容易复现 |
| 需要超大离散词表，服务 AR 图像生成 | **LFQ / BSQ** | 二值组合天然支持大词表 |
| 图像与视频统一 Tokenizer | **LFQ / BSQ 系路线** | 已有统一视觉建模实践 |
| 神经压缩、音频 Codec、可变码率 | **RVQ / RQ-VAE / RFSQ** | 多阶段残差码更自然 |
| 希望保留可学习语义原型 | **VQ / SimVQ** | codeword 本身可以承载可学习结构 |
| 使用 Diffusion / Flow 作为生成器 | **连续 VAE / SoftVQ** | 连续潜空间更匹配去噪与流模型 |
| 同时做视觉理解与生成 | **语义对齐 Tokenizer，如 QLIP** | 重建 Token 与语言语义联合训练 |
| 研究更优非参数量化几何 | **球面格点量化** | 直接优化码点空间填充效率 |

一个实用原则是：

> 不要先问“最新方法是什么”，而要先问“下游模型需要什么形式的潜变量”。

---

## 11. 时间线：这条路径并不是直线

| 时间 | 方法 | 关键变化 |
|---:|---|---|
| 2013 | **VAE** | 用变分推断构造可采样的连续潜空间 |
| 2017 | **VQ-VAE** | 使用可学习向量码本获得离散表示 |
| 2020 | **VQGAN** | 用感知与对抗目标增强高分辨率重建 |
| 2022 | **RQ-VAE** | 使用多级残差量化缩短空间 Token 序列 |
| 2023-09 | **FSQ** | 用低维有限标量量化替代传统 VQ |
| 2023-10 | **MAGVIT-v2 / LFQ** | 用 lookup-free 二值量化构造大词表 |
| 2024-06 | **BSQ** | 将二值量化放到超球面几何中 |
| 2024-11 | **SimVQ** | 重新参数化可学习码本，改善表示坍塌 |
| 2024-12 | **SoftVQ-VAE** | 用软码本组合构造连续 Token |
| 2025-02 | **QLIP** | 将视觉重建与语言语义对齐结合 |
| 2025-08 | **RFSQ** | 解决多级 FSQ 的残差幅度衰减 |
| 2025-12 | **Spherical Leech Quantization** | 用更优球面格点改进非参数量化 |

时间线也进一步说明：FSQ 之后不是一条单线升级，而是多个问题被同时展开。

---

## 12. 最值得记住的版本

如果只想记住一条主线，可以写成：

```text
AE
→ VAE：让连续潜空间可采样
→ VQ-VAE：把潜空间变成离散 Token
→ FSQ：去掉显式可学习码本
→ BSQ：把固定量化点放到球面上
→ Lattice Quantization：寻找更优的量化几何
```

但如果要准确描述整个研究领域，更合理的图是：

```text
                          ┌→ SimVQ：修复可学习码本
                          │
VQ-VAE → VQGAN / RQ-VAE ──┼→ FSQ → RFSQ
                          │
                          ├→ LFQ
                          │
                          └→ BSQ → 球面格点量化

VAE / VQ-VAE ───────────────→ SoftVQ / 连续 Tokenizer

BSQ / 离散 Tokenizer ───────→ QLIP / 语义对齐 Tokenizer
```

---

## 小结

从 AE 到 FSQ，真正发生的变化可以概括为四步：

1. **AE** 学习压缩与重建；
2. **VAE** 给连续潜空间加入概率结构；
3. **VQ-VAE** 把潜变量离散化，使视觉、音频和视频可以像语言一样被 Token 模型处理；
4. **FSQ** 用规则的隐式码本替代可学习向量码本，显著简化离散化训练。

而 FSQ 之后，研究分成了几条路线：

- **LFQ / BSQ**：进一步扩大隐式词表，并改善量化几何；
- **RFSQ**：发展多级残差式标量量化；
- **球面格点量化**：寻找比二值超立方体更高效的固定码点；
- **SimVQ**：保留可学习码本，但重新设计它的优化方式；
- **SoftVQ**：针对 Diffusion 与 Flow 模型重新采用连续 Token；
- **QLIP 等语义对齐方法**：让 Token 同时服务于重建、理解与生成。

所以，“FSQ 下面是什么”的最准确回答不是某个单独缩写，而是：

> **从“是否使用可学习码本”，继续走向“量化点具有什么几何结构、是否需要分层、是否应保持连续，以及 Token 是否具有语义”的系统性竞争。**

---

## 参考文献

[^vae]: Diederik P. Kingma, Max Welling. [Auto-Encoding Variational Bayes](https://arxiv.org/abs/1312.6114), 2013.

[^vqvae]: Aaron van den Oord, Oriol Vinyals, Koray Kavukcuoglu. [Neural Discrete Representation Learning](https://arxiv.org/abs/1711.00937), 2017.

[^vqgan]: Patrick Esser, Robin Rombach, Björn Ommer. [Taming Transformers for High-Resolution Image Synthesis](https://arxiv.org/abs/2012.09841), 2020.

[^rqvae]: Doyup Lee et al. [Autoregressive Image Generation using Residual Quantization](https://arxiv.org/abs/2203.01941), 2022.

[^fsq]: Fabian Mentzer, David Minnen, Eirikur Agustsson, Michael Tschannen. [Finite Scalar Quantization: VQ-VAE Made Simple](https://arxiv.org/abs/2309.15505), 2023.

[^magvit2]: Lijun Yu et al. [Language Model Beats Diffusion — Tokenizer is Key to Visual Generation](https://arxiv.org/abs/2310.05737), 2023.

[^bsq]: Yue Zhao, Yuanjun Xiong, Philipp Krähenbühl. [Image and Video Tokenization with Binary Spherical Quantization](https://arxiv.org/abs/2406.07548), 2024.

[^simvq]: Yongxin Zhu, Bocheng Li, Yifei Xin, Linli Xu. [Addressing Representation Collapse in Vector Quantized Models with One Linear Layer](https://arxiv.org/abs/2411.02038), 2024.

[^softvq]: Hao Chen et al. [SoftVQ-VAE: Efficient 1-Dimensional Continuous Tokenizer](https://arxiv.org/abs/2412.10958), 2024.

[^qlip]: Yue Zhao et al. [QLIP: Text-Aligned Visual Tokenization Unifies Auto-Regressive Multimodal Understanding and Generation](https://arxiv.org/abs/2502.05178), 2025.

[^rfsq]: Xiaoxu Zhu. [Robust Residual Finite Scalar Quantization for Neural Compression](https://arxiv.org/abs/2508.15860), 2025.

[^leech]: Yue Zhao et al. [Spherical Leech Quantization for Visual Tokenization and Generation](https://arxiv.org/abs/2512.14697), 2025.

---

**完**
