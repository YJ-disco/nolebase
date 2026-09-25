---
tags:
  - AI/infra
---

# CUDA 与算子优化专栏导览

本目录收**在 GPU 上写 kernel** 的那一层：线程怎么组织、访存怎么优化、经典算子怎么从朴素实现一步步逼近硬件上限。

它与相邻专栏的边界是**关注点而不是目录层级**：

| 不收 | 去哪看 | 本专栏只关心 |
| --- | --- | --- |
| 硬件本身 —— SM 微架构、存储层次、算力与带宽、互联拓扑 | [[00-GPU 与加速器专栏导览\|GPU 与加速器]] | 这些硬件能力如何约束 kernel 的写法 |
| 用多块硬件训练 —— 框架、并行策略、通信、显存账本 | [[00-训练专栏导览\|训练]] | 单个 kernel 内部的优化 |
| 把模型服务出去 —— 调度、显存管理、推理引擎、数值精度 | [[00-AI Infra 专栏导览\|AI Infra]] | 算子本身怎么逼近硬件上限 |

换个说法：`gpu/` 回答“这块卡有多少算力、多少带宽”，`train/` 回答“多块卡怎么组织起来训练”，本专栏回答“**怎么让一个 kernel 跑满这些能力**”。

## 阅读顺序

| 篇 | 回答什么 |
| --- | --- |
| [[01-CUDA 开发环境与第一个 Kernel]] | 驱动 / Toolkit / nvcc 三者的关系，以及一个 kernel 从 CPU 到 GPU 的五步 |
| [[02-CUDA 编程模型与执行模型]] | Grid / Block / Thread 三级层次，Warp 与 SIMT，Divergence 的代价，Warp Shuffle |
| [[03-CUDA 内存模型与访存优化]] | 内存层次、合并访问、Bank Conflict 与向量化 —— 两个最大的性能杠杆 |
| [[04-Occupancy、同步与原子操作]] | 一个 SM 能塞多少 Block，线程怎么协调，竞争同一地址的代价 |
| [[05-Reduce 算子优化]] | 三个瓶颈、八个版本的完整优化循环，从 15% 到 85% 带宽利用率 |
| [[06-GEMM 性能优化]] | 八个台阶从 6% 爬到 99% cuBLAS —— 深度学习最核心算子的完整优化阶梯 |
| [[07-Softmax 与 Online Softmax]] | 把三遍扫描压成两遍的递推公式 —— FlashAttention 的数学前提 |
| [[08-FlashAttention]] | 不减 FLOPs 只减访存：IO 复杂度从 $O(N^2)$ 降到 $O(N^2d^2/M)$，以及 V2 的三个改进 |

**01 → 04 是基础**（怎么写、怎么调度、怎么搬数据、怎么管资源），**05 → 08 是四个经典算子**。基础四篇讲的所有机制，都会在这四个算子里各以不同形式出现一遍：

| 算子 | 主要修的瓶颈 | 关键手段 |
| --- | --- | --- |
| Reduce | Warp Divergence、Bank Conflict、同步开销、访存效率 | 步长反转、Warp Shuffle、`float4` |
| GEMM | 数据复用、访存延迟 | Tiling、寄存器外积、双缓冲、`cp.async` |
| Softmax | 扫描次数 | Online 递推、寄存器缓存 |
| FlashAttention | HBM 读写量 | Tiling + Online Softmax、Causal 块跳过 |

**两条贯穿全目录的判据**：

> **一、绝大多数 kernel 的瓶颈不在计算而在访存。** 优化的主线是**让同一份数据被算更多次**（提升算术强度）。Reduce 靠共享内存与 Shuffle 减少全局访存，GEMM 靠 tiling 把复用提到寄存器，Softmax 靠 Online 算法少扫一遍，FlashAttention 靠分块把 $O(N^2)$ 的 HBM 读写降到 $O(N^2d^2/M)$。硬件判据见 [[01-GPU 硬件架构与存储层次]]。
>
> **二、生产环境用库，手写 kernel 的价值在于理解瓶颈。** CUB 的 `DeviceReduce` 稳定在 90%+ 带宽利用率，cuBLAS / CUTLASS 在大矩阵上到 90%+ 峰值，FlashAttention 是 Attention 的既定答案。**手写的意义是「读 profiler 时知道该往哪看」**，不是替代它们。

## 相关

- [[01-GPU 硬件架构与存储层次]] —— 本目录所有优化的硬件约束来自这里
- [[05-Attention 后端与图优化]] —— kernel 之上的那一层：可插拔后端与 CUDA Graph
- [[11-Self-Attention 机制]] —— 算子服务的模型结构
- [[01-数值计算与精度]] —— 归约顺序、低精度累加与数值稳定

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/12-cuda%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B
- https://docs.nvidia.com/cuda/cuda-c-programming-guide/
- https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- https://docs.nvidia.com/nsight-compute/
- https://arxiv.org/abs/2205.14135
