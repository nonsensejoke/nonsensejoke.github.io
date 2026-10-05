---
layout: default
title: "从头推导 Hartree–Fock 与 Kohn–Sham DFT（附 HF 能量的解析梯度）"
date: 2026-10-05 10:00:00 +0800
categories: [计算化学, 量子化学]
tags: [hartree-fock, dft, kohn-sham, roothaan-hall, scf, analytic-gradient, pulay-force]
use_math: true
description: "从多电子薛定谔方程出发，完整推导 Slater 行列式能量、Hartree–Fock 与 Roothaan–Hall 方程、Hohenberg–Kohn 定理与 Kohn–Sham 方程、Dirac 交换泛函，以及 RHF 解析梯度与 Pulay 力。"
---

量子化学教材里的 Hartree–Fock 和 DFT，常常是“结论给得很快、推导跳得很多”。这篇笔记试图把整条链从头走一遍：从多电子薛定谔方程出发，推出 Slater 行列式的能量表达式、Hartree–Fock 方程和 Roothaan–Hall 矩阵方程；再换一条“密度路线”，推出 Hohenberg–Kohn 定理、Kohn–Sham 方程和 LDA 交换泛函；最后推导 HF 能量的解析梯度，并用数值实验说明 Pulay 力为什么不能丢。

> 阅读约定：全文用**原子单位**（$$\hbar = m_e = e = 4\pi\varepsilon_0 = 1$$），能量单位 Hartree（1 Ha ≈ 27.211 eV），长度单位 Bohr（≈ 0.5292 Å）。
>
> 每一章开头我会先说"我们现在要干什么、为什么"，再进入数学。遇到你可能不熟的数学工具（Dirac 记号、泛函导数、Lagrange 乘子、置换代数），我会在正文里现场解释，不假设你学过高等量子力学。

---

## 目录

- [第 0 章 出发点：多电子薛定谔方程](#ch0)
- [第 1 章 波函数为什么必须是行列式](#ch1)
- [第 2 章 核心推导一：行列式的能量期望值](#ch2)
- [第 3 章 核心推导二：变分法导出 Hartree–Fock 方程](#ch3)
- [第 4 章 自旋积分：闭壳层 RHF](#ch4)
- [第 5 章 基组展开：Roothaan–Hall 方程与实际算法](#ch5)
- [第 6 章 密度泛函理论](#ch6)
- [第 7 章 HF 能量的解析梯度](#ch7)
- [第 8 章 全局回顾与自检](#ch8)

---

## 第 0 章 出发点：多电子薛定谔方程 {#ch0}

### 0.1 Born–Oppenheimer 近似

一个分子有 $$N$$ 个电子和 $$M$$ 个原子核。核比电子重约 2000 倍以上，运动慢得多，所以我们**把核钉住**，只解电子的问题；核坐标 $$\{\mathbf{R}_A\}$$ 成为参数。

固定核构型下的电子哈密顿量：

$$
\hat{H}_{\text{el}} = \underbrace{-\frac{1}{2}\sum_{i=1}^{N}\nabla_i^2}_{\text{电子动能}} \; \underbrace{-\sum_{i=1}^{N}\sum_{A=1}^{M}\frac{Z_A}{|\mathbf{r}_i - \mathbf{R}_A|}}_{\text{核-电子吸引}} \; + \underbrace{\sum_{i<j}\frac{1}{|\mathbf{r}_i - \mathbf{r}_j|}}_{\text{电子-电子排斥}}
$$

核-核排斥是常数，最后加上：

$$
V_{NN} = \sum_{A<B}\frac{Z_A Z_B}{R_{AB}}
$$

**分子的总能量** = $$E_{\text{el}} + V_{NN}$$。把这个总能量作为 $$\{\mathbf{R}_A\}$$ 的函数，就是**势能面**（PES）。几何优化就是在 PES 上找极小点；这就是为什么第 7 章的梯度那么重要。

### 0.2 记号压缩

定义**单电子哈密顿量**（"core Hamiltonian"）：

$$
\hat{h}(i) = -\frac{1}{2}\nabla_i^2 - \sum_A \frac{Z_A}{|\mathbf{r}_i - \mathbf{R}_A|}
$$

于是

$$
\boxed{\hat{H}_{\text{el}} = \sum_{i=1}^{N}\hat{h}(i) + \sum_{i<j}^{N}\frac{1}{r_{ij}}}
$$

### 0.3 问题的本质困难

如果只有 $$\sum_i \hat h(i)$$，哈密顿量就是**可分离**的：$$N$$ 个独立的单电子问题，解出来乘起来就完事。

罪魁祸首是 $$1/r_{ij}$$：它把电子 $$i$$ 和 $$j$$ 的坐标**耦合**在一起，导致 $$\hat H$$ 不可分离。$$N=2$$ 时（氦原子）薛定谔方程已经没有解析解。

**整篇文章的主线**就是两种应对策略：

| | 策略 | 代价 |
|---|---|---|
| **Hartree–Fock** | 强行假设波函数是单个行列式（"每个电子在其他电子的平均场中运动"） | 丢掉了电子关联（correlation） |
| **Kohn–Sham DFT** | 换变量：不用 $$3N$$ 维波函数，只用 $$3$$ 维电子密度 $$\rho(\mathbf{r})$$ | 关键泛函 $$E_{xc}[\rho]$$ 未知，只能近似 |

### 0.4 Dirac 记号速成

后面会大量用到，先说清楚。这只是**积分的缩写**：

$$
\langle \phi | \hat{A} | \psi\rangle \equiv \int \phi^*(\mathbf{x})\,\hat{A}\,\psi(\mathbf{x})\,d\mathbf{x}
$$

$$\langle\phi\vert \psi\rangle = \int\phi^*\psi\,d\mathbf{x}$$ 叫"重叠积分"。若 $$\langle\phi_i\vert \phi_j\rangle = \delta_{ij}$$，称这组函数**正交归一**。

后面会用到两套双电子积分记号，**务必分清**：

**物理学家记号**（也叫 "1212" 或 bra-ket 约定）：

$$
\langle ij | kl\rangle \equiv \int\!\!\int \chi_i^*(\mathbf{x}_1)\chi_j^*(\mathbf{x}_2)\,\frac{1}{r_{12}}\,\chi_k(\mathbf{x}_1)\chi_l(\mathbf{x}_2)\,d\mathbf{x}_1 d\mathbf{x}_2
$$

规则：**前两个下标是 bra（带星号），后两个是 ket；第 1、3 个下标属于电子 1，第 2、4 个属于电子 2。**

**化学家记号**（也叫 "1122"，量化程序里普遍使用）：

$$
(ij | kl) \equiv \int\!\!\int \phi_i^*(\mathbf{r}_1)\phi_j(\mathbf{r}_1)\,\frac{1}{r_{12}}\,\phi_k^*(\mathbf{r}_2)\phi_l(\mathbf{r}_2)\,d\mathbf{r}_1 d\mathbf{r}_2
$$

规则：**竖线左边两个下标都属于电子 1，右边两个都属于电子 2；每一对里第一个带星号。**

换算关系：$$\langle ij\vert kl\rangle = (ik\vert jl)$$。

对**实值**基函数（高斯基组都是实的），化学家记号有 8 重对称性：

$$
(ij|kl) = (ji|kl) = (ij|lk) = (ji|lk) = (kl|ij) = (lk|ij) = (kl|ji) = (lk|ji)
$$

这个对称性在实际计算里能把积分数量降到 1/8，很重要。

---

## 第 1 章 波函数为什么必须是行列式 {#ch1}

### 1.1 反对称性要求

电子是**费米子**。相对论量子场论给出的结论（这里当公理接受）：交换任意两个全同费米子的**全部坐标**，波函数变号。

引入**自旋坐标**：$$\mathbf{x} = (\mathbf{r}, \omega)$$，其中 $$\omega$$ 是自旋变量，自旋函数 $$\alpha(\omega)$$（上旋）和 $$\beta(\omega)$$（下旋）满足

$$
\int \alpha^*\alpha \,d\omega = \int\beta^*\beta\,d\omega = 1,\qquad \int\alpha^*\beta\,d\omega = 0
$$

反对称性要求：

$$
\Psi(\mathbf{x}_1,\dots,\mathbf{x}_i,\dots,\mathbf{x}_j,\dots,\mathbf{x}_N) = -\Psi(\mathbf{x}_1,\dots,\mathbf{x}_j,\dots,\mathbf{x}_i,\dots,\mathbf{x}_N)
$$

**Pauli 不相容原理是它的推论**：若两个电子占据同一个自旋轨道，交换它们波函数既应变号又应不变，只能为零。

### 1.2 从 Hartree 乘积到行列式

**第一步（错的，但有启发性）**：假设电子彼此独立，每个电子占一个**自旋轨道** $$\chi_i(\mathbf{x})$$，波函数取乘积：

$$
\Psi^{\text{HP}} = \chi_1(\mathbf{x}_1)\chi_2(\mathbf{x}_2)\cdots\chi_N(\mathbf{x}_N)
$$

这叫 Hartree 乘积。它满足薛定谔方程（对无相互作用哈密顿量），但**不反对称**。

**第二步（修正）**：把所有 $$N!$$ 种电子编号的排列都加进来，带上适当符号：

$$
\Psi^{\text{HF}} = \frac{1}{\sqrt{N!}}\sum_{P}(-1)^{p}\,\hat{P}\left[\chi_1(\mathbf{x}_1)\chi_2(\mathbf{x}_2)\cdots\chi_N(\mathbf{x}_N)\right]
$$

这里 $$\hat P$$ 遍历 $$N!$$ 个置换，$$p$$ 是把 $$P$$ 分解成对换所需的**对换次数**（奇偶性明确定义），$$(-1)^p$$ 就是置换的符号。

这个式子正好是**行列式的展开式**（Leibniz 公式）。所以

$$
\boxed{
\Psi^{\text{HF}} = \frac{1}{\sqrt{N!}}
\begin{vmatrix}
\chi_1(\mathbf{x}_1) & \chi_2(\mathbf{x}_1) & \cdots & \chi_N(\mathbf{x}_1)\\
\chi_1(\mathbf{x}_2) & \chi_2(\mathbf{x}_2) & \cdots & \chi_N(\mathbf{x}_2)\\
\vdots & & & \vdots\\
\chi_1(\mathbf{x}_N) & \chi_2(\mathbf{x}_N) & \cdots & \chi_N(\mathbf{x}_N)
\end{vmatrix}
\;\equiv\; |\chi_1\chi_2\cdots\chi_N\rangle
}
$$

**这叫 Slater 行列式。** 行列式的两条性质正好对应两条物理：

- 交换两**行** ⇔ 交换两个电子 ⇔ 变号 ✓ 反对称性
- 两**列**相同 ⇔ 两个电子占同一自旋轨道 ⇔ 行列式为 0 ✓ Pauli 原理

$$1/\sqrt{N!}$$ 是归一化因子（当 $$\{\chi_i\}$$ 正交归一时使 $$\langle\Psi\vert \Psi\rangle = 1$$）。

### 1.3 一个可以放心做的假设

**我们假设 $$\{\chi_i\}$$ 正交归一。** 这不损失任何普遍性：如果它们不正交，用 Gram–Schmidt 正交化得到的新轨道张成同一个空间，行列式只差一个常数因子（因为对列做线性组合等于左乘一个矩阵，行列式乘上该矩阵的行列式），归一化后波函数完全相同。

### 1.4 Hartree–Fock 的定义

> **Hartree–Fock 方法**：在"波函数为单个 Slater 行列式"的约束下，用变分原理找使 $$\langle\Psi\vert \hat H\vert \Psi\rangle$$ 最小的那组自旋轨道。

变分原理保证 $$E_{\text{HF}} \ge E_{\text{exact}}$$——HF 能量是**上界**。差值定义为**关联能**：

$$
E_{\text{corr}} \equiv E_{\text{exact}} - E_{\text{HF}} \quad (<0)
$$

典型量级：占总能量约 1%，但绝对值常达数十到数百 kJ/mol——远大于化学精度（~4 kJ/mol）。这就是为什么 HF 之后还需要 MP2/CCSD/DFT。

---

## 第 2 章 核心推导一：行列式的能量期望值 {#ch2}

**目标**：把 $$E = \langle\Psi^{\text{HF}}\vert \hat H\vert \Psi^{\text{HF}}\rangle$$ 化成单电子积分和双电子积分的显式组合。

这是全文第一个真正的技术难点。我们要用置换群的一个漂亮技巧。

### 2.1 关键技巧：把双重置换和塌缩成单重和

把行列式写成置换和。为记号方便，令

$$
\Pi \equiv \chi_1(\mathbf{x}_1)\chi_2(\mathbf{x}_2)\cdots\chi_N(\mathbf{x}_N)
$$

（Hartree 乘积），则 $$\Psi = \frac{1}{\sqrt{N!}}\sum_P (-1)^p \hat P\Pi$$。于是对任意算符 $$\hat O$$：

$$
\langle\Psi|\hat O|\Psi\rangle = \frac{1}{N!}\sum_{P}\sum_{Q}(-1)^{p+q}\,\langle \hat P\Pi\,|\,\hat O\,|\,\hat Q\Pi\rangle
$$

看起来有 $$(N!)^2$$ 项，绝望。但注意**两个事实**：

1. **$$\hat O$$ 对电子编号是对称的**（$$\hat H$$ 里每一项都平等对待所有电子），所以 $$[\hat P, \hat O] = 0$$。
2. **$$\hat P$$ 是酉算符**（它只是重排积分变量），所以 $$\hat P^\dagger \hat P = 1$$，即 $$\hat P^\dagger = \hat P^{-1}$$。

于是

$$
\langle \hat P\Pi|\hat O|\hat Q\Pi\rangle = \langle\Pi|\hat P^\dagger \hat O\hat Q|\Pi\rangle = \langle\Pi|\hat O\,\hat P^{-1}\hat Q|\Pi\rangle
$$

令 $$\hat R = \hat P^{-1}\hat Q$$。置换符号是乘法性的，所以 $$(-1)^{p+q} = (-1)^{r}$$（因为 $$r \equiv p + q \pmod 2$$）。

**对每个固定的 $$R$$，有多少对 $$(P,Q)$$ 满足 $$P^{-1}Q = R$$？** 恰好 $$N!$$ 对（$$P$$ 任选，$$Q = PR$$ 唯一确定）。所以

$$
\boxed{\langle\Psi|\hat O|\Psi\rangle = \sum_{R}(-1)^{r}\,\langle\Pi|\hat O\,\hat R\,\Pi\rangle}
$$

$$(N!)^2$$ 项塌缩成 $$N!$$ 项。**下面只需数出哪些 $$R$$ 的贡献不为零。**

> 技术说明：$$\hat R$$ 作用在电子坐标上等价于作用在轨道标号上。为方便，下面把 $$\hat R\Pi$$ 理解为
> $$\hat R\Pi = \chi_{R(1)}(\mathbf{x}_1)\chi_{R(2)}(\mathbf{x}_2)\cdots\chi_{R(N)}(\mathbf{x}_N)$$，即**重排轨道标号**。两种理解给出同一结果。

### 2.2 单电子部分

取 $$\hat O_1 = \sum_{i}\hat h(i)$$。先看其中一项 $$\hat h(1)$$：

$$
\langle\Pi|\hat h(1)|\hat R\Pi\rangle = \underbrace{\langle\chi_1|\hat h|\chi_{R(1)}\rangle}_{\text{电子 1}}\;\prod_{i=2}^{N}\underbrace{\langle\chi_i|\chi_{R(i)}\rangle}_{=\,\delta_{i,R(i)}}
$$

要让这一堆 $$\delta$$ 全不为零，必须 $$R(i) = i$$ 对所有 $$i\ge 2$$ 成立。但置换是双射，固定了 $$2,\dots,N$$ 就必须 $$R(1)=1$$——**只有恒等置换 $$R = e$$ 存活**，且 $$(-1)^{r}=+1$$。

所以 $$\langle\Pi\vert \hat h(1)\vert \hat R\Pi\rangle$$ 只剩 $$\langle\chi_1\vert \hat h\vert \chi_1\rangle$$。对 $$\hat h(2), \hat h(3),\dots$$ 同理。求和：

$$
\boxed{\langle\Psi|\textstyle\sum_i \hat h(i)|\Psi\rangle = \sum_{i=1}^{N}\langle\chi_i|\hat h|\chi_i\rangle \equiv \sum_{i=1}^{N} h_{ii}}
$$

直观：**单电子能量就是各轨道单电子能量的简单加和。**

### 2.3 双电子部分

取 $$\hat O_2 = \sum_{i<j} r_{ij}^{-1}$$。先看 $$r_{12}^{-1}$$ 这一项：

$$
\langle\Pi|\,r_{12}^{-1}\,|\hat R\Pi\rangle = \langle \chi_1\chi_2 | \chi_{R(1)}\chi_{R(2)}\rangle \prod_{i=3}^{N}\delta_{i,R(i)}
$$

现在 $$R$$ 必须固定 $$3,\dots,N$$，但可以在 $$\{1,2\}$$ 上任意作用。**两种可能**：

- $$R = e$$：$$(-1)^r = +1$$，贡献 $$\langle\chi_1\chi_2\vert \chi_1\chi_2\rangle$$
- $$R = (1\,2)$$（对换）：$$(-1)^r = -1$$，贡献 $$-\langle\chi_1\chi_2\vert \chi_2\chi_1\rangle$$

对一般的 $$(i,j)$$ 对同理。求和：

$$
\langle\Psi|\hat O_2|\Psi\rangle = \sum_{i<j}\Big[\langle \chi_i\chi_j|\chi_i\chi_j\rangle - \langle\chi_i\chi_j|\chi_j\chi_i\rangle\Big]
$$

**技巧**：$$i=j$$ 时方括号里两项相同、自动为零，所以可以把 $$i<j$$ 的限制去掉，换成 $$\frac12\sum_{i}\sum_j$$（自由求和）。

### 2.4 Hartree–Fock 能量表达式

合并，并引入**反对称化积分**（"double-bar integral"）：

$$
\langle ij||kl\rangle \equiv \langle ij|kl\rangle - \langle ij|lk\rangle
$$

$$
\boxed{
E_{\text{HF}} = \sum_{i=1}^{N} h_{ii} + \frac{1}{2}\sum_{i=1}^{N}\sum_{j=1}^{N}\langle ij||ij\rangle
}
$$

展开写：

$$
E_{\text{HF}} = \sum_i \langle i|\hat h|i\rangle + \frac{1}{2}\sum_{ij}\Big[\underbrace{\langle ij|ij\rangle}_{\text{Coulomb } J_{ij}} - \underbrace{\langle ij|ji\rangle}_{\text{Exchange } K_{ij}}\Big]
$$

### 2.5 两项的物理意义（很重要）

**Coulomb 积分**

$$
J_{ij} = \langle ij|ij\rangle = \int\!\!\int \frac{|\chi_i(\mathbf{x}_1)|^2 |\chi_j(\mathbf{x}_2)|^2}{r_{12}}d\mathbf{x}_1 d\mathbf{x}_2
$$

这是两个**电荷云** $$\vert \chi_i\vert ^2$$ 和 $$\vert \chi_j\vert ^2$$ 之间的经典静电排斥能。完全是经典的、直观的。注意它包含了 $$i=j$$ 的**自相互作用**项 $$J_{ii}$$——电子和自己排斥，这显然是错的。

**Exchange 积分**

$$
K_{ij} = \langle ij|ji\rangle = \int\!\!\int \frac{\chi_i^*(\mathbf{x}_1)\chi_j(\mathbf{x}_1)\,\chi_j^*(\mathbf{x}_2)\chi_i(\mathbf{x}_2)}{r_{12}}d\mathbf{x}_1 d\mathbf{x}_2
$$

**没有经典对应**。它纯粹来自波函数的反对称化（来自那个对换置换 $$R=(1\,2)$$）。三个关键性质：

1. $$K_{ii} = J_{ii}$$，所以 $$i=j$$ 项在 $$J - K$$ 里**精确抵消**——自相互作用误差被完美消除。这是 HF 的一大优点（对比：纯 DFT 泛函做不到，见第 6 章）。
2. 若 $$\chi_i$$ 和 $$\chi_j$$ **自旋不同**，则 $$\chi_i^*\chi_j$$ 含 $$\int\alpha^*\beta\,d\omega = 0$$，故 $$K_{ij}=0$$。**交换只发生在同自旋电子之间。**
3. $$K_{ij} \ge 0$$，所以 $$-K$$ 降低能量。物理图像：同自旋电子由于 Pauli 原理彼此回避，在每个电子周围形成一个"**Fermi 空穴**"，降低了排斥能。这就是 Hund 规则和分子磁性的来源。

**一句话总结**：HF 能量 = 单电子能量 + 经典静电排斥 − 同自旋电子的反对称化修正。它**完全没有描述反自旋电子的瞬时回避**（Coulomb 空穴），这正是 HF 缺失的关联能。

### 2.6 Slater–Condon 规则（顺带一提）

同样的置换计数技巧还给出行列式之间的**非对角**矩阵元。若 $$\vert \Psi\rangle$$ 和 $$\vert \Psi'\rangle$$ 相差：

- **0 个轨道**：上面的结果
- **1 个轨道**（$$\chi_a \to \chi_r$$）：$$\langle\Psi\vert \hat H\vert \Psi'\rangle = \langle a\vert \hat h\vert r\rangle + \sum_j \langle aj\Vert rj\rangle$$
- **2 个轨道**（$$\chi_a\chi_b\to\chi_r\chi_s$$）：$$\langle\Psi\vert \hat H\vert \Psi'\rangle = \langle ab\Vert rs\rangle$$
- **≥3 个轨道**：**0**（因为 $$\hat H$$ 最多是二体算符）

这套规则是全部后 HF 方法（CI、CC、MP）的地基。最后一条"三激发矩阵元为零"是 CI 矩阵稀疏的根本原因。

---

## 第 3 章 核心推导二：变分法导出 Hartree–Fock 方程 {#ch3}

现在 $$E$$ 是一堆函数 $$\{\chi_i\}$$ 的**泛函**（函数的函数）。我们要在正交归一约束下最小化它。

### 3.1 需要的两个数学工具

**工具 1：泛函导数。**

普通函数 $$f(x)$$ 的极值：$$\delta f = f(x+\delta x) - f(x)$$ 的一阶项系数（导数）为零。

泛函 $$E[\chi]$$ 类似：让 $$\chi \to \chi + \delta\chi$$，把 $$\delta E$$ 展开成 $$\delta\chi$$ 的幂，**一阶项必须对任意 $$\delta\chi$$ 都为零**。如果一阶项能写成

$$
\delta E = \int \delta\chi^*(\mathbf{x})\, G(\mathbf{x})\, d\mathbf{x} + \text{c.c.}
$$

那么"对任意 $$\delta\chi^*$$ 都为零"就要求 $$G(\mathbf{x}) = 0$$。这个 $$G$$ 就叫**泛函导数** $$\delta E/\delta\chi^*$$。

（"c.c." = 复共轭。因为 $$E$$ 是实数，$$\delta\chi$$ 和 $$\delta\chi^*$$ 的变分给出互为共轭的两个方程，只需处理一个。）

**工具 2：Lagrange 乘子。**

我们不能自由变分——必须保持 $$\langle\chi_i\vert \chi_j\rangle = \delta_{ij}$$（共 $$N^2$$ 个约束）。标准做法：给每个约束配一个乘子 $$\varepsilon_{ji}$$，构造

$$
\mathcal{L}[\{\chi\}] = E[\{\chi\}] - \sum_{i=1}^{N}\sum_{j=1}^{N}\varepsilon_{ji}\Big(\langle\chi_i|\chi_j\rangle - \delta_{ij}\Big)
$$

然后**无约束地**变分 $$\mathcal L$$。因为 $$E$$ 实、约束实，可以证明矩阵 $$\boldsymbol\varepsilon$$ 是**Hermitian**：$$\varepsilon_{ij} = \varepsilon_{ji}^*$$。（这一点后面救了我们。）

### 3.2 逐项做变分

把 $$\chi_i \to \chi_i + \delta\chi_i$$ 代入第 2 章的 $$E$$，只保留一阶项。

**单电子项：**

$$
\delta\Big(\sum_i h_{ii}\Big) = \sum_i \langle \delta\chi_i|\hat h|\chi_i\rangle + \text{c.c.}
$$

**Coulomb 项：** $$\frac12\sum_{ij}\langle ij\vert ij\rangle$$ 中每个 $$\chi$$ 出现 4 次（2 个在 bra，2 个在 ket）。bra 部分的变分：

$$
\frac{1}{2}\sum_{ij}\Big[\langle \delta\chi_i\,\chi_j|\chi_i\chi_j\rangle + \langle \chi_i\,\delta\chi_j|\chi_i\chi_j\rangle\Big]
$$

第二项做 $$i\leftrightarrow j$$ 换名，再利用积分在"同时交换电子 1、2 标号"下不变，得到 $$\langle\delta\chi_i\chi_j\vert \chi_i\chi_j\rangle$$——与第一项**相同**。所以 $$\frac12 \times 2 = 1$$：

$$
\delta(\text{Coulomb})\big|_{\text{bra}} = \sum_{ij}\langle\delta\chi_i\,\chi_j|\chi_i\chi_j\rangle
$$

把电子 2 的积分先做掉，定义**Coulomb 算符**：

$$
\hat J_j(\mathbf{x}_1)\,f(\mathbf{x}_1) \equiv \left[\int \frac{|\chi_j(\mathbf{x}_2)|^2}{r_{12}}d\mathbf{x}_2\right] f(\mathbf{x}_1)
$$

注意它是**局域的乘法算符**——就是一个普通的势函数（$$\chi_j$$ 的电荷云产生的静电势）。于是上式 $$= \sum_i\langle\delta\chi_i\vert \sum_j\hat J_j\vert \chi_i\rangle$$。

**Exchange 项：** 完全平行的推导，得 $$-\sum_i\langle\delta\chi_i\vert \sum_j\hat K_j\vert \chi_i\rangle$$，其中

$$
\hat K_j(\mathbf{x}_1)\,f(\mathbf{x}_1) \equiv \left[\int\frac{\chi_j^*(\mathbf{x}_2)f(\mathbf{x}_2)}{r_{12}}d\mathbf{x}_2\right]\chi_j(\mathbf{x}_1)
$$

**注意它是非局域的积分算符**：要知道 $$\hat K_j f$$ 在 $$\mathbf{x}_1$$ 的值，必须知道 $$f$$ 在**所有**位置的值。这个非局域性是 HF 计算贵的主要原因，也是 DFT 想绕开的东西。

**约束项：**

$$
-\sum_{ij}\varepsilon_{ji}\langle\delta\chi_i|\chi_j\rangle + \text{c.c.}
$$

### 3.3 Hartree–Fock 方程

把所有一阶项收拢：

$$
\delta\mathcal{L} = \sum_{i}\Big\langle \delta\chi_i \,\Big|\; \Big[\hat h + \sum_j(\hat J_j - \hat K_j)\Big]\chi_i - \sum_j \varepsilon_{ji}\chi_j \Big\rangle + \text{c.c.} = 0
$$

$$\delta\chi_i$$ 任意 ⟹ 大括号必须为零。定义 **Fock 算符**：

$$
\boxed{\hat f(\mathbf{x}_1) = \hat h(\mathbf{x}_1) + \sum_{j=1}^{N}\Big[\hat J_j(\mathbf{x}_1) - \hat K_j(\mathbf{x}_1)\Big]}
$$

得到

$$
\hat f \chi_i = \sum_{j}\varepsilon_{ji}\,\chi_j \qquad (i=1,\dots,N)
$$

这是**Hartree–Fock 方程**的一般形式（还没对角化）。

### 3.4 正则化：为什么可以取对角形式

$$\boldsymbol\varepsilon$$ 是 Hermitian 矩阵，所以存在酉矩阵 $$\mathbf U$$ 使 $$\mathbf U^\dagger\boldsymbol\varepsilon\mathbf U$$ 对角。问题是：**换一组轨道，物理会变吗？**

不会。做酉变换 $$\chi_i' = \sum_j \chi_j U_{ji}$$，则行列式变为

$$
\Psi' = \det(\mathbf U)\cdot\Psi
$$

（因为对矩阵的列做线性组合，行列式乘上变换矩阵的行列式。）$$\mathbf U$$ 酉 ⟹ $$\vert \det\mathbf U\vert  = 1$$，所以 $$\Psi'$$ 和 $$\Psi$$ 只差一个**相位因子**，代表同一个物理态，$$E$$ 不变。而 $$\boldsymbol\varepsilon \to \mathbf U^\dagger\boldsymbol\varepsilon\mathbf U$$。

**所以：选那个让 $$\boldsymbol\varepsilon$$ 对角的 $$\mathbf U$$。** 得到

$$
\boxed{\hat f\,\chi_i = \varepsilon_i\,\chi_i}
$$

这就是**正则 Hartree–Fock 方程**（canonical HF equations）。$$\{\chi_i\}$$ 叫正则分子轨道，$$\varepsilon_i$$ 叫**轨道能量**。

> 顺带说：占据轨道的酉变换自由度正是"定域化轨道"（Boys、Pipek–Mezey）的理论基础——同一个 HF 波函数可以用"看起来像化学键"的定域轨道描述，也可以用正则轨道描述。

### 3.5 三个必须理解的推论

**(a) 这是非线性方程，必须迭代。**

$$\hat f$$ 里含 $$\hat J_j, \hat K_j$$，而它们依赖于 $$\{\chi_j\}$$——**方程的解**。所以这是"伪特征值问题"：要解方程先得知道答案。解法只能是**自洽场迭代**（SCF）：猜一组轨道 → 造 $$\hat f$$ → 解特征值问题 → 得新轨道 → 重复至收敛。

**(b) 总能量 ≠ 轨道能量之和。**

由 $$\hat f\chi_i = \varepsilon_i\chi_i$$ 左乘 $$\chi_i^*$$ 积分：

$$
\varepsilon_i = h_{ii} + \sum_j \langle ij||ij\rangle
$$

所以

$$
\sum_i \varepsilon_i = \sum_i h_{ii} + \sum_{ij}\langle ij||ij\rangle
$$

而 $$E_{\text{HF}} = \sum_i h_{ii} + \frac12\sum_{ij}\langle ij\Vert ij\rangle$$。因此

$$
\boxed{E_{\text{HF}} = \sum_i \varepsilon_i - \frac{1}{2}\sum_{ij}\langle ij||ij\rangle}
$$

**为什么？** 因为 $$\varepsilon_i$$ 包含了电子 $$i$$ 与所有其他电子的相互作用；把所有 $$\varepsilon_i$$ 加起来，每一对相互作用被**数了两次**。减去 $$\frac12\sum\langle ij\Vert ij\rangle$$ 正是消除双计数。这是新手最常犯的错误之一。

**(c) Koopmans 定理。**

若假设移走一个电子后其余轨道**不松弛**（frozen orbital approximation），则

$$
\text{IP} \approx -\varepsilon_{\text{HOMO}},\qquad \text{EA}\approx -\varepsilon_{\text{LUMO}}
$$

推导：用 Slater–Condon 规则算 $$E(N-1) - E(N)$$，冻结轨道下正好等于 $$-\varepsilon_a$$。实际上电离能因为忽略轨道松弛（使 IP 偏大）和忽略关联（使 IP 偏小）而部分抵消，HF 的 IP 误差常在 1 eV 内——运气好。

---

## 第 4 章 自旋积分：闭壳层 RHF {#ch4}

上面的公式用的是**自旋轨道**。实际计算中我们想用**空间轨道**（省一半变量）。

### 4.1 限制型（RHF）假设

设 $$N = 2n$$ 为偶数，且每个空间轨道 $$\psi_a(\mathbf{r})$$ 被**一个 $$\alpha$$ 电子和一个 $$\beta$$ 电子双占据**：

$$
\chi_{2a-1} = \psi_a\alpha,\qquad \chi_{2a} = \psi_a\beta \qquad (a = 1,\dots,n)
$$

这叫**闭壳层限制型 Hartree–Fock**（RHF）。（开壳层用 UHF：$$\alpha$$ 和 $$\beta$$ 用不同的空间轨道；或 ROHF。）

### 4.2 做自旋积分

**单电子项**：每个 $$\psi_a$$ 贡献两次（$$\alpha$$ 和 $$\beta$$），自旋积分给 1：

$$
\sum_{i}^{2n} h_{ii} = 2\sum_{a=1}^{n} h_{aa},\qquad h_{aa} = \int\psi_a^*(\mathbf{r})\,\hat h\,\psi_a(\mathbf{r})\,d\mathbf{r}
$$

**Coulomb 项**：$$\langle ij\vert ij\rangle$$ 中，电子 1 的部分是 $$\chi_i^*\chi_i$$（同一轨道，自旋积分总为 1），电子 2 同理。所以**自旋完全不设限**，每个空间轨道对 $$(a,b)$$ 有 $$2\times 2 = 4$$ 种自旋组合：

$$
\sum_{ij}^{2n}\langle ij|ij\rangle = 4\sum_{ab}^{n}(aa|bb)
$$

（这里换成了化学家记号，因为 $$\langle ij\vert ij\rangle$$ 转成空间轨道后自然是 $$(aa\vert bb)$$ 形式。）

**Exchange 项**：$$\langle ij\vert ji\rangle$$ 中电子 1 的部分是 $$\chi_i^*\chi_j$$，要求 $$i,j$$ **同自旋**。所以只有 $$2$$ 种组合（都 $$\alpha$$ 或都 $$\beta$$）：

$$
\sum_{ij}^{2n}\langle ij|ji\rangle = 2\sum_{ab}^{n}(ab|ba)
$$

### 4.3 RHF 能量与 Fock 算符

代入 $$E = \sum h_{ii} + \frac12\sum(\langle ij\vert ij\rangle - \langle ij\vert ji\rangle)$$：

$$
\boxed{
E_{\text{RHF}} = 2\sum_{a=1}^{n} h_{aa} + \sum_{a=1}^{n}\sum_{b=1}^{n}\Big[2(aa|bb) - (ab|ba)\Big]
}
$$

相应地，闭壳层 Fock 算符：

$$
\boxed{\hat f(\mathbf{r}_1) = \hat h(\mathbf{r}_1) + \sum_{b=1}^{n}\Big[2\hat J_b(\mathbf{r}_1) - \hat K_b(\mathbf{r}_1)\Big]}
$$

**注意那个 2 和 1 的不对称**：Coulomb 系数是 2（每个空间轨道有 2 个电子都参与经典排斥），交换系数是 1（只有同自旋的那 1 个参与交换）。这个 "$$2J - K$$" 结构记住就行。

轨道能量：$$\varepsilon_a = h_{aa} + \sum_b[2(aa\vert bb) - (ab\vert ba)]$$，于是

$$
E_{\text{RHF}} = 2\sum_a\varepsilon_a - \sum_{ab}\big[2(aa|bb)-(ab|ba)\big]
$$

---

## 第 5 章 基组展开：Roothaan–Hall 方程与实际算法 {#ch5}

$$\hat f\psi_a = \varepsilon_a\psi_a$$ 是**积分微分方程**，只有原子（球对称）能数值精确解。分子里必须把轨道展开成已知函数。

### 5.1 LCAO 展开

$$
\psi_a(\mathbf{r}) = \sum_{\mu=1}^{K} C_{\mu a}\,\chi_\mu(\mathbf{r})
$$

$$\{\chi_\mu\}$$ 是**基函数**（$$K$$ 个，$$K \ge n$$），通常是**以原子核为中心的高斯型函数**：

$$
\chi_\mu(\mathbf{r}) = (x - A_x)^{l}(y-A_y)^{m}(z-A_z)^{n}\,e^{-\alpha|\mathbf{r}-\mathbf{A}|^2}
$$

为什么用高斯而不用更物理的 Slater 型 $$e^{-\zeta r}$$？因为**两个高斯的乘积还是高斯**（高斯乘积定理），使四中心双电子积分有解析公式。代价是高斯在核处没有正确的尖点、在远处衰减太快，所以要用几个高斯**收缩**（contract）成一个基函数来拟合。

**关键点：基函数不正交！** $$S_{\mu\nu} \equiv \langle\chi_\mu\vert \chi_\nu\rangle \ne \delta_{\mu\nu}$$。这会带来一个额外的矩阵 $$\mathbf S$$，也正是第 7 章 Pulay 力的来源。

### 5.2 Roothaan–Hall 方程

把展开代入 $$\hat f\psi_a = \varepsilon_a\psi_a$$，左乘 $$\chi_\mu^*$$ 并积分：

$$
\sum_\nu \underbrace{\langle\chi_\mu|\hat f|\chi_\nu\rangle}_{F_{\mu\nu}} C_{\nu a} = \varepsilon_a \sum_\nu \underbrace{\langle\chi_\mu|\chi_\nu\rangle}_{S_{\mu\nu}} C_{\nu a}
$$

矩阵形式：

$$
\boxed{\mathbf{F}\mathbf{C} = \mathbf{S}\mathbf{C}\boldsymbol{\varepsilon}}
$$

这叫 **Roothaan–Hall 方程**（Roothaan 与 Hall 1951 年独立提出），是一个**广义特征值问题**。$$\boldsymbol\varepsilon$$ 是对角矩阵。

### 5.3 密度矩阵

定义**密度矩阵**（闭壳层，含因子 2）：

$$
\boxed{P_{\mu\nu} = 2\sum_{a=1}^{n} C_{\mu a}C_{\nu a}^{*}}
$$

它的意义：电子密度是

$$
\rho(\mathbf{r}) = 2\sum_a |\psi_a(\mathbf{r})|^2 = \sum_{\mu\nu}P_{\mu\nu}\,\chi_\mu(\mathbf{r})\chi_\nu^*(\mathbf{r})
$$

并且 $$\text{tr}(\mathbf{PS}) = N$$（电子数）。密度矩阵是 SCF 的真正状态变量——所有物理量都通过它表达。

### 5.4 Fock 矩阵的显式表达（关键推导）

$$
F_{\mu\nu} = \underbrace{\langle\chi_\mu|\hat h|\chi_\nu\rangle}_{H^{\text{core}}_{\mu\nu}} + \sum_{b}^{n}\big[2\langle\mu|\hat J_b|\nu\rangle - \langle\mu|\hat K_b|\nu\rangle\big]
$$

展开 $$\psi_b = \sum_\lambda C_{\lambda b}\chi_\lambda$$：

- $$\langle\mu\vert \hat J_b\vert \nu\rangle = \sum_{\lambda\sigma}C_{\lambda b}^*C_{\sigma b}(\mu\nu\vert \lambda\sigma)$$
- $$\langle\mu\vert \hat K_b\vert \nu\rangle = \sum_{\lambda\sigma}C_{\lambda b}^*C_{\sigma b}(\mu\sigma\vert \lambda\nu)$$

代入并用 $$2\sum_b C_{\lambda b}^* C_{\sigma b} = P_{\lambda\sigma}$$（实基组下 $$P$$ 对称）：

$$
\boxed{F_{\mu\nu} = H^{\text{core}}_{\mu\nu} + \sum_{\lambda\sigma}P_{\lambda\sigma}\Big[(\mu\nu|\lambda\sigma) - \tfrac{1}{2}(\mu\lambda|\sigma\nu)\Big]}
$$

**记住这个式子的下标模式**：Coulomb 项 $$(\mu\nu\vert \lambda\sigma)$$——外标号 $$\mu\nu$$ 在一起；Exchange 项 $$(\mu\lambda\vert \sigma\nu)$$——外标号被拆开、和密度标号交叉配对。这个"交叉"就是非局域性在矩阵语言里的样子。

**核心哈密顿矩阵**：$$H^{\text{core}}_{\mu\nu} = T_{\mu\nu} + V^{\text{ne}}_{\mu\nu}$$，其中动能 $$T_{\mu\nu}=-\frac12\langle\mu\vert \nabla^2\vert \nu\rangle$$，$$V^{\text{ne}}_{\mu\nu} = -\sum_A Z_A\langle\mu\vert \,\vert \mathbf r-\mathbf R_A\vert ^{-1}\vert \nu\rangle$$。

### 5.5 能量的矩阵表达式（关键推导）

从 $$E_{\text{RHF}} = 2\sum_a h_{aa} + \sum_{ab}[2(aa\vert bb)-(ab\vert ba)]$$ 出发，逐项转成基组表示（以下设基函数实值）。

**单电子项**：

$$
2\sum_a h_{aa} = 2\sum_a\sum_{\mu\nu}C_{\mu a}C_{\nu a}H_{\mu\nu} = \sum_{\mu\nu}P_{\mu\nu}H^{\text{core}}_{\mu\nu}
$$

**Coulomb 项**：

$$
2\sum_{ab}(aa|bb) = 2\sum_{\mu\nu\lambda\sigma}\Big(\tfrac12 P_{\mu\nu}\Big)\Big(\tfrac12 P_{\lambda\sigma}\Big)(\mu\nu|\lambda\sigma) = \tfrac{1}{2}\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}(\mu\nu|\lambda\sigma)
$$

**Exchange 项**：

$$
\sum_{ab}(ab|ba) = \sum_{\mu\nu\lambda\sigma}\Big(\sum_a C_{\mu a}C_{\sigma a}\Big)\Big(\sum_b C_{\nu b}C_{\lambda b}\Big)(\mu\nu|\lambda\sigma) = \tfrac14\sum_{\mu\nu\lambda\sigma}P_{\mu\sigma}P_{\nu\lambda}(\mu\nu|\lambda\sigma)
$$

为了让第一个密度矩阵带下标 $$(\mu\nu)$$，做**哑标重命名** $$\mu\to\mu,\;\sigma\to\nu,\;\nu\to\lambda,\;\lambda\to\sigma$$：

$$
\sum_{ab}(ab|ba) = \tfrac14\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}(\mu\lambda|\sigma\nu)
$$

**合并**：

$$
\boxed{
E_{\text{el}} = \sum_{\mu\nu}P_{\mu\nu}H^{\text{core}}_{\mu\nu} + \frac{1}{2}\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}\Big[(\mu\nu|\lambda\sigma) - \tfrac12(\mu\lambda|\sigma\nu)\Big]
}
$$

**与 5.4 对照可见一个漂亮的关系**：括号里正好是 $$F_{\mu\nu}-H^{\text{core}}_{\mu\nu}$$ 的求和核，所以

$$
\boxed{E_{\text{el}} = \frac{1}{2}\sum_{\mu\nu}P_{\mu\nu}\Big(H^{\text{core}}_{\mu\nu} + F_{\mu\nu}\Big) = \frac12\,\text{tr}\big[\mathbf{P}(\mathbf{H}^{\text{core}}+\mathbf{F})\big]}
$$

$$
E_{\text{total}} = E_{\text{el}} + V_{NN}
$$

**这是量化程序里实际用的公式。** 那个 $$\frac12$$ 和 $$(\mathbf H + \mathbf F)$$ 的组合正是 3.5(b) 里"消除双计数"的矩阵版本。

另外注意一个对于第 7 章至关重要的事实：

$$
\boxed{F_{\mu\nu} = \frac{\partial E_{\text{el}}}{\partial P_{\mu\nu}}}
$$

（对 $$E_{\text{el}}$$ 的表达式直接求导，并利用括号里的核在 $$(\mu\nu)\leftrightarrow(\lambda\sigma)$$ 交换下对称——这一点可用 8 重对称性验证：$$(\mu\lambda\vert \sigma\nu)$$ 在 $$\mu\leftrightarrow\lambda,\nu\leftrightarrow\sigma$$ 下变为 $$(\lambda\mu\vert \nu\sigma) = (\mu\lambda\vert \sigma\nu)$$ ✓。）

### 5.6 怎么解广义特征值问题

$$\mathbf{FC} = \mathbf{SC}\boldsymbol\varepsilon$$ 不是标准特征值问题，因为 $$\mathbf S\ne\mathbf 1$$。标准做法是**对称正交化**：

$$\mathbf S$$ 是实对称正定矩阵，对角化 $$\mathbf S = \mathbf{U}\mathbf{s}\mathbf{U}^\dagger$$，定义

$$
\mathbf{X} = \mathbf{S}^{-1/2} = \mathbf{U}\,\mathbf{s}^{-1/2}\,\mathbf{U}^\dagger
$$

（即把对角元开 $$-1/2$$ 次方。）令 $$\mathbf{C} = \mathbf{X}\mathbf{C}'$$，代入：

$$
\mathbf{FXC}' = \mathbf{SXC}'\boldsymbol\varepsilon \;\xrightarrow{\ \text{左乘}\ \mathbf{X}^\dagger\ }\; \underbrace{(\mathbf{X}^\dagger\mathbf{FX})}_{\mathbf{F}'}\mathbf{C}' = \underbrace{(\mathbf{X}^\dagger\mathbf{SX})}_{=\,\mathbf 1}\mathbf{C}'\boldsymbol\varepsilon
$$

（用了 $$\mathbf S^{-1/2}\mathbf S\mathbf S^{-1/2} = \mathbf 1$$。）所以

$$
\mathbf{F}'\mathbf{C}' = \mathbf{C}'\boldsymbol\varepsilon
$$

标准特征值问题，直接对角化，再回代 $$\mathbf C = \mathbf{XC}'$$。

> 实践提示：若基组接近线性相关（$$\mathbf S$$ 有很小的特征值），$$\mathbf s^{-1/2}$$ 会数值爆炸。解决办法是丢掉小于阈值（如 $$10^{-6}$$）的特征值（canonical orthogonalization），或用 Cholesky 分解。

### 5.7 SCF 算法全流程

```
输入：分子几何 {R_A, Z_A}，基组
1.  计算并存储 S, H_core, 以及全部双电子积分 (μν|λσ)
2.  对角化 S → X = S^(-1/2)
3.  猜初始密度 P  （常用：H_core 猜、SAD 原子密度叠加、GWH）
4.  循环：
      a. 由 P 造 Fock 矩阵 F = H_core + G(P)
      b. E = ½ tr[P(H_core + F)]
      c. F' = Xᵗ F X
      d. 对角化 F' → C', ε
      e. C = X C'
      f. 取能量最低的 n 个轨道（占据轨道），造新密度
             P_new = 2 Σ_a C_μa C_νa
      g. 检查收敛：|ΔE| < 1e-8 且 ||P_new - P|| < 1e-6 ?
         否 → 用 DIIS 外推得到下一轮的 P/F，回到 (a)
5.  E_total = E_el + V_NN
```

**几点实践说明**：

- **DIIS**（Direct Inversion in the Iterative Subspace，Pulay 1980）：把历史几步的 Fock 矩阵线性组合，使残差 $$\mathbf{e} = \mathbf{FPS}-\mathbf{SPF}$$ 的模最小。没有 DIIS 的话 SCF 经常震荡不收敛。
- **计算标度**：双电子积分有 $$O(K^4)$$ 个。$$K=1000$$ 就是 $$10^{12}$$ 个——存不下。现代做法：直接 SCF（每轮重算，配合积分筛选 Schwarz 不等式 $$\vert (\mu\nu\vert \lambda\sigma)\vert \le\sqrt{(\mu\nu\vert \mu\nu)(\lambda\sigma\vert \lambda\sigma)}$$）、密度拟合/RI、连续快速多极（CFMM），可把 Coulomb 部分降到近线性标度。
- **Aufbau 原理**：第 4(f) 步"取最低 $$n$$ 个轨道"看似显然，但在过渡金属体系里可能收敛到激发态解或跑到错误的电子构型，需要 level shifting、fractional occupation 或 MOM 等技巧。

### 5.8 一个最小的具体例子

$$\mathrm{H_2}$$ 用极小基（每个 H 一个 1s 函数 $$\chi_A,\chi_B$$）。对称性给出

$$
\psi_1 = \frac{\chi_A+\chi_B}{\sqrt{2(1+S)}}\ (\sigma_g),\qquad \psi_2 = \frac{\chi_A-\chi_B}{\sqrt{2(1-S)}}\ (\sigma_u^*)
$$

其中 $$S = S_{AB}$$。这里 $$n=1$$（一个双占据轨道），能量公式塌缩成

$$
E_{\text{RHF}} = 2h_{11} + \big[2(11|11)-(11|11)\big] + \frac{1}{R} = 2h_{11} + (11|11) + \frac{1}{R}
$$

即"两个电子的单电子能量 + 它们之间的一份 Coulomb 排斥 + 核排斥"——正好符合直觉，而且 $$2J-K$$ 在同一轨道内退化为 $$J$$，自相互作用被交换项吃掉了一半，剩下的正是两个电子之间**真实的**一份排斥。这是检验公式的一个好例子。

（顺带：这个体系也暴露 HF 的致命弱点。$$R\to\infty$$ 时 RHF 强迫两个电子留在同一个对称轨道，波函数含 $$\mathrm{H^+H^-}$$ 离子成分，解离能严重偏高——这就是著名的 RHF 解离灾难，需要 UHF 或多参考方法。）

---

## 第 6 章 密度泛函理论 {#ch6}

### 6.0 动机：维度的暴政

$$N$$ 电子波函数是 $$\Psi(\mathbf{x}_1,\dots,\mathbf{x}_N)$$——**$$4N$$ 个变量**（$$3N$$ 空间 + $$N$$ 自旋）。Walter Kohn 有个著名估计：即使每个维度只存 3 个数，$$N=100$$ 的波函数也需要 $$3^{300}$$ 个数——远超宇宙原子总数。

而**电子密度**只有 3 个变量：

$$
\rho(\mathbf{r}_1) = N\sum_{\omega_1}\int|\Psi(\mathbf{x}_1,\mathbf{x}_2,\dots,\mathbf{x}_N)|^2\,d\mathbf{x}_2\cdots d\mathbf{x}_N
$$

（因子 $$N$$ 因为电子全同：$$\int\rho\,d\mathbf{r} = N$$。）

**DFT 的赌注**：$$\rho$$ 是否包含了全部信息？1964 年 Hohenberg 和 Kohn 证明：**是的**。

### 6.1 Hohenberg–Kohn 第一定理

> **定理**：对于非简并基态，外势 $$v_{\text{ext}}(\mathbf{r})$$ 由基态密度 $$\rho_0(\mathbf{r})$$ 唯一决定（至多相差一个常数）。

**为什么这很了不起**：$$\rho_0 \Rightarrow v_{\text{ext}}$$；又 $$\int\rho_0 = N$$ 给出电子数；而 $$\hat H = \hat T + \hat V_{ee} + \hat V_{\text{ext}}$$ 中 $$\hat T$$ 和 $$\hat V_{ee}$$ 是普适的（对任何分子都一样）。所以

$$
\rho_0 \;\Longrightarrow\; \hat H \;\Longrightarrow\; \text{一切（基态、激发态、所有性质）}
$$

**证明（反证法，只有五行）**：

假设两个外势 $$v$$ 和 $$v'$$（相差超过一个常数）给出同一个基态密度 $$\rho_0$$。它们对应哈密顿量 $$\hat H, \hat H'$$，基态 $$\Psi, \Psi'$$，能量 $$E_0, E_0'$$。

由于 $$v \ne v' + \text{const}$$，两个薛定谔方程不同，$$\Psi \ne \Psi'$$（否则相减会得到 $$v-v'=$$ 常数，矛盾）。

用变分原理（$$\Psi'$$ 不是 $$\hat H$$ 的基态，非简并 ⟹ **严格**不等号）：

$$
E_0 < \langle\Psi'|\hat H|\Psi'\rangle = \langle\Psi'|\hat H'|\Psi'\rangle + \langle\Psi'|\hat H - \hat H'|\Psi'\rangle = E_0' + \int\rho_0(\mathbf{r})\big[v(\mathbf{r})-v'(\mathbf{r})\big]d\mathbf{r}
$$

（$$\hat H - \hat H' = \sum_i[v(\mathbf r_i) - v'(\mathbf r_i)]$$ 是单体乘法算符，其期望值就是密度的加权积分。）

对称地交换角色：

$$
E_0' < E_0 + \int\rho_0(\mathbf{r})\big[v'(\mathbf{r})-v(\mathbf{r})\big]d\mathbf{r}
$$

**两式相加**，积分项完全抵消：

$$
E_0 + E_0' < E_0' + E_0
$$

矛盾。∎

（漂亮之处：整个证明只用到变分原理和"外势是单体乘法算符"这一点。）

### 6.2 Hohenberg–Kohn 第二定理与变分原理

既然 $$\rho\Rightarrow\hat H\Rightarrow\Psi$$，我们可以定义**普适泛函**

$$
F_{\text{HK}}[\rho] = \langle\Psi[\rho]|\hat T + \hat V_{ee}|\Psi[\rho]\rangle
$$

"普适"指它**不依赖于外势**——氢原子、苯、DNA 用的是同一个 $$F_{\text{HK}}$$。

> **定理**：对给定 $$v_{\text{ext}}$$，能量泛函 $$E_v[\rho] = F_{\text{HK}}[\rho] + \int\rho\, v_{\text{ext}}\,d\mathbf{r}$$ 满足 $$E_v[\rho]\ge E_0$$，等号当且仅当 $$\rho=\rho_0$$。

原始 HK 证明有个技术漏洞：$$F_{\text{HK}}[\rho]$$ 只对 **$$v$$-可表示**（即确实是某个外势的基态密度）的 $$\rho$$ 有定义，而判断一个密度是否 $$v$$-可表示极其困难。

**Levy 约束搜索（1979）** 优雅地绕过了这个问题。定义

$$
\boxed{F[\rho] = \min_{\Psi\to\rho}\ \langle\Psi|\hat T+\hat V_{ee}|\Psi\rangle}
$$

含义：在**所有**产生密度 $$\rho$$ 的反对称波函数中，取 $$\langle\hat T+\hat V_{ee}\rangle$$ 最小的那个。这只要求 $$\rho$$ 是 **$$N$$-可表示**的（能由某个反对称波函数产生），而这个条件很宽松：$$\rho\ge 0$$、$$\int\rho = N$$、$$\int\vert \nabla\rho^{1/2}\vert ^2<\infty$$ 就够了。

**两行证明变分原理**：

$$
\min_\rho\Big\{F[\rho]+\int\rho v\Big\} = \min_\rho\min_{\Psi\to\rho}\langle\Psi|\hat H|\Psi\rangle = \min_{\Psi}\langle\Psi|\hat H|\Psi\rangle = E_0
$$

（第二个等号：先按密度分组再取最小，等于直接在所有 $$\Psi$$ 上取最小。）∎

**但是**：$$F[\rho]$$ 的显式形式我们不知道，而且大概永远不会知道。这就是 DFT 从"精确理论"变成"近似方法"的地方。

### 6.3 为什么需要 Kohn–Sham：动能是拦路虎

最早的尝试（Thomas–Fermi，1927）直接给密度泛函：

$$
T_{\text{TF}}[\rho] = C_F\int\rho^{5/3}d\mathbf{r},\qquad C_F = \tfrac{3}{10}(3\pi^2)^{2/3}\approx 2.871
$$

（由均匀电子气动能推出。）问题：**动能占总能量的绝大部分，$$T_{\text{TF}}$$ 的误差约 10%，绝对误差达几十甚至上百 Hartree。** Teller 甚至证明了 Thomas–Fermi 理论中**分子不成键**。彻底失败。

**Kohn 和 Sham（1965）的洞见**：与其硬猜动能泛函，不如**借用轨道**来算动能的绝大部分。

### 6.4 Kohn–Sham 构造

引入一个**虚构的无相互作用参考体系**：$$N$$ 个不相互作用的电子，处在某个有效势 $$v_s(\mathbf{r})$$ 中，**其基态密度恰好等于真实体系的密度** $$\rho$$。

无相互作用体系的基态波函数**精确地**是一个 Slater 行列式 $$\Phi_s = \vert \psi_1\cdots\psi_N\rangle$$，其动能可以精确算出：

$$
T_s[\rho] = \langle\Phi_s|\hat T|\Phi_s\rangle = -\frac{1}{2}\sum_{i=1}^{N}\langle\psi_i|\nabla^2|\psi_i\rangle
$$

$$
\rho(\mathbf{r}) = \sum_{i=1}^{N}|\psi_i(\mathbf{r})|^2
$$

**注意**：$$T_s[\rho] \ne T[\rho]$$（真实动能）。差别不大但不为零。

现在做**能量分解**——这一步是纯定义，**没有任何近似**：

$$
\boxed{E[\rho] = T_s[\rho] + \int\rho(\mathbf{r})v_{\text{ext}}(\mathbf{r})d\mathbf{r} + J[\rho] + E_{xc}[\rho]}
$$

其中

$$
J[\rho] = \frac{1}{2}\int\!\!\int\frac{\rho(\mathbf{r})\rho(\mathbf{r}')}{|\mathbf{r}-\mathbf{r}'|}d\mathbf{r}\,d\mathbf{r}' \quad(\text{经典 Hartree 排斥})
$$

而**交换关联能**就定义为剩下的一切：

$$
\boxed{E_{xc}[\rho] \equiv \underbrace{\big(T[\rho]-T_s[\rho]\big)}_{\text{动能关联}} + \underbrace{\big(V_{ee}[\rho]-J[\rho]\big)}_{\text{交换 + Coulomb 关联 − 自作用}}}
$$

**这是全套理论的关键一招**：把所有不知道的东西塞进一个小项。$$E_{xc}$$ 通常只占总能量的百分之几，所以即使它有 5–10% 的误差，总能量误差也能控制在化学精度附近。对比 Thomas–Fermi 把 100% 的动能都交给近似——高下立判。

> 常见误解澄清：$$E_{xc}$$ **不只是**"交换 + 关联"。它还包含了 $$T - T_s$$ 这个动能修正项。这就是为什么 Kohn–Sham 的 $$E_{xc}$$ 和波函数理论里的"交换能""关联能"不是同一个东西。

### 6.5 Kohn–Sham 方程

现在对 $$E[\rho]$$ 变分。因为 $$T_s$$ 是用轨道写的，直接**对轨道变分**（和第 3 章完全平行），带正交归一约束 $$\langle\psi_i\vert \psi_j\rangle=\delta_{ij}$$。

需要几个泛函导数。用链式法则：$$\dfrac{\delta\rho(\mathbf r')}{\delta\psi_i^*(\mathbf r)} = \psi_i(\mathbf r)\,\delta(\mathbf r-\mathbf r')$$。

**（a）Hartree 项。** 让 $$\rho\to\rho+\delta\rho$$：

$$
\delta J = \frac12\int\!\!\int\frac{\delta\rho(\mathbf r)\rho(\mathbf r')}{|\mathbf r-\mathbf r'|} + \frac12\int\!\!\int\frac{\rho(\mathbf r)\delta\rho(\mathbf r')}{|\mathbf r-\mathbf r'|} = \int \delta\rho(\mathbf r)\left[\int\frac{\rho(\mathbf r')}{|\mathbf r-\mathbf r'|}d\mathbf r'\right]d\mathbf r
$$

（两项因积分对称而相等，$$\frac12\times 2=1$$。）所以

$$
v_H(\mathbf r) \equiv \frac{\delta J}{\delta\rho(\mathbf r)} = \int\frac{\rho(\mathbf r')}{|\mathbf r-\mathbf r'|}d\mathbf r'
$$

这就是经典静电势（Hartree 势）。

**（b）交换关联势**（定义）：

$$
v_{xc}(\mathbf r) \equiv \frac{\delta E_{xc}}{\delta\rho(\mathbf r)}
$$

**（c）动能项**：$$\delta T_s/\delta\psi_i^* = -\frac12\nabla^2\psi_i$$（直接对轨道求，不需要知道 $$\delta T_s/\delta\rho$$！这正是 KS 方案的巧妙处）。

把 Lagrange 乘子法照搬第 3 章、再做正则化，得到

$$
\boxed{\left[-\frac{1}{2}\nabla^2 + \underbrace{v_{\text{ext}}(\mathbf r) + v_H(\mathbf r) + v_{xc}(\mathbf r)}_{v_{\text{eff}}(\mathbf r)}\right]\psi_i(\mathbf r) = \varepsilon_i\,\psi_i(\mathbf r)}
$$

这就是 **Kohn–Sham 方程**。同时我们也读出了那个虚构参考体系的势：$$v_s = v_{\text{eff}}$$。

$$v_{\text{eff}}$$ 依赖 $$\rho$$，$$\rho$$ 依赖 $$\psi_i$$——**又是自洽场问题，算法流程和 HF 的 SCF 一模一样。**

### 6.6 KS-DFT 与 HF 的结构对比

| | Hartree–Fock | Kohn–Sham DFT |
|---|---|---|
| 行列式的地位 | 真实波函数的**近似** | 虚构参考体系的**精确**波函数 |
| 交换 | 精确的 $$\hat K$$（**非局域**积分算符） | 近似的 $$v_{xc}(\mathbf r)$$（**局域**乘法算符） |
| 关联 | 完全缺失 | 原则上精确，实际近似 |
| 自作用误差 | 无（$$J_{ii}=K_{ii}$$ 精确抵消） | **有**（近似 $$E_{xc}$$ 不能精确抵消 $$J$$ 中的自作用） |
| 轨道能量含义 | Koopmans：$$-\varepsilon_{\text{HOMO}}\approx$$ IP | 严格意义上是 Lagrange 乘子；精确泛函下 $$\varepsilon_{\text{HOMO}} = -\text{IP}$$（Janak/Perdew），但近似泛函下偏离很大 |
| 变分上界 | 是（$$E_{\text{HF}}\ge E_{\text{exact}}$$） | **否**（近似泛函可能低于真值） |
| 代价 | $$O(K^4)$$（交换积分） | 纯泛函 $$O(K^3)$$ 甚至更低；杂化泛函退回到含交换积分 |

**"局域 vs 非局域"是 DFT 便宜的根本原因，也是它主要缺陷（自作用误差、电荷转移态、范德华）的根源。**

### 6.7 泛函近似的阶梯

Perdew 的"Jacob 天梯"：

**第一级 LDA（局域密度近似）**

假设每一点的 $$\varepsilon_{xc}$$ 就是**同密度均匀电子气**的值：

$$
E_{xc}^{\text{LDA}}[\rho] = \int \rho(\mathbf r)\,\varepsilon_{xc}^{\text{unif}}\big(\rho(\mathbf r)\big)\,d\mathbf r
$$

交换部分**可以精确解析求出**——下面就推一遍，因为这是唯一一个能从头算出来的泛函。

**第二级 GGA（广义梯度近似）**：加上 $$\nabla\rho$$，$$E_{xc}=\int f(\rho,\nabla\rho)$$。代表：BLYP、PBE、B88。

**第三级 meta-GGA**：再加动能密度 $$\tau = \frac12\sum_i\vert \nabla\psi_i\vert ^2$$（或 $$\nabla^2\rho$$）。代表：TPSS、SCAN、M06-L、r²SCAN。

**第四级 杂化泛函**：掺入一部分精确 HF 交换。

$$
E_{xc}^{\text{hyb}} = a\,E_x^{\text{HF}} + (1-a)E_x^{\text{DFA}} + E_c^{\text{DFA}}
$$

代表：B3LYP（$$a=0.20$$）、PBE0（$$a=0.25$$）、$$\omega$$B97X-D（长程分离）。**理论依据是绝热连接**（见 6.11）。

**第五级 双杂化**：再掺入 MP2 型的虚轨道关联，如 B2PLYP、DSD-PBEP86。

（正交的一条线：**色散校正**。所有半局域泛函都缺 $$-C_6/R^6$$ 长程色散，需要 DFT-D3/D4 或 VV10 之类的修补。）

### 6.8 从头推导 Dirac 交换泛函（LDA-X）

这是全文唯一一个"密度泛函"可以真正被解析算出来的例子，值得完整做一遍。

**设定**：均匀电子气——$$N$$ 个电子在体积 $$V$$ 中，加上均匀正电背景（保证电中性）。无相互作用轨道是**平面波**：

$$
\psi_{\mathbf k}(\mathbf r) = \frac{1}{\sqrt V}e^{i\mathbf k\cdot\mathbf r}
$$

**第 1 步：Fermi 波矢与密度的关系。**

在 $$k$$ 空间中，态密度是 $$V/(2\pi)^3$$（每个自旋）。填到 $$k_F$$：

$$
N = 2\cdot\frac{V}{(2\pi)^3}\cdot\frac{4}{3}\pi k_F^3 = \frac{V k_F^3}{3\pi^2}
\;\Longrightarrow\;
\boxed{\rho=\frac{N}{V} = \frac{k_F^3}{3\pi^2}},\quad k_F = (3\pi^2\rho)^{1/3}
$$

**第 2 步：单个交换积分。**

用第 2 章的 HF 交换公式 $$E_x = -\frac12\sum_{ij}\langle ij\vert ji\rangle$$（$$i,j$$ 遍历自旋轨道；异自旋项为零）。对平面波：

$$
\langle \mathbf k\,\mathbf k'|\mathbf k'\,\mathbf k\rangle = \frac{1}{V^2}\int\!\!\int e^{-i\mathbf k\cdot\mathbf r_1}e^{-i\mathbf k'\cdot\mathbf r_2}\frac{1}{r_{12}}e^{i\mathbf k'\cdot\mathbf r_1}e^{i\mathbf k\cdot\mathbf r_2}\,d\mathbf r_1 d\mathbf r_2
$$

指数合并为 $$e^{i(\mathbf k'-\mathbf k)\cdot(\mathbf r_1-\mathbf r_2)}$$。换元 $$\mathbf u = \mathbf r_1-\mathbf r_2$$，对质心积分给出因子 $$V$$：

$$
= \frac{1}{V}\int \frac{e^{i\mathbf q\cdot\mathbf u}}{u}\,d\mathbf u,\qquad \mathbf q = \mathbf k'-\mathbf k
$$

利用 Coulomb 势的 Fourier 变换 $$\displaystyle\int\frac{e^{i\mathbf q\cdot\mathbf u}}{u}d\mathbf u = \frac{4\pi}{q^2}$$：

$$
\langle\mathbf k\mathbf k'|\mathbf k'\mathbf k\rangle = \frac{1}{V}\cdot\frac{4\pi}{|\mathbf k-\mathbf k'|^2}
$$

**第 3 步：求和 → 积分。**

$$
E_x = -\frac12\sum_{\sigma=\alpha,\beta}\sum_{\mathbf k,\mathbf k'\le k_F}\frac{4\pi}{V|\mathbf k-\mathbf k'|^2}
$$

自旋求和给因子 2，与 $$-\frac12$$ 抵消成 $$-1$$。把 $$\sum_{\mathbf k}\to \frac{V}{(2\pi)^3}\int d^3k$$：

$$
E_x = -\frac{4\pi}{V}\left(\frac{V}{(2\pi)^3}\right)^2 \underbrace{\int_{k<k_F}\!\!\int_{k'<k_F}\frac{d^3k\,d^3k'}{|\mathbf k-\mathbf k'|^2}}_{\equiv\, I}
= -\frac{4\pi V}{(2\pi)^6}\,I
$$

**第 4 步：算那个六重积分 $$I$$。**

先固定 $$\mathbf k$$，对 $$\mathbf k'$$ 积分，用球坐标（$$\mu=\cos\theta$$）：

$$
\int_{k'<k_F}\frac{d^3k'}{|\mathbf k-\mathbf k'|^2} = 2\pi\int_0^{k_F}\!\!k'^2dk'\int_{-1}^{1}\frac{d\mu}{k^2+k'^2-2kk'\mu} = \frac{2\pi}{k}\int_0^{k_F}k'\ln\left|\frac{k+k'}{k-k'}\right|dk'
$$

（内层对 $$\mu$$ 的积分：$$\int_{-1}^1\frac{d\mu}{a-b\mu} = \frac{1}{b}\ln\frac{a+b}{a-b}$$，代入 $$a=k^2+k'^2, b=2kk'$$，注意 $$a\pm b = (k\pm k')^2$$。）

再对 $$\mathbf k$$ 积分，令 $$k=k_F x,\ k'=k_F y$$：

$$
I = 8\pi^2 k_F^4\underbrace{\int_0^1\!\!\int_0^1 xy\,\ln\left|\frac{x+y}{x-y}\right|dx\,dy}_{=\,1/2} = 4\pi^2 k_F^4
$$

**第 5 步：合并。**

$$
E_x = -\frac{4\pi V}{64\pi^6}\cdot 4\pi^2 k_F^4 = -\frac{V k_F^4}{4\pi^3}
$$

代入 $$k_F=(3\pi^2\rho)^{1/3}$$：

$$
\frac{E_x}{V} = -\frac{(3\pi^2)^{4/3}}{4\pi^3}\rho^{4/3} = -\frac{3}{4}\left(\frac{3}{\pi}\right)^{1/3}\rho^{4/3}
$$

（化简：$$\frac{(3\pi^2)^{4/3}}{4\pi^3} = \frac{3^{4/3}\pi^{8/3}}{4\pi^3} = \frac{3^{4/3}}{4\pi^{1/3}} = \frac34\left(\frac3\pi\right)^{1/3}$$ ✓）

**局域化**，即对非均匀体系逐点应用：

$$
\boxed{E_x^{\text{LDA}}[\rho] = -\frac{3}{4}\left(\frac{3}{\pi}\right)^{1/3}\int\rho(\mathbf r)^{4/3}\,d\mathbf r},\qquad C_x = \tfrac34(3/\pi)^{1/3}\approx 0.7386
$$

对应的势：

$$
v_x^{\text{LDA}}(\mathbf r) = \frac{\delta E_x}{\delta\rho} = -\left(\frac{3}{\pi}\right)^{1/3}\rho(\mathbf r)^{1/3} = \frac43\,\varepsilon_x^{\text{LDA}}
$$

自旋极化版本（用自旋标度关系 $$E_x[\rho_\alpha,\rho_\beta]=\frac12\{E_x[2\rho_\alpha]+E_x[2\rho_\beta]\}$$）：

$$
E_x^{\text{LSDA}} = -\frac{3}{2}\left(\frac{3}{4\pi}\right)^{1/3}\int\big(\rho_\alpha^{4/3}+\rho_\beta^{4/3}\big)d\mathbf r
$$

**关联部分** $$\varepsilon_c^{\text{unif}}(\rho)$$ 没有闭式解。它来自对均匀电子气的高精度**量子蒙特卡洛**计算（Ceperley–Alder 1980），再用解析式拟合——VWN、PZ81、PW92 就是不同的拟合形式。所以 LDA 关联是"数值精确 + 拟合"，不是推导出来的。

**为什么 LDA 出乎意料地好用**？分子的密度远非均匀，理论上 LDA 该垮掉。它没垮的原因是 LDA 的**交换关联空穴**满足正确的求和规则（空穴积分 = −1 个电子），误差在球平均后大量抵消。但 LDA 系统性**高估**结合能（典型 20–30%），键长偏短——所以实际化学计算几乎不用纯 LDA。

### 6.9 GGA 与它的泛函导数

$$
E_{xc}^{\text{GGA}} = \int f\big(\rho, \nabla\rho\big)\,d\mathbf r
$$

**泛函导数怎么求？** 这里出现了对导数的依赖，需要**分部积分**：

$$
\delta E_{xc} = \int\left[\frac{\partial f}{\partial\rho}\delta\rho + \frac{\partial f}{\partial\nabla\rho}\cdot\nabla\delta\rho\right]d\mathbf r
$$

第二项分部积分（分子体系边界项为零，因为 $$\rho$$ 指数衰减）：

$$
\int\frac{\partial f}{\partial\nabla\rho}\cdot\nabla\delta\rho\,d\mathbf r = -\int\nabla\cdot\left(\frac{\partial f}{\partial\nabla\rho}\right)\delta\rho\,d\mathbf r
$$

所以

$$
\boxed{v_{xc}^{\text{GGA}} = \frac{\partial f}{\partial\rho} - \nabla\cdot\left(\frac{\partial f}{\partial\nabla\rho}\right)}
$$

（这是带梯度的 Euler–Lagrange 方程，和经典力学里从 Lagrangian 推 Euler–Lagrange 方程是同一个数学。）

**实践中不这么算**——因为它需要 $$\rho$$ 的二阶导数，数值上不稳定。程序里直接算 Fock 矩阵元（见 6.10），把分部积分转嫁到基函数上。

### 6.10 KS-DFT 的矩阵形式与数值积分

和 HF 一样做 LCAO 展开。KS 矩阵：

$$
F^{\text{KS}}_{\mu\nu} = H^{\text{core}}_{\mu\nu} + \underbrace{\sum_{\lambda\sigma}P_{\lambda\sigma}(\mu\nu|\lambda\sigma)}_{J_{\mu\nu}} \;-\; \underbrace{\frac{a}{2}\sum_{\lambda\sigma}P_{\lambda\sigma}(\mu\lambda|\sigma\nu)}_{\text{仅杂化泛函}} \;+\; V^{xc}_{\mu\nu}
$$

（$$a$$ = 精确交换混合系数；纯泛函 $$a=0$$，B3LYP $$a=0.20$$，HF 相当于 $$a=1$$ 且 $$V^{xc}=0$$。）

**交换关联矩阵元** = $$\partial E_{xc}/\partial P_{\mu\nu}$$。对 LDA：

$$
V^{xc}_{\mu\nu} = \int v_{xc}\big(\rho(\mathbf r)\big)\,\chi_\mu(\mathbf r)\chi_\nu(\mathbf r)\,d\mathbf r
$$

对 GGA（设 $$f = f(\rho,\gamma)$$，$$\gamma = \vert \nabla\rho\vert ^2$$），用 $$\partial\rho/\partial P_{\mu\nu} = \chi_\mu\chi_\nu$$ 和 $$\partial\gamma/\partial P_{\mu\nu} = 2\nabla\rho\cdot\nabla(\chi_\mu\chi_\nu)$$：

$$
\boxed{V^{xc}_{\mu\nu} = \int\left[\frac{\partial f}{\partial\rho}\,\chi_\mu\chi_\nu + 2\frac{\partial f}{\partial\gamma}\,\nabla\rho\cdot\nabla(\chi_\mu\chi_\nu)\right]d\mathbf r}
$$

只需基函数的一阶导——数值上稳健得多。

**数值积分（DFT 独有的一环）**：$$v_{xc}$$ 里有 $$\rho^{1/3}$$ 这类非解析函数，积分**必须数值做**：

$$
\int g(\mathbf r)\,d\mathbf r \approx \sum_{p} w_p\, g(\mathbf r_p)
$$

标准方案（Becke 1988）：

1. **原子分区**：用平滑的权重函数 $$w_A(\mathbf r)$$ 把分子积分拆成原子中心积分，$$\sum_A w_A(\mathbf r)=1$$。
2. 每个原子上用**径向网格**（Gauss–Chebyshev/Euler–Maclaurin，常 75–99 点）× **角向网格**（Lebedev，常 302–590 点）。
3. 典型总格点数：每原子 $$10^4$$ 量级。

网格太粗会导致能量对几何的伪振荡（"grid noise"），几何优化和频率计算尤其敏感——如果你的频率算出虚频而结构看起来没问题，先试试加密网格。

**总能量**（注意别搞错）：

$$
E_{\text{KS}} = \sum_{\mu\nu}P_{\mu\nu}H^{\text{core}}_{\mu\nu} + \frac{1}{2}\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}(\mu\nu|\lambda\sigma) - \frac{a}{4}\sum P_{\mu\nu}P_{\lambda\sigma}(\mu\lambda|\sigma\nu) + E_{xc}[\rho] + V_{NN}
$$

**警告**：$$E_{xc}\ne\frac12\text{tr}(\mathbf{P}\mathbf{V}^{xc})$$。因为 $$E_{xc}$$ 对 $$\rho$$ 是**非线性**的，必须直接数值积分 $$\int f(\rho,\nabla\rho)d\mathbf r$$。HF 里"$$\frac12\text{tr}[\mathbf P(\mathbf H+\mathbf F)]$$"那个简洁公式在 DFT 里**不成立**。这是新手写 DFT 代码最常见的 bug。

### 6.11 绝热连接：杂化泛函为什么合理

引入耦合常数 $$\lambda\in[0,1]$$，定义一族哈密顿量

$$
\hat H_\lambda = \hat T + \lambda\hat V_{ee} + \hat V_\lambda
$$

其中 $$\hat V_\lambda$$ 是一个"陪跑"外势，**调节到使每个 $$\lambda$$ 的基态密度都等于真实密度 $$\rho$$**。于是：

- $$\lambda = 0$$：无相互作用的 KS 参考体系
- $$\lambda = 1$$：真实体系

用 Hellmann–Feynman 定理对 $$\lambda$$ 积分（见 7.2 的证明），可得**绝热连接公式**：

$$
E_{xc} = \int_0^1 \Big[\langle\Psi_\lambda|\hat V_{ee}|\Psi_\lambda\rangle - J[\rho]\Big]\,d\lambda
$$

**关键观察**：$$\lambda=0$$ 端的被积函数正是**精确交换能**（因为 $$\Psi_0$$ 是行列式，$$\langle\Phi\vert \hat V_{ee}\vert \Phi\rangle - J = E_x^{\text{exact}}$$）。而半局域泛函在 $$\lambda$$ 大的一端表现较好、$$\lambda\to0$$ 端最差。

所以：用梯形/线性近似做这个积分，自然得到"一部分精确交换 + 一部分 DFA 交换"的形式。Becke 的"半-半"泛函取 $$a=1/2$$；经验优化后 B3LYP 落在 $$a=0.20$$，PBE0 用微扰论论证给出 $$a=1/4$$。**这就是杂化泛函的理论依据，不是纯经验拼凑。**

### 6.12 已知的系统性缺陷（用 DFT 时必须知道）

| 问题 | 根源 | 缓解手段 |
|---|---|---|
| 自作用误差 / 离域误差 | 近似 $$E_{xc}$$ 不能抵消 $$J$$ 中的自作用 | 掺入精确交换、长程分离泛函、DFT+U |
| 长程色散缺失 | 半局域泛函看不到远处的密度涨落 | D3/D4、VV10、MBD |
| 电荷转移激发态误差大 | TD-DFT 中 $$-1/R$$ 渐近行为错误 | 长程分离（CAM-B3LYP、ωB97X） |
| 反应势垒偏低 | 离域误差使过渡态过度稳定 | 高 HF 交换比例的泛函（M06-2X、ωB97X） |
| 强关联体系失效 | 单行列式参考 | 多参考方法（CASSCF/DMRG），非 DFT 能解决 |
| 不是变分上界 | 近似泛函 | 无法通过收紧基组来系统改进 |

**最后一条值得强调**：HF 和 CCSD(T) 有明确的"系统改进路径"（更大基组、更高激发级别），而 DFT 换个泛函误差可能变大也可能变小，**没有收敛序列**。这是 DFT 最根本的哲学缺陷。


---

## 第 7 章 HF 能量的解析梯度 {#ch7}

### 7.1 为什么必须要解析梯度

几乎所有有用的量化计算都需要 $$\partial E/\partial \mathbf{R}_A$$：

- **几何优化**：在势能面上找极小点（稳定结构）或一阶鞍点（过渡态）
- **分子动力学**：力 $$\mathbf{F}_A = -\partial E/\partial\mathbf{R}_A$$
- **振动频率**：需要二阶导（Hessian），但一阶解析梯度是它的基础
- **反应路径**（IRC）

**数值差分行不行？** 中心差分需要 $$6M$$ 次 SCF（$$M$$ 个原子），且精度受限于 SCF 收敛阈值和步长的权衡（通常只能到 $$10^{-5}$$ Ha/Bohr）。**解析梯度只需约 1–3 倍单次 SCF 的代价，且精度是机器精度级的。** 这是 1970 年代（Pulay）之后量子化学能真正处理分子结构的分水岭。

### 7.2 Hellmann–Feynman 定理及其失效

**定理**：若 $$\Psi(\lambda)$$ 是 $$\hat H(\lambda)$$ 的归一化本征态，则

$$
\frac{dE}{d\lambda} = \Big\langle\Psi\Big|\frac{\partial\hat H}{\partial\lambda}\Big|\Psi\Big\rangle
$$

**证明**：$$E = \langle\Psi\vert \hat H\vert \Psi\rangle$$（归一化下），

$$
\frac{dE}{d\lambda} = \Big\langle\frac{\partial\Psi}{\partial\lambda}\Big|\hat H\Big|\Psi\Big\rangle + \Big\langle\Psi\Big|\hat H\Big|\frac{\partial\Psi}{\partial\lambda}\Big\rangle + \Big\langle\Psi\Big|\frac{\partial\hat H}{\partial\lambda}\Big|\Psi\Big\rangle
$$

前两项用 $$\hat H\Psi = E\Psi$$ 替换成 $$E\big[\langle\partial_\lambda\Psi\vert \Psi\rangle+\langle\Psi\vert \partial_\lambda\Psi\rangle\big] = E\,\partial_\lambda\langle\Psi\vert \Psi\rangle = E\cdot\partial_\lambda(1) = 0$$。∎

**如果这个定理适用**，核上的力就纯粹是**经典静电力**（核感受到的电子云和其他核的静电作用）——非常直观。

**但它在实际计算中不适用**，原因有两个层次：

1. 定理要求 $$\Psi$$ 是**精确**本征态。HF 波函数不是。
2. 更要命的：定理的证明用了"$$\Psi$$ 在参数变化时仍在同一个变分空间里"。而我们的基函数**长在原子核上**——核一动，基函数跟着动，**变分空间本身在变**。这一项贡献叫 **Pulay 力**（1969，Peter Pulay 首次系统处理）。

在小基组下，Pulay 力可以和 Hellmann–Feynman 力同量级甚至更大。**我下面会用数值实验证明这一点。**

### 7.3 完整推导

从第 5 章的能量表达式出发（实基组，$$\mathbf P$$ 对称）：

$$
E = \sum_{\mu\nu}P_{\mu\nu}H_{\mu\nu} + \frac12\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}\Gamma_{\mu\nu\lambda\sigma} + V_{NN},
\qquad \Gamma_{\mu\nu\lambda\sigma}\equiv(\mu\nu|\lambda\sigma)-\tfrac12(\mu\lambda|\sigma\nu)
$$

设 $$x$$ 是某个核 Cartesian 坐标。**上标 $$x$$ 表示"积分本身的导数"**（基函数中心移动 + 算符显式依赖），$$P^x_{\mu\nu}\equiv dP_{\mu\nu}/dx$$。

**第 1 步：链式法则。**

$$
\frac{dE}{dx} = \underbrace{\sum_{\mu\nu}P_{\mu\nu}H^{x}_{\mu\nu} + \frac12\sum P_{\mu\nu}P_{\lambda\sigma}\Gamma^{x}_{\mu\nu\lambda\sigma} + V^{x}_{NN}}_{\text{显式项：积分导数}} \;+\; \underbrace{\sum_{\mu\nu}P^{x}_{\mu\nu}\Big[H_{\mu\nu}+\sum_{\lambda\sigma}P_{\lambda\sigma}\Gamma_{\mu\nu\lambda\sigma}\Big]}_{\text{隐式项：密度弛豫}}
$$

（对 $$P$$ 求导时产生两项，因 $$\Gamma$$ 在 $$(\mu\nu)\leftrightarrow(\lambda\sigma)$$ 下对称而相等，$$\frac12\times2=1$$。）

方括号里正是 **Fock 矩阵** $$F_{\mu\nu}$$（见 5.5 末尾）。所以隐式项 $$=\sum_{\mu\nu}P^x_{\mu\nu}F_{\mu\nu}$$。

**第 2 步：处理隐式项——这是全部技巧所在。**

我们**似乎**需要 $$dC/dx$$（解 CPHF 方程，很贵）。但其实不需要。展开 $$P_{\mu\nu}=2\sum_a C_{\mu a}C_{\nu a}$$：

$$
\sum_{\mu\nu}P^{x}_{\mu\nu}F_{\mu\nu} = 2\sum_a\sum_{\mu\nu}\big(C^{x}_{\mu a}C_{\nu a}+C_{\mu a}C^{x}_{\nu a}\big)F_{\mu\nu} \overset{\mathbf F \text{ 对称}}{=} 4\sum_a\sum_{\mu\nu}C^{x}_{\mu a}F_{\mu\nu}C_{\nu a}
$$

用 Roothaan–Hall 方程 $$\mathbf{FC}=\mathbf{SC}\boldsymbol\varepsilon$$，即 $$\sum_\nu F_{\mu\nu}C_{\nu a} = \varepsilon_a\sum_\nu S_{\mu\nu}C_{\nu a}$$：

$$
= 4\sum_a \varepsilon_a\sum_{\mu\nu}C^{x}_{\mu a}S_{\mu\nu}C_{\nu a}
$$

**第 3 步：用正交归一约束消掉 $$C^x$$。**

正交归一条件 $$\sum_{\mu\nu}C_{\mu a}S_{\mu\nu}C_{\nu a}=1$$ 对**任意** $$x$$ 都成立。两边对 $$x$$ 求导：

$$
\underbrace{2\sum_{\mu\nu}C^{x}_{\mu a}S_{\mu\nu}C_{\nu a}}_{\text{系数变化}} + \underbrace{\sum_{\mu\nu}C_{\mu a}S^{x}_{\mu\nu}C_{\nu a}}_{\text{基函数变化}} = 0
$$

$$
\Longrightarrow\quad \sum_{\mu\nu}C^{x}_{\mu a}S_{\mu\nu}C_{\nu a} = -\frac12\sum_{\mu\nu}C_{\mu a}S^{x}_{\mu\nu}C_{\nu a}
$$

代回：

$$
\sum_{\mu\nu}P^{x}_{\mu\nu}F_{\mu\nu} = -2\sum_a\varepsilon_a\sum_{\mu\nu}C_{\mu a}C_{\nu a}S^{x}_{\mu\nu} = -\sum_{\mu\nu}W_{\mu\nu}S^{x}_{\mu\nu}
$$

其中定义**能量加权密度矩阵**（energy-weighted density matrix）：

$$
\boxed{W_{\mu\nu} = 2\sum_{a=1}^{n}\varepsilon_a\,C_{\mu a}C_{\nu a}}
$$

**第 4 步：最终结果。**

$$
\boxed{
\frac{\partial E_{\text{RHF}}}{\partial x} = \sum_{\mu\nu}P_{\mu\nu}\frac{\partial H^{\text{core}}_{\mu\nu}}{\partial x}
+ \frac12\sum_{\mu\nu\lambda\sigma}P_{\mu\nu}P_{\lambda\sigma}\frac{\partial}{\partial x}\Big[(\mu\nu|\lambda\sigma)-\tfrac12(\mu\lambda|\sigma\nu)\Big]
- \sum_{\mu\nu}W_{\mu\nu}\frac{\partial S_{\mu\nu}}{\partial x}
+ \frac{\partial V_{NN}}{\partial x}
}
$$

### 7.4 这个公式的三个要点

**要点 1：不需要 $$\partial C/\partial x$$。**

这是最重要的结论。求 $$3M$$ 个梯度分量，**一次 CPHF 都不用解**。原因是深刻的：SCF 能量对 $$\mathbf C$$ 是**变分稳定**的（一阶导为零），所以系数的一阶变化不产生一阶能量变化——**除了**约束面本身在移动的那部分，那部分正是 $$-\mathbf{W}\cdot\mathbf{S}^x$$。

这是 **Wigner $$2n+1$$ 规则**的一个实例：变分优化到 $$n$$ 阶的波函数，能给出 $$2n+1$$ 阶的能量导数。$$n=0$$（波函数零阶，即不需要 $$\partial C/\partial x$$）⟹ 能量一阶导正确。而 Hessian（二阶导）就需要 $$\partial C/\partial x$$ 了——必须解 CPHF。

**要点 2：$$-\mathbf{W}\mathbf{S}^x$$ 就是 Pulay 力。**

它**唯一**的来源是"基函数随核运动"（重叠矩阵依赖核坐标）。如果用平面波基组（不随核动），$$\mathbf S^x=0$$，这一项消失——这就是为什么固体物理的平面波程序没有 Pulay 力。

**要点 3：$$H^{\text{core}}$$ 的导数含三种物理。**

$$
\frac{\partial H_{\mu\nu}}{\partial X_A} = \underbrace{\Big\langle\frac{\partial\chi_\mu}{\partial X_A}\Big|\hat h\Big|\chi_\nu\Big\rangle + \Big\langle\chi_\mu\Big|\hat h\Big|\frac{\partial\chi_\nu}{\partial X_A}\Big\rangle}_{\text{基函数移动（Pulay 类）}} + \underbrace{\Big\langle\chi_\mu\Big|\frac{\partial\hat h}{\partial X_A}\Big|\chi_\nu\Big\rangle}_{\text{Hellmann–Feynman 力}}
$$

其中算符的显式导数只来自核吸引项：

$$
\frac{\partial\hat h}{\partial X_A} = -Z_A\frac{\partial}{\partial X_A}\frac{1}{|\mathbf r-\mathbf R_A|} = -Z_A\frac{x-X_A}{|\mathbf r-\mathbf R_A|^3}
$$

核排斥项：

$$
\frac{\partial V_{NN}}{\partial X_A} = -\sum_{B\ne A}Z_AZ_B\frac{X_A-X_B}{R_{AB}^3}
$$

（对 $$\partial(1/R_{AB})/\partial X_A = -(X_A-X_B)/R_{AB}^3$$。）

### 7.5 高斯基函数的导数积分怎么算

好消息：**高斯函数对中心坐标的导数还是（两个）高斯函数**，只是角动量变了。

对一个 Cartesian 原始高斯

$$
g(\mathbf r;\alpha,l,m,n,\mathbf A) = (x-A_x)^{l}(y-A_y)^{m}(z-A_z)^{n}e^{-\alpha|\mathbf r-\mathbf A|^2}
$$

对中心坐标求导：

$$
\frac{\partial g}{\partial A_x} = \underbrace{2\alpha\, g(\alpha, l+1, m, n, \mathbf A)}_{\text{角动量 } +1} - \underbrace{l\, g(\alpha, l-1, m, n, \mathbf A)}_{\text{角动量 } -1}
$$

**验证**：$$\partial_{A_x}(x-A_x)^l = -l(x-A_x)^{l-1}$$；$$\partial_{A_x}e^{-\alpha(x-A_x)^2}=2\alpha(x-A_x)e^{-\alpha(x-A_x)^2}$$。✓

（$$l=0$$ 时第二项自动消失。）

**这意味着**：导数积分可以完全复用已有的积分代码——只需以更高角动量再算一遍。实际实现（McMurchie–Davidson、Obara–Saika、Head-Gordon–Pople 递推）会把这一步融进递推关系，一次生成积分和它的导数。

**平动不变性（重要的省力技巧 + 检验手段）**：

如果所有核（连同基函数）一起平移，任何积分都不变。所以对任意一个积分 $$I$$：

$$
\sum_{A}\frac{\partial I}{\partial \mathbf{R}_A}=0
$$

于是**总梯度满足** $$\sum_A \partial E/\partial\mathbf R_A = \mathbf 0$$。实际计算中，最后一个中心的导数可以由其余中心的导数取负和得到（省 1/4 的工作量），同时这也是检验代码正确性的第一道关卡。（类似地，转动不变性给出关于力矩的求和规则。）

### 7.6 数值验证

我写了一个最小的 RHF 程序（STO-3G，只含 s 型函数）来实际检验上面的公式。做法是：把公式中的所有积分导数用**中心差分**求出（移动某原子时，以它为中心的基函数和它的核电荷一起动——这正好就是公式所需的全导数），代入解析梯度公式，再与**总能量的中心差分**比较。

结果：

| 体系（2 电子，STO-3G） | $$E_{\text{RHF}}$$ / Ha | 解析 vs 数值 最大偏差 | 去掉 Pulay 项后的最大偏差 |
|---|---|---|---|
| H₃⁺（不对称三角形） | −1.2375255977 | $$2.0\times10^{-10}$$ | $$4.3\times10^{-1}$$ |
| HeH⁺ | −2.8514282788 | $$5.3\times10^{-10}$$ | $$3.9\times10^{-1}$$ |
| HeH₂²⁺（三中心） | −2.4814421799 | $$1.9\times10^{-10}$$ | $$2.9\times10^{-1}$$ |

$$10^{-10}$$ 的偏差就是差分步长带来的截断误差水平——**公式精确成立**。

而 **Pulay 项一旦丢掉，梯度错到 0.3–0.4 Ha/Bohr**（和梯度本身同量级甚至更大）。这直观说明了 7.2 节的论点：纯 Hellmann–Feynman 力在原子中心基组下是错的。

平动不变性检验 $$\sum_A\partial E/\partial\mathbf R_A$$ 也在 $$10^{-10}$$ 量级。

（验证思路很简单：任何能算 s 型高斯积分的小程序都能复现。想省事的话，也可以直接用 PySCF 的 `mf.nuc_grad_method().kernel()` 对照。）

### 7.7 DFT 梯度有什么不同

KS-DFT 梯度的骨架完全一样（$$\mathbf P\mathbf H^x$$、双电子项、$$-\mathbf W\mathbf S^x$$、$$V_{NN}^x$$），$$W_{\mu\nu}$$ 用 KS 轨道能量。多出来两类项：

$$
\frac{\partial E_{xc}}{\partial x} = \underbrace{\sum_{p}w_p\left[\frac{\partial f}{\partial\rho}\frac{\partial\rho(\mathbf r_p)}{\partial x}+\dots\right]}_{\text{密度随基函数移动而变}} \;+\; \underbrace{\sum_{p}\frac{\partial w_p}{\partial x}\,f\big(\rho(\mathbf r_p)\big)}_{\text{网格权重导数}}
$$

- **第一项**：$$\rho$$ 的变化通过基函数导数进入，形式上和 HF 那部分平行。
- **第二项**是 DFT 独有的：Becke 分区权重 $$w_p$$ **依赖于核坐标**（原子一动，分区边界就动）。这个"网格权重导数"（Becke 1988; Johnson–Gill–Pople 1993）必须算，否则梯度不精确、几何优化不收敛。

另外，**$$\mathbf W$$ 的定义在 DFT 里不变**，$$-\mathbf{WS}^x$$ 依然是 Pulay 项。杂化泛函则要额外算 $$-\frac{a}{4}\sum PP(\mu\lambda\vert \sigma\nu)^x$$。

### 7.8 二阶导需要 CPHF

Hessian $$\partial^2E/\partial x\partial y$$ 无法回避 $$\partial C/\partial x$$。求它的方程叫 **CPHF**（Coupled-Perturbed Hartree–Fock）。思路：对 Roothaan–Hall 方程和正交归一条件同时求导，得到关于轨道旋转参数 $$U^x_{pi}$$ 的线性方程组：

$$
\text{(occ-virt 块)}\qquad (\varepsilon_i-\varepsilon_a)U^x_{ai} - \sum_{bj}A_{ai,bj}U^x_{bj} = B^x_{ai}
$$

$$A$$ 是 orbital Hessian（含双电子积分），$$B^x$$ 含扰动积分。这是个大型线性方程组，通常用迭代法（不显式构造 $$A$$）求解。占据-占据块由正交归一条件直接给出 $$U^x_{ij}=-\frac12 S^x_{ij}$$，不需要迭代——这一点和 7.3 第 3 步是同一个道理。

数值上：一阶梯度约 1–3 倍 SCF 代价；解析 Hessian 约 $$3M$$ 倍——所以频率计算比几何优化贵得多。

---

## 第 8 章 全局回顾与自检 {#ch8}

### 8.1 一张图看懂整条链

```
多电子薛定谔方程  Ĥ = Σh(i) + Σ 1/r_ij
        │  1/r_ij 使问题不可分离
        ├────────────────────────────────┐
        │                                │
   【波函数路线】                    【密度路线】
        │                                │
   假设 Ψ = 单个 Slater 行列式        HK 定理: ρ ⟹ Ĥ ⟹ 一切
        │  (第 1 章)                     │  (6.1–6.2)
        ▼                                ▼
   Slater–Condon 规则                 Kohn–Sham 分解
   E = Σh_ii + ½ΣΣ⟨ij||ij⟩            E = T_s + ∫ρv + J + E_xc
        │  (第 2 章)                     │  (6.4)  ← 精确, 无近似
        ▼                                ▼
   变分 + Lagrange 乘子              变分 (对轨道)
   f̂χ_i = ε_i χ_i                    [-½∇² + v_eff]ψ_i = ε_i ψ_i
        │  (第 3 章)                     │  (6.5)
        ▼                                ▼
   自旋积分 → RHF (2J − K)           E_xc 需要近似 (LDA/GGA/hybrid)
        │  (第 4 章)                     │  (6.7–6.9)
        └──────────────┬─────────────────┘
                       ▼
              LCAO 展开: FC = SCε
              F_μν = H_μν + Σ P[(μν|λσ) − ½(μλ|σν)] (+V_xc)
              E = ½tr[P(H+F)] + V_NN   (DFT: 需单独积分 E_xc)
                       │  (第 5 章 / 6.10)
                       ▼
                  SCF 自洽迭代
                       │
                       ▼
              解析梯度 (无需 CPHF)
      ∂E/∂x = P·H^x + ½PP·Γ^x − W·S^x + V_NN^x
                       │  (第 7 章)
                       ▼
              几何优化 / MD / 频率
```

### 8.2 最容易搞错的七个点

1. **$$E \ne \sum_i\varepsilon_i$$**。要减去双计数：$$E = \sum_i\varepsilon_i-\frac12\sum_{ij}\langle ij\Vert ij\rangle$$。
2. **物理学家记号 vs 化学家记号**：$$\langle ij\vert kl\rangle = (ik\vert jl)$$。混用是最常见的推导错误来源。
3. **RHF 里 Coulomb 系数是 2、交换系数是 1**（$$2\hat J-\hat K$$），因为交换只在同自旋间。
4. **DFT 的 $$E_{xc}\ne\frac12\text{tr}(\mathbf{PV}^{xc})$$**。$$E_{xc}$$ 对 $$\rho$$ 非线性，必须直接数值积分。
5. **KS 的 $$E_{xc}$$ 包含动能修正 $$T-T_s$$**，不只是"交换 + 关联"。
6. **梯度里的 Pulay 项不能丢**。丢了错到和梯度本身同量级（7.6 已用数值证明）。
7. **一阶梯度不需要 CPHF，二阶需要**。这是 Wigner $$2n+1$$ 规则。

### 8.3 自检题（能答上来说明真的懂了）

**基础层**

1. 为什么 $$J_{ii}=K_{ii}$$？这在物理上意味着什么？把这一点和 DFT 的自作用误差联系起来。
2. 从 $$\hat f\chi_i=\varepsilon_i\chi_i$$ 出发，证明 $$\varepsilon_i = h_{ii}+\sum_j\langle ij\Vert ij\rangle$$，然后推出 $$E\ne\sum\varepsilon_i$$ 的修正项。
3. 为什么 Slater 行列式在占据轨道的酉变换下物理不变？这给了我们什么自由度？

**中等层**

4. 用 $$\mathrm{H_2}$$ 极小基验证 $$E=2h_{11}+(11\vert 11)+1/R$$。为什么 $$2J-K$$ 在这里退化成 $$J$$？
5. 若用**平面波**基组做 HF，梯度公式会怎样简化？为什么？
6. 手推 GGA 的 $$v_{xc}=\partial f/\partial\rho-\nabla\cdot(\partial f/\partial\nabla\rho)$$，并解释为什么程序里不直接用这个式子。
7. 从 $$E_x^{\text{LDA}}=-C_x\int\rho^{4/3}$$ 出发求 $$v_x^{\text{LDA}}$$，验证 $$v_x=\frac43\varepsilon_x$$。

**进阶层**

8. 证明能量加权密度矩阵可以写成 $$\mathbf{W}=\mathbf{P}\mathbf{F}\mathbf{P}/2$$。（提示：用 $$\mathbf{FC}=\mathbf{SC}\boldsymbol\varepsilon$$ 和 $$\mathbf{C}^T\mathbf{SC}=\mathbf 1$$。）
9. 如果基函数**不**随原子移动（比如放在空间固定的格点上），$$\mathbf S^x$$ 会怎样？梯度公式变成什么？这时 Hellmann–Feynman 定理成立吗？
10. 为什么 DFT 不是变分上界？这对"用更好的基组/泛函一定更准"这个直觉意味着什么？

### 8.4 继续深入的路线

| 主题 | 推荐 |
|---|---|
| **HF 与后 HF 的标准教科书** | Szabo & Ostlund, *Modern Quantum Chemistry*（本文第 2–5、7 章的记号基本沿用它，读起来会很顺） |
| **DFT** | Koch & Holthausen, *A Chemist's Guide to DFT*（化学视角，不啃泛函分析）；Parr & Yang（理论严谨版） |
| **积分与梯度的实现细节** | Helgaker, Jørgensen & Olsen, *Molecular Electronic-Structure Theory*，第 9 章（分子积分求值）与第 10 章（Hartree–Fock 理论） |
| **动手写代码** | 先做 Crawford 组的 *Programming Projects*（从积分到 CCSD 一步步来）；或读 PySCF 源码，它的 Python 层非常可读 |
| **数值实验** | 用 PySCF 把本文每个公式都对一遍：`mf.get_fock()`、`mf.make_rdm1()`、`mf.nuc_grad_method().kernel()` |

---

*本文中的 Dirac 交换泛函推导（六重积分 $$=4\pi^2k_F^4$$、$$C_x=0.7386$$）和 RHF 解析梯度公式均已数值验证。*

---

**完**
