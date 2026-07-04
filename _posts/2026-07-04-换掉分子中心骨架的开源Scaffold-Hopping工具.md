---
layout: default
title: "换掉分子中心骨架：开源 / 免费 Scaffold Hopping 工具与实操路线"
date: 2026-07-03 12:00:00 +0800
categories: [药物设计, 工具]
tags: [scaffold-hopping, core-replacement, rdkit, chembounce, cheminformatics, drug-design]
description: "手上有一个分子想换掉中心骨架、保留外围取代基？这篇梳理了免费/开源的 scaffold hopping 与 core replacement 工具，从查替换先例、自动生成到自建 RDKit pipeline 和结构基础生成模型，并给出按场景推荐的组合。"
---

手上有一个分子，想把它的中心部分换成另一套骨架，同时尽量保留外围取代基、关键药效团、3D 形状或结合模式——这类需求在药物设计里叫 **scaffold hopping**，更具体一点就是 **core replacement**。它和普通的相似性搜索不太一样：相似性搜索是"找整体相似分子"，而这里的目标更明确——**固定一部分结构，只替换中心骨架**。本文梳理一下目前**免费或开源**的可用工具和实操路线，从查替换先例、自动生成，到自建 RDKit pipeline 以及结构基础的生成模型，最后按不同场景给几条推荐组合。

> 这里的"免费/开源"包括免费网页工具、开源代码库，以及可免费使用的桌面/工作流工具。具体商业使用、部署、再分发和数据许可，请以各项目自己的许可证为准。

---

## 1. 问题定义与任务分类

### 1.1 问题定义

当前需求是：

> 已有一个分子，希望保留外围取代基或关键药效团，把分子中心部分替换成其他 scaffold。

这类问题通常可以称为：

- **Core replacement**
- **Scaffold replacement**
- **Scaffold hopping**
- **Central linker replacement**
- **Bioisosteric scaffold replacement**

它和普通 ligand-based similarity search 不完全一样。普通相似性搜索往往是"找整体相似分子"，而这里的目标更明确：**固定部分结构，只替换中心骨架**。

### 1.2 先判断你的任务属于哪一类（按连接点数量）

如果中心部分只有 **2 个连接点**，很多工具会把它当作 **linker replacement / linker design**。如果中心部分有 **3 个或更多连接点**，更接近经典 **core / scaffold replacement**。SwissBioisostere 的教程也按 attachment point 数量区分：side chain 为 1 个 R group，linker 为 2 个 R groups，scaffold 为 3 个 R groups。[8]

推荐把原分子先拆成：

```text
R1 -- core/scaffold -- R2
```

或者多连接点情况：

```text
R1 / R2 / R3 -- core/scaffold
```

然后根据 exit vectors（连接点）的数量选择工具路线：

- **2 个连接点** → 优先考虑 linker design 路线（DeLinker / DiffLinker / Link-INVENT）。
- **3 个及以上** → 优先考虑经典 core replacement（SwissBioisostere / ChemBounce / RDKit 自建）。

---

## 2. 方案总览表

| 方案类别 | 推荐工具 | 适用场景 | 优点 | 局限 |
|---|---|---|---|---|
| 可替换片段证据库 | SwissBioisostere | 想查"这个中心能换成什么"，且要有文献先例 | 有 ChEMBL/文献活性数据支撑，不发散 | 是证据库而非完整生成器 |
| 专门 scaffold hopping 工具 | ChemBounce | 输入 SMILES 快速替换中心 scaffold | 上手快，目标明确，有 Colab | 仍需后处理和验证 |
| 手工可控 pipeline | RDKit | 已知要保留哪些片段和连接点 | 可解释、可控、适合药化设计 | 需自备 scaffold 库和规则 |
| 生成式分子设计 | REINVENT4 / Link-INVENT | 想生成更多 novel scaffolds、多目标优化 | 化学空间大，可多目标 | 配置、评分函数、过滤要求高 |
| Linker design | DeLinker / DiffLinker / REINVENT4 / RDKit | 中心部分本质上是 linker | 适合固定两端、重建中间连接体 | 需定义 fragment 和连接点 |
| 结构基础生成 | DiffHopp | 有蛋白-配体复合物或可靠 pose | 能考虑 binding pocket 三维环境 | 更偏研究型，部署成本较高 |
| 相似性 / 形状筛选 | SwissSimilarity / Shape-it / USRCAT | 想找 2D 不像但 3D/药效团相似的 hop | 能发现拓扑差异大的 scaffold hop | 不强制保留原 side chains |
| GUI / 工作流 | DataWarrior / KNIME + RDKit nodes | 不想完全写代码 | 可视化、拖拽式流程 | 灵活度不如纯代码 |

---

## 3. 核心方案详解

### 3.1 方案一：SwissBioisostere —— 查历史上可替换的中心片段

**类型**：免费网页数据库 / bioisosteric replacement 资源
**入口**：https://www.swissbioisostere.ch/
**补充说明**：https://www.expasy.org/resources/swissbioisostere

SwissBioisostere 非常适合"我有一个中心 scaffold，想看看能换成什么"的需求。Expasy 对它的说明是：它包含数百万个 molecular replacements 及其在 biochemical assays 中的表现，目的是给药物发现项目提供 bioisosteric modification 思路。[5] SwissBioisostere 自己的网站也说明它可以查询 historical knowledge on molecular replacements and their effect on bioactivity。[6] 2021 年更新论文明确把 scaffold hopping / linker replacement 作为用例；其中也说明：三点连接的片段通常对应 scaffold replacement，两点连接的片段通常对应 linker replacement。[7]

**适合场景**：

- 你已经能明确画出要替换的中心 core/scaffold；
- 想得到"有 ChEMBL / 文献先例支撑"的替换建议；
- 想看替换前后的 bioactivity 趋势，而不仅仅是枚举一堆结构。

**局限**：

- 更像"替换证据数据库"，不是完整的自动分子生成器；
- 结果仍要结合目标蛋白、构象、SAR、合成路线和 ADMET 进一步过滤。

---

### 3.2 方案二：ChemBounce —— 输入 SMILES 自动做 scaffold hopping

**类型**：开源 scaffold hopping 生成框架（近年比较直接面向该需求的工具）
**论文**：https://academic.oup.com/bioinformatics/article/41/9/btaf501/8251522
**GitHub**：https://github.com/jyryu3161/chembounce
**Colab 示例**：https://colab.research.google.com/github/jyryu3161/chembounce/blob/main/chembounce_colab.ipynb

ChemBounce 的目标就是做 scaffold hopping：给定一个输入分子，识别其 core scaffold，并用 scaffold/fragment 库进行替换 [1,2]。Bioinformatics 论文介绍它会基于输入结构生成 scaffold-hopped 分子，使用来自 ChEMBL 的 scaffold library（规模约 300 万+ 片段），同时考虑 shape-based similarity constraints（Tanimoto 与 electron shape similarity）和 synthetic accessibility，以尽量保留药效团和潜在活性。[1] 它内部用 ScaffoldGraph 的 HierS 算法对输入结构做碎片化，并允许你**保留特定部分**（指定哪些片段不被替换）——这正好对应"只换中心、保留外围"的诉求。GitHub 说明中输出包括 "Final structure: the final result SMILES of scaffold hopping"，并提供预计算 scaffold fingerprints [2,3]；Colab 示例还支持自定义 hopping library。[4]

**基本思路**：

```text
输入 hit SMILES
→ ChemBounce 识别 core scaffold
→ 使用 scaffold database 替换核心结构
→ 输出 scaffold-hopped candidates
→ 后续用 RDKit / docking / shape / pharmacophore 过滤
```

**适合场景**：

- 只有一个 hit/lead molecule，想快速生成一批替代中心 scaffold 的候选；
- 希望比纯 MMPA 更自动化，又比深度生成模型更容易上手；
- 不想一开始就自己写复杂的 RDKit pipeline；
- 希望输出候选具有一定结构多样性和合成可及性。

**局限**：

- 生成结构不代表一定可合成、可结合、可保持活性；
- 需要进一步检查连接点、价态、构象和药效团保持情况；
- 若对"要保留哪些部分"有非常严格要求，仍建议结合 RDKit 自定义规则。

**推荐的后处理**：

```text
ChemBounce 输出
→ RDKit 标准化 → 去盐 / 去重 → valence check
→ PAINS / reactive group filter
→ Murcko scaffold clustering
→ 2D similarity / 3D shape similarity → pharmacophore matching
→ docking / MM-GBSA / FEP 等进一步验证
```

---

### 3.3 方案三：RDKit 自定义 Core Replacement Pipeline —— 最灵活、最可控

**类型**：开源 cheminformatics toolkit（BSD license）
**官网/文档**：https://www.rdkit.org/docs/Overview.html
**核心 API**：https://www.rdkit.org/docs/source/rdkit.Chem.rdmolops.html
**R-group 分解文档**：https://www.rdkit.org/docs/source/rdkit.Chem.rdRGroupDecomposition.html

RDKit 官方 overview 写明它有 BSD license、C++ 核心、Python wrapper、2D/3D molecule operations、descriptor generation、PostgreSQL cartridge 以及 KNIME nodes。[9] 如果你已经明确知道：哪个中心 scaffold 要被替换、哪些外围取代基必须保留、有几个连接点、连接点方向或距离是否重要——那么 RDKit 是最可控的方案 [11,12,13]。

**抽象形式**：

```text
R1 - old_core - R2   →   R1 - new_core - R2
```

或多连接点：

```text
R1
 |
old_core - R2      →      new_core（同样接回 R1/R2/R3）
 |
R3
```

**基本工作流**：

```text
1. 输入原始分子
2. 用 SMARTS 或 atom mapping 标记 old_core
3. 将分子拆分为 old_core + R groups
4. 准备带 dummy atoms 的 new scaffold library
5. 按连接点编号重新拼接
6. 生成候选分子
7. 做化学合理性、性质、形状和活性相关过滤
```

**连接点表示**（dummy atoms + atom map）：

```text
[*:1]-new_core-[*:2]
R1-[*:1] + [*:1]-new_core-[*:2] + [*:2]-R2  →  R1-new_core-R2
```

**RDKit 中最相关的 API / 模块**：

- `ReplaceCore()`：移除分子的 core，并在保留下来的 side chains 上用 dummy atoms 标记连接点。[10]
- `ReplaceSidechains()`：反过来保留 core，把 side chains 替换成 dummy atoms，用于识别 attachment points。[10]
- `ReplaceSubstructs()` / reaction SMARTS / `molzip` / 自定义连接逻辑：把带 attachment labels 的新 scaffold 和原 R-groups 接回去。[10]
- `rdkit.Chem.rdRGroupDecomposition` / `RGroupDecompose`：按给定 core 分解 core 与 R groups。
- Murcko scaffold 相关工具、MMPA / fragmentation（`rdMMPA.FragmentMol`）相关工具。

**优点**：最高可控度、可精确定义保留/替换片段、可解释、便于接入内部规则/专利过滤/合成可及性评估，并可与 docking、pharmacophore、shape similarity 无缝衔接。

**局限**：需要写 Python；要自己处理 attachment labeling、价态、芳香性、立体化学、互变异构、去盐、canonicalization、重复结构和不合理结构过滤；生成的 2D 合理结构不代表 3D binding pose 合理。

**特别适合的场景**：ring replacement、bioisostere replacement、central linker hopping、hinge binder scaffold replacement、kinase inhibitor core hopping、GPCR ligand central core replacement、PROTAC linker/scaffold replacement、macrocycle 或 constrained scaffold 枚举。

---

### 3.4 方案四：REINVENT4 / Link-INVENT —— 生成式 scaffold hopping

**类型**：开源生成式分子设计框架
**论文**：https://link.springer.com/article/10.1186/s13321-024-00812-5
**GitHub**：https://github.com/MolecularAI/REINVENT4

REINVENT4 是一个开源生成式分子设计框架，支持 de novo design、scaffold hopping、R-group replacement、linker design、molecule optimization 和 library design 等任务 [15,16]。

**适合场景**：想生成更大的化学空间；希望引入多目标优化；不只是从现成 scaffold library 中替换；想同时优化性质、相似性、对接分数、药效团匹配等。

**可以设置的约束（示例）**：

```text
保留关键外围片段 / 替换中心 scaffold
MW < 500 / cLogP 合理范围 / TPSA < 100
避免 PAINS / reactive groups
保持关键 pharmacophore / 3D shape similarity
docking score 不明显变差 / SA score 可接受
```

**任务模式与需求的关系**：

| REINVENT4 任务 | 与当前需求的关系 |
|---|---|
| Scaffold hopping | 直接生成替代核心骨架 |
| Linker design（Link-INVENT） | 若中心部分连接两个端基，非常适合 |
| Molecule optimization | 在结构约束下优化性质 |
| R-group replacement | 主要换外围取代基，和当前需求相反，但可用于后续优化 |

**局限**：配置复杂度高于 ChemBounce；需要合理设计 scoring function；生成分子需严格去重、过滤和人工审核；若没有高质量评分函数，结果可能新颖但药理意义有限。

---

### 3.5 方案五：把中心部分视为 Linker Design（深度生成模型）

有些分子的"中心 scaffold"本质上是连接两个或多个药效团的 linker。此时可以把问题重新定义为：**固定左端片段和右端片段，重新生成中间 linker。**

```text
Left fragment - old linker - Right fragment  →  Left fragment - new linker - Right fragment
```

**适合工具**：

- **DeLinker**：开源深度生成模型，可把两个 RDKit mol 对象连接起来，用于 fragment linking、scaffold hopping 和 PROTAC design。GitHub 提供 scaffold hopping notebook。[17,18]
- **DiffLinker**：开源 3D diffusion linker design。给定 3D disconnected fragments 后在中间放置缺失原子并设计完整分子，**可连接超过两个片段，并能设置 pocket 约束**（让 linker 契合给定口袋）。[19,20]
- **SyntaLinker**：深度生成，展示了 fragment linking、lead optimization 和 scaffold hopping 用例。[21]
- **REINVENT4 / Link-INVENT** 与 **RDKit linker enumeration**（见上）。

**推荐过滤指标**：连接点数量匹配、连接点距离接近原分子、连接点方向合理、关键 HBD/HBA/hydrophobic centers 保留、3D shape similarity 可接受、pharmacophore feature alignment 可接受、docking pose 保留关键相互作用、合成可及性可接受。

**什么时候优先用 linker design**：如果你的分子可以明显切成 `pharmacophore A - central part - pharmacophore B`，那么 linker design 往往比泛泛的 scaffold hopping 更合适。这类任务中 2D replacement 往往不够，关键是保持 exit vector geometry 和 3D pharmacophore arrangement。

---

### 3.6 方案六：有蛋白结构时使用 Pocket-conditioned 方法（DiffHopp）

**类型**：pocket-conditioned scaffold hopping 生成模型（图扩散）
**论文**：https://arxiv.org/abs/2308.07416
**GitHub**：https://github.com/jostorge/diffusion-hopping

如果你有共晶结构、cryo-EM 结构、docking pose、AlphaFold 结构加可靠 binding site、或关键相互作用信息，可以考虑结构约束 scaffold hopping。DiffHopp 面向的问题是：给定 protein-ligand complex，生成保留关键分子特征但替换 scaffold 的新分子 [22,23]。

**适合场景**：有明确 binding pocket；原配体 pose 比较可信；替换 scaffold 时必须保留三维相互作用；希望新 scaffold 在 pocket 中几何合理。

**优点**：能考虑三维蛋白口袋环境；比纯 2D scaffold replacement 更贴近结构基础药物设计；适合结构生物学数据较充分的项目。

**局限**：更偏研究型；环境配置和模型使用成本较高；输出仍需 docking、MD、合成可行性和人工药化判断；不适合作为唯一决策依据。

> **相关方法补充**：早期的深度 scaffold hopping 工作 **DeepHop（Deep scaffold hopping with multimodal transformer neural networks，Zheng et al., 2021, J. Cheminform.）** 把 scaffold hopping 当作 SMILES 序列翻译问题来做，是这一方向的代表性早期文献。[24]

---

## 4. 其他值得用的免费 / 开源工具

| 工具 | 类型 | 适合做什么 | 备注 |
|---|---|---|---|
| **mmpdb** | 开源命令行 / Python | Matched Molecular Pair Analysis；从已有 SAR 数据中找可替换片段或生成新结构 | PyPI 和 GitHub 都说明 mmpdb 用于识别 matched molecular pairs、预测 property changes 并生成 new molecular structures。[25][26] |
| **ScaffoldGraph** | 开源 Python / CLI | 从一批活性分子构建 scaffold network / scaffold tree，找同靶点不同 scaffold 系列 | 基于 RDKit + NetworkX，`pip install scaffoldgraph` 即可。更偏 scaffold-space 分析和导航，不是直接替换生成器；ChemBounce 的碎片化步骤即基于它。[27][28] |
| **SwissSimilarity** | 免费网页工具 | 用 2D/3D similarity、fingerprint、shape/superposition 等 ligand-based screening 找相似但 scaffold 不同的分子 | 适合"保留功能/活性相似性，不强制保留原 side chains"的 scaffold hopping。[29][30] |
| **DataWarrior** | 免费 / 开源 GUI | 可视化、相似性、scaffold/substitution pattern 分析、组合库/进化库枚举 | 官方说明其为 open-source program，支持 chemical intelligence、scaffold/substitution pattern 趋势可视化。[31] |
| **KNIME + RDKit nodes** | 免费 GUI workflow + RDKit 节点 | 拖拽式流程做 scaffold/R-group 分解、枚举、过滤、打分 | KNIME 的 RDKit 页面说明 RDKit 提供 molecule I/O、substructure search、chemical reactions、2D/3D coordinate generation、fingerprinting 等功能。[32] |
| **REINVENT4 / Link-INVENT** | 开源生成式模型 | De novo design、scaffold hopping、R-group replacement、linker design、property optimization | 论文和 GitHub 均列出 scaffold hopping、R-group replacement、linker design 等任务。[15][16] |
| **DeLinker** | 开源深度生成模型 | 固定两端 fragments，生成中间 linker；可用于 scaffold hopping | GitHub 有 scaffold hopping notebook；bio.tools 说明用于 fragment linking、scaffold hopping 和 PROTAC design。[17][18] |
| **DiffLinker** | 开源 3D diffusion linker design | 给定 3D disconnected fragments，补全中间 linker / scaffold | GitHub 说明给定 3D fragments 后在中间放置缺失原子并设计完整分子；Nature Machine Intelligence 论文介绍其为 3D-conditional diffusion model for molecular linker design。[19][20] |
| **SyntaLinker** | 深度生成，论文/代码生态 | Fragment linking、lead optimization、scaffold hopping | RSC 论文展示了 fragment linking、lead optimization 和 scaffold hopping 用例。[21] |
| **ZINCPharmer / Pharmer** | 免费网页药效团搜索 | 保留关键 pharmacophore features 和空间距离，筛选 purchasable compounds | NAR 论文说明 ZINCPharmer 搜索来自 ZINC purchasable compounds 的 conformations；bio.tools 说明可识别分子结构中的 pharmacophore features。[33][34] |
| **Shape-it / RDKit shape tools / USRCAT** | 开源 3D shape 或 shape+feature screening | 用 3D shape similarity 和 pharmacophoric constraints 找 2D fingerprint 不易发现的 scaffold hops | Shape-it GitHub 是 shape-based alignment 工具；USRCAT 论文强调 shape-based 方法能找拓扑不相似但形状相似的 scaffold-hopped compounds；RDKit 也有 shape alignment API。[35][36][14] |
| **OpenPharmacophore** | 开源 Python 药效团建模 | 从 ligand、ligand-receptor、receptor 或 MD trajectory 中建立 pharmacophore 并做 virtual screening | GitHub/文档说明其目标是处理 pharmacophore models 和 virtual screening。[37] |
| **DeepHop** | 深度生成（multimodal transformer） | 端到端 scaffold hopping（SMILES 翻译式） | Zheng et al., 2021, J. Cheminform.，代表性早期深度学习方法。[24] |

---

## 5. 商业工具备注（非免费，仅供对照）

药化界常用的两个经典商业 scaffold hopping / bioisostere 软件，功能强大但**不免费**，列出供对照：

- **Cresset Spark**：基于 field point / bioisostere 的 field-based scaffold hopping 与 bioisosteric replacement，自带来自商业筛选库、ChEMBL、VeHICLE 等的片段库。[38]
- **OpenEye BROOD**：经典的 fragment replacement 商业软件。[39]

如果只能用免费/开源方案，上文的 SwissBioisostere + ChemBounce + RDKit + DiffLinker 组合基本可以覆盖这两者的主要用途。

---

## 6. 针对"把中心部分换成其他 scaffold"的实操路线

### 路线 A：最快得到有文献先例的替换建议

```text
原分子 → 标注中心 scaffold/linker → SwissBioisostere → 候选替换片段
→ RDKit 接回原 R-groups → 过滤和排序
```

推荐用于：项目早期，需要快速找几个可信的 scaffold hopping ideas，尤其希望避免纯生成模型发散。

步骤：

1. 在原分子中定义要替换的中心片段；
2. 用 R1/R2/R3 标注 attachment points；
3. 在 SwissBioisostere 查询 replacement；
4. 优先选择 attachment point 数量一致、几何关系相近、文献上下文接近的替换；
5. 用 RDKit 把新 scaffold 接回原来的 R-groups；
6. 去重、sanitize、性质过滤和 3D overlay。

---

### 路线 B：自动生成一批 scaffold-hopped 完整分子

```text
输入 SMILES → ChemBounce → 生成 scaffold-hopped SMILES
→ RDKit / SwissADME / 自有模型过滤 → docking / shape / pharmacophore 复筛
```

推荐用于：一次性生成几十到几千个候选分子，并通过后处理筛掉不合理结构。

建议过滤指标：

- **结构合法性**：RDKit sanitize、valence、aromaticity、charge、salt/mixture；
- **基础性质**：MW、cLogP、tPSA、HBD/HBA、rotatable bonds、Fsp3、QED、SA score；
- **相似性**：ECFP/Tanimoto 不宜过高也不宜过低；必要时加 pharmacophore fingerprint；
- **3D**：conformer generation、shape overlay、pharmacophore alignment；
- **结构警报**：PAINS、reactive/toxicophores、unstable groups；
- **项目相关**：docking score、interaction fingerprint、已知 SAR rules、selectivity filters。

---

### 路线 C：自建可控 pipeline，适合长期使用

```text
原分子 + core SMARTS
  → RDKit ReplaceCore() 得到带 dummy labels 的 R-groups
  → 候选 scaffold 库（要求 attachment labels 匹配）
  → reaction SMARTS / molzip / 自定义连接
  → enumerate → canonicalize + deduplicate
  → property / 2D / 3D / docking filters
```

推荐用于：团队内希望把 scaffold hopping 做成可复用工作流，并且有内部 compound library、SAR 数据或 scoring model。

候选 scaffold 库可以来自：

- SwissBioisostere replacement；
- ChEMBL / BindingDB / PubChem 中同靶点活性分子拆解；
- mmpdb 从内部 SAR 中挖出的 transformation rules；
- ScaffoldGraph 从 active set 里总结出的 scaffold network；
- 自己定义的 bioisostere scaffold list；
- ChemBounce 或其他生成式模型输出。

---

### 路线 D：有 3D 结合构象时，优先考虑 linker/scaffold 生成模型

```text
共晶 ligand 或 docking pose
  → 删除中心 scaffold/linker
  → 固定两端或多个关键 fragments 的 3D 坐标
  → DeLinker / DiffLinker / Link-INVENT / DiffHopp
  → 生成新 linker/core
  → 3D overlay + docking + interaction fingerprint
  → MD 或 MM-GBSA → 合成可行性检查
```

推荐用于：你知道原分子两端分别抓住 binding pocket 的关键相互作用，中间 core 主要负责几何连接、构象限制或 physicochemical tuning。这类任务中 2D replacement 往往不够，关键是保持 exit vector geometry 和 3D pharmacophore arrangement。

---

## 7. 工具选择建议

| 你的情况 | 首选方案 |
|---|---|
| 想查"这个中心能换成什么"且要文献先例 | SwissBioisostere |
| 只有一个 hit，想快速获得替代 scaffold | ChemBounce |
| 明确知道中心 core 和外围片段 | RDKit 自定义 core replacement |
| 中心部分是连接两个端基的 linker | DiffLinker / DeLinker / REINVENT4 Link-INVENT / RDKit linker enumeration |
| 想探索更大新颖化学空间 | REINVENT4 |
| 有蛋白-配体复合物 | DiffHopp / pocket-conditioned 生成模型 |
| 想找 2D 不像但 3D/药效团相似的 hop | SwissSimilarity / Shape-it / USRCAT / ZINCPharmer |
| 想要最可解释的药化设计流程 | RDKit + scaffold library |
| 更偏 GUI、不想完全写代码 | DataWarrior / KNIME RDKit nodes |
| 想要最快原型验证 | ChemBounce + RDKit 后处理 |

---

## 8. 优先级建议（推荐组合）

**如果你现在只有一个分子，想快速得到候选**：

```text
SwissBioisostere → ChemBounce → SwissSimilarity / shape-based screening
→ RDKit 精修枚举 → docking / pharmacophore / ADMET 复筛
```

**如果你们团队会写 Python，想做成可复用流程**：

```text
RDKit + mmpdb + ScaffoldGraph + 自定义 scaffold 库 + 3D shape/pharmacophore filtering
```

**如果你们更偏 GUI / 不想完全写代码**：

```text
SwissBioisostere + SwissSimilarity + DataWarrior / KNIME RDKit nodes
```

**如果你们有蛋白结构或可靠 ligand pose**：

```text
DeLinker / DiffLinker / REINVENT4-Link-INVENT / DiffHopp + docking + interaction fingerprint
```

---

## 9. 最小可行流程（MVP）

对于当前需求，最小可行流程可以是：

```text
1. 准备输入分子的 SMILES
2. 明确中心 scaffold 的原子范围
3. 明确需要保留的外围片段
4. 使用 ChemBounce 生成第一批 scaffold-hopped molecules
   （或先用 SwissBioisostere 找有先例的替换片段）
5. 用 RDKit 检查：
   - 是否保留关键外围结构
   - 是否成功替换中心 scaffold
   - valence 是否正确 / 是否去重
   - MW / cLogP / TPSA / HBD / HBA 是否合理
6. 如果有蛋白结构，进一步 docking
7. 如果无蛋白结构，做 3D shape / pharmacophore similarity
8. 人工挑选 20–100 个候选进入下一轮评估
```

如果你已经非常清楚中心部分和连接点，则推荐可控枚举流程：

```text
1. 用 SMARTS 定义 old_core
2. 用 RDKit ReplaceCore() 拆分 R groups
3. 准备 [*:1]-new_scaffold-[*:2] 形式的 scaffold library
4. 用 molzip / reaction SMARTS / 自定义拼接生成新分子
5. 过滤和验证
```

---

## 10. 小结

对于"把分子中心部分换成别的 scaffold"这个问题，优先建议：

1. **先用 SwissBioisostere** 找有历史先例的 scaffold/linker replacement；
2. **再用 ChemBounce** 快速生成完整 scaffold-hopped 分子；
3. **用 RDKit** 做可控的拆解、连接、去重和性质过滤；
4. **用 SwissSimilarity / Shape-it / USRCAT / pharmacophore search** 找 2D 不像但 3D/药效团相似的 scaffold hops；
5. **如果有 3D pose**，再上 DeLinker、DiffLinker、REINVENT4/Link-INVENT 或 DiffHopp 这类生成模型。

最稳的实际组合是：

```text
SwissBioisostere + RDKit + ChemBounce + 3D shape/pharmacophore 复筛
```

---

## 11. 参考文献

> 全文引用统一为连续编号 `[1]–[39]`，重复的文献（如 ChemBounce 论文、REINVENT4 论文/仓库等）已合并。

[1] ChemBounce: a computational framework for scaffold hopping in drug discovery. URL: https://academic.oup.com/bioinformatics/article/41/9/btaf501/8251522
[2] ChemBounce GitHub repository. URL: https://github.com/jyryu3161/chembounce
[3] ChemBounce README: scaffold fingerprint database notes. URL: https://github.com/jyryu3161/chembounce/blob/main/README.md
[4] ChemBounce Colab notebook. URL: https://colab.research.google.com/github/jyryu3161/chembounce/blob/main/chembounce_colab.ipynb
[5] Expasy / SIB: SwissBioisostere resource page. URL: https://www.expasy.org/resources/swissbioisostere
[6] SwissBioisostere official website. URL: https://www.swissbioisostere.ch/
[7] SwissBioisostere 2021: updated structural, bioactivity and physicochemical data. URL: https://academic.oup.com/nar/article/50/D1/D1382/6426059
[8] SwissBioisostere Tutorial 4: replacement of linkers and scaffolds; side chain/linker/scaffold R-group definitions. URL: https://www.swissbioisostere.ch/tutorials.html
[9] RDKit overview documentation. URL: https://www.rdkit.org/docs/Overview.html
[10] RDKit `rdkit.Chem.rdmolops` API documentation: `ReplaceCore`, `ReplaceSidechains`, `ReplaceSubstructs`. URL: https://www.rdkit.org/docs/source/rdkit.Chem.rdmolops.html
[11] RDKit R-group decomposition documentation. URL: https://www.rdkit.org/docs/source/rdkit.Chem.rdRGroupDecomposition.html
[12] RDKit blog: R-group decomposition and molzip. URL: https://greglandrum.github.io/rdkit-blog/posts/2022-03-14-rgd-and-molzip.html
[13] RDKit blog: R-group decomposition tutorial. URL: https://greglandrum.github.io/rdkit-blog/posts/2023-01-09-rgd-tutorial.html
[14] RDKit shape alignment API documentation. URL: https://www.rdkit.org/docs/source/rdkit.Chem.rdShapeAlign.html
[15] Reinvent 4: Modern AI-driven generative molecule design. URL: https://link.springer.com/article/10.1186/s13321-024-00812-5
[16] REINVENT4 GitHub repository. URL: https://github.com/MolecularAI/REINVENT4
[17] DeLinker scaffold hopping notebook. URL: https://github.com/fimrie/DeLinker/blob/master/examples/DeLinker_scaffold_hopping.ipynb
[18] DeLinker on bio.tools. URL: https://bio.tools/delinker
[19] DiffLinker GitHub repository. URL: https://github.com/igashov/DiffLinker
[20] DiffLinker paper in Nature Machine Intelligence. URL: https://www.nature.com/articles/s42256-024-00815-9
[21] SyntaLinker paper. URL: https://pubs.rsc.org/sc/article/11/31/8312/705774/SyntaLinker-automatic-fragment-linking-with-deep
[22] DiffHopp: A Graph Diffusion Model for Novel Drug Design via Scaffold Hopping. URL: https://arxiv.org/abs/2308.07416
[23] DiffHopp GitHub repository. URL: https://github.com/jostorge/diffusion-hopping
[24] Deep scaffold hopping with multimodal transformer neural networks (DeepHop). Zheng S, et al. J. Cheminform. 2021;13(1):87. URL: https://jcheminf.biomedcentral.com/articles/10.1186/s13321-021-00565-5
[25] mmpdb on PyPI. URL: https://pypi.org/project/mmpdb/
[26] mmpdb GitHub repository. URL: https://github.com/rdkit/mmpdb
[27] ScaffoldGraph paper in Bioinformatics. URL: https://academic.oup.com/bioinformatics/article/36/12/3930/5814205
[28] ScaffoldGraph GitHub repository. URL: https://github.com/UCLCheminformatics/ScaffoldGraph
[29] SwissSimilarity on Expasy. URL: https://www.expasy.org/resources/swisssimilarity
[30] The SwissSimilarity 2021 Web Tool. URL: https://www.mdpi.com/1422-0067/23/2/811
[31] DataWarrior official site. URL: https://openmolecules.org/datawarrior/
[32] RDKit Nodes for KNIME. URL: https://www.knime.com/rdkit
[33] ZINCPharmer paper in Nucleic Acids Research. URL: https://academic.oup.com/nar/article/40/W1/W409/1072552
[34] ZINCPharmer on bio.tools. URL: https://bio.tools/zincpharmer
[35] Shape-it GitHub repository. URL: https://github.com/rdkit/shape-it
[36] USRCAT paper. URL: https://pmc.ncbi.nlm.nih.gov/articles/PMC3505738/
[37] OpenPharmacophore GitHub repository. URL: https://github.com/uibcdf/OpenPharmacophore
[38] Cresset Spark (field-based scaffold hopping / bioisosteric replacement，商业软件). URL: https://www.cresset-group.com/software/spark/
[39] OpenEye BROOD (fragment replacement，商业软件). URL: https://www.eyesopen.com/brood

---

**完**
