---
layout: default
title: "给 Boltz-2/AlphaFold3 的 mmCIF 模型补上配体键级和蛋白–配体成键"
date: 2026-07-15 12:00:00 +0800
categories: [结构生物学, 工具]
tags: [mmcif, boltz-2, alphafold3, chem_comp_bond, struct_conn, rdkit, gemmi]
description: "Boltz-2 等 cofolding 工具输出的 mmCIF 只有坐标，没有配体键级、也没有蛋白–配体成键。本文记录补上这两类信息的完整方法，重点讲清一个最容易踩的坑：判断原子顺序要用连接图而不是坐标。"
---

用 Boltz-2、AlphaFold3、Chai 这类 cofolding 工具跑完蛋白–配体复合物，拿到的 mmCIF 里往往**只有原子坐标**：配体内部哪个原子和哪个原子成键、是单键还是双键，没有；配体和蛋白之间那根共价键（如果有），也没有。很多下游流程（参数化、对接后处理、可视化标注、提交 PDB）都需要这些成键信息。本文记录怎么把它们补进 mmCIF，并重点讲清一个**最容易翻车的坑**：判断原子对应关系该用什么、不该用什么。文中的例子来自一个真实的 Boltz-2 输出——一个 ADC（抗体偶联药物）复合物，配体是 deruxtecan 的 payload。

---

## 0. 先分清两个类别：模板 vs 实例

mmCIF 里"键"分两层，放在**不同**的类别里，这是最容易搞混、也最关键的一点：

| 类别 | 描述对象 | 作用域 | 键什么 |
|------|---------|--------|--------|
| `_chem_comp_bond` | **化学组分的模板** | 按 `comp_id`，对该组分**每一份拷贝都生效** | 组分**内部**原子间的键 |
| `_struct_conn` | **模型里具体的原子实例** | 精确到"哪条链/哪个残基/哪个原子" | 跨组分、跨链的连接（含蛋白–配体） |

一句话记忆：**"这个分子长什么样"用 `_chem_comp_bond`（模板，一份定义管所有拷贝）；"这两个具体原子连在一起"用 `_struct_conn`（实例，精确定位）。**

所以蛋白–配体的共价键**不能**放进 `_chem_comp_bond`——模板按 `comp_id` 走，表达不了"E 链的配体连到 A 链某个 Cys"这种实例级、跨组分的关系。

---

## 1. 配体内部键级：`_chem_comp_bond`

### 1.1 目标格式

```
loop_
_chem_comp_bond.comp_id
_chem_comp_bond.atom_id_1
_chem_comp_bond.atom_id_2
_chem_comp_bond.value_order
_chem_comp_bond.pdbx_aromatic_flag
_chem_comp_bond.pdbx_ordinal
LIG C1  C2  sing N 1
LIG C2  C3  doub N 2
LIG C3  N1  sing Y 3
...
```

- `atom_id_1/2` 是**原子名**（不是序号），必须和 `_atom_site.label_atom_id` 里该配体用的名字逐字一致（大小写敏感）。
- `value_order`：`sing` / `doub` / `trip` / `arom` / `delo`。
- `pdbx_aromatic_flag`：`Y`/`N`，标该键是否在芳环上。

芳香键有两种写法，按下游工具口味二选一并保持一致：PDB CCD 惯例是 `value_order` 用 Kekulé 单/双键 + 芳环键 `pdbx_aromatic_flag=Y`（推荐，兼容性最好）；也可以直接 `value_order=arom`。

### 1.2 键级从哪来

CIF 里没有键级，得从外部参考拿：最省事的是**配对的 SDF/MOL**（V2000 键块第 3 列就是键级：1 单、2 双、3 三、4 芳香）；没有 SDF 就用 **SMILES + RDKit** 建模板做子结构匹配；如果配体是**真实 CCD 代码**，可从 `files.rcsb.org/ligands/download/XXX.cif` 下载自带 `chem_comp_bond`。注意 Boltz 常给配体分配 5 字符自定义代码（如 `ZOYQV`），这种不在 CCD 里，只能自备 SDF/SMILES。

### 1.3 核心坑：判断原子顺序要用连接图，不要用坐标

把 SDF 的键级搬到 CIF 原子上，前提是知道"SDF 第 k 个原子 ↔ CIF 哪个原子"。这一步最容易错：

- ❌ **元素数目一致 ≠ 顺序一致**。两边都是"52 个 C、13 个 O…"完全说明不了 index 对得上。
- ❌ **不能比坐标/距离矩阵**。cofolding 输出的是预测构象，SDF 可能是另一个构象；同一分子摆的姿势不同，坐标和距离矩阵都会对不上——但这**不代表**原子顺序不同。

真实例子：我把某 Boltz-2 输出里的配体和它的参考 SDF 比距离矩阵，最大差到 26 Å，看起来"完全不匹配"。但这只是构象不同，原子顺序其实**逐个一致**。

✅ **正确判据是成键连接图（graph），它与构象无关**——一根键无论分子怎么弯折都还是那根键。做法：

1. 从 SDF 读显式键 → 键集合（用 index 表示）。
2. 从 CIF 坐标用**共价半径**推断键 → 另一个键集合。
3. 在"index i ↔ index i"假设下比较两个键集合是否完全相等；相等即顺序一致。
4. 若不等，做**图同构**（按元素标记节点）得到真实映射再重排。

```python
import numpy as np, networkx as nx

COV = {'H':0.31,'C':0.76,'N':0.71,'O':0.66,'F':0.57,'S':1.05,'P':1.07,'Cl':1.02,'Br':1.20}

def perceive_bonds(el, xyz, tol=0.45):
    """用共价半径从坐标推断连接（无键级）。"""
    n = len(el); d = np.linalg.norm(xyz[:,None]-xyz[None], axis=-1); bonds=set()
    for i in range(n):
        for j in range(i+1, n):
            if 0.4 < d[i,j] < COV[el[i]] + COV[el[j]] + tol:
                bonds.add((i, j))
    return bonds

def order_matches(sdf_el, sdf_bonds, cif_el, cif_bonds):
    """先测 identity 映射，不行再图同构。返回 (是否顺序一致, 映射 sdf->cif)。"""
    sset = set((min(a,b), max(a,b)) for a,b,_ in sdf_bonds)
    if sset == cif_bonds:
        return True, {i:i for i in range(len(sdf_el))}
    Gs = nx.Graph(); [Gs.add_node(i,e=sdf_el[i]) for i in range(len(sdf_el))]; Gs.add_edges_from(sset)
    Gc = nx.Graph(); [Gc.add_node(i,e=cif_el[i]) for i in range(len(cif_el))]; Gc.add_edges_from(cif_bonds)
    nm = nx.algorithms.isomorphism.categorical_node_match('e', None)
    GM = nx.algorithms.isomorphism.GraphMatcher(Gs, Gc, node_match=nm)
    if GM.is_isomorphic():
        return all(k==v for k,v in GM.mapping.items()), GM.mapping
    return False, None
```

芳香标记可以用 RDKit 从 SDF 感知，`value_order` 仍保留 SDF 的 Kekulé，芳环键置 `pdbx_aromatic_flag=Y`：

```python
from rdkit import Chem
mol = Chem.MolFromMolFile('ligand.sdf', sanitize=True)
arom = {tuple(sorted((b.GetBeginAtomIdx(), b.GetEndAtomIdx()))): b.GetIsAromatic()
        for b in mol.GetBonds()}
```

一句话经验：**判断"是不是同一个分子/顺序对不对"用连接图；坐标只用来看构象。**

生成好的 block 按惯例插在 `_chem_comp` loop 之后（mmCIF 类别顺序其实无所谓，只影响可读性）。

---

## 2. 蛋白–配体成键：`_struct_conn`

### 2.1 目标格式

```
loop_
_struct_conn_type.id
_struct_conn_type.criteria
_struct_conn_type.reference
covale ? ?
#
loop_
_struct_conn.id
_struct_conn.conn_type_id
_struct_conn.ptnr1_label_asym_id
_struct_conn.ptnr1_label_comp_id
_struct_conn.ptnr1_label_seq_id
_struct_conn.ptnr1_label_atom_id
_struct_conn.ptnr1_auth_asym_id
_struct_conn.ptnr1_auth_seq_id
_struct_conn.ptnr2_label_asym_id
_struct_conn.ptnr2_label_comp_id
_struct_conn.ptnr2_label_seq_id
_struct_conn.ptnr2_label_atom_id
_struct_conn.ptnr2_auth_asym_id
_struct_conn.ptnr2_auth_seq_id
_struct_conn.pdbx_value_order
_struct_conn.pdbx_dist_value
covale1 covale E LIG . C43 E 1 A CYS 221 SG A 221 sing 1.81
```

- 每根键一行，用 **ptnr1 / ptnr2** 两组标识符精确定位两端原子：`label_asym_id`（链）+ `label_comp_id`（残基名）+ `label_seq_id`（序号）+ `label_atom_id`（原子名）。建议同时给 `auth_*` 一套，兼容性最好。
- **非聚合物配体**的 `label_seq_id` 通常是 `.`，用 `auth_seq_id`（一般是 1）定位；蛋白残基有正常的 `label_seq_id`。
- 需要配一个 `_struct_conn_type` 声明用到的每种类型。

`conn_type_id` 常见取值：`covale`（共价键，如共价抑制剂、ADC linker 接 Cys、糖基化）、`metalc`（金属配位）、`disulf`（二硫键）、`hydrog`（氢键，一般不入模型）。

### 2.2 重要原则：只写真实存在的键

**绝大多数配体是非共价结合的**（靠氢键、疏水、静电堆在口袋里），这种情况**不写任何 `_struct_conn`**，配体就是一堆坐标待着。只有确实是共价配体或金属配位才加。不要凭空"想连就连"，`_struct_conn` 必须反映真实化学。

### 2.3 用几何距离定位配对伙伴

你通常知道**化学上**哪根键（比如"配体 C43 连 Cys 的 SG"），但不知道**具体哪条链哪个 Cys**。用距离来定：取出配体反应原子和所有候选伙伴原子，对每个配体原子找最近的候选，落在合理键长内（共价 C–S 约 1.8 Å，判据可用 < 2.1 Å；金属配位放宽到 ~2.5–3.0 Å）即配对，并确认"最近"显著近于"第二近"以排除歧义。

```python
import numpy as np
def load_atoms(path, sel):        # sel(c)->bool，c 为 atom_site 拆分字段
    out = []
    for ln in open(path):
        if ln.startswith(('ATOM','HETATM')):
            c = ln.split()
            if sel(c):
                out.append(dict(atom=c[3], comp=c[5], lseq=c[6], aseq=c[7],
                                lasym=c[9], aasym=c[15],
                                xyz=np.array([float(c[10]), float(c[11]), float(c[12])])))
    return out

lig = load_atoms('model.cif', lambda c: c[5]=='LIG' and c[3]=='C43')
sgs = load_atoms('model.cif', lambda c: c[5]=='CYS' and c[3]=='SG')
for L in lig:
    ds = sorted(sgs, key=lambda s: np.linalg.norm(s['xyz']-L['xyz']))
    d0 = np.linalg.norm(ds[0]['xyz']-L['xyz'])
    d1 = np.linalg.norm(ds[1]['xyz']-L['xyz'])
    print(f"C43[{L['lasym']}] -> {ds[0]['lasym']}/CYS{ds[0]['aseq']}/SG "
          f"= {d0:.2f} Å (次近 {d1:.2f} Å)")   # d0<2.1 且 d0<<d1 → 可靠配对
```

真实例子里配体有 6 份拷贝（E–J 链），每个 C43 都找到唯一的 Cys SG 搭档，距离 0.96–1.80 Å，且都远近于第二近的 SG（如 1.52 对 5.93 Å），配对毫无歧义——于是生成 6 条 `covale` 记录。

> 一个能反映模型质量的细节：这里有的键长只有 0.96、1.12 Å，远短于正常 C–S 键（~1.8 Å），说明预测模型在偶联位点的几何并不理想。**连接关系是对的，几何需要后续 minimization**。`pdbx_dist_value` 存实测值时对此要心里有数，也可以留空 `?`。

多拷贝时要注意：`_chem_comp_bond` 只写**一份**（模板对所有拷贝生效），但 `_struct_conn` 要为**每一份**共价拷贝各写一条（实例级）。block 按惯例插在 `_atom_site` 之前。

---

## 3. 校验

改完别肉眼看，用 **gemmi**（`pip install gemmi`）过一遍：

```python
import gemmi
doc = gemmi.cif.read('model_edited.cif')     # 能读通 = 语法合法
b = doc.sole_block()

# atom_site 里真实存在的原子集合
at = b.find('_atom_site.', ['label_asym_id','label_comp_id','label_atom_id','auth_seq_id'])
present = set((r[0], r[1], r[2], r[3]) for r in at)

# 逐条核对 struct_conn 两端原子是否都能在 present 里定位到；
# 逐条核对 chem_comp_bond 的 atom_id 是否都在该配体原子名里。
```

校验清单：文件仍能被 gemmi/PDB 工具解析；`_chem_comp_bond` 每个 `atom_id` 都存在于该配体原子；键数与单/双/芳香分布符合预期、无孤立原子；`_struct_conn` 每条记录两端原子都能精确定位；`_struct_conn_type` 已声明所有用到的类型。

---

## 4. cofolding 输出特有的坑

- **配体代码是自定义 5 字符**（如 `ZOYQV`），不在 CCD 里，别指望下载键定义。
- **`_chem_comp` 的 formula 常为空**（`.`），元素信息只能从 `_atom_site.type_symbol` 拿。
- **同一配体多份拷贝**：`_chem_comp_bond` 写一份模板，`_struct_conn` 每份共价拷贝各一条。
- **预测几何可能不理想**：偶联位点键长可能明显偏离理想值，连接对但需 minimization。
- **原子命名逐字一致**：`_chem_comp_bond`/`_struct_conn` 里的原子名必须和 `_atom_site` 完全一致（大小写敏感）。
- **label 与 auth 标识符**：cofolding 输出里两套 ID 可能相同也可能不同，写 `_struct_conn` 时两套都填最稳妥。

---

## 小结

给 cofolding 的 mmCIF 补成键信息，分两条线：

1. **配体内部键级** → `_chem_comp_bond`（模板，一份管所有拷贝）。键级从 SDF/SMILES/CCD 拿；**判断原子顺序用连接图，不要用坐标**；芳香键用 Kekulé + flag。
2. **蛋白–配体键** → `_struct_conn`（实例，ptnr1/ptnr2 精确定位）。只写真实存在的键；用几何距离定位配对伙伴；多拷贝各写一条。

最后用 gemmi 校验语法与所有引用原子能否定位。核心那句话值得再念一遍：**同一个分子摆的姿势不同，坐标会天差地别，但连接图不会变——所以对应关系交给图，不要交给坐标。**

---

**完**
