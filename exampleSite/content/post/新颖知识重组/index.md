+++
author = "Joseph"
title = "知识重组新颖度计算"
date = "2026-08-07"
description = "Novel Knowledge Recombination or Recombinatory Novelty"
slug= "knowledge-recombinatory-novelty"
image= "cover.png"
categories = [
    "Patent indicator"
]
tags = [
    "Patent"
]
+++
## 一、指标背景与定义

### 1.1 理论基础

如文末参考文献所示，发表于AMJ和SMJ的两篇论文提出并应用了"新颖知识组合"或“知识重组新颖性”（Novel Knowledge Recombination or Recombinatory Novelty）的指标，而本文回顾了他们的测量思路并设计了Python脚本。

创新被认为是现有知识元素的重新组合（Schumpeter, 1934; Fleming, 2001），而真正具有突破性的创新往往涉及**远距离知识组合**——即那些在技术领域中罕见或前所未有的知识连接。

### 1.2 LINK 指标定义

**LINK(t, i, j)** 表示技术领域 i 的专利在年份 t 引用技术领域 j 的链接强度：

$$
LINK(t, i, j) = \frac{\text{过去5年内领域 } i \text{ 引用领域 } j \text{ 的总次数}}{\text{过去5年内领域 } i \text{ 的总引用次数}}
$$

**参数说明**：
- $t$: 引用发生的年份（citing patent 的申请年份）
- $i$: 引用方专利的技术领域（IPC 前 4 位）
- $j$: 被引方专利的技术领域（IPC 前 4 位）
- 回溯窗口：5 年（可配置）

**解读**：
- $LINK$ 值接近 0：说明领域 i 在过去很少引用领域 j，这是一个**罕见/远距离**的知识组合
- $LINK$ 值接近 1：说明领域 i 经常引用领域 j，这是一个**常见/本地**的知识组合

### 1.3 Novelty 指标定义

$Novelty$ 用于衡量单个专利的新颖程度：

$$
Novelty = 1 - \min(LINK \text{of all citations})
$$

即用 1 减去该专利所有后向引用中最低的 $LINK$ 值。

**解读**：
- $Novelty$ 值接近 1：专利至少包含一条非常远距离的知识组合（引用了一个罕见的领域连接）
- $Novelty$ 值接近 0：专利的所有引用都是常见的领域连接

### 1.4 RADICAL 指标定义

$RADICAL$ 是二元变量，使用从大到小依次排列所处的分位数来识别"激进创新"专利：

$$
RADICAL_{p} = \begin{cases}
1 & \text{if } Novelty > \text{Percentile}_{p}(Novelty \text{in same year and IPC}) \\
0 & \text{otherwise}
\end{cases}
$$

其中 *p* 通常取 90、95 或 99，表示 novelty 值是否处于同年份同技术领域的前 （1-*p*）%，或者说该专利的novelty值已经超过了 *p* %的其他专利

### 1.5 Normalized Novelty 指标定义

$Normalized\ Novelty$ 用于消除同年份同技术领域的系统性差异，使得不同领域和年份的 novelty 值可以比较：

$$
Normalized\ Novelty = Novelty - \bar{\mu}_{\text{year-1}, \text{IPC}}
$$

其中 $\bar{\mu}_{\text{year-1}, \text{IPC}}$ 表示**前一年**同 IPC 技术领域所有专利的平均 novelty 值。

**计算步骤**：
1. 按 `(year, ipc_4d)` 分组，计算各组的平均 novelty 值
2. 将各组的平均 novelty 值向前推移一年（即 t 年的值来自 t-1 年）
3. 用每个专利的 novelty 减去其对应年份和 IPC 的前一年平均值

**解读**：
- Normalized Novelty > 0：该专利的新颖性高于同期同领域前一年的平均水平
- Normalized Novelty < 0：该专利的新颖性低于同期同领域前一年的平均水平
- 该指标消除了技术和时间趋势的影响，便于跨领域跨时期比较
### 1.5 与其他相似指标的对比
> [Eggers & Kaul, 2018](#eggers2018) Our measure has similarities to several prior measures in the literature. First, our measure is similar to Fleming’s (2001) measure of “component familiarity,” except that Fleming’s measure captures combinations that are new to the inventor, while our measure identifies combinations that are new to the field. Second, our measure is similar to Trajtenberg, Henderson, and Jaffe’s (1997) measure of “originality,” except that their measure focuses on the diversity of a patent’s citations, while ours focuses on their novelty. Third, our measure is similar in spirit to one developed by Dahlin and Behrens (2005), except that they measure novelty by comparing the pattern of citations at the patent level, while our measure of novelty is based on technology class level comparisons, making our measure both more sensitive to rare connections made by patents with otherwise conventional citation patterns, and less prone to bias from examiner-added citations (Alcacer  & Gittelman, 2006). Finally, Aharonson and Schilling (2016) develop a measure of “outlier patents” based on the co-occurrence of multiple classifications in the same patent, which is quite similar to ours; the main difference being that our measure focuses on citation links between the class of the citing patent and the class of the cited patent. 

## 二、数据结构

### 2.1 输入数据

代码从 `patent_network.pkl` 文件加载数据，该文件包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `patents` | dict | 专利信息字典，key 为标准化专利号 |
| `citation_info` | list | 引用信息列表 [(source, target, type, date)] |
| `ipc_data` | dict | 专利 IPC 分类信息 |

### 2.2 IPC 分类

技术领域按 **IPC 前 4 位**确定，例如：
- `G06F` - 计算；推算；计数
- `H01L` - 半导体器件
- `C07D` - 杂环化合物


## 三、计算流程

### 3.1 总体流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     1. 数据加载阶段                          │
├─────────────────────────────────────────────────────────────┤
│  patent_network.pkl                                         │
│       │                                                     │
│       ├── patents (专利基本信息)                            │
│       ├── ipc_data (IPC分类)                                │
│       └── citation_info (引用关系)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  2. 数据预处理阶段                            │
├─────────────────────────────────────────────────────────────┤
│  2.1 提取有效专利 (1980-2025年，有IPC分类)                   │
│  2.2 构建专利信息表: {norm_pub: {year, ipc_4d, ...}}       │
│  2.3 筛选有效引用 (两端专利都在有效集中)                     │
│  2.4 构建引用网络: citing → [{cited, cited_year, cited_ipc}]│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  3. LINK 矩阵计算                            │
├─────────────────────────────────────────────────────────────┤
│  3.1 遍历所有引用记录，统计:                                 │
│      - (year, citing_ipc, cited_ipc) → count               │
│      - (year, citing_ipc) → total                          │
│                                                              │
│  3.2 对每对 (t, i, j) 计算:                                 │
│      LINK(t,i,j) = count(t-5至t-1, i→j) / total(t-5至t-1, i)│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 4. Novelty 计算                              │
├─────────────────────────────────────────────────────────────┤
│  对每个专利 p:                                              │
│    1. 获取专利年份 t 和领域 i                               │
│    2. 获取所有被引专利的领域 j                               │
│    3. 查表获取每条引用的 LINK(t, i, j)                      │
│    4. Novelty = 1 - min(LINK values)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 5. Normalized Novelty 计算                     │
├─────────────────────────────────────────────────────────────┤
│  按 (year, ipc_4d) 分组，计算平均 novelty                     │
│  将平均值向前推移一年                                          │
│  Normalized Novelty = Novelty - prior_year_mean               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 6. RADICAL 计算                              │
├─────────────────────────────────────────────────────────────┤
│  按 (year, ipc_4d) 分组，计算 Novelty 分位数                 │
│  - RADICAL_90: Novelty > 90% 分位                           │
│  - RADICAL_95: Novelty > 95% 分位                           │
│  - RADICAL_99: Novelty > 99% 分位                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     7. 结果输出                              │
├─────────────────────────────────────────────────────────────┤
│  novelty_indicators.parquet                                  │
│    ├── norm_pub, year, ipc_4d                              │
│    ├── n_citations, min_link, novelty                       │
│    ├── mean_link, max_link                                  │
│    ├── normalized_novelty, prior_year_mean_novelty         │
│    └── radical_90, radical_95, radical_99                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 详细步骤说明

#### 步骤 1: 数据加载

```python
patents, citation_info, ipc_data = load_data_from_pkl(pkl_path)
```

从 pickle 文件加载所有数据。

#### 步骤 2: 构建专利信息表

```python
patent_info = {
    'CN1010408': {'year': 1986, 'ipc_4d': 'C05F', ...},
    'CN102889914': {'year': 2012, 'ipc_4d': 'G01F', ...},
    ...
}
```

只保留 1980-2025 年且有有效 IPC 分类的专利。

#### 步骤 3: 处理引用网络

```python
citation_network = process_citations_in_chunks(citation_info, patent_info)
```

构建引用网络，只保留两端专利都在有效集中的引用：

```python
citation_network = {
    'CN1010408': [
        {'cited_patent': 'CN1001234', 'cited_year': 1980, 'cited_ipc_4d': 'C05B'},
        {'cited_patent': 'CN1005678', 'cited_year': 1982, 'cited_ipc_4d': 'A01K'},
        ...
    ],
    ...
}
```

#### 步骤 4: 计算 LINK 矩阵

```python
link_dict = calculate_link_matrix(citation_network, patent_info, lookback=5)
```

LINK 矩阵结构：

```python
link_dict = {
    (1986, 'G06F', 'H01L'): 0.023,   # G06F 在1986年引用H01L的LINK值
    (1986, 'G06F', 'G06F'): 0.845,   # G06F 引用自身的LINK值（通常较高）
    ...
}
```

**计算公式**：

$$
LINK(t, i, j) = \frac{\sum_{y=t-5}^{t-1} \text{count}(y, i, j)}{\sum_{y=t-5}^{t-1} \text{total}(y, i)}
$$

#### 步骤 5: 计算 Novelty

```python
df_novelty = calculate_patent_novelty(citation_network, patent_info, link_dict)
```

对每个专利，找出其所有引用中最小的 LINK 值：

$$
Novelty_p = 1 - \min_{c \in \text{citations}(p)} LINK(t_p, i_p, j_c)
$$

#### 步骤 6: 计算 RADICAL

```python
df_novelty = calculate_radical_indicators(df_novelty)
```


## 四、代码结构

### 4.1 核心函数

| 函数名 | 功能 |
|--------|------|
| `load_data_from_pkl()` | 从 pickle 文件加载数据 |
| `extract_ipc_4d()` | 提取 IPC 前 4 位 |
| `build_patent_year_ipc()` | 构建专利年份和 IPC 信息表 |
| `process_citations_in_chunks()` | 处理引用信息，构建引用网络 |
| `calculate_link_matrix()` | 计算 LINK 矩阵 |
| `calculate_patent_novelty()` | 计算每个专利的 Novelty |
| `calculate_radical_indicators()` | 计算 RADICAL 二元指标 |

### 4.2 关键算法

#### LINK 计算伪代码

```python
def calculate_link_matrix(citation_network, patent_info, lookback=5):
    # 1. 收集所有引用记录
    records = []
    for citing_patent, citations in citation_network.items():
        citing_year = patent_info[citing_patent]['year']
        citing_ipc = patent_info[citing_patent]['ipc_4d']
        for cited in citations:
            records.append((citing_year, citing_ipc, cited['cited_ipc_4d']))
    
    # 2. 按 (year, citing_ipc, cited_ipc) 统计
    # 3. 按 (year, citing_ipc) 统计总数
    # 4. 计算 LINK 值
    for (year, ipc_i, ipc_j) in pairs:
        total_i_to_j = sum(count[y, ipc_i, ipc_j] for y in range(year-lookback, year))
        total_i = sum(total[y, ipc_i] for y in range(year-lookback, year))
        link[(year, ipc_i, ipc_j)] = total_i_to_j / total_i if total_i > 0 else 0
    
    return link
```

#### Novelty 计算伪代码

```python
def calculate_patent_novelty(citation_network, patent_info, link_dict):
    results = []
    for norm_pub in patent_info:
        year = patent_info[norm_pub]['year']
        ipc = patent_info[norm_pub]['ipc_4d']
        citations = citation_network.get(norm_pub, [])
        
        if len(citations) < MIN_CITATIONS:
            novelty = NaN
        else:
            # 获取每条引用的 LINK 值
            links = []
            for cited in citations:
                link_val = link_dict.get((year, ipc, cited['cited_ipc_4d']), 0)
                links.append(link_val)
            
            # Novelty = 1 - 最小 LINK
            novelty = 1 - min(links)
        
        results.append({
            'norm_pub': norm_pub,
            'novelty': novelty,
            ...
        })
    
    return DataFrame(results)
```

## 五、运行指南

### 5.1 环境要求

```bash
# Python 3.8+
pip install pandas numpy tqdm
```

### 5.2 基本用法

```bash
# 使用默认参数
python calculate_link_novelty.py

# 自定义参数
python calculate_link_novelty.py \
    --pkl "path/to/patent_network.pkl" \
    --output "path/to/output.parquet" \
    --lookback 5 \
    --min-citations 1
```

### 5.3 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--pkl` | patent_network.pkl 路径 | 输入的 pickle 文件 |
| `--output` | novelty_indicators.parquet | 输出文件路径 |
| `--lookback` | 5 | 回溯窗口年数（论文推荐5年） |
| `--min-citations` | 1 | 计算 novelty 的最少引用数 |

### 5.4 输出文件格式

输出为 Parquet 格式，包含以下字段：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `norm_pub` | string | 专利号（标准化） |
| `year` | int | 申请年份 |
| `ipc_4d` | string | IPC 前 4 位 |
| `n_citations` | int | 后向引用数量 |
| `min_link` | float | 最小 LINK 值（0-1），无引用时为 NaN |
| `novelty` | float | 新颖性指标（0-1），无引用时为 NaN |
| `mean_link` | float | 平均 LINK 值，无引用时为 NaN |
| `max_link` | float | 最大 LINK 值，无引用时为 NaN |
| `normalized_novelty` | float | 标准化新颖性（相对于同期同领域平均值） |
| `prior_year_mean_novelty` | float | 前一年份同 IPC 领域的平均 novelty 值 |
| `radical_90` | int | 是否为同年份同领域前 10% 高新颖专利 |
| `radical_95` | int | 是否为同年份同领域前 5% 高新颖专利 |
| `radical_99` | int | 是否为同年份同领域前 1% 高新颖专利 |


## 六、示例计算

### 6.1 示例数据

假设有以下引用关系：

| 专利 | 年份 | IPC领域 | 引用 |
|------|------|---------|------|
| P1 | 1986 | G06F | 引用 G06F(×10), H01L(×2) |
| P2 | 1986 | G06F | 引用 G06F(×8), A01K(×1), C07D(×1) |
| P3 | 1986 | A01K | 引用 A01K(×5), G06F(×3) |

### 6.2 LINK 计算

对于 1986 年 G06F 引用 H01L：

- 过去5年（1981-1985）G06F → H01L 引用次数：假设为 50 次
- 过去5年 G06F 总引用次数：假设为 2000 次
- **LINK(1986, G06F, H01L) = 50 / 2000 = 0.025**

对于 G06F 引用 G06F（自引用）：

- 过去5年 G06F → G06F 引用次数：假设为 1500 次
- **LINK(1986, G06F, G06F) = 1500 / 2000 = 0.75**

### 6.3 Novelty 计算

专利 P1 的所有引用 LINK 值：
- G06F → G06F: 0.75 (10次)
- G06F → H01L: 0.025 (2次)

- **min_link = 0.025**
- **Novelty = 1 - 0.025 = 0.975**

这表明 P1 至少有一条非常远距离的知识组合（引用了一个罕见的技术领域知识）。


## 七、注意事项

### 7.1 内存优化

由于专利数据量大（2000万+），代码采用以下优化策略：
- 使用分块处理引用信息
- 及时释放不需要的大对象（`gc.collect()`）
- 使用 `defaultdict` 优化字典操作

### 7.2 时间窗口选择

- 论文原版使用 **5 年**回溯窗口
- 可根据研究需要调整为 3 年、7 年等
- 较短的窗口对近期变化更敏感
- 较长的窗口更稳定但可能滞后于技术变革

### 7.3 引用数量要求

- 默认对所有专利计算，即使只有 1 条引用
- 对于研究高度探索性创新，可以设置 `MIN_CITATIONS=3` 或更高
- **没有后向引用的专利**：其 `novelty` 值为 `NaN`（因为无法计算 LINK 值）

### 7.4 缺失值处理

- `novelty = NaN`：专利没有后向引用（无法计算 LINK）
- `min_link = NaN`：同上
- `mean_link = NaN`：同上
- `n_citations = 0`：专利没有后向引用

### 7.5 IPC 分类层级

- IPC 前 4 位（如 G06F）是常用选择
- 也可以使用前 3 位（section+class，如 G06）更宽泛
- 或前 8 位（如 G06F11/00）更细粒度

---

## 八、参考文献

1. <span id="eggers2018"></span> Eggers, J. P., & Kaul, A. (2018). Motivation and Ability? A Behavioral Perspective on the Pursuit of Radical Invention in Multi-Technology Incumbents. *Academy of Management Journal*, 61(1), 67-91. https://doi.org/10.5465/amj.2016.0486

   **摘要（译）**：
   本文从行为理论的视角，研究在位企业如何平衡追求激进技术发明的动机与能力。我们区分了企业追求激进发明的动机（受绩效相对于期望水平的影响）和成功开发激进发明的能力（受现有知识基础的影响）。使用1980-1997年美国专利数据，我们发现动机和能力在追求激进发明方面呈现倒U型关系：当绩效略低于期望时，企业倾向于过度投资于激进发明；而当绩效远高于期望时，则投资不足。我们的研究揭示了动机与能力之间的关键失配区域，为理解在位企业的技术追求提供了新的理论视角。

   本文提出的 LINK 和 DISTANT 指标为测量专利的知识组合新颖性提供了基础框架。

2. Qu, Y., Fang, Y., & Park, Y. (2025). Unlocking novel knowledge recombination: How AI bridges technological domains. *Strategic Management Journal*. (Advance online publication). https://doi.org/10.1002/smj.70080

   **摘要（译）**：
   人工智能（AI）如何使以前不可行的知识组合成为可能？本文提出并检验了一种理论，解释了AI作为使能技术如何通过"桥接"效应促进新颖知识重组。研究表明，AI通过其预测能力和传输共享解决方案的能力，作为强大的共享层发挥作用，从而在过去不相关的技术元素之间创建桥梁。我们的分析表明，基于AI的发明比不基于AI的类似发明更可能涉及新颖的知识重组，且AI能够桥接和连接以前分散的知识领域。这些发现表明，AI不仅仅是"发明新方法"（作为研发工具），而且从根本上重塑了发明活动本身的性质。
   
   本文采用 Eggers & Kaul (2018) 的 LINK 指标测量方法，研究AI发明的知识组合新颖性特征。



