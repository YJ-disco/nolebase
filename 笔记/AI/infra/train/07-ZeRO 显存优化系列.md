---
tags:
  - AI/infra/显存管理
  - AI/infra/并行策略
---

# ZeRO 显存优化系列

[[02-分布式训练总论与显存账本]] 算过一笔账：BF16 + Adam 下每参数静态显存约 $16\Psi$，7B 模型就要 112 GB。**ZeRO（Zero Redundancy Optimizer）是 DeepSpeed 的核心技术，它从「冗余在哪里」这个问题出发，逐阶段切掉这些冗余。**

## 一、冗余分析

DDP 里**每张卡都冗余存储完整的三份东西**：

| 组成 | 每卡存储 |
| --- | --- |
| 优化器状态 | $12\Psi$ |
| 梯度 | $2\Psi$ |
| 参数 | $2\Psi$ |

$N$ 卡集群里，$N$ 份完全相同的副本 —— **其中 $(N-1)/N$ 的存储是纯粹浪费的**。

> **设计思想是「切分-聚合」范式**：平时每卡只存 $1/N$，需要时通过通信获取完整数据，用完即弃。
>
> 三阶段的顺序**从最容易切的开始**（优化器状态），到最难切的（参数）—— 因为越往前切，通信代价越小。

## 二、三个阶段

### ZeRO-1：只切优化器状态

**切什么**：Adam 的 FP32 参数副本 + 一阶动量 + 二阶动量，共 $12\Psi$。

**通信**：梯度**仍需 AllReduce**（与 DDP 相同），参数更新后需要 AllGather 同步参数。

$$\text{每卡显存}：16\Psi \;\longrightarrow\; 4\Psi + \frac{12\Psi}{N}$$

**这一刀最划算的原因**：Adam 状态占了 $16\Psi$ 里的 12（见 [[02-分布式训练总论与显存账本]]），**切它等于切掉了大头，而通信只增加一次参数 AllGather**。

### ZeRO-2：再切梯度

**切什么**：在 ZeRO-1 基础上，梯度也按分片存储。

**通信模式的关键变化**：反向传播时用 **ReduceScatter 替代 AllReduce** —— 每卡只保留自己负责分片的聚合梯度。

$$\text{每卡显存}：16\Psi \;\longrightarrow\; 2\Psi + \frac{14\Psi}{N}$$

$$\text{通信量}：\text{ReduceScatter} = \Psi \quad(\text{AllReduce 的 } 2\Psi \text{ 的一半})$$

> **注意这里有个容易漏的点**：ReduceScatter 的 $\Psi$ 比 AllReduce 的 $2\Psi$「少一半」，但 AllReduce 隐含的那次 AllGather 并没有消失 —— 只是被推迟到了参数更新时。**通信量的对比要按完整一步算，不能只比单次调用。**

### ZeRO-3：连参数也切

**切什么**：参数、梯度、优化器状态全部切分。

**通信模式**：

```
前向：AllGather 重组当前层参数 → 计算 → 释放
反向：AllGather 参数 → 计算梯度 → ReduceScatter 梯度 → 释放参数
```

$$\text{每卡显存}：16\Psi \;\longrightarrow\; \frac{16\Psi}{N} \quad(\text{理想线性缩放})$$

$$\text{通信量}：3\Psi \quad(\underbrace{\Psi}_{\text{前向 AllGather}} + \underbrace{\Psi}_{\text{反向 AllGather}} + \underbrace{\Psi}_{\text{ReduceScatter}})，比 DDP 多 50\%$$

> **ZeRO-3 与前两个阶段有性质上的差别**：ZeRO-1/2 是「省显存，通信代价小」，ZeRO-3 是**用 50% 的额外通信换线性显存缩放**。它只在「参数本身单卡装不下」时才值得 —— 这是选型的分水岭。

## 三、Offload：把状态搬到 CPU 和 SSD

| 方案 | 做法 | 适用 | 代价 |
| --- | --- | --- | --- |
| **ZeRO-Offload** | 优化器状态与梯度计算卸载到 CPU，GPU 只做前向/反向 | 少卡（1–4 卡）训大模型 | **PCIe 带宽成瓶颈**（PCIe 4.0 约 32 GB/s） |
| **ZeRO-Infinity** | 在 Offload 基础上进一步利用 **NVMe SSD** | 万亿参数在有限 GPU 上训练 | I/O 带宽；靠分块预取（prefetch）与计算-I/O 重叠掩盖 |

> **Offload 的本质是「拿一条慢得多的链路换容量」**：PCIe 4.0 的 32 GB/s 对比 NVLink 4.0 的 900 GB/s（见 [[03-多卡互联与集群网络]]）—— 差 28 倍。所以它只适合「少卡、无别的选择」的场景，不是通用优化。

## 四、选型

| 阶段 | 省什么 | 通信代价 | 什么时候用 |
| --- | --- | --- | --- |
| ZeRO-1 | 优化器状态（$\frac{12\Psi}{N}$） | 与 DDP 接近 | 参数 + 梯度单卡装得下 |
| **ZeRO-2** | + 梯度 | 略高于 DDP | **参数单卡装得下** |
| ZeRO-3 | 全部（$\frac{16\Psi}{N}$） | 比 DDP 多 50% | 参数也装不下 |
| Offload | 用 CPU 内存 | + PCIe 传输 | 少卡大模型 |

**判据是「装不下的到底是哪一部分」** —— 优化器状态装不下用 1，梯度也装不下用 2，参数都装不下才用 3。

## 五、ZeRO 与 FSDP 的对应

| ZeRO | PyTorch FSDP |
| --- | --- |
| ZeRO-2 | `SHARD_GRAD_OP` |
| **ZeRO-3** | **`FULL_SHARD`** |

三者的分工：

- **ZeRO 是算法/论文层面的概念**（Microsoft, `arXiv:1910.02054`）
- **FSDP 是 PyTorch 原生实现**
- **DeepSpeed 是微软的独立实现**

> **怎么选**：PyTorch 生态内**优先 FSDP**（原生、与 `torch.compile` / `torch.distributed` 集成好）；需要 Offload / Infinity，或已有 DeepSpeed 配置时用 DeepSpeed。

## 相关

- [[02-分布式训练总论与显存账本]] —— $16\Psi$ 的逐项账本与五大策略全景
- [[06-数据并行：DP、DDP 与 FSDP]] —— DDP 是 ZeRO 的对照基线
- [[03-集合通信与 NCCL]] —— ReduceScatter 与 AllGather 的通信量
- [[03-多卡互联与集群网络]] —— PCIe 与 NVLink 的带宽落差
- [[10-混合精度与显存优化]] —— 正交的单卡显存手段

## 参考

- **AIInfraGuide 第5章 ZeRO 显存优化系列**（冗余分析、三阶段的切分对象与显存/通信公式、Offload 与 Infinity、选型表、与 FSDP 的对应）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%89-%E5%88%86%E5%B8%83%E5%BC%8F%E8%AE%AD%E7%BB%83/%E7%AC%AC5%E7%AB%A0-zero%E7%B3%BB%E5%88%97
- **ZeRO: Memory Optimizations Toward Training Trillion Parameter Models**：https://arxiv.org/abs/1910.02054
- **ZeRO-Offload: Democratizing Billion-Scale Model Training**：https://arxiv.org/abs/2101.06840
- **ZeRO-Infinity: Breaking the GPU Memory Wall for Extreme Scale Deep Learning**：https://arxiv.org/abs/2104.07857
- **PyTorch FSDP**：https://pytorch.org/docs/stable/fsdp.html
