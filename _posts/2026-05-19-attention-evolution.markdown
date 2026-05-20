---
layout: post
title:  "注意力机制进化"
date:   2026-05-19 21:52:31 +0800
categories: attention
math: true
---
# 注意力机制的进化：**从 MHA 到 CSA/HCA：注意力机制如何在“记住”与“遗忘”间寻找最优解**

> 一个 DeepSeek-V3 级别的模型（$h=128$，61 层），如果用标准 MHA，在 4096 token 的上下文窗口中做推理，KV Cache 会吃掉约 7GB 显存。如果你的序列长度拉到 128K，这个数字会膨胀到约 224GB。这样的显存消耗在实际生产环境，显然是不可接受的。
>
> 这就是为什么过去 5 年，注意力机制的几乎所有创新都围绕一个核心问题：**如何让 KV Cache 变小、变快，同时尽量不损失模型质量。**
>
> 这篇文章沿着 MHA → MQA → GQA → MLA → DSA → CSA/HCA 这条主线，梳理注意力机制进化逻辑——每一步解决了什么问题，付出了什么代价，以及背后的「为什么」。


---

## 1. Multi-Head Attention：一切的起点

2017 年，Transformer 带着 Multi-Head Attention（MHA）横空出世。它的核心思想很优雅：**不要让模型只从一个角度「看」序列，而是用多个头并行地从不同子空间捕捉关系**。

原始 embedding 向量 $h \in \mathbb{R}^{d_{model}}$ 与 $W_Q$、$W_K$、$W_V$ 参数矩阵相乘，得到 Q、K、V。假设头数 $h = 8$，每个头有自己独立的 $W_Q^{(i)}$、$W_K^{(i)}$、$W_V^{(i)}$，共 24 组参数矩阵。

![Multi-head-Attention](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/mha_attention.excalidraw.png)

注意力计算公式：

$$
\text{Attention}(Q, K, V) = \text{Softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right) V
$$

其中 $d_k = d_{model} / h$，是每个注意力头的维度。除以 $\sqrt{d_k}$ 是为了防止点积过大导致 Softmax 梯度消失——这是原始论文里一个精巧的细节。

### 推理时的「隐形成本」：KV Cache

在自回归（Auto-Regressive）生成中，每个新 token 都需要与**之前所有 token** 计算注意力。

为了避免重复计算，推理时会把每个 token 的 K 和 V 缓存下来——这就是 KV Cache。
![qk_matrix.png](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/qk_matrix.png)
**算一笔账**：以 DeepSeek-V3 的参数规模为例，$d_{model}$=7168，h=128(头数)，共 61 层。在标准多头注意力（MHA）下，每个头的维度 $d_{k}=d_{model}/h=56$。

每个 token **每一层**的 KV Cache 大小为：
2×h×$d_{k}$=2×128×56=14336 个 float16=**28KB**

>float16占用2个字节

**乘以层数**：61 层总计，每个 token 的 KV Cache = 28KB×61≈**1.71MB**

序列长度 L=4096 时，总 KV Cache = 4096×1.71MB≈**7.0GB**。  

当 L=128K时，这个数字变成约 **224GB**——而且这还只是一个 batch 里的一条序列。

KV Cache 就是注意力机制进化的「第一性原理」——**所有后续优化，本质上都是在跟这 224GB 较劲。**

> ⚠️ **训练 vs 推理**：KV Cache 只在推理时存在。训练时，所有 token 的 Q、K、V 可以一次性并行计算（teacher forcing），不需要缓存。所以本文讨论的优化主要针对推理阶段。

```python
# MHA 的核心逻辑（PyTorch 伪代码）
# B: batch, L: seq_len, h: num_heads, d_k: head_dim
Q = x @ W_Q  # (B, L, h*d_k)
K = x @ W_K
V = x @ W_V

Q = Q.view(B, L, h, d_k).transpose(1, 2)  # (B, h, L, d_k)
K = K.view(B, L, h, d_k).transpose(1, 2)
V = V.view(B, L, h, d_k).transpose(1, 2)

attn = softmax(Q @ K.transpose(-2, -1) / sqrt(d_k))  # (B, h, L, L)
out  = attn @ V  # (B, h, L, d_k)
```

> 📄 Vaswani et al., *Attention Is All You Need*, NeurIPS 2017

---

## 2. MQA 和 GQA：让多个头「共享」K 和 V

Transformer 的作者之一 Noam Shazeer 很早就意识到了 KV Cache 的问题。他在 2019 年提出了 **Multi-Query Attention（MQA）**，思路非常直接：**既然 KV Cache 的大头来自「每个头都有独立的 K、V」，那就让所有头共享同一对 K、V。**

![Multi-group-Attention](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/mqa_attention.excalidraw.png)

GQA（Group Query Attention）则是 2023 年 Google 提出的折中方案——不共享全部，也不独立全部，而是分组共享。

![Multi-group-Attention](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/gqa_attention.excalidraw.png)

### 2.1 Multi-Query Attention（MQA）

**核心思想**：所有注意力头共享同一对 K 和 V，Query 仍保持多头独立。

|                                       | MHA                                  | MQA                                |
| ------------------------------------- | ------------------------------------ | ---------------------------------- |
| K, V 组数                               | $h$ 组                                | **1 组**                            |
| KV Cache（$h=128$, $d_k=56$, $L=4096$） | $2 \times 128 \times 56 \times 4096$ | $2 \times 1 \times 56 \times 4096$ |
| 缓存缩减                                  | —                                    | **减少 128 倍**                       |

**直觉类比**：MQA 就像所有头看同一张地图（共享的 K、V），但各自拿不同的放大镜（独立的 Q）去找自己关心的区域。

**优点**：推理速度显著提升，内存带宽压力大幅降低。
**缺点**：表达能力下降——不同头无法捕获多样化的 K-V 模式，模型质量通常比 MHA 低 3%~5%。

**为什么后来被 GQA 取代？** 因为 MQA 的 $g=1$ 太极端了，质量损失 3%~5% 在大模型上太痛了。

MQA 曾被部分早期模型采用，但后来大多被 GQA 取代。

```python
# MQA 的核心改动：K 和 V 只有 1 组，广播到所有头
K = x @ W_K  # (B, L, d_k)，W_K 形状 (d_model, d_k)，只有 1 组
V = x @ W_V  # (B, L, d_k)，W_V 形状 (d_model, d_k)，只有 1 组

K = K.unsqueeze(1).expand(-1, h, -1, -1)  # 广播到 h 个头: (B, h, L, d_k)
V = V.unsqueeze(1).expand(-1, h, -1, -1)
# Q 仍然保持多头独立，后续计算不变
```

> 📄 Noam Shazeer, *Fast Transformer Decoding: One Write-Head is All You Need*, 2019

### 2.2 Group Query Attention（GQA）

GQA 是 MHA 和 MQA 的折中：将 $h$ 个 Query 头分成 $g$ 个组，同组内的头共享一对 K、V。

- 当 $g = h$ → 退化为 MHA
- 当 $g = 1$ → 退化为 MQA

**以 DeepSeek-V3 的参数规模为例**：$h=128$，$g=8$。KV Cache 减少至 MHA 的 **1/16**，而质量损失不到 1%。

**为什么 GQA 赢了？** 因为它找到了一个「甜点区」：$g=8$ 时，KV Cache 已经足够小，而 8 组 K、V 仍然能保留足够的多样性。

GQA 成为主流大模型的默认选择：LLaMA 3、Qwen2.5 等模型都在用。

```python
# GQA：K, V 有 g 组，每个 Query 头映射到对应的组
K = x @ W_K  # (B, L, g*d_k)，W_K 形状 (d_model, g*d_k)
V = x @ W_V

K = K.view(B, L, g, d_k).transpose(1, 2)  # (B, g, L, d_k)
# 将 g 组广播到 h 个头：每个组重复 h/g 次
K = K.repeat_interleave(h // g, dim=1)  # (B, h, L, d_k)
```

> 📄 Ainslie et al., *GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints*, EMNLP 2023

---

## 3. Multi-Head Latent Attention（MLA）：DeepSeek 的「矩阵吸收」魔法

如果说 MQA/GQA 是在「减少 K、V 的组数」上做文章，那 DeepSeek-V2 提出的 **Multi-Head Latent Attention（MLA）** 则换了一个思路：**不减少组数，而是把 K 和 V 压缩到一个低秩空间里，推理时再「解压」——而且解压和注意力计算可以合并成一步。**

![Multi-latent-attention](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/mla_attention.excalidraw.png)

### 3.1 为什么 DeepSeek 需要 MLA？

DeepSeek-V2 是一个 MoE（Mixture of Experts）模型，总参数量极大（236B，激活 21B）。在 MoE 架构下，KV Cache 的瓶颈比 Dense 模型更严重——因为模型「身子」已经很大了，KV Cache 再大就真的装不下了。GQA 的 8 倍压缩不够用，需要更激进的方案。

### 3.2 核心机制：低秩投影 + 矩阵吸收

MLA 分两步走：

**第一步：低秩投影（压缩）**

原始 embedding $h \in \mathbb{R}^{d_{model}}$ 被投影到低秩空间：

$$
c_{KV} = W_{DKV} \cdot h \quad \in \mathbb{R}^{d_c}, \quad d_c \ll d_{model}
$$

$$
c_Q = W_{DQ} \cdot h \quad \in \mathbb{R}^{d_c'}
$$

推理时，只需要缓存低秩的 $c_{KV}$，而不是完整的 K 和 V——这就是 KV Cache 压缩的来源。

**第二步：上投影 + 矩阵吸收（关键洞察）**

计算注意力时，通过上投影矩阵恢复到每个头的维度。**注意：MLA 是多头注意力，每个头有自己独立的上投影矩阵。** 对于第 $i$ 个头：

$$
K^{(i)} = W_{UK}^{(i)} \cdot c_{KV}, \quad Q^{(i)} = W_{UQ}^{(i)} \cdot c_Q
$$

注意力分数为 $Q^{(i)} {K^{(i)}}^T$。展开来看：

$$
Q^{(i)} {K^{(i)}}^T = (c_Q \cdot W_{UQ}^{(i)}) \cdot (c_{KV} \cdot W_{UK}^{(i)})^T
$$

利用矩阵转置的性质 $(AB)^T = B^T A^T$：

$$
= c_Q \cdot W_{UQ}^{(i)} \cdot {W_{UK}^{(i)}}^T \cdot c_{KV}^T
$$

**关键一步**：$W_{UQ}^{(i)} \cdot {W_{UK}^{(i)}}^T$ 是一个与输入无关的静态矩阵，可以在推理前**提前算好**。于是：

$$
Q^{(i)} {K^{(i)}}^T = c_Q \cdot \underbrace{(W_{UQ}^{(i)} {W_{UK}^{(i)}}^T)}_{\text{预计算，每个头各一份}} \cdot c_{KV}^T
$$

**这意味着什么？** 推理时根本不需要显式地存储和计算完整的 K！只需要存低秩的 $c_{KV}$，然后用预计算好的「吸收矩阵」一步到位算出注意力分数。

**直觉类比**：MLA 就像不存整张高清图，只存压缩后的 JPEG 缩略图（$c_{KV}$）。用的时候不是先解压再看，而是把「解压」和「查找」合并成一步——因为解压矩阵（$W_{UK}$）和查找矩阵（$W_{UQ}$）可以提前「吸收」成一个矩阵。

### 3.3 V 的处理：不能吸收，但也不需要缓存

K 的投影矩阵可以被 Q 的投影矩阵吸收，但 **V 不行**——因为 V 是在 Softmax 之后才乘上去的：

$$
\text{Output}^{(i)} = \text{Softmax}(Q^{(i)} {K^{(i)}}^T) \cdot V^{(i)}
$$

所以 V 必须显式地通过 $V^{(i)} = W_{UV}^{(i)} \cdot c_{KV}$ 恢复出完整维度，**不能**像 K 那样做吸收。

但 MLA 的精妙之处在于：即使 V 需要恢复，你仍然**只需要缓存 $c_{KV}$**。因为 V 是在推理时从 $c_{KV}$ 实时计算的——缓存的是压缩后的「原料」，而不是解压后的「成品」。

### 3.4 实际压缩效果

以 DeepSeek-V2 的配置为例：$d_c = 512$，$h = 128$，$d_k = 128$。

- MHA：每 token 每层缓存 $2 \times 128 \times 128 = 32768$ 个 float16
- MLA：每 token 每层只缓存 $c_{KV}$，即 $512$ 个 float16

**缩减倍数 = 32768 / 512 = 64 倍！** 这比 GQA 的 8 倍压缩高了一个数量级。

### 3.5 位置编码解耦

MLA 的另一个精巧设计是将位置编码从内容中解耦出来。内容和位置信息通过不同的上投影参数处理（内容 128 维 + 位置 64 维），最终注意力分数 = 内容注意力 + 位置注意力。

![projection_dim](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/projection_dim.excalidraw.png)

这个设计使得从 GQA 到 MLA 可以实现**无缝迁移（upcycling）**——不需要从头训练，只需在预训练 GQA 模型的基础上微调转换。

> ⚠️ **训练 vs 推理**：MLA 的矩阵吸收只在推理时有效。训练时，所有 token 的 Q、K、V 并行计算，不需要 KV Cache，也不需要吸收——直接算完整的 Q、K、V 即可。

> 详细的数学推导见 [[MLA注意力计算过程]]

> 📄 DeepSeek-AI, *DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model*, 2024

---

## 4. 滑动窗口注意力（Sliding Window Attention）

在讨论 DeepSeek 的稀疏注意力之前，先介绍一个更简单、更广泛部署的稀疏注意力形式。

**Sliding Window Attention（SWA）** 的思路极其直接：每个 token 只关注最近 $W$ 个 token，而不是整个序列。DeepSeek-V4 的滑动窗口使用 $W=128$。

![swa](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/swa_pick_token.excalidraw.png)
### 4.1 为什么滑动窗口有效？

大多数语言现象的依赖关系是局部的——一个词主要和它附近的词有关系。SWA 利用这个先验，将注意力的计算范围从 $O(L^2)$ 降到 $O(L \cdot W)$。

但 SWA 有一个明显的问题：**它完全看不到远处的 token**。一个 128 的窗口意味着第 1 个 token 和第 200 个 token 之间没有任何直接交互。

### 4.2 SWA 的定位

SWA 是最简单的稀疏注意力——不需要任何选择机制，直接硬编码窗口大小。DeepSeek-V4 正是以 SWA 为基础，在此基础上叠加 CSA/HCA 机制来覆盖中远距离依赖，形成了完整的多尺度注意力方案。

```python
# SWA 核心逻辑：只关注最近的 W 个 token
# W: 窗口大小, L: 序列长度
W = 128

# 构建因果掩码：只保留最近 W 个位置
mask = torch.zeros(L, L, dtype=torch.bool)
for i in range(L):
    # token i 只能看到 max(0, i-W) 到 i 的 token
    mask[i, :max(0, i - W + 1)] = True  # 屏蔽远处 token

# 标准注意力计算，但应用窗口掩码
attn_scores = Q @ K.transpose(-2, -1) / sqrt(d_k)
attn_scores = attn_scores.masked_fill(mask, float('-inf'))
attn = softmax(attn_scores)
out = attn @ V
```

> ⚠️ **KV Cache 优化**：窗口外的 token 不需要缓存 K/V，因为永远不会被后续 token 访问。KV Cache 只需保留最近 W 个 token，实现 O(W) 的固定内存。

---

## 5. 稀疏注意力（DeepSeek Sparse Attention）：不是每个 token 都值得关注

MQA/GQA/MLA 解决的是「KV 本身存多大」的问题，但还有一个维度没动：**每个 token 仍然需要与之前所有 token 计算注意力**。序列长度 $L$ 时，注意力计算量是 $O(L^2)$。

**稀疏注意力**的思路是：不是所有前序 token 都值得关注，只挑最相关的 top-K 个来计算。

### 5.1 Lightning Indexer：一个「廉价版」注意力层

DeepSeek 引入了 **Lightning Indexer**（闪电索引器）——一个维度远低于标准注意力的「轻量级」打分层。它的任务是快速评估当前 token 与所有前序 token 的相关性，排除掉绝对不相关的，只保留 top-K。

训练时，Lightning Indexer 通过**与标准注意力分数的分布对齐**来学习——相当于让一个「小学生」模仿「教授」的打分行为，但速度快得多。

![lightening](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/lightening_indexer_pick_token.excalidraw.png)
### 5.2 工作流程

1. **Lightning Indexer 快速打分** → 排除不相关 token
2. **Top-K 筛选** → 只保留最相关的 K 个
3. **注意力计算** → 在筛选后的子集上做精确计算

```python
# DSA（DeepSeek Sparse Attention）核心逻辑
# 假设已有预训练的 Lightning Indexer

# 1. Lightning Indexer 快速打分（轻量级注意力）
# W_index: 降维后的投影矩阵 (d_model -> d_index, d_index << d_k)
q_idx = current_token @ W_index_Q    # (B, 1, d_index)
k_idx = all_prev_tokens @ W_index_K  # (B, L, d_index)

# 快速相关性打分
scores_idx = q_idx @ k_idx.transpose(-2, -1)  # (B, 1, L)

# 2. Top-K 筛选：只保留最相关的 K 个 token
K_sparse = 4096  # 例如保留 4096 个 token
topk_indices = scores_idx.topk(K_sparse, dim=-1).indices  # (B, 1, K)

# 3. 在筛选后的子集上做精确注意力
K_selected = torch.gather(K_full, dim=2, index=topk_indices.expand(-1, h, -1, d_k))
V_selected = torch.gather(V_full, dim=2, index=topk_indices.expand(-1, h, -1, d_k))

# 标准注意力计算（但只在 K_sparse 个 token 上）
attn_scores = Q @ K_selected.transpose(-2, -1) / sqrt(d_k)
attn = softmax(attn_scores)
out = attn @ V_selected  # (B, h, 1, d_k)

# 计算复杂度从 O(L^2) 降到 O(L * K_sparse)
```

这个机制被沿用到了 DeepSeek-V4。

> 📄 DeepSeek-AI, *DeepSeek-V2 / V3 Technical Report*, 2024

---

## 6. CSA 和 HCA：DeepSeek-V4 的「多尺度注意力」

DeepSeek-V4 在稀疏注意力的基础上更进一步：**不仅挑 token，还把 token 分组压缩后再挑。** 这就是 CSA（Compressed Sparse Attention）和 HCA（Heavily Compressed Attention）。

### 6.1 三层注意力架构

DeepSeek-V4 的注意力机制像一个「多尺度观测系统」：

| 层级       | 覆盖范围         | 压缩比                 | 注意力类型             | 类比      |
| -------- | ------------ | ------------------- | ----------------- | ------- |
| **滑动窗口** | 最近 128 token | 无压缩（精确）             | *配合CSA和HCA*       | 肉眼看近处   |
| **CSA**  | 中等距离         | 每 4 token → 1 个条目   | Shared KV + Top-K | 望远镜看中距离 |
| **HCA**  | 超长距离         | 每 128 token → 1 个条目 | Shared KV         | 卫星图看全局  |

**直觉类比**：
- **滑动窗口** = 你在读这句话时，清楚地看到前后几个词
- **CSA** = 你翻到前一页，能回忆起大概内容（每 4 个词压缩成一个要点）
- **HCA** = 你回忆整本书的目录结构（每 128 个词压缩成一个主题词）

### 6.2 CSA（Compressed Sparse Attention）

CSA 使用一个**可学习的 token 级压缩器**，将每 $m$ 个连续 token 压缩为一个条目。然后通过 Lightning Indexer + Top-K 选择最相关的压缩条目，进行 shared KV Attention 计算。

- KV 序列长度变为原来的 $1/m$
- 适合中等距离（数万到数十万 token）
- 保留了足够的细节精度



![csa](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/csa.excalidraw.png)

```python
# CSA（Compressed Sparse Attention）核心逻辑
# m: 压缩比率（每 m 个 token 压缩为 1 个条目）
m = 4

# 1. Token 级压缩器（可学习的 CNN 或线性投影）
# 将连续 m 个 token 压缩为 1 个条目
compressed_len = L // m
K_compressed = K_full.unfold(dimension=2, size=m, step=m)  # (B, h, L//m, m, d_k)
K_compressed = K_compressed.mean(dim=-2)  # 平均池化 -> (B, h, L//m, d_k)

V_compressed = V_full.unfold(dimension=2, size=m, step=m)
V_compressed = V_compressed.mean(dim=-2)  # (B, h, L//m, d_k)

# 2. Lightning Indexer 在压缩后的序列上打分
q_idx = current_token @ W_index_Q  # (B, 1, d_index)
k_idx = K_compressed @ W_index_K   # (B, h, L//m, d_index)
scores_idx = q_idx @ k_idx.transpose(-2, -1)  # (B, h, 1, L//m)

# 3. Top-K 筛选压缩条目
K_csa = 1024  # 在压缩空间中保留 top-K
topk_indices = scores_idx.topk(K_csa, dim=-1).indices

# 4. 在筛选后的压缩子集上做 shared KV 注意力
K_sel = torch.gather(K_compressed, dim=2, index=topk_indices.expand(-1, h, -1, d_k))
V_sel = torch.gather(V_compressed, dim=2, index=topk_indices.expand(-1, h, -1, d_k))

attn = softmax(Q @ K_sel.transpose(-2, -1) / sqrt(d_k))
out = attn @ V_sel

# KV Cache: 只存压缩后的条目，长度 = L//m
```

### 6.3 HCA（Heavily Compressed Attention）

HCA 采用更激进的压缩策略（每 128 token 一组），压缩后的 KV 全部参与 Shared KV Attention 计算（不再做 Top-K 筛选，因为条目已经足够少了）。

- KV 序列长度变为原来的 $1/128$
- 适合超长距离（百万级 token）
- 牺牲细节，换取全局视野

```python
# HCA（Heavily Compressed Attention）核心逻辑
# 每 128 个 token 压缩为 1 个条目
m_hca = 128

# 1. 粗粒度压缩（比 CSA 更激进）
compressed_len = L // m_hca
K_hca = K_full.unfold(dimension=2, size=m_hca, step=m_hca)
K_hca = K_hca.mean(dim=-2)  # (B, h, L//128, d_k)

V_hca = V_full.unfold(dimension=2, size=m_hca, step=m_hca)
V_hca = V_hca.mean(dim=-2)  # (B, h, L//128, d_k)

# 2. HCA 使用 MQA（所有头共享 K/V）
K_hca = K_hca.mean(dim=1, keepdim=True)  # (B, 1, L//128, d_k) - MQA
V_hca = V_hca.mean(dim=1, keepdim=True)  # (B, 1, L//128, d_k)

# 3. 稠密计算：所有压缩条目全部参与（不做 Top-K）
# 因为 L//128 已经很小，稀疏筛选没必要
attn_scores = Q @ K_hca.transpose(-2, -1) / sqrt(d_k)  # (B, h, 1, L//128)
attn = softmax(attn_scores)
out = attn @ V_hca  # (B, h, 1, d_k)

# KV Cache: 只存压缩条目，长度 = L//128
# 例如 L=1M 时，HCA 只需缓存 ~7813 个条目
```

![HCA](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/hca.excalidraw.png)

### 6.4 关键设计

一个关键设计决策：CSA 用稀疏选择，HCA 用稠密计算。为什么？CSA 压缩比适中，每个压缩条目仍然包含较多信息，值得用 Top-K 筛选来保留最重要的内容，精度优先；HCA 压缩比极高，压缩条目数量已经很少，KV Cache 不再是瓶颈，用更简单的稠密计算即可，简洁优先。这体现了 DeepSeek-V4 的设计哲学：在每一层选择最合适的工具，而不是一刀切。

这种多尺度混合架构让 DeepSeek-V4 在支持百万级上下文的同时，KV Cache 内存占用相比前代 DeepSeek-V3.2 减少了高达 93%，相比传统 GQA 架构减少了约 98%。

> 📄 DeepSeek-AI, *DeepSeek-V4 Technical Report*, 2025

---

## 7. 总结：一张图看清注意力机制的进化

| 方法          | 提出时间 | 核心思路        | KV Cache 缩减             | 计算复杂度                    | 质量损失       | 代表模型             |
| ----------- | ---- | ----------- | ----------------------- | ------------------------ | ---------- | ---------------- |
| **MHA**     | 2017 | 每个头独立 K、V   | 基准（1×）                  | $O(L^2 d)$               | —          | 原始 Transformer   |
| **MQA**     | 2019 | 所有头共享 K、V   | ~$1/h$                  | $O(L^2 d)$               | 3%~5%      | 部分早期模型           |
| **GQA**     | 2023 | 分组共享 K、V    | ~$g/h$                  | $O(L^2 d)$               | <1%（$g=8$） | LLaMA 3, Qwen2.5 |
| **MLA**     | 2024 | 低秩投影 + 矩阵吸收 | ~$d_k h / d_c$（实测 ~64×） | $O(L^2 d)$（常数因子更优）       | 极小         | DeepSeek-V2/V3   |
| **SWA**     | 2020 | 固定窗口        | 窗口外不缓存                  | $O(LWd)$                 | 中远距离丢失     | DeepSeek-V4      |
| **DSA**     | 2025 | 稀疏 Top-K 选择 | 计算量缩减                   | $O(LKd)$                 | 可控         | DeepSeek-V3.2    |
| **CSA/HCA** | 2026 | 多尺度压缩 + 稀疏  | 减少 80%+                 | $O(L(K_{CSA}+K_{HCA})d)$ | 可控         | DeepSeek-V4      |

### 进化主线
![evolution_path](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/PixPin_2026-05-11_15-17-30.png)
每一步都在回答同一个问题：**如何在「记住足够多」和「花足够少」之间找到最优平衡。**

---

## 推荐了解：Flash Attention

前面讨论的所有优化都是模型架构层面的改进。但注意力机制还有一个工程实现层面的优化值得了解——**Flash Attention**。

标准注意力计算的瓶颈不在 FLOPs（浮点运算量），而在 **HBM（高带宽内存）访问**。注意力计算需要在 GPU 的 HBM（几十 GB，带宽 ~2TB/s）和 SRAM（几十 MB，带宽 ~19TB/s）之间反复搬运中间结果（$QK^T$、Softmax 输出等），这些中间结果的形状是 $O(L^2)$，访问开销巨大。

### 核心思路：IO-aware Tiling

Flash Attention（2022）的核心洞察是：**不要在 HBM 中存储完整的 $L \times L$ 注意力矩阵，而是分块（tiling）在 SRAM 中完成 Softmax 计算。**

具体做法：
1. 将 Q、K、V 按块加载到 SRAM
2. 在 SRAM 中计算局部注意力分数 + Softmax（使用在线 Softmax 算法）
3. 累加输出，只把最终结果写回 HBM

这样 HBM 访问量从 $O(N^2 d)$ 降到 $O(N^2 d^2 / M)$（$M$ 是 SRAM 大小），实际推理/训练速度提升 **2-4 倍**。

### Flash Attention 不减少 KV Cache

这是很多人容易混淆的点：**Flash Attention 不改变注意力的数学结果，也不减少 KV Cache 大小。** 它只是让同样的计算跑得更快。它和 MQA/GQA/MLA 是正交的优化维度——你可以同时用 GQA + Flash Attention。

```python
# Flash Attention 的使用（PyTorch 2.0+）
# 数学上等价于标准注意力，但内存访问模式完全不同
from torch.nn.functional import scaled_dot_product_attention

out = scaled_dot_product_attention(Q, K, V, attn_mask=None)
# 内部自动选择 Flash Attention 或 Memory-Efficient Attention
```

**直觉类比**：如果架构优化是在「减少食材的量」，那 Flash Attention 就是在「改进烹饪的手法」——同样的菜，用更高效的方式做出来。

> 📄 Dao et al., *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness*, NeurIPS 2022
> 📄 Dao, *FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning*, 2023

---

# 附录
开篇的计算数据假设

 📐 **本文计算假设**：以 DeepSeek-V3 的模型配置为基准，如果使用MHA，那么 KV Cache 大小估算基于以下参数：
>
> | 参数 | 符号 | 值 | 说明 |
> |------|------|-----|------|
> | 隐藏维度 | $d_{model}$ | 7168 | embedding 向量维度 |
> | 注意力头数 | $h$ | 128 | 注意力头的总数 |
> | 每头维度 | $d_k$ | 128 | 统一比较基准（与 MLA 上投影对齐）|
> | 注意力层数 | $n_{layers}$ | 61 | Transformer 层数 |
> | 压缩维度（MLA）| $d_c$ | 512 | 低秩投影后的维度 |
> | GQA 分组数 | $g$ | 8 | K/V 共享的组数 |
> | 数据类型 | — | float16 | 每个 element 占 2 字节 |
>
> **KV Cache 计算公式**（每 token 每层）：
>
> $$\text{KV Cache per token per layer} = 2 \times n_{kv} \times d_k \times \text{sizeof(float16)}$$
>
> 其中 $n_{kv}$ 为 K/V 的组数：MHA 中 $n_{kv}=h$，GQA 中 $n_{kv}=g$，MQA 中 $n_{kv}=1$，MLA 中 $n_{kv}=d_c / d_k$（仅缓存压缩向量 $c_{KV}$）。
>
> **总 KV Cache** = 上述值 $\times n_{layers} \times L$（序列长度）

*感谢 DeepSeek 团队的开源精神，让这些精妙的设计能够被研究和传播。*

---

## 参考文献

1. Vaswani, A., Shazeer, N., Parmar, N., et al. (2017). **Attention Is All You Need**. NeurIPS 2017. [arXiv:1706.03762](https://arxiv.org/abs/1706.03762)

2. Shazeer, N. (2019). **Fast Transformer Decoding: One Write-Head is All You Need**. [arXiv:1911.02150](https://arxiv.org/abs/1911.02150)

3. Ainslie, J., Lee-Thorp, J., de Jong, M., et al. (2023). **GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints**. EMNLP 2023. [arXiv:2305.13245](https://arxiv.org/abs/2305.13245)

4. DeepSeek-AI (2024). **DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model**. [arXiv:2405.04434](https://arxiv.org/abs/2405.04434)

5. Beltagy, I., Peters, M. E., Cohan, A. (2020). **Longformer: The Long-Document Transformer**. [arXiv:2004.05150](https://arxiv.org/abs/2004.05150)

6. DeepSeek-AI (2025). **DeepSeek-V3.2: Pushing the Frontier of Open Large Language Models**. [arXiv:2512.02556](https://arxiv.org/abs/2512.02556)

7. DeepSeek-AI (2026). **DeepSeek-V4: Towards Highly Efficient Million-Token Context Intelligence**. Technical Report.



