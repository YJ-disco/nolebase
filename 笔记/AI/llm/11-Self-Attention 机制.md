---
tags:
  - AI/llm/架构
  - AI/llm/Attention
---

# Self-Attention 机制

[[02-Transformer 架构]] 给了整体骨架，这一篇把其中最重、也最讲究的那一块拆开：**Attention 到底在算什么、为什么长成这个形状、它的代价在哪**。

## 演进脉络：为什么是点积

| 时间 | 方案 | 匹配函数 | 贡献 |
| --- | --- | --- | --- |
| 2014 | **Bahdanau Attention** | 一个额外的前馈网络（additive） | 首次让 Decoder 回头「看」Encoder 的所有隐状态，而不是强行压缩成一个定长向量 |
| 2015 | **Luong Attention** | **直接用点积** | 计算效率远高于前馈网络，且**天然是矩阵乘法**，适合 GPU 并行 |
| 2017 | **Self-Attention** | 点积 | 完全抛弃 RNN，只用 Attention 建模序列；从「跨序列」变成「自关注」 |

两条演进线索值得记住：

- **匹配函数从「学出来的网络」变成「点积」** —— 因为点积可以被表示为矩阵乘法，从而吃到 GPU 的并行度。这是所有后续优化的前提。
- **Self-Attention 的两个收益**：**并行性**（所有位置同时算，不像 RNN 依赖上一步）与**长距离依赖**（任意两位置一步直达，不存在信号衰减）。代价是 $O(N^2)$ 复杂度 —— **这个平方项是后面所有 Attention 优化工作的出发点**。

## QKV：从信息检索理解

三个词直接借自信息检索。想象走进图书馆：

- 你脑子里的需求「我想了解并行计算」→ **Query**
- 每本书书脊上的标签 → **Key**
- 每本书的实际内容 → **Value**

拿 Query 和每本书的 Key 比对得到匹配分数，然后**按分数对 Value 加权汇总** —— 取的是内容（Value）而留下标签（Key）。

形式化：给定输入 $X \in \mathbb{R}^{N \times d_{model}}$，三个独立线性变换：

$$Q = XW_Q, \qquad K = XW_K, \qquad V = XW_V$$

| 投影 | 语义 |
| --- | --- |
| $W_Q$ | 「我需要什么信息」 |
| $W_K$ | 「我能提供什么信息的**线索**」 |
| $W_V$ | 「我实际携带的信息」 |

**为什么不用 $X$ 本身而要做三次不同投影？** 因为一个 token「需要什么」和「能提供什么」是两个不同的角色。动词可能需要关注主语和宾语（Query 方向），但它作为被关注对象时提供的是动作语义（Key/Value 方向）。三个独立投影让模型有自由度学这两套映射。

QKV 借的是信息检索里的三个角色。

```
走进图书馆
  你脑子里的需求「我想了解并行计算」  ──▶  Query：我需要什么信息
  每本书书脊上的标签                 ──▶  Key  ：我能被什么查询命中
  每本书的实际内容                   ──▶  Value：我实际携带的信息

拿 Query 与每本书的 Key 比对得到匹配分数 ──▶ 按分数对 Value 加权汇总
取的是内容（Value），标签（Key）本身不进结果

落到张量上是三个独立线性变换，而非一个
  Q = X W_Q      W_Q：我需要什么信息
  K = X W_K      W_K：我能提供什么信息的线索
  V = X W_V      W_V：我实际携带的信息

为什么不能直接用 X 代替这三次投影
  一个 token「需要什么」和「能提供什么」是两个不同的角色。
  动词可能需要关注主语和宾语（Query 方向），
  但它作为被关注对象时提供的是动作语义（Key / Value 方向）。
  三次独立投影让模型有自由度分别学这两套映射。
```

## 五步计算

$$\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V \cdot W_O$$

| 步 | 运算 | 形状（单头） |
| --- | --- | --- |
| 1 | 线性投影 | $(N, d)@(d,d) \to (N,d)$，三次 |
| 2 | 分数 $S = QK^\top$ | $(N,d)@(d,N) \to (N,N)$ |
| 3 | 缩放 $S/\sqrt{d_k}$ | $(N,N)$ |
| 4 | 按行 Softmax | $(N,N)$，每行和为 1 |
| 5 | 加权求和 $AV$ | $(N,N)@(N,d) \to (N,d)$ |

**第 2 步产生 $N \times N$ 矩阵，这就是平方复杂度的根源。**

两处工程细节：

- **三个投影通常合并成一次 GEMM** —— 把 $W_Q$、$W_K$、$W_V$ 拼成 $W_{QKV} \in \mathbb{R}^{d \times 3d}$，一次矩阵乘再 split，GPU 利用率更高
- **输出投影 $W_O$ 不是可选项** —— 它负责把多个头拼接后的表示重新混合

### 维度全程跟踪（具体数值）

设定 `seq_len=6`、`d_model=512`、`num_heads=8`、`head_dim=64`、`batch=1`：

```
设定 seq_len=6、d_model=512、num_heads=8、head_dim=64、batch=1

X                       (1, 6, 512)
   │  @ W^T（三次，通常合并成一次 GEMM 再 split）
   ▼
Q / K / V               (1, 6, 512)
   │  view(1, 6, 8, 64)
   ▼
                        (1, 6, 8, 64)
   │  transpose(1, 2)：把 head 维提到 seq_len 前面
   ▼
                        (1, 8, 6, 64)      ← 8 个头从这一步起各自独立
   │  Q @ K^T（K 先 transpose(-2,-1) 成 (1, 8, 64, 6)）
   ▼
分数矩阵                 (1, 8, 6, 6)
   │  ÷ sqrt(64) = 8
   ▼
                        (1, 8, 6, 6)
   │  softmax(dim=-1)：每行 6 个元素，和为 1
   ▼
注意力权重               (1, 8, 6, 6)
   │  @ V
   ▼
                        (1, 8, 6, 64)
   │  transpose(1, 2) → view(1, 6, 512)：8 × 64 拼回 d_model
   ▼
context                 (1, 6, 512)
   │  @ W_O^T
   ▼
输出                     (1, 6, 512)

输入与输出形状都是 (batch, seq_len, d_model) —— 这是多个 Transformer Block
能像积木一样堆叠的前提。
```

**输入与输出形状都是 $(batch, seq\_len, d_{model})$** —— 这是多个 Transformer Block 能像积木一样堆叠的前提。

那个 `transpose(1, 2)` 的作用：把 head 维移到 `seq_len` 前面，后续矩阵乘就能**在每个头内部独立进行，同时利用 batch 维一次算完所有头**。

## 为什么要除以 $\sqrt{d_k}$

这一步常被当作「调参技巧」，但它有严格的推导。

设 $q_i, k_i$ 独立同分布、均值 0、方差 1。单个分量乘积：

$$\mathbb{E}[q_i k_i] = 0, \qquad \operatorname{Var}(q_i k_i) = \mathbb{E}[q_i^2]\mathbb{E}[k_i^2] = 1$$

点积是 $d_k$ 个独立项之和，方差可加：

$$\operatorname{Var}(q \cdot k) = \sum_{i=1}^{d_k} \operatorname{Var}(q_i k_i) = d_k$$

**点积的标准差是 $\sqrt{d_k}$** —— $d_k = 64$ 时约 8，$d_k = 128$ 时约 11.3。维度越大，分数绝对值越大、分布越分散。

Softmax 对输入量级极敏感：输入差距大时（比如 50 与 −10），输出极度集中接近 one-hot，**梯度趋近于零，训练停滞**。

除以 $\sqrt{d_k}$ 后：

$$\operatorname{Var}\left(\frac{q\cdot k}{\sqrt{d_k}}\right) = \frac{d_k}{d_k} = 1$$

**方差被拉回 1，无论 $d_k$ 取多少** —— 这就是名字里 "Scaled" 的由来。基础推导见 [[04-概率、Softmax 与信息论]]。

## Softmax 的数值稳定性与 Online Softmax

稳定形式与它的三遍扫描问题见 [[04-概率、Softmax 与信息论]]：先求 $\max$、再求 $\exp$ 与和、最后相除 —— **三遍扫描就是三次 HBM 读写**。

序列到 128K 时这个 IO 开销不可忽略。**Online Softmax**（Milakov & Gimelshein, 2018）把它压到一遍（加最后归一化共两遍）：遍历时动态维护 $m_i$（前 $i$ 个的最大值）与 $d_i$（以 $m_i$ 为基的指数和）：

$$m_i = \max(m_{i-1}, z_i)$$

$$d_i = d_{i-1} \cdot e^{\,m_{i-1} - m_i} + e^{\,z_i - m_i}$$

**关键在第二行那个修正因子 $e^{m_{i-1}-m_i}$** —— 最大值更新时，之前累积的指数和要「换基」。这就是「可组合状态」的具体形态：分块计算时只需在块间传递 $(m, d)$ 两个标量。

> **对 FlashAttention 来说这是前提条件。** 它把 Attention 矩阵切成 tile 逐块处理，每处理一块都要更新 Softmax 中间结果 —— 如果不能在线更新，就必须把整个 $N \times N$ 矩阵写回 HBM 再做全局 Softmax，分块就失去了意义。Online Softmax 让分块与精确 Softmax 得以兼容。
>
> 这也印证了 [[01-数值计算与精度]] 收尾那条判据：**数值稳定的算法通常也更容易写成分块 kernel。**

## Multi-Head

### 为什么多头

单头只有一组 QKV 投影，只能学一种「关注模式」。但语言里的关系是多维度的 —— 同一个词可能同时存在句法关系（主谓一致）、语义关系（同义替换）、位置关系（局部模式）。

多头就是「派一个评审团」：每个头有独立的 $W_Q^i, W_K^i, W_V^i$，在**不同子空间**捕捉不同类型的关系，最后拼接并混合。

### 参数量：头数不改变总量

$$\text{MultiHead}(Q,K,V) = \text{Concat}(\text{head}_1,\ldots,\text{head}_h)W_O$$

| 矩阵 | 形状 | 参数量（$d=512$） |
| --- | --- | --- |
| $W_Q$ / $W_K$ / $W_V$ / $W_O$ | 各 $(512,512)$ | 各 262,144 |
| 合计 | | $4d^2 = 1{,}048{,}576$ |

**MHA 的参数量是 $4d_{model}^2$，与头数无关** —— 每加一个头，每头的维度相应减小，总投影维度始终等于 $d_{model}$。**头数只影响切分方式，不影响参数量。**

实现上**不会真的维护 $h$ 组小矩阵**，而是用一个大矩阵做一次投影再 reshape 切分 —— 二者等价（大矩阵可看作 8 个小矩阵纵向拼接）。

### 多头的工程意义

8 个头的计算完全独立，这带来两个并行机会：

- **GPU 内并行** —— 利用 batch 维，一次 kernel 启动处理所有头
- **张量并行** —— 不同头分到不同 GPU。8 个头分到 4 卡，每卡算 2 个头，最后通过一次 AllReduce 汇总输出投影结果

第 2 条是后面张量并行的实现基础 —— **它之所以能切，是因为头之间在数学上互不依赖**。

## MHA → MQA → GQA → MLA

演进的驱动力只有一个：**KV Cache 太大**。

| 方案 | Q 头 | KV 头 | 共享关系 |
| --- | --- | --- | --- |
| **MHA** | $h$ | $h$ | 每组独立 |
| **MQA**（Shazeer, 2019） | $h$ | **1** | 所有 Q 头共享同一组 KV |
| **GQA**（Ainslie et al., 2023） | $h$ | $g$ | 每 $h/g$ 个 Q 头共享一组 |
| **MLA**（DeepSeek-V2） | $h$ | — | 不减少头数，而是**把 K/V 压缩到低维潜在空间** |

GQA 是 MHA 与 MQA 的折中：$g=h$ 退化为 MHA，$g=1$ 退化为 MQA。

### 具体数字（$d_{model}=4096$、32 头、$head\_dim=128$、FP16、$N=4096$）

| 指标 | MHA（$g=32$） | GQA（$g=8$） | MQA（$g=1$） |
| --- | --- | --- | --- |
| KV 头数 | 32 | 8 | 1 |
| $W_K$ 参数量 | 16 M | 4 M | 0.5 M |
| 单 token KV Cache（每层） | 16 KB | **4 KB** | 0.5 KB |
| 4096 token KV Cache（每层） | 64 MB | **16 MB** | 2 MB |
| 模型质量 | 最好 | 接近 MHA | 有下降 |

$$\text{KV Cache} = 2 \times B \times L \times n_{kv} \times d_{head} \times \text{bytes}$$

（2 是 K 和 V，$B$ batch，$L$ 已生成序列长度。）

> **GQA 从 MHA 到 $g=8$ 把 KV Cache 缩到 1/4，而质量几乎不受影响** —— 这就是它成为主流默认的原因。完整账本与 MHA/GQA/MQA 的取舍见 [[10-KV Cache 与推理优化]]。

### MLA 是另一条路线

**GQA 减少 KV 头的数量，MLA 降低每个表示的维度。** MLA 不缓存完整 K/V，而是缓存一个低维潜在表示：

```
MHA：X → W_K → K            （缓存 K）
MLA：X → W_DKV → c          （缓存 c，维度远小于 K）
     推理时 c → W_UK → K     （按需解压）
```

DeepSeek-V2 报告了约 **93.3%** 的压缩率，同时通过精心设计的上投影矩阵保持质量。两条路线的目标一致：**让推理时的 KV Cache 尽可能小**。

KV Cache 的大小由 KV 头数决定，四代布局都在压这个量。

```
MHA（Q 头 h，KV 头 h）
   Q1  Q2  Q3  Q4  Q5  Q6  Q7  Q8
    │   │   │   │   │   │   │   │
   K1  K2  K3  K4  K5  K6  K7  K8
        └─ 每组 K/V 独立，缓存最大

MQA（Q 头 h，KV 头 1）—— 所有 Q 头共享同一组 KV
   Q1  Q2  Q3  Q4  Q5  Q6  Q7  Q8
   └───┴───┴───┴───┴───┴───┴───┘
                 K

GQA（Q 头 h，KV 头 g）—— 每 h/g 个 Q 头共享一组，是上面两者的连续折中
   Q1  Q2  Q3  Q4   Q5  Q6  Q7  Q8
   └───┴───┴───┘    └───┴───┴───┘
        K1               K2
   └─ g = h 退化为 MHA，g = 1 退化为 MQA

MLA（不减少头数，降低每个表示的维度）
   MHA：X ──▶ W_K   ──▶ K          缓存 K
   MLA：X ──▶ W_DKV ──▶ c          缓存 c，维度远小于 K
               推理时 c ──▶ W_UK ──▶ K    按需解压

d_model = 4096、32 头、head_dim = 128、FP16、N = 4096 时的对照
   KV 头数              32        8          1
   单 token KV（每层）  16 KB     4 KB       0.5 KB
   4096 token（每层）   64 MB     16 MB      2 MB
   └─ GQA 从 32 缩到 8，缓存到 1/4，质量接近 MHA
      MQA 压得更狠却没能普及 ──▶ 这条曲线在 1 那个位置已经过头了
```

## Causal Mask

自回归模型的训练目标是「根据前文预测下一个 token」，所以每个位置**只能看到自己和之前的 token**，不能偷看未来 —— 否则训练时就看到答案，学不到任何预测能力。

实现是一个上三角为 False 的掩码，把未来位置的分数设为 $-\infty$：

```
mask（上三角为 False）                softmax 之后的权重
┌ T  F  F  F ┐                       ┌ w00  0    0    0   ┐
│ T  T  F  F │   ── softmax ──▶      │ w10  w11  0    0   │
│ T  T  T  F │                       │ w20  w21  w22  0   │
└ T  T  T  T ┘                       └ w30  w31  w32  w33 ┘

下三角为 True 的部分保留，其余位置在分数矩阵上被填成 −1e9，
过 softmax 后概率为 0 —— 未来位置被排除了，但「跳过计算」是另一件事
（上三角的 tile 在 FlashAttention-2 里可以直接不处理）。
```

两处实现细节：

- **用 `-1e9` 而不是 `-inf`** —— 半精度下 `-inf` 参与运算容易出 NaN。这是数值稳定性对数学纯洁性的让步，同类做法见 [[01-数值计算与精度]]
- **上三角可以跳过计算** —— 有效部分只有下三角。**FlashAttention-2 正是利用这一点，在处理纯上三角的 tile 时直接跳过，省掉接近一半的计算量**

## 瓶颈：两个项，两种场景

六次主要矩阵乘的 FLOPs：

| 运算 | FLOPs |
| --- | --- |
| $Q, K, V = XW$（三次） | $3 \times 2Nd^2$ |
| $S = QK^\top$ | $2N^2 d$ |
| $O = SV$ | $2N^2 d$ |
| $\text{Out} = OW_O$ | $2Nd^2$ |

$$\text{总计} = \underbrace{8Nd^2}_{\text{投影}} + \underbrace{4N^2 d}_{\text{Attention 矩阵}}$$

**这一分为二给出了一个判据**：

| 条件 | 主导项 | 瓶颈 |
| --- | --- | --- |
| $N \ll d$ | $8Nd^2$ | **QKV 投影的 GEMM** —— 算力受限 |
| $N \gg d$ | $4N^2 d$ | **Attention 矩阵** —— 且它是 memory-bound 的重灾区 |

第二条正是 [[01-GPU 硬件架构与存储层次]] 里「算术强度远低于 295 FLOP/Byte」的典型例子：$N \times N$ 的分数矩阵要写回 HBM、再读回来做 Softmax、再读一次做加权求和 —— **访存量是 $O(N^2)$，计算量也是 $O(N^2)$，但访存占了绝对大头**。

> **这就是 FlashAttention 存在的全部理由**：不把 $N \times N$ 中间矩阵写回 HBM，而在片上分块完成稳定 Softmax 与 $PV$ 累积。它不改变 FLOPs，只改变访存。

一篇具体的显存估算：$B=1$、$N_h=32$、$S=32768$ 时，显式物化 FP16 的分数矩阵需要 $2 \times 1 \times 32 \times 32768^2 = 64$ GiB —— 还没算概率、反向中间量与其它层。

FLOPs 一分为二，这个拆分本身就是一条判据。

```
总 FLOPs = 8Nd²（QKV 投影与输出投影） + 4N²d（Attention 矩阵）
           N 序列长度、d 表示维度

N 与 d 的大小关系决定哪一项主导

N ≪ d（短序列、宽表示）
   主导项 8Nd² ──▶ 瓶颈在投影的 GEMM，算力受限
   例：N = 512、d = 4096

N ≫ d（长上下文）
   主导项 4N²d ──▶ 瓶颈是 N × N 的分数矩阵，且它是 memory-bound 的重灾区
   例：N = 32768、d = 128
   └─ 分数矩阵要写回 HBM、读回来做 Softmax、再读一次做加权求和
      访存量与计算量都是 O(N²)，但访存占绝对大头
      算术强度远低于硬件那条 295 FLOP/Byte 的分界

这就是 FlashAttention 存在的全部理由：不把 N × N 中间矩阵写回 HBM，
而在片上分块完成稳定 Softmax 与 PV 累积 —— 它不改变 FLOPs，只改变访存。

一个具体的显存估算：B = 1、N_h = 32、S = 32768 时，显式物化 FP16 的
分数矩阵需要 2 × 1 × 32 × 32768² = 64 GiB，还没算概率、反向中间量与其他层。
```

## 相关

- [[02-Transformer 架构]] —— 整体骨架，本篇是其中 Attention 子块的展开
- [[12-FFN 与激活函数]] —— 同一个 Block 里的另一个子模块
- [[13-归一化与残差连接]] —— 子模块之间的那两层
- [[09-位置编码]] —— Attention 本身对顺序不敏感，顺序信息靠它注入
- [[10-KV Cache 与推理优化]] —— MHA/GQA/MQA 的完整显存账本
- [[04-概率、Softmax 与信息论]] —— 稳定 Softmax、LogSumExp 与方差推导
- [[01-数值计算与精度]] —— 为什么用 `-1e9` 而不是 `-inf`

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/transformer/33-self-attention%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3
- https://arxiv.org/abs/1706.03762
- https://arxiv.org/abs/1409.0473
- https://arxiv.org/abs/1508.04025
- https://arxiv.org/abs/1911.02150
- https://arxiv.org/abs/2305.13245
- https://arxiv.org/abs/2405.04434
- https://arxiv.org/abs/1805.02867
- https://arxiv.org/abs/2205.14135
