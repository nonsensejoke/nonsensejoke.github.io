---
layout: default
title: "MHC Class I/II、HLA 与 Allele：从蛋白结构到 HLA-A/B/C"
date: 2026-09-10
categories: [biology, immunology]
tags: [MHC, HLA, allele, antigen-presentation, immunology]
---

最近在理解 MHC（Major Histocompatibility Complex，主要组织相容性复合体）时，发现很多概念其实是层层嵌套的：

- MHC Class I 和 Class II 到底是什么？
- `α1`、`α2` 是不是两条肽链？
- MHC 的 allele（等位基因）是什么意思？
- `HLA-A*02:01` 到底代表什么？
- `HLA-A`、`HLA-B`、`HLA-C` 又是什么关系？

这篇文章把这些概念从蛋白结构一路整理到基因和 allele，希望把它们之间的关系一次理清。

---

## 一、MHC 是什么？

MHC 分子的核心作用可以简单理解成：

> **把细胞中的蛋白片段（peptide）展示在细胞表面，让 T 细胞检查。**

人体中的 MHC 系统称为 **HLA（Human Leukocyte Antigen）**。

MHC 主要分为两类：

- **MHC Class I**
- **MHC Class II**

两者都负责呈递 peptide，但来源、表达细胞以及识别它们的 T 细胞不同。

| 特征 | MHC Class I | MHC Class II |
|---|---|---|
| 主要表达位置 | 几乎所有有核细胞 | 专业抗原呈递细胞 |
| 常见抗原来源 | 细胞内蛋白 | 细胞外摄取的蛋白 |
| 主要被谁识别 | CD8⁺ T cell | CD4⁺ T cell |
| peptide-binding groove | α1 + α2 | α1 + β1 |
| 常见 peptide 长度 | 约 8–10 aa | 约 13–18 aa |
| groove 两端 | 相对封闭 | 相对开放 |

---

## 二、MHC Class I 的结构

MHC Class I 由两部分组成：

1. 一条较大的 **α chain / heavy chain**
2. 一条较小的 **β2-microglobulin（β2m）**

其中 α chain 是真正跨膜的那条链。

结构可以简化成：

```text
                  peptide
              ==============
                α1        α2
                 \________/
                     |
                     α3
                     |
                transmembrane
                     |
               cytoplasmic tail
```

β2-microglobulin 会从侧面与 α chain 非共价结合，但它本身不跨膜。

### α1、α2、α3 是三条链吗？

不是。

这是一个特别容易误解的地方。

`α1`、`α2`、`α3` 是**同一条 α chain 上的三个结构域（domain）**：

```text
N-terminus

[ α1 ] — [ α2 ] — [ α3 ] — [ transmembrane ] — [ cytoplasmic tail ]

                                                               C-terminus
```

也就是说：

> **α1 + α2 + α3 实际上属于同一条连续的多肽链。**

其中：

- `α1 + α2` 共同构成 peptide-binding groove
- `α3` 位于更靠近细胞膜的位置，并参与与 CD8 的相互作用

所以完整的 MHC Class I 分子可以粗略写成：

```text
MHC Class I
=
一条 α heavy chain
+
一条 β2-microglobulin
+
结合在 groove 中的 peptide
```

---

## 三、为什么 MHC Class I 通常只能装比较短的 peptide？

MHC Class I 的 peptide-binding groove 由 `α1 + α2` 构成。

其两端相对封闭，因此 peptide 通常不能无限向外延伸。

所以 MHC I 最常见的是结合：

```text
8–10 amino acids
```

其中 9-mer peptide 很常见。

可以把它想象成一个长度相对固定的凹槽：

```text
       peptide
    ┌───────────┐
    │           │
────┘           └────
      MHC I groove
```

---

## 四、MHC Class II 的结构有什么不同？

MHC Class II 不是“一条大 α chain + β2m”。

它本身就是由两条跨膜链组成：

- α chain
- β chain

结构大致为：

```text
             peptide
         =================

          α1          β1
           |           |
          α2          β2
           |           |
          TM          TM
```

其中：

```text
α1 + β1
```

共同构成 peptide-binding groove。

与 MHC I 不同，MHC II 的 groove 两端更加开放，因此可以容纳更长的 peptide。

常见长度大约：

```text
13–18 amino acids
```

甚至可以更长。

---

## 五、MHC 的 allele 到底是什么意思？

理解 HLA 最关键的一步，是把下面几个词分开：

```text
gene / locus
allele
protein
```

### gene / locus

例如：

```text
HLA-A
```

是一个基因位点。

### allele

同一个基因，在不同人群中可以存在很多不同版本。

这些不同版本就叫：

> **allele（等位基因）**

例如：

```text
HLA-A*01:01
HLA-A*02:01
HLA-A*24:02
```

它们都属于 **HLA-A 这个 gene/locus**，但代表不同的 allele。

可以类比成：

```text
HLA-A
= 一本书

HLA-A*01:01
HLA-A*02:01
HLA-A*24:02
= 这本书的不同版本
```

这些版本总体结构非常相似，但 DNA 序列存在差异，因此编码出的 HLA-A 蛋白在部分氨基酸位置也会不同。

---

## 六、为什么 MHC allele 特别重要？

HLA allele 之间最重要的差异之一集中在：

```text
α1 + α2
```

也就是 peptide-binding groove 所在区域。

因此，不同 allele 的 groove 形状、空间、电荷环境会有所区别。

结果就是：

> **不同 HLA allele 偏好结合不同类型的 peptide。**

例如：

```text
某个病毒蛋白
      |
      v
被切成大量 peptide
      |
      +---- peptide A
      +---- peptide B
      +---- peptide C
      +---- peptide D
```

可能：

```text
HLA-A*02:01 更适合结合 peptide A

HLA-A*24:02 更适合结合 peptide C
```

因此，不同人的 HLA genotype 不同，他们能够高效呈递的抗原 peptide repertoire 也会有所差异。

---

## 七、HLA-A*02:01 代表一整个 MHC 分子吗？

严格来说，不完全是。

`HLA-A*02:01` 指的是：

> **HLA-A gene 的一个特定 allele。**

它编码的是一条特定序列的 **HLA-A α heavy chain**。

也就是：

```text
HLA-A*02:01
      |
      v
[ α1 ] — [ α2 ] — [ α3 ] — [ TM ] — [ cytoplasmic tail ]
```

但是完整的 MHC Class I 分子还需要：

```text
HLA-A α chain
      +
β2-microglobulin
      +
peptide
```

因此：

```text
HLA-A*02:01
≠ 整个 MHC Class I complex
```

更准确地说：

```text
HLA-A*02:01
→ 指定一种 HLA-A α heavy chain
```

β2-microglobulin 由另外的 `B2M` 基因编码，并不属于 `HLA-A*02:01` 的序列。

---

## 八、HLA-A*02:01 的编号怎么看？

HLA allele 有统一的命名系统。

例如：

```text
HLA-A*02:01
```

可以拆成：

```text
HLA-A  *  02  :  01
  |        |      |
 gene    allele   specific
 locus   group    protein
```

简单理解：

- `HLA-A`：基因位点
- `02`：allele group
- `01`：进一步区分具体蛋白序列

还可能看到更长的名字：

```text
HLA-A*02:01:01:01
```

后面的字段可以进一步区分：

- 编码区中的同义 DNA 差异
- 非编码区差异
- 更精细的 genomic sequence 差异

因此，`HLA-A*02:01` 更适合理解为：

> **在蛋白分辨率上指定一种 HLA-A heavy-chain 序列。**

如果要精确到完整 DNA / genomic sequence，则通常需要更高字段分辨率的 allele 名称。

---

## 九、人体里的 HLA-A 种类到底有多少？

这里要区分：

```text
一个人
vs
整个人类群体
```

### 对一个人而言

人是二倍体。

一个 HLA-A allele 来自父亲，另一个来自母亲。

例如：

```text
父亲：HLA-A*02:01
母亲：HLA-A*24:02
```

那么这个人的 HLA-A genotype 可以写成：

```text
HLA-A*02:01 / HLA-A*24:02
```

由于 HLA 是 **codominant expression（共显性表达）**，两个 allele 通常都会表达。

因此：

> **一个人在 HLA-A locus 上最多通常表达两种不同的 HLA-A 蛋白。**

如果父母给的是同一个 allele，例如：

```text
HLA-A*02:01 / HLA-A*02:01
```

那么这个 locus 实际上只有一种 HLA-A protein 类型。

### 对整个人类群体而言

情况完全不同。

HLA 是人体中多态性最强的基因系统之一。

在人群中存在大量 HLA-A allele，例如：

```text
HLA-A*01:01
HLA-A*02:01
HLA-A*02:06
HLA-A*03:01
HLA-A*24:02
...
```

这些 allele 会被标准化编号并登记。

随着新的个体被测序，还可能继续发现新的 allele，因此这个 allele catalog 并不是一个永远固定不变的静态列表。

---

## 十、那 HLA-A、HLA-B、HLA-C 又是什么？

这是另一个非常关键的层级关系。

`HLA-A`、`HLA-B`、`HLA-C` **不是同一个基因的三个 allele**。

它们是：

> **三个不同的 MHC Class I 基因位点。**

都位于 chromosome 6 的 MHC 区域中。

可以简单画成：

```text
Chromosome 6

------ HLA-A ------ HLA-B ------ HLA-C ------
```

它们分别编码不同的 MHC I α heavy chain。

不过，因为这三个基因具有共同的进化来源，它们编码出的蛋白结构高度相似：

```text
HLA-A protein:
[ α1 ] — [ α2 ] — [ α3 ] — TM

HLA-B protein:
[ α1 ] — [ α2 ] — [ α3 ] — TM

HLA-C protein:
[ α1 ] — [ α2 ] — [ α3 ] — TM
```

所以：

```text
HLA-A
HLA-B
HLA-C
```

是三个不同 gene / locus。

而：

```text
HLA-A*02:01
HLA-A*24:02
```

才是**同一个 HLA-A gene 的不同 allele**。

---

## 十一、HLA-A 和 HLA-B 功能上有什么区别？

从大框架看，它们做的是同一类工作：

```text
呈递 intracellular peptide
            |
            v
       CD8+ T cell
```

但 HLA-A 和 HLA-B 本身的氨基酸序列不同。

而且：

```text
HLA-A 有自己的一套 alleles
HLA-B 有自己的一套 alleles
HLA-C 也有自己的一套 alleles
```

例如：

```text
HLA-A
├── A*01:01
├── A*02:01
└── A*24:02

HLA-B
├── B*07:02
├── B*27:05
└── B*15:01

HLA-C
├── C*03:04
├── C*07:01
└── C*07:02
```

不同 HLA 分子的 peptide-binding groove 不完全相同，因此能够高效结合的 peptide repertoire 也不同。

这相当于同一个细胞同时安装了多种不同规格的“peptide 展示槽”。

---

## 十二、为什么一个人能表达多种 MHC Class I？

因为我们至少要同时考虑三个经典 Class I locus：

```text
HLA-A
HLA-B
HLA-C
```

每个 locus 又分别从父亲和母亲各继承一个 allele。

例如：

```text
              paternal            maternal

HLA-A         A*02:01             A*24:02
HLA-B         B*07:02             B*15:01
HLA-C         C*03:04             C*07:02
```

由于它们共显性表达，因此如果三个 locus 都是 heterozygous，那么理论上可以产生：

```text
2 HLA-A
+
2 HLA-B
+
2 HLA-C
=
6 种经典 MHC Class I heavy-chain 类型
```

它们都可以与 β2-microglobulin 结合，并在细胞表面呈递 peptide。

---

## 十三、把整个层级关系画在一起

最后把最容易混淆的几个层级放在一起：

```text
MHC
│
├── Class I
│   │
│   ├── HLA-A              ← gene / locus
│   │   ├── A*01:01        ← allele
│   │   ├── A*02:01        ← allele
│   │   └── A*24:02        ← allele
│   │
│   ├── HLA-B              ← gene / locus
│   │   ├── B*07:02
│   │   ├── B*27:05
│   │   └── ...
│   │
│   └── HLA-C              ← gene / locus
│       ├── C*03:04
│       ├── C*07:02
│       └── ...
│
└── Class II
    │
    ├── HLA-DP
    ├── HLA-DQ
    └── HLA-DR
```

如果只记一句话，可以记成：

> **HLA-A、HLA-B、HLA-C 是不同的 Class I 基因；HLA-A*02:01、HLA-A*24:02 是 HLA-A 这个基因的不同 allele。**

---

## 十四、几个最容易混淆的概念

### 1. α1 和 α2 不是两条 peptide chain

它们是：

```text
同一条 HLA α heavy chain 上的两个 structural domains
```

### 2. HLA-A*02:01 不是完整的 MHC I complex

它指定的是：

```text
一种 HLA-A α heavy chain
```

完整 MHC I 还需要 β2-microglobulin 和 peptide。

### 3. HLA-A 和 HLA-B 不是 alleles

它们是：

```text
不同 gene / locus
```

而：

```text
A*02:01
A*24:02
```

才是 HLA-A 的不同 alleles。

### 4. “人类有很多 HLA allele”不代表一个人有很多

整个人群的 allele catalog 非常庞大。

但对于一个具体的人：

```text
每个 autosomal HLA locus
通常只有两个 allele copies
```

分别来自父亲和母亲。

---

## 总结

可以用下面这一条链把整篇文章串起来：

```text
Chromosome 6
     |
     v
HLA-A gene
     |
     +---- allele A*02:01
     |
     v
HLA-A α heavy chain
     |
     +---- α1
     +---- α2
     +---- α3
     +---- transmembrane domain
     |
     + β2-microglobulin
     + peptide
     |
     v
MHC Class I complex
     |
     v
呈递 peptide 给 CD8+ T cell
```

而人体并不只有 HLA-A：

```text
HLA-A
HLA-B
HLA-C
```

是三个经典 MHC Class I loci。

每个 locus 又有大量不同 allele，因此 HLA 系统在人群中具有非常高的遗传多态性。

理解了：

```text
class → gene/locus → allele → protein/domain
```

这几个层级以后，MHC/HLA 的命名体系就会清晰很多。
