---
tags:
  - AI/infra/并行策略
  - AI/infra/MoE
---

# 专家并行与 MoE

MoE 是当前千亿/万亿参数模型（DeepSeek-V3、Mixtral 等）的主流架构 —— **用稀疏激活在不显著增加计算量的前提下扩大参数规模**。

它引入了一个**前面所有并行维度都不覆盖的新维度**：专家并行（EP）。这一篇的重点是它的标志性通信模式 **All-to-All**，以及它独有的负载均衡问题。

## 一、MoE 结构回顾

**稠密 FFN → 稀疏 MoE：用 $E$ 个 Expert 替换单个 FFN。**

| 组件 | 作用 |
| --- | --- |
| **Router（Gating Network）** | 为每个 token 选择 Top-$K$ 个 Expert |
| Expert | 每个都是一个独立 FFN（见 [[12-FFN 与激活函数]]） |
| **容量因子（Capacity Factor）** | 每个 Expert 能接收的 token 上限 |

**稀疏激活的核心权衡**：

```
参数量    × E        ← 容量大幅扩大
单 token 计算量 × K  ← 计算量小幅增加
```

例如 64 个 Expert、Top-2：**参数是 64 倍，但每个 token 只算 2 个**。这就是「参数与计算解耦」。

**容量因子与 token 丢弃**：容量因子是每个 Expert 的 token 上限。超出的 token 会被丢弃（**跳过这个 Expert 的计算**）—— 丢弃是为了保证所有 Expert 的批形状一致，代价是**那些 token 的信息没被处理**。

## 二、Expert Parallelism（EP）

**核心思想：不同 Expert 放在不同 GPU 上**，每卡只存 $E/N$ 个 Expert。

**EP 与 TP 的区别是切的对象不同**：

| | 切什么 |
| --- | --- |
| TP | 切**单个权重矩阵**的行/列 |
| **EP** | 切「**哪些 Expert 在哪**」—— Expert 本身是完整的 |

> **这个区别决定了通信模式**：TP 切的是矩阵，通信是 AllReduce（同样的形状、求和）；EP 切的是「归属」，通信是 **All-to-All**（数据要按归属重新分发，形状会变）。

## 三、All-to-All：MoE 的主要瓶颈

**每个 MoE 层要做两次 All-to-All**：

```
① dispatch（分发）
   每个 token 被发往它选中的 K 个 Expert 所在的 GPU

② combine（收回）
   各 Expert 算完后，结果发回 token 原始的 GPU
```

| 阶段 | 传什么 |
| --- | --- |
| dispatch | **token 的隐藏表示** → 发往 Expert 所在卡 |
| combine | **Expert 的输出** → 发回原卡 |

**通信量取决于**：token 数、Top-$K$、Expert 分布。

> **为什么 All-to-All 是瓶颈**：
> 1. 它是**全互联**通信模式，对网络拓扑的带宽要求最高（见 [[03-多卡互联与集群网络]]）
> 2. 每个 MoE 层都要做两次，频率高
> 3. **通信量随 token 数（即 batch × 序列长度）线性增长** —— 不像 PP 那样与层数无关
>
> 与 TP 的对比：TP 是每层 2 次 AllReduce，EP 是每层 2 次 All-to-All。**All-to-All 的数据重排比 AllReduce 更贵** —— 因为每个 rank 发给其他每个 rank 的数据量不一定相等。

**通信优化方向**：分组 All-to-All（把通信域切小）、**计算通信重叠**（Expert 算的同时传下一批 token）、专用通信库（如 DeepEP）。

## 四、负载均衡

### 问题：Router 倾斜

**少数 Expert 被过度选择 → 部分 GPU 过载、部分空闲。** 因为 Router 是根据输入学出来的，而真实数据分布本身不均匀。

后果很直接：**MoE 层的耗时由最忙的那个 Expert 决定**，其他 Expert 提前算完就得等 —— 稀疏激活本应带来的收益被摊薄。

### 两类解法

| 解法 | 机制 | 代价 |
| --- | --- | --- |
| **Auxiliary Loss（辅助损失）** | 在训练损失上加一项，引导 Router 均匀分发 token | **与主任务目标冲突** —— 是在「让模型学好」和「让负载均衡」之间做妥协 |
| **Aux-loss-free（DeepSeek 的做法）** | 不改进损失，而是**动态调整每个 Expert 的 bias** 来纠偏 | 不污染主任务目标 |

> **aux-loss-free 的价值在于「不把工程约束写进优化目标」** —— 辅助损失会让模型在「预测得准」和「负载均衡」之间取折中，而 bias 调整把这件事挪到了损失函数之外。这是一个值得记的设计取向：**能用工程手段解决的，就不要污染目标函数。**

**Expert 容量与 drop/pad 策略**是负载均衡的下游手段：容量满了就丢 token（drop）或补齐（pad）—— 前者损失信息，后者浪费算力。

## 五、EP 与其他维度的组合

| 组合 | 怎么切 |
| --- | --- |
| **EP × DP** | EP 组与 DP 组**正交划分**（不同维度独立分组） |
| **EP × TP** | Expert 内部再做张量并行 —— 用于**单个 Expert 就很大的情况** |
| **EP × PP** | MoE 层与稠密层在流水线中的分配 |

实例：**DeepSeek-V3 / Mixtral 的并行配置**是多维组合，不是单开 EP。

## 六、什么时候该上 EP

**只有模型用 MoE 架构时才需要。** 它是**架构驱动的并行维度** —— 不像 TP/PP 那样「任何模型都能用」，EP 的前提是模型里有「多个可分布的 Expert」。

反过来说：**MoE 的收益（参数与计算解耦）与代价（All-to-All 通信 + 负载均衡）是一体的** —— 值不值得用 MoE，取决于你的任务是否真的需要「大容量、低激活」。

## 相关

- [[12-FFN 与激活函数]] —— MoE 替换的正是 FFN
- [[03-多卡互联与集群网络]] —— All-to-All 对拓扑带宽的要求
- [[03-集合通信与 NCCL]] —— All-to-All 与 AllReduce 的语义差异
- [[13-3D 并行与混合并行策略]] —— EP 如何并入多维编排
- [[07-ZeRO 显存优化系列]] —— EP 组的参数如何再分片

## 参考

- **AIInfraGuide 第10章 MoE 并行**（MoE 结构、Router Top-K 与容量因子、EP 与 TP 的区别、两次 All-to-All 的分工与通信量、负载均衡的 aux loss 与 aux-loss-free 两类解法、EP×DP/TP/PP 的组合）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%89-%E5%88%86%E5%B8%83%E5%BC%8F%E8%AE%AD%E7%BB%83/%E7%AC%AC10%E7%AB%A0-moe%E5%B9%B6%E8%A1%8C
- **Switch Transformers: Scaling to Trillion Parameter Models with Simple and Efficient Sparsity**：https://arxiv.org/abs/2101.03961
- **Mixtral of Experts**：https://arxiv.org/abs/2401.04088
- **DeepSeek-V3 Technical Report**（aux-loss-free 负载均衡与 EP 配置）：https://arxiv.org/abs/2412.19437
- **DeepEP**（MoE 专用高性能通信库）：https://github.com/deepseek-ai/DeepEP
