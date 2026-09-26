---
tags:
  - AI/infra/CUDA
  - AI/infra/算子优化
---

# Softmax 与 Online Softmax

Softmax 是 Transformer 里最关键的**非线性**操作，也是理解 FlashAttention 的前置知识。它的优化分两个层面：**工程层**（把三遍扫描做得更快）与**算法层**（把三遍变成两遍）。

后者的那个修正因子，就是 FlashAttention 能够分块计算的数学依据。

## 先解决数值问题：Safe Softmax

直接算 $e^{x_i}$ 会溢出 —— $x_i = 1000$ 时 $e^{1000}$ 超出任何浮点格式。**解法是利用 Softmax 的平移不变性**（见 [[04-概率、Softmax 与信息论]]）：

$$y_i = \frac{e^{x_i - m}}{\sum_j e^{x_j - m}}, \qquad m = \max_j x_j$$

最大的指数输入变成 0，因此最大指数值为 1，不会上溢。这就是 **Safe Softmax**。

### 代价是三遍扫描

$$m = \max_j(x_j) \;\rightarrow\; d = \sum_j e^{x_j - m} \;\rightarrow\; y_i = \frac{e^{x_i - m}}{d}$$

**三遍各读一次输入，总内存读取量 $3N$。** 对 memory-bound 操作来说，**减少一遍就等于性能提升 33%**。

问题在于：第 2 遍求 sum 必须知道第 1 遍的 max —— 看起来没法合并。

扫描次数直接决定要读几遍 HBM（$N$ 是行长度）：

```
  Safe Softmax（三遍）：
    第 1 遍  读 x ──▶ m = max(x)                    ┐
    第 2 遍  读 x ──▶ d = Σ e^(x−m)                 ├ 合计 3N 次读取
    第 3 遍  读 x ──▶ y = e^(x−m)/d，写出 y          ┘

  Online Softmax（两遍）：
    第 1 遍  读 x ──▶ 同时维护 (m, d)（递推 + 修正）  ┐ 合计 2N
    第 2 遍  读 x ──▶ y = e^(x−m)/d，写出 y           ┘

  One-Pass（一遍，整行能装进寄存器时）：
    读 x ──▶ 寄存器 ──▶ 递推得 (m, d) ──▶ 直接算 y 并写出        合计 N

  ⇒ 每少一遍就少读一次 HBM。对 memory-bound 的 Softmax，3N → 2N 是 33%，
     2N → N 再翻近一倍。
```

## Online Softmax：用乘法修正代替重算

核心洞察：**max 与 sum 可以在一遍扫描中同时维护**，代价只是在发现新最大值时「修正」之前累积的 sum。

类比：一边翻花名册一边累加「调整后的总分」。翻到第 50 人时发现他比之前最高分还高 —— 之前算的全偏了。**但偏多少是确切知道的**：之前的 $e^{x_i - m_{old}}$ 应该变成 $e^{x_i - m_{new}}$，总和只需乘一个修正因子 $e^{m_{old} - m_{new}}$。

### 递推关系

设已处理前 $k$ 个元素，维护状态：

- $m_k = \max(x_0,\ldots,x_{k-1})$
- $d_k = \sum_{j=0}^{k-1} e^{x_j - m_k}$（**基于当前 max 的指数和**）

处理第 $k$ 个元素时：

$$m_{k+1} = \max(m_k, x_k)$$

$$d_{k+1} = d_k \cdot e^{\,m_k - m_{k+1}} + e^{\,x_k - m_{k+1}}$$

推导只有一步 —— 把新基准代回去：

$$d_{k+1} = \sum_{j=0}^{k-1} e^{x_j - m_{k+1}} + e^{x_k - m_{k+1}} = \underbrace{\sum_{j=0}^{k-1} e^{x_j - m_k}}_{d_k} \cdot e^{\,m_k - m_{k+1}} + e^{\,x_k - m_{k+1}}$$

> **关键在 $e^{m_k - m_{k+1}}$ 这个修正因子。** 若新元素**不是**新的最大值（$m_{k+1} = m_k$），则 $e^0 = 1$，修正退化为普通累加 —— **没有任何额外开销**。所以「修正」只在少数几次新高峰时发生。

修正因子：发现新最大值时，之前的累加不用重算，乘一个因子就能续上

```
  已处理前 k 个元素，状态是 (mₖ, dₖ)，新元素 xₖ 到来：

        ├── 情形一：xₖ ≤ mₖ ⇒ 基准不变 ⇒ 修正因子 = 1
        │            m_{k+1} = mₖ
        │            d_{k+1} = dₖ + e^(xₖ − mₖ)        ← 普通累加，零额外开销
        │
        └── 情形二：xₖ > mₖ ⇒ 基准抬高
                     m_{k+1} = xₖ
                     d_{k+1} = dₖ · e^(mₖ − m_{k+1}) + 1
                               └───── 修正因子 ≤ 1，把旧的 sum 按新基准缩一次

  ⇒ 把「必须知道全部数据的 max 才能求 sum」换成「先按当前 max 算，遇新高峰
     再按比例缩」—— 于是 m 与 d 能在一遍扫描里同时维护。
```

### 算法

```
// 第 1 遍：同时得到 m 和 d
m = -∞ ; d = 0
for j = 0 to N-1:
    m_new = max(m, x[j])
    d = d * exp(m - m_new) + exp(x[j] - m_new)
    m = m_new

// 第 2 遍：归一化
for j = 0 to N-1:
    y[j] = exp(x[j] - m) / d
```

**从 3N 降到 2N，理论提升 33%。**

### 数值稳定性与 Safe Softmax 完全等价

三条论证：

- $m$ 始终是已见元素的最大值 → 所有 $x_j - m \le 0$ → **不上溢**
- 修正因子 $e^{m_k - m_{k+1}} \le 1$（因为 $m_{k+1} \ge m_k$）→ **也不上溢**
- 最终 $d$ 与三遍结果**数学上完全等价**

> **Online Softmax 不是近似算法** —— 它的结果与 Safe Softmax 逐位可比（除浮点非结合性带来的微小差异，见 [[01-数值计算与精度]]）。这是它能被 FlashAttention 采纳的前提。

## 工程层的优化阶梯（Safe Softmax）

| 版本 | 核心优化点 | 带宽利用率 | 相对加速 |
| --- | --- | --- | --- |
| V0 单线程/行 | 无 | **约 5%** | 1.0× |
| V1 Block 并行 | 多线程协作 + 合并访存 + 共享内存规约 | 约 35% | **7.0×** |
| V2 Warp Shuffle | 寄存器级两级规约 | 约 52% | 10.4× |
| V3 向量化加载 | `float4` 减少指令数 | 约 65% | 13.0× |
| V4 两遍融合 | 减少一次全局读取 | 约 75% | 15.0× |

**V1 一步就拿到 7 倍** —— 因为 V0（一个线程处理一整行）完全没有并行，是纯粹的串行实现。**这是「先做对并行划分，再谈细节优化」的典型**：后面三级加起来才把 35% 提到 75%。

V4 的「两遍融合」是把第 2、3 遍合并：

```cpp
// 第 1 遍算 m 与 d
// 第 2 遍一次写出 y_i = exp(x_i - m) / d
```

代价是**每个元素要读两次**（第 1 遍读一次、第 2 遍再读一次）—— 除非寄存器能装下整行。这就引出了下一节的 One-Pass。

## Online Softmax 的优化阶梯

| 版本 | 核心优化点 | 扫描次数 | 带宽利用率 | 相对加速 |
| --- | --- | --- | --- | --- |
| V0 单线程/行 | 无 | 2 | 约 7% | 1.0× |
| V1 Block 并行 + Warp Shuffle | 寄存器级合并规约 | 2 | 约 58% | **8.3×** |
| V2 向量化加载 | `float4` | 2 | 约 70% | 10.0× |
| **V3 寄存器缓存** | **One-Pass** | **1** | **约 82%** | **11.7×** |
| V4 Grid Stride | 多行并行 + 适配任意 $M$ | 2 | 约 72% | 10.3× |

### V3 的 One-Pass 是怎么做到的

如果**整行能装进寄存器**，那么一遍扫描就能同时完成 $(m, d)$ 的计算和输出的写出 —— 因为元素已经在寄存器里，不需要第二次从 HBM 读。

```
读入整行到寄存器（float4 分块）
  → 一遍 Online 递推同时更新 (m, d)
  → 用最终的 (m, d) 直接算 y
```

**V3 达到 82%，是本组最高** —— 但**受寄存器容量限制**：$N \le 8192$ 时才可行（每线程 255 个寄存器的上限，见 [[04-Occupancy、同步与原子操作]]）。

### 「合并规约」是什么

V1 的难点在于：Online 递推是**顺序**的，但 Block 内多个线程各持一部分数据。解法是把「合并两个 $(m, d)$ 状态」写成可并行的操作：

$$m_{new} = \max(m_a, m_b), \qquad d_{new} = d_a e^{\,m_a - m_{new}} + d_b e^{\,m_b - m_{new}}$$

**这与单元素的递推是同一个公式，只是操作数从「标量」变成「子状态」。** 于是可以像普通归约一样、用 Warp Shuffle 做树形合并 —— 硬件不关心合并的是什么，只关心它是结合的。这就是 Online 递推「可组合状态」的价值，与 [[11-Self-Attention 机制]] 里讲的是同一件事。

### V4 的取舍

V2/V4 的 Two-Pass 方案**通用性更强** —— 任意 $N$ 下都能到 70%+，代价是 72% 略低于 V3 的 82%。

| $N$ | 该选 |
| --- | --- |
| $\le 8192$ | V3 One-Pass（82%） |
| 任意 $N$ | V2 / V4 Two-Pass（70%+） |

两组优化阶梯（左边是 Safe Softmax 的工程层，右边是 Online Softmax）：

```
  Safe Softmax（三遍扫描）                  Online Softmax（两遍 / 一遍）
    V0 单线程/行      约  5%   1.0×          V0 单线程/行         约  7%   1.0×
    V1 Block 并行     约 35%   7.0× ▲ 最大   V1 Block + Shuffle   约 58%   8.3× ▲ 最大
    V2 Warp Shuffle   约 52%  10.4×          V2 向量化加载        约 70%  10.0×
    V3 向量化加载     约 65%  13.0×          V3 寄存器缓存（1 遍） 约 82%  11.7× ▲ 最高
    V4 两遍融合       约 75%  15.0×          V4 Grid Stride       约 72%  10.3×

  ⇒ 两组都是「先把并行划分做对」拿大头（V1 一步就是 7–8 倍），
     后面各细节加起来才把 35%→75% / 58%→82%
  ⇒ V3 的一遍扫描最高，但要求整行装进寄存器（N ≤ 8192）；超了就退回两遍方案
```

## 从这里到 FlashAttention

**Online Softmax 就是 FlashAttention 的核心。** 标准 Attention：

$$\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V$$

传统实现要先算出完整的 $N \times N$ 矩阵、再 Softmax、再乘 $V$ —— **这个矩阵必须落在 HBM 里**。$N = 128\text{K}$ 时它是 **64 GB**（算例见 [[11-Self-Attention 机制]]）。

FlashAttention 把 $K$、$V$ 分块加载到片上，每算一块 $QK^\top$ 就**增量更新 Softmax 状态 $(m, d)$**，同时修正已累积的输出 —— 全程不存完整 Attention 矩阵。

**修正输出的公式**（在线 Softmax 的修正因子在这里多了一件事：还要修正已经累加的输出）：

$$O_{new} = O_{old} \cdot \frac{d_{old}}{d_{new}} \cdot e^{\,m_{old} - m_{new}} \;+\; \frac{e^{\,m_{block} - m_{new}}}{d_{new}} \cdot P_{block} V_{block}$$

> **两点变化，其余完全一致**：
> 1. 分母从 $d_{old}$ 变成 $d_{new}$ —— 归一化基准变了
> 2. **之前的输出 $O_{old}$ 也要按比例缩放** —— 因为它是用旧的 $(m_{old}, d_{old})$ 归一化出来的
>
> 理解了 Online Softmax 的递推与合并公式，**FlashAttention 的分块策略只是在此基础上多了一个矩阵乘的增量更新**。

```cpp
// 核心循环（伪代码）
for each block of K, V:
    S = Q @ K_block^T                           // 分块分数
    m_new = max(m_old, rowmax(S))
    P = exp(S - m_new)
    d_new = d_old * exp(m_old - m_new) + rowsum(P)
    O = O * (d_old * exp(m_old - m_new) / d_new) + (P / d_new) @ V_block
    m_old, d_old = m_new, d_new
```

## 工程结论

> **融合才是终极答案。** 在 vLLM、TensorRT-LLM 这类推理框架里，**Softmax 不会单独存在** —— 它和 Scale、Mask、MatMul 融合成一个大 kernel，中间结果不落地到全局内存。
>
> 单独优化 Softmax 的价值在于**理解原理**，以及特定独立场景（如分类层的 Softmax）。

| 场景 | 方案 |
| --- | --- |
| 独立 Softmax（分类层） | V2 或 V3（看 $N$） |
| **Attention 中的 Softmax** | **用 FlashAttention 的融合实现** |
| 理解原理 | V1 |
| 自定义 Attention 变体 | 基于 V2 的模式扩展 |
| 追求极致 | cuDNN / FlashAttention 的融合实现 |

## 相关

- [[04-概率、Softmax 与信息论]] —— 稳定 Softmax、LogSumExp 与平移不变性
- [[11-Self-Attention 机制]] —— Online Softmax 的「可组合状态」在 Attention 里的位置
- [[05-Reduce 算子优化]] —— 同一套 Warp Shuffle 两级归约
- [[06-GEMM 性能优化]] —— Attention 里的两个矩阵乘从哪来
- [[01-数值计算与精度]] —— 浮点非结合性与归约顺序

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/52-cuda-online-softmax%E5%AE%9E%E7%8E%B0
- https://arxiv.org/abs/1805.02867
- https://arxiv.org/abs/2205.14135
- https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html
