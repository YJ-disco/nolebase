---
tags:
  - AI/infra
---

# GPU 与加速器专栏导览

本目录收**模型跑在上面的那块硬件**：芯片内部怎么组织、存储层次有多深、代际怎么演进、多卡之间怎么连。

它与相邻专栏的边界是**关注点而不是目录层级**：

| 不收 | 去哪看 | 本专栏只关心 |
| --- | --- | --- |
| 怎么写 kernel —— 编程模型、访存优化、经典算子 | [[00-CUDA 与算子优化专栏导览\|CUDA 与算子优化]] | 硬件提供了哪些可被 kernel 利用的能力 |
| 多块硬件怎么组织训练 —— 框架、并行策略、通信、显存优化 | [[00-训练专栏导览\|训练]] | 卡与卡之间的物理链路与带宽 |
| 模型怎么跑起来对外服务 —— 调度、显存管理、推理引擎 | [[00-AI Infra 专栏导览\|AI Infra]] | 这些策略受哪条硬件上限约束 |

换个说法：`cuda/` 回答“怎么把算子写到接近上限”，`train/` 回答“多块卡怎么组织起来”，本专栏回答“**这些上限各自是多少、为什么是这么多**”。

**与推理侧的呼应**：本目录讲的是「这块卡有多少算力、多少带宽」，[[01-推理性能指标与瓶颈定位]] 讲的是「这两条上限如何决定推理快慢」。Roofline 的平衡点在两处都出现 —— 这里算的是硬件的固有比值，那里用的是同一个比值来判断某类任务会不会撞墙。

## 阅读顺序

| 篇 | 回答什么 |
| --- | --- |
| [[01-GPU 硬件架构与存储层次]] | 为什么深度学习选 GPU？SM 和 Warp 是什么？显存带宽为什么比算力更常成为瓶颈？ |
| [[02-NVIDIA GPU 架构演进：Volta 到 Blackwell]] | 从 2017 到 2024 五代架构，每一代解决了什么问题、引入了什么精度格式与互联带宽 |
| [[03-多卡互联与集群网络]] | NVLink / NVSwitch / InfiniBand / RoCE 各自的带宽量级，以及为什么它们决定了并行策略的形状 |

**02 与 03 是两条正交的线**：02 讲「单卡一代比一代强多少」，03 讲「多卡之间搬数据有多快」。后者对分布式训练的影响往往更大 —— 一块卡的算力提升是线性的，而互联带宽不够会让并行扩展效率断崖式下跌。

**一条反复出现的判据**：

本目录的每一篇最终都指向同一个结论：

> **当模型足够大时，瓶颈不在计算而在访存。** 这句话有两层含义 —— 单卡内的 HBM 带宽，以及跨卡的互联带宽。前者决定了算子的算术强度够不够（见 [[01-GPU 硬件架构与存储层次]]），后者决定了并行维度该放在机内还是跨机（见 [[03-多卡互联与集群网络]]）。

它也是后续所有优化技术（FlashAttention、Kernel Fusion、ZeRO、张量并行）的共同动机。

## 相关

- [[01-推理性能指标与瓶颈定位]] —— 算力与带宽这两条上限如何决定推理快慢
- [[02-PagedAttention：KV Cache 的分页管理]] —— 显存容量与带宽如何塑造推理引擎的设计

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/gpu/gpu-basics
- https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper
- https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf
- https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/
- https://www.nvidia.com/en-us/data-center/nvlink/
