---
tags:
  - AI/infra/并行策略
  - AI/infra/显存管理
---

# 数据并行：DP、DDP 与 FSDP

数据并行瞄准的是**训练太慢** —— 数据量太大，单卡一个 epoch 要跑很久。它的做法是「多找几位老师同时批卷，但每人手里都得有一份完整的评分标准」。

**关键约束：每卡都要装得下完整模型。** 所以它只解决「跑不完」，不解决「装不下」—— 这条边界，以及后来 FSDP 如何跨过它，就是本篇的主线。

## 一、数学基础：它是严格等价的

**数据并行不是近似技巧，在同步 SGD 下与单卡训练严格等价。** 理解这点才能明白为什么梯度必须「取平均」。

全局 batch 大小 $B$，损失取样本平均：

$$L = \frac{1}{B}\sum_{i=1}^{B}\ell(x_i;\theta), \qquad g = \nabla_\theta L = \frac{1}{B}\sum_{i=1}^{B}\nabla_\theta \ell(x_i;\theta)$$

把 $B$ 个样本均分给 $N$ 块卡（每卡 $B/N$），第 $k$ 卡的**本地梯度**（内部已做过平均）：

$$g_k = \frac{N}{B}\sum_{i \in \mathcal{D}_k}\nabla_\theta \ell(x_i;\theta)$$

对 $N$ 个本地梯度**取平均**：

$$\bar g = \frac{1}{N}\sum_{k=1}^{N} g_k = \frac{1}{N}\sum_{k=1}^{N}\frac{N}{B}\sum_{i\in\mathcal{D}_k}\nabla_\theta \ell = \frac{1}{B}\sum_{i=1}^{B}\nabla_\theta \ell = g$$

> **$\bar g$ 精确等于单卡在全局 batch 上的梯度。** 所以「$N$ 卡数据并行」与「单卡跑一个 $B$ 大小的 batch」，每一步的参数更新在数学上一模一样。

两个直接推论：

| 推论 | 说明 |
| --- | --- |
| **梯度要 AllReduce 求平均，不是求和** | 本地损失已在 local batch 内除过 $B/N$，跨卡再平均才还原全局平均。**求和等于把学习率放大 $N$ 倍** |
| **有效 batch 变大了** | Effective Batch = 单卡 local batch × $N$（× 梯度累积步数，见 [[10-混合精度与显存优化]]）—— 这也是扩卡通常要调学习率的原因 |

> [!warning] 等价性有三个前提
> **同步梯度**、**各卡 local batch 大小相同**、**同一份初始参数**。任一条被破坏（最后一个 batch 不满、异步更新、参数不一致），等价性就不成立 —— 而这正是下面 DP 与 DDP 的差别所在。

## 二、三代演进

### 第一代：DP（DataParallel）—— 主卡中心化

单进程多线程，一个进程管所有 GPU，反复做「分发—收集」：

```
GPU0（主卡）持有模型与输入
  → Scatter 输入到各卡
  → 各卡复制模型副本 + 前向
  → Gather 输出回主卡
  → 主卡算 loss + 反向
  → 各卡梯度汇总回主卡求和
  → 主卡更新参数，下次迭代再复制
```

**三个致命缺陷**：

| 缺陷 | 原因 |
| --- | --- |
| **GIL 限制** | 单进程多线程，Python 全局解释器锁让多线程无法真正并行调度 |
| **负载不均** | 主卡额外承担 Scatter/Gather、loss 计算、梯度汇总 |
| **通信低效** | 每步都重新复制模型；数据走主卡中转，**容易挤在 PCIe 上而非 NVLink** |

**典型症状是「主卡 OOM，其他卡显存还很空」** —— 因为输出汇总与 loss 计算都堆在主卡。

> **DP 的病根是「单进程 + 主卡中心化」。DDP 的所有改进，本质都是在拆掉这两个前提。**

### 第二代：DDP —— 多进程、去中心化

每块 GPU 一个独立进程，各持有完整模型副本，**彼此地位对等，没有主卡**，梯度同步走去中心化的 AllReduce。

#### Bucket 机制：把通信藏进计算

最朴素的做法是「等整个反向算完，再统一发起一次 AllReduce」—— 但**通信时 GPU 在干等**。

**DDP 的洞察**：反向是**从最后一层往前**逐层算的，**靠后的层梯度先就绪**。既然如此，为什么要等全部算完？

于是 DDP 把梯度按反向计算顺序打包成若干 **Bucket**（默认约 **25 MB**）：

- 某个 Bucket 内所有梯度就绪 → **立即**发起 AllReduce
- 与此同时，更靠前的层继续算反向

**通信与计算就这样重叠起来。**

```python
model = DDP(model, device_ids=[local_rank], bucket_cap_mb=25)
```

`bucket_cap_mb` 是双向权衡：**太小** → 通信次数多、每次的固定开销占比高；**太大** → 要等更久才凑满一桶，重叠效果差。默认值对多数模型够用。

> [!warning] 未参与前向的参数会让 AllReduce 死锁
> 如果模型里有**没被走到**的参数（条件分支），它们的梯度永远不就绪，对应 Bucket 永远等不满，**AllReduce 卡住 → deadlock**。
>
> 解法是 `find_unused_parameters=True`，让 DDP 主动标记未使用参数。**但它有额外开销，能避免则避免。**

#### 超大规模会失效

DDP 在中等规模下几乎线性加速，但**红利在超大规模失效**：

- 卡数增长 → 协调开销与网络需求显著增长，通信逐渐盖不住计算
- **512+ GPU 时通信开始受限于「环延迟（ring latency）」** —— 信号绕 Ring 传播一圈的时间，DP 通信无法再被完全重叠

**到这一点就该转向其他并行维度（TP / PP / ZeRO），而不是继续堆 DP。**

#### DDP 的根本局限

**每卡仍要装下完整的「参数 + 梯度 + 优化器状态」$= 16\Psi$**（见 [[02-分布式训练总论与显存账本]]）。

> **DDP 加卡只能摊薄计算时间，不能摊薄单卡显存。** 7B 模型的 $16\Psi \approx 112$ GB，单张 80 GB 的 H100 直接装不下 —— **无论加多少卡，DDP 都救不了这个数字。**

### 第三代：FSDP —— 把那 $16\Psi$ 也切开

**参数分片，按需 AllGather。**

```
前向：AllGather 拼出当前 unit 的完整参数 → 算完 → 释放
反向：AllGather 再次拼参数 → 算梯度 → ReduceScatter 同步并分片 → 释放
```

#### 四种分片策略：一个「显存 vs 通信」的旋钮

| 策略 | 分片内容 | 显存 | 通信 | 场景 |
| --- | --- | --- | --- | --- |
| **`FULL_SHARD`** | 参数 + 梯度 + 优化器（≈ ZeRO-3） | 最高 | 最高 | 大模型，显存紧张 |
| **`SHARD_GRAD_OP`** | 梯度 + 优化器，**参数不分片**（≈ ZeRO-2） | 中 | 中 | 中等模型，想省通信 |
| **`HYBRID_SHARD`** | **机内 FULL_SHARD + 机间数据并行** | 高 | 机间较低 | **多机大模型首选** |
| `NO_SHARD` | 不分片（等价 DDP） | 无 | 最低 | 调试对照 |

> **`HYBRID_SHARD` 的洞察很值得记**：机内有高带宽 NVLink，适合通信密集的 FULL_SHARD；机间只有较慢的 IB，就退化成通信量小的数据并行。
>
> **把「高频通信」关在机内，「低频通信」才跨机** —— 这与 [[13-3D 并行与混合并行策略]] 的通信域划分原则是同一条判据。

#### FSDP2

PyTorch 正在推进新一代 API `fully_shard`。与 FSDP1 的「整个模块打包分片」不同，**FSDP2 是 per-parameter 分片，底层基于 `DTensor`**：

- 分片粒度到单个参数，避免整块打包带来的显存/通信浪费
- 基于 DTensor，**与 TP / SP 组合时接口统一** —— 这是搭建 2D/3D 并行的基础
- 更清晰的初始化与 checkpoint 语义

## 三、显存账本：三代对比

$$\underbrace{2\Psi}_{\text{FP16 参数}} + \underbrace{2\Psi}_{\text{FP16 梯度}} + \underbrace{4\Psi}_{\text{FP32 master}} + \underbrace{4\Psi + 4\Psi}_{\text{Adam 动量}} = 16\Psi$$

（后三项合称**优化器状态**，共 $12\Psi$。）$N$ 卡时：

| 方案 | 参数 | 梯度 | 优化器状态 | 单卡合计 | $N=8$，7B |
| --- | --- | --- | --- | --- | --- |
| **DDP** | $2\Psi$ | $2\Psi$ | $12\Psi$ | $16\Psi$ | **约 112 GB** |
| SHARD_GRAD_OP（ZeRO-2） | $2\Psi$ | $\frac{2\Psi}{N}$ | $\frac{12\Psi}{N}$ | $2\Psi + \frac{14\Psi}{N}$ | 约 26 GB |
| **FULL_SHARD**（ZeRO-3） | $\frac{2\Psi}{N}$ | $\frac{2\Psi}{N}$ | $\frac{12\Psi}{N}$ | $\frac{16\Psi}{N}$ | **约 14 GB** |

三条结论：

1. **DDP 的显存与卡数无关** —— 加卡不减负
2. **优化器状态是最大头**（$12\Psi$，占 75%）→ **ZeRO-1/2 只切它就能省掉大半**
3. **FULL_SHARD 随卡数线性下降** —— 理论上能训任意大的模型（只要卡够多）

> [!warning] 这张表不算激活值
> 激活值随 batch / 序列长度 / 深度增长，**长序列训练时往往才是显存大头**。FSDP 分的是模型状态，**对激活值无能为力** —— 那要靠激活重计算、SP / CP 等手段（见 [[10-混合精度与显存优化]]、[[11-长序列训练与上下文并行]]）。
>
> **选型时先估模型状态，再给激活与碎片留 1.2–1.5 倍余量。**

## 四、通信量：DDP $2\Psi$ vs FSDP $3\Psi$

FSDP 用**更多通信**换**更少显存**。

| | 每步单卡通信量 | 拆解 |
| --- | --- | --- |
| **DDP** | **$2\Psi$** | 一次 AllReduce |
| **FSDP** | **$3\Psi$** | 前向 AllGather $\Psi$ + 反向 AllGather $\Psi$ + 反向 ReduceScatter $\Psi$ |

**为什么 AllReduce 是 $2\Psi$ 而 AllGather / ReduceScatter 各只是 $\Psi$？**

因为 `AllReduce = ReduceScatter + AllGather`（见 [[03-集合通信与 NCCL]]）—— 它本身就是两个 $\Psi$ 操作的组合。**FSDP 把这两半拆开用，再额外多一次前向 AllGather，所以是 $3\Psi$。**

| 维度 | DDP | FSDP（FULL_SHARD） |
| --- | --- | --- |
| 核心思路 | 每卡完整模型，梯度 AllReduce | 参数分片，按需 AllGather |
| 每卡通信量/步 | $2\Psi$ | $3\Psi$（**多 50%**） |
| 单卡显存 | $16\Psi$ | $\frac{16\Psi}{N}$ |
| 重叠手段 | Bucket 机制 | **prefetch 预取下一层参数** |

> **DDP 与 FSDP 是一组清晰的权衡对偶**：DDP 省通信费显存，FSDP 省显存费通信。**没有免费午餐 —— 选哪个取决于你的瓶颈是显存还是通信。**
>
> 而且 $3\Psi$ 的额外通信能否被掩盖，**很依赖网络带宽与 prefetch 效果**：高带宽机内（NVLink / NVSwitch）基本能重叠掉；低带宽跨机场景则会暴露成瓶颈 —— **这正是 `HYBRID_SHARD` 存在的意义**。

## 五、选型

**问题只有一个：参数 + 梯度 + 优化器状态 + 激活值，单卡装得下吗？**

```
装得下 → DDP（最简单、通信最省）
装不下 → 先试 SHARD_GRAD_OP（通信更省）
         还不够 → FULL_SHARD
         多机跨节点 → HYBRID_SHARD
```

**判据是「装不下的到底是哪一项」**（完整分支见 [[10-混合精度与显存优化]] 的全景表）：

| 装不下的是 | 该上 |
| --- | --- |
| 优化器状态 | ZeRO-1 |
| 梯度 | ZeRO-2 / `SHARD_GRAD_OP` |
| 参数 | ZeRO-3 / `FULL_SHARD` / TP |
| 激活值 | 激活重计算 / SP / CP |

## 相关

- [[02-分布式训练总论与显存账本]] —— $16\Psi$ 账本与五大策略全景
- [[07-ZeRO 显存优化系列]] —— FSDP 四策略与 ZeRO 阶段的对应
- [[03-集合通信与 NCCL]] —— AllReduce / AllGather / ReduceScatter 的通信量
- [[03-多卡互联与集群网络]] —— `HYBRID_SHARD` 的带宽依据
- `01-PyTorch 框架与训练循环` —— `no_sync()` 与梯度的累加语义
- [[13-3D 并行与混合并行策略]] —— DP 在混合并行里的位置

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%89-%E5%88%86%E5%B8%83%E5%BC%8F%E8%AE%AD%E7%BB%83/41-%E6%95%B0%E6%8D%AE%E5%B9%B6%E8%A1%8C%E8%AF%A6%E8%A7%A3
- https://pytorch.org/docs/stable/notes/ddp.html
- https://pytorch.org/docs/stable/fsdp.html
- https://arxiv.org/abs/1910.02054
- https://pytorch.org/docs/stable/distributed.fsdp.fully_shard.html
