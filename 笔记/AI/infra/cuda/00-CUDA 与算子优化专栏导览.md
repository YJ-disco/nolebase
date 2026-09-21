---
tags:
  - AI/infra
---

# CUDA 与算子优化专栏导览

本目录收**在 GPU 上写 kernel** 的那一层：线程怎么组织、访存怎么优化、经典算子怎么从朴素实现一步步逼近硬件上限。

它在 `AI/infra/` 这棵子树里的位置：

| 目录 | 关注什么 |
| --- | --- |
| `AI/infra/gpu/` | 硬件本身 —— SM、存储层次、算力与带宽、互联 |
| **`AI/infra/cuda/`**（本目录） | 在硬件上写 kernel —— 编程模型、访存优化、经典算子 |
| `AI/infra/train/` | 用多块硬件训练 —— 框架、并行策略、通信、显存 |
| `AI/infra/`（根） | 把模型服务出去 —— 调度、显存管理、推理引擎、数值精度 |

四块是一条链：**硬件 → 写算子 → 训练 → 服务**。本目录是链条第二环。

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

## 待建

| 概念 | 说明 |
| --- | --- |
| **AI 编译器** | Triton 的 Block-level 编程模型、torch.compile 的 Dynamo + Inductor 与 Graph Break 问题、TVM / XLA 的定位差异 |
| **性能分析工具链** | Nsight Systems 的 CPU-GPU 全链路 trace 与 GPU idle gap 定位、Nsight Compute 的 SOL 面板与 Kernel 对比分析 |
| **PagedAttention 的 CUDA 实现** | 虚拟页到物理页的映射在 GPU 上怎么落地 |
| **FlashAttention-3 与 Decode 侧优化** | Hopper 的 `wgmma` / TMA / FP8 异步流水线；Flash-Decoding 系列面向小 batch 长序列的并行策略 |

前两项在源教程里**只有章节大纲、没有正文**（已在源站侧确认）；后两项是 V1/V2 之后的演进，源教程未覆盖。torch.compile 的部分内容已在 [[05-Attention 后端与图优化]]。

## 相关

- [[01-GPU 硬件架构与存储层次]] —— 本目录所有优化的硬件约束来自这里
- [[05-Attention 后端与图优化]] —— kernel 之上的那一层：可插拔后端与 CUDA Graph
- [[11-Self-Attention 机制]] —— 算子服务的模型结构
- [[01-数值计算与精度]] —— 归约顺序、低精度累加与数值稳定

## 参考

- **AIInfraGuide 模块二（CUDA 编程与算子优化）**：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/12-cuda%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B
- **CUDA C++ Programming Guide**：https://docs.nvidia.com/cuda/cuda-c-programming-guide/
- **CUDA C++ Best Practices Guide**：https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- **NVIDIA Nsight Compute**：https://docs.nvidia.com/nsight-compute/
- **FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness**：https://arxiv.org/abs/2205.14135
