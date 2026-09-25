---
tags:
  - AI/infra/架构
  - AI/infra/存储层次
---

# GPU 硬件架构与存储层次

深度学习选 GPU，原因在于它的设计目标和深度学习的计算特征刚好对上。理解这件事是后面所有优化的起点 —— 你优化的每一个 kernel、设计的每一种并行策略，最终都受这块硬件的算力与带宽两条上限约束。

## 一、CPU 与 GPU 的设计哲学

| 维度 | CPU | GPU |
| --- | --- | --- |
| 设计目标 | **低延迟**处理复杂任务 | **高吞吐**处理大量简单任务 |
| 核心数量 | 几个到几十个「大核」 | 数千到上万个「小核」 |
| 单核能力 | 强（复杂分支预测、乱序执行） | 弱（简单 ALU，按序执行） |
| 缓存占比 | 芯片面积的大部分 | 芯片面积的小部分 |
| 控制逻辑 | 复杂（乱序执行、分支预测器） | 简单（大量核心共享控制单元） |
| 内存带宽 | 较低（DDR5 约 100 GB/s） | 极高（HBM3 3.35 TB/s） |
| 适合任务 | 串行逻辑、操作系统、网络服务 | 矩阵运算、数据并行、图形渲染 |

差距的量级值得记一下：**H100 SXM 的 FP16 Tensor 算力约 989 TFLOPS，高端服务器 CPU（如 Intel Xeon w9-3595X）的 FP16 算力在个位数 TFLOPS 量级** —— 两个数量级以上。

### 深度学习为什么天然适合 GPU

拆到最底层，深度学习的计算就是海量矩阵乘法与逐元素运算：

- **前向**：每一层都是 $Y = XW + b$
- **反向**：$\frac{\partial L}{\partial W} = X^\top \frac{\partial L}{\partial Y}$，依然是矩阵乘
- **Attention**：$\text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V$，核心还是矩阵乘

这些运算有两个关键特征，恰好是 GPU 的长项：**数据并行度极高**（矩阵元素间相互独立，天然可分配）与**计算模式规则**（无需复杂分支，所有线程执行相同指令）。

> **反过来说**：一旦你的计算不满足这两条（大量分支、串行依赖、不规则访存），GPU 的优势就会迅速消失。这是后面很多「为什么这个算子优化不动」的根源。

## 二、芯片内部的组织层次

以 NVIDIA GPU 为例，从外到内：

```
GPU 芯片
├── GPC (Graphics Processing Cluster)
│   ├── TPC (Texture Processing Cluster)
│   │   ├── SM (Streaming Multiprocessor)     ← GPU 的基本调度单元
│   │   │   ├── CUDA Core × N                 ← FP32/INT32 运算
│   │   │   ├── Tensor Core × M               ← 矩阵乘累加
│   │   │   ├── SFU (Special Function Unit)   ← sin / cos / exp 等
│   │   │   ├── Register File
│   │   │   ├── Shared Memory
│   │   │   └── L1 Cache
│   │   └── ...
│   └── ...
├── GPC ...
├── L2 Cache（全卡共享）
└── Memory Controller → HBM
```

**SM 是资源分配的最小粒度。** 写 CUDA kernel 时，一个 Thread Block 会被调度到一个 SM 上执行；SM 内的寄存器、共享内存等资源由落在这个 SM 上的所有 Thread Block 共同分配。这条约束的意义会在占用率（Occupancy）那类问题里反复出现。

以 H100 的 SM 为例：

| 组件 | 数量 | 功能 |
| --- | --- | --- |
| CUDA Core（FP32） | 128 | 浮点与整数运算 |
| Tensor Core（第四代） | 4 | 矩阵乘累加（MMA） |
| Load/Store Unit | 32 | 内存读写 |
| SFU | 16 | 超越函数 |
| Warp Scheduler | 4 | 每周期各调度 1 个 Warp |
| Register File | 256 KB | 线程私有寄存器 |
| Shared Memory / L1 | 228 KB（可配置划分） | SM 内共享的高速存储 |

## 三、Warp：执行的最小单位

GPU 的执行单位是 **Warp** —— **32 个线程锁步执行同一条指令**。这个模式叫 **SIMT**（Single Instruction, Multiple Threads）。

```
Warp（32 个线程）
├── Thread 0 :  add r1, r2, r3
├── Thread 1 :  add r1, r2, r3   ← 同一条指令，不同数据
├── ...
└── Thread 31:  add r1, r2, r3
```

> [!warning] Warp Divergence 是性能杀手
> Warp 内线程遇到分支（`if`/`else`）时，走不同分支的线程会被**掩码（mask）**，两条路径串行执行，未命中路径的线程空转。分支的粒度如果比 warp 还细，等于把 32 路并行退化成串行。
>
> 这是「为什么这个算子写出来比别人慢一截」的最常见原因之一，也是后面贯穿的优化主题。

## 四、存储层次：Memory is the new compute

GPU 的存储层次与 CPU 类似，越靠近计算核心越快、越小：

| 层级 | 容量 | 延迟量级 | 作用域 |
| --- | --- | --- | --- |
| Register | 每线程最多 255 个 | 约 0 cycle | 单线程私有 |
| Shared Memory | 每 SM 约 228 KB | 约 30 cycle | SM 内所有线程 |
| L1 Cache | 与 Shared Memory 共享物理存储 | 约 30 cycle | SM 内 |
| L2 Cache | 全卡约 50 MB（H100） | 约 200 cycle | 全局共享 |
| HBM | 80 GB | 约 600 cycle | 全局 |

**重要的一档落差**：寄存器到 HBM 的延迟差了两个数量级。这就是为什么 kernel 优化的核心动作几乎都是「把数据往上搬一层、并让搬上来的数据被复用尽可能多次」。

### HBM 与 GDDR

| 类型 | 代表 GPU | 带宽 | 容量 |
| --- | --- | --- | --- |
| GDDR6X | RTX 4090 | 1,008 GB/s | 24 GB |
| HBM2e | A100 80GB SXM | 2,039 GB/s | 80 GB |
| HBM3 | H100 SXM | 3,350 GB/s | 80 GB |
| HBM3e | B200 | 8,000 GB/s | 192 GB |

HBM（High Bandwidth Memory）把多层 DRAM 堆叠起来，通过硅中介层（Silicon Interposer）与 GPU 芯片直连，用极宽的位宽换带宽。从 A100 到 B200 带宽涨了近 4 倍 —— **这对推理意义重大**：Decode 阶段是带宽受限的（见 [[01-推理性能指标与瓶颈定位]]），带宽每翻一倍，理论上限就接近翻倍。

### Roofline：算力与带宽的平衡点

判断一个算子受**算力**还是受**带宽**限制，用**算术强度**：

$$\text{算术强度} = \frac{\text{浮点运算量 (FLOPs)}}{\text{访存字节数 (Bytes)}}$$

GPU 有一个固有的 **ops:byte 比** = 峰值算力 ÷ 显存带宽。算术强度低于它 → Memory Bound；高于它 → Compute Bound。以 H100 SXM 为例：

$$\frac{989 \times 10^{12}\ \text{FLOP/s}}{3.35 \times 10^{12}\ \text{Byte/s}} \approx 295\ \text{FLOP/Byte}$$

**每从显存读 1 Byte，需要执行至少 295 次 FP16 浮点运算才能把算力喂饱**，否则 GPU 就是在等数据。

> 大多数深度学习算子的算术强度都远低于 295 —— 尤其是 Attention、LayerNorm、激活函数这类「读得多、算得少」的操作。**FlashAttention 与 Kernel Fusion 的全部意义就是提升算术强度：让同一份数据被算更多次，而不是单纯把计算变快。**

平衡点这条判据在 [[01-推理性能指标与瓶颈定位]] 里被用来解释「为什么单请求 Decode 的算术强度只有约 1 FLOP/Byte」，那里有完整推导。

## 五、Tensor Core

CUDA Core 是通用计算核心，一个时钟周期执行一次 FMA（fused multiply-add）。**Tensor Core 专为矩阵乘累加（MMA）设计**，一个指令完成一个小矩阵块的乘加。

### MMA 的形状随代际变化

这是容易记混的一处。每一代的 MMA 指令形状不同：

| 架构 | 指令 | 形状 | 说明 |
| --- | --- | --- | --- |
| Volta | `mma`（4×4×4） | m8n8k4 | 每 Tensor Core 每周期 64 次 FMA |
| Ampere | `mma.sync.aligned` | **m16n8k16** | 以 **warp** 为单位执行，操作数走 `ld.shared` → 寄存器 → Tensor Core → 寄存器 |
| Hopper | `wgmma` | **m64nNk16**，N ∈ {64, 128, 256} | 由 **warpgroup（4 warp / 128 线程）** 协同执行；A/B 可直接从 shared memory 进 Tensor Core，不必先过寄存器 |
| Blackwell | `tcgen05.mma` | 最大单 CTA 原子 **m128n256k16** | 累加器落在专用的 **TMEM**，而非线程寄存器；由单线程发射 |

一次 MMA 的乘加次数可以直接算：`m16n8k16` 是 2,048 次，`m64n128k16` 是 131,072 次。

**Hopper 那一行的两个变化值得单独记**：

1. **操作数不再必须过寄存器** —— `wgmma` 允许 A/B 从 shared memory 直接流入 Tensor Core，把寄存器压力降下来，同时省掉一次 `ld.shared`。
2. **warpgroup 内建同步** —— 传统 `mma` 要求 4 个 warp 各自算完部分积后再通过 shared memory 归约；`wgmma` 由硬件保证 128 个线程的累加结果一致。

代价是**调度粒度变粗**：`wgmma` 的最小 tile 是 64 行，如果实际矩阵远小于这个尺寸（例如推理场景 batch=1），打包与 padding 的开销会吃掉收益。

> [!warning] 一处与来源不一致的说法
> 来源教程写「以 Hopper 架构为例，单个 Tensor Core 一个时钟周期可以完成 `16×8×16` 的 FP16 矩阵乘累加」。`m16n8k16` 是 **Ampere** 的 `mma` 形状；Hopper 的对应指令是 `wgmma.m64nNk16`，且**以 warpgroup 为执行单位**，不是「单个 Tensor Core 一个周期」。上表按 NVIDIA PTX 文档口径写。
>
> 依据：NVIDIA PTX ISA（`wgmma` 与 `mma` 指令形状）、NVIDIA Hopper 架构博客的逐 SM 规格表。

#### 各代单 SM 的 Tensor 吞吐（可自行验算）

Tensor Core 的每周期吞吐逐代翻倍，拿它乘 SM 数和频率就能还原出厂商公布的算力：

| 架构 | 每 Tensor Core 每周期 FMA | 每 SM Tensor Core 数 | 每 SM 每周期 FP16 FLOPs |
| --- | --- | --- | --- |
| Volta | 64 | 8 | 1,024 |
| Ampere | — | 4 | 2,048 |
| Hopper | — | 4 | 4,096 |
| Blackwell | — | 4 | 8,192 |

用 Hopper 验算：`132 SM × 4,096 FLOP/cycle × 1.830 GHz ≈ 989 TFLOPS`，与厂商公布的 989.4 TFLOPS 吻合。

### 精度格式

| 精度 | 位宽 | 指数位 | 尾数位 | 动态范围 | 首次支持 |
| --- | --- | --- | --- | --- | --- |
| FP32 | 32 | 8 | 23 | 高 | — |
| TF32 | 19 | 8 | 10 | 同 FP32 | Ampere |
| FP16 | 16 | 5 | 10 | 窄 | Volta |
| BF16 | 16 | 8 | 7 | 同 FP32 | Ampere |
| FP8 E4M3 | 8 | 4 | 3 | 小 | Hopper |
| FP8 E5M2 | 8 | 5 | 2 | 中 | Hopper |
| INT8 | 8 | 整数 | — | — | Turing |
| FP4 | 4 | — | — | 极窄 | Blackwell |

两条判据：

- **精度与动态范围是两件事。** FP16 尾数比 BF16 多 3 位（同量级下表示更细），但指数少 3 位，最大有限值只有 65,504 —— 所以 FP16 常需 Loss Scaling，BF16 通常不需要。细节见 [[01-数值计算与精度]]。
- **BF16 是大模型训练的默认。** 它与 FP32 共享 8 位指数，动态范围一致，因此训练更不易溢出，同时把显存与通信量减半。现代大模型训练几乎都用 BF16 而非 FP16。

### 算力倍增与对齐要求

以 H100 SXM 为例，Tensor Core 相对 CUDA Core 的加速比：

| 精度 | CUDA Core | Tensor Core | 加速比 |
| --- | --- | --- | --- |
| FP32 | 67 TFLOPS | 495 TFLOPS（TF32） | 约 7.4× |
| FP16 | 134 TFLOPS | 989 TFLOPS | 约 7.4× |
| FP8 | — | 1,979 TFLOPS | — |

> [!warning] 对齐要求会让算力打折
> 要触发 Tensor Core，矩阵维度需要满足对齐要求（通常 8 或 16 的倍数）。**如果模型的 hidden_size 不是 16 的倍数，Tensor Core 可能无法被充分利用** —— 这类问题在自定义模型或小模型上很常见，表现是「明明没做错什么，MFU 就是上不去」。

## 六、评估一块卡的四个指标

| 指标 | 决定什么 |
| --- | --- |
| **算力**（TFLOPS） | Compute-bound 任务的上限 |
| **显存带宽**（GB/s） | Memory-bound 任务的上限 |
| **显存容量**（GB） | 能装下多大的模型与多长的上下文 |
| **互联带宽**（GB/s） | 多卡并行的扩展效率（见 [[03-多卡互联与集群网络]]） |

CUDA Core 的理论峰值算力可以手算：

$$\text{FP16 算力} = \text{SM 数} \times \text{每 SM 的 FP16 Core 数} \times 2 \times \text{时钟频率}$$

系数 2 来自 FMA 含一次乘法和一次加法。Tensor Core 的算法不同 —— 要看每周期完成的 MMA 规模（见上一节的验算）。

> [!warning] **厂商标称的 TFLOPS 通常带稀疏（Sparsity）**。NVIDIA 从 Ampere 起支持 2:4 结构化稀疏，开启后吞吐翻倍，但**需要权重经过专门剪枝**，绝大多数实际负载用的是 Dense 算力。做性能分析时应以实测为准，有效算力通常是标称 Dense 的 30%–60%。

## 七、训练显存账本

显存往往是最先撞到的瓶颈。以 Adam + FP16 混合精度为例，**每个参数**的显存开销：

| 组成部分 | 每参数字节 | 说明 |
| --- | --- | --- |
| FP16 参数 | 2 B | 前向反向使用 |
| FP32 参数副本（master weights） | 4 B | Adam 更新在 FP32 上做 |
| FP32 梯度 | 4 B | 反向产生 |
| Adam 一阶动量 $m$ | 4 B | 梯度的指数移动平均 |
| Adam 二阶动量 $v$ | 4 B | 梯度平方的指数移动平均 |
| **合计** | **18 B** | — |

一个 7B 模型的固定开销 = $7 \times 10^9 \times 18 = 126$ GB，**这还没算激活值**。所以「7B 模型要 14 GB 显存」这个直觉只在推理时成立。

> [!note] 16 B 与 18 B 两种口径
> 这里的 18 B 对应「梯度以 FP32 累加」；若梯度直接存 BF16，合计为 16 B，7B 模型算出来是 112 GB。**差别只在梯度那一行**，两种都是真实配置。完整对照表与取舍见 [[02-分布式训练总论与显存账本]]。

> [!warning] 为什么必须保留 FP32 master weights
> 低精度权重上过小的更新会被直接舍掉 —— 若 $|\delta|$ 远小于当前 $x$ 附近的可表示间距，$\operatorname{fl}(x+\delta) = x$，这一步的更新等于没做。这条判据的完整解释在 [[01-数值计算与精度]]。

### 五类显存优化策略

| 策略 | 原理 | 省什么 | 代价 |
| --- | --- | --- | --- |
| **混合精度** | 前向反向用 FP16/BF16，更新用 FP32 | 约 50% 参数显存 | FP16 需 Loss Scaling |
| **梯度累积** | 多个 micro-batch 累积梯度，等效大 batch | 降低激活峰值 | 增加训练步数 |
| **梯度检查点** | 前向只保留部分激活，反向重算 | 激活显存降到 $O(\sqrt{N})$ | 约 +33% 计算量 |
| **ZeRO** | 优化器状态 / 梯度 / 参数分片到多卡 | 每卡显存线性下降 | 增加通信量 |
| **Offloading** | 部分数据卸载到 CPU 内存或 NVMe | 突破单卡显存上限 | PCIe / NVMe 带宽成瓶颈 |

这些策略**不互斥**。实践中 7B–70B 模型的常见配置是 ZeRO Stage 2 + 混合精度 + 梯度检查点。

## 相关

- [[01-推理性能指标与瓶颈定位]] —— 算力与带宽如何决定推理快慢、Decode 为什么带宽受限
- [[02-NVIDIA GPU 架构演进：Volta 到 Blackwell]] —— 这些组件逐代怎么变
- [[03-多卡互联与集群网络]] —— 第四个指标（互联带宽）的展开
- [[01-数值计算与精度]] —— 精度格式、动态范围与混合精度的完整机制
- [[10-KV Cache 与推理优化]] —— 显存容量与带宽如何塑造 KV Cache 的设计

## 参考

- https://docs.nvidia.com/cuda/cuda-c-programming-guide/
- https://blogs.nvidia.com.tw/blog/nvidia-hopper-architecture-in-depth/
- https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper
- https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf
- https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/
- https://docs.nvidia.com/cuda/parallel-thread-execution/
- https://dl.acm.org/doi/10.1145/1498765.1498785
- https://arxiv.org/abs/1910.02054
- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/gpu/gpu-basics
