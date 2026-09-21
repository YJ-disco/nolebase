---
tags:
  - AI/infra/CUDA
  - AI/infra/算子优化
---

# GEMM 性能优化

**GEMM（通用矩阵乘法）是深度学习最核心的算子** —— 线性层、Attention 的 QKV 投影、FFN，本质都是 GEMM。它也是连接硬件与上层框架的桥梁：理解了 GEMM 的优化阶梯，就理解了 cuBLAS / CUTLASS 为什么长成那个样子。

[[05-Reduce 算子优化]] 是「单点突破」的故事 —— 每一步修一个瓶颈。GEMM 是**八个台阶逐级爬升**的故事，每一级都有明确的收益区间。

## 一、指标与目标

$$\text{TFLOPS} = \frac{2 \times M \times N \times K}{\text{执行时间（秒）} \times 10^{12}}$$

（系数 2 来自一次乘加算 2 个 FLOP。）**用占理论峰值的百分比评估优化效果 —— cuBLAS 在大矩阵上通常能到 90%+。**

## 二、方法论：三个工具

### Roofline 与平衡点

| 物理上限 | V100 上的值 |
| --- | --- |
| 计算上限（FP32） | 约 15.7 TFLOPS |
| 带宽上限（HBM） | 约 900 GB/s |

$$\text{平衡点} = \frac{15.7 \times 10^{12}}{900 \times 10^9} \approx 17.4\ \text{FLOP/Byte}$$

算术强度低于 17.4 → 带宽受限；高于 → 计算受限。

> **GEMM 优化的核心目标就是提高算术强度** —— 通过数据复用，让每个字节参与尽可能多的计算。整篇的八个台阶都在做这一件事。

### 用带宽而不是延迟做判据

GPU 的海量线程能用流水线隐藏延迟，**但带宽是物理硬限**。判断一个优化是否有效，看它有没有减少对某一级存储的带宽需求：

| 存储层次 | 带宽 | 延迟 |
| --- | --- | --- |
| 寄存器 | 约 **20 TB/s** | 0 cycle |
| 共享内存 | 约 **12 TB/s** | 20–30 cycles |
| L2 Cache | 约 2 TB/s | 约 200 cycles |
| HBM | 约 900 GB/s | 300–400 cycles |

**从 HBM 到共享内存是 13 倍，到寄存器是 22 倍** —— 每一级台阶的本质都是「把数据往上搬一层」。

### Wave 模型与 L2 命中率

**Wave（执行波）** = GPU 同一时刻能并行执行的 Block 集合。V100 有 80 个 SM、每 SM 容纳 2 个 Block → 一个 Wave 最多 **160 个 Block**。

Block Tile 在 M/N 方向上的排布影响 L2 命中率，近似公式：

$$\text{L2 命中率} \approx 1 - \frac{W_m W_n}{W_m W_n + W_m N_{block} + W_n M_{block}}$$

> [!warning] 别忽略 Block 的调度顺序
> 行优先 / 列优先 / Z 形遍历会显著影响 L2 命中率 —— **大矩阵上可产生 10% 以上的性能差异**。这是一条「不改 kernel 逻辑、只改 Grid 映射」就能拿到的收益，而且很容易被漏掉。

## 三、八个台阶

| 阶段 | 相对 cuBLAS | 核心改进 |
| --- | --- | --- |
| Naive | **6% – 11%** | 无优化基线 |
| **Shared Memory Tiling** | 25% – 40% | 数据复用 |
| **Thread Tiling**（寄存器外积） | 50% – 65% | 减少共享内存访问 |
| Vectorized Load（`float4`） | 60% – 75% | 减少指令数 |
| Bank Conflict 消除 | 70% – 80% | 提高共享内存吞吐 |
| **Warp Tiling** | 80% – 87% | 优化 warp 内访问 |
| **Double Buffering** | **90% – 97%** | 隐藏延迟 |
| SASS 手调 | 95% – 99% | 寄存器调度 |

**最大的两级跨越是 Thread Tiling（+25pp）和 Double Buffering（+10pp 到 97%）** —— 前者把复用从共享内存提到寄存器，后者把访存延迟藏起来。

## 四、各台阶在做什么

### 台阶 1：Shared Memory Tiling

朴素三重循环让**每个乘加都从全局内存取数**。Tiling 的做法是把矩阵切成 Tile 载入共享内存，再在片上算。

**为什么沿 K 维度切分**：$C = AB$ 的累加是沿 $K$ 方向的，沿 $K$ 切成多段后各段部分和相加即可 —— 这与分块矩阵乘法的恒等式一致（见 [[03-线性代数与 GEMM]]）。所以外层可以沿 $K$ 循环，每轮加载一块 $A$、一块 $B$，逐步累加。

**计算访存比怎么提升**：Block Tile 为 $BM \times BN$、沿 $K$ 切 $BK$ 时，一次加载的数据被复用了 $BM \times BN$ 次。Tile 越大，复用越高 —— 但受共享内存容量限制。

**协作加载的线程排布**很讲究：256 个线程要加载 $BM \times BK$ 与 $BK \times BN$ 两块数据，需要把线程按二维映射，且**保证每个 Tile 维度能被整除**，否则要加边界判断（而边界判断本身是分支）。

### 台阶 2：Thread Tiling（寄存器外积）

此时共享内存成了新瓶颈（12 TB/s 对 20 TB/s，差近一半）。**解法是把复用再往上提一层到寄存器。**

做法是**外积（outer product）分解**：每个线程在寄存器里维护一个 $TM \times TN$ 的小累加块，每次从共享内存读一行 $A$（$TM$ 个）和一列 $B$（$TN$ 个），做 $TM \times TN$ 次乘加：

$$\text{共享内存访问次数} \propto \frac{1}{TM} + \frac{1}{TN}$$

**$TM = TN = 8$ 时，共享内存访问量降到 1/4** —— 这就是这一级能拿到 +25pp 的原因。

**寄存器预算**是硬约束：$TM \times TN = 64$ 个累加器 + 操作数 + 地址与索引，实际约 120–128 个寄存器/线程（上限 255）。**这里正是「Occupancy 不是越高越好」的战场** —— 128 寄存器只有 25% Occupancy，但数据复用换来的效率远超损失（见 [[04-Occupancy、同步与原子操作]]）。

### 台阶 3：向量化访存

`float4` 一次取 4 个元素，访存指令数降到 1/4。配套要求：

- **布局要能被向量化** —— 常需要把 A 做一次转置存储，使向量化后的坐标映射与 Tile 布局对齐
- **16 字节对齐**（见 [[03-CUDA 内存模型与访存优化]]）

### 台阶 4：消除 Bank Conflict

**`float4` 会改变冲突的形态** —— 一次取 16 字节，覆盖 4 个连续 Bank，所以冲突分析要按「**Memory Transaction**」重新算，不能沿用标量时代的结论。

**Warp 形状（线程在 Tile 内的排布）直接决定冲突**：$4 \times 8$、$8 \times 4$、$2 \times 16$ 等不同形状在 float4 下的 transaction 分布完全不同。解法仍是 **Padding** 或 **Swizzle**（XOR 地址变换，见 [[03-CUDA 内存模型与访存优化]]）。

> **这一级的关键认知是：向量化与 Bank Conflict 是耦合的。** 先向量化再调冲突，顺序不能反 —— 用标量时代的冲突结论去指导向量化后的代码，多半是错的。

### 台阶 5：Warp Tiling

把 Tile 结构按 Warp 组织，使**每个 Warp 内部访问的 Bank 分布与线程映射对齐**，减少 Warp 内冗余计算与同步开销。

**Warp Tile 的形状影响计算访存比** —— 与 Thread Tiling 是同一套推导，只是尺度从「线程」换到「Warp」。`4 × 8` 的推荐形状是「平衡 A/B 两侧访问效率」的结果。

### 台阶 6：Double Buffering（最大的单级收益）

前面所有优化都没解决一件事：**从全局内存加载数据时 GPU 在等**。双缓冲在共享内存里开两份 Buffer，让**加载下一块与计算当前块重叠**：

| 层级 | 缓冲什么 | 作用 |
| --- | --- | --- |
| **p-Loop 级** | 共享内存 → 寄存器 | 隐藏共享内存延迟（20–30 cycles） |
| **K-Loop 级** | 全局内存 → 共享内存 | 隐藏 HBM 延迟（300–400 cycles） |

**两级都要做** —— 只做 K-Loop 级，瓶颈会从 HBM 转移到共享内存。

### 台阶 7：异步拷贝（Ampere 及以后）

双缓冲的加载仍占用寄存器和指令 slot。**`cp.async` 让全局内存 → 共享内存的拷贝异步化，绕过寄存器**：

```cpp
__pipeline_memcpy_async(&smem[buf][i], &gmem[i], sizeof(float4));
__pipeline_commit();
__pipeline_wait_prior(1);       // 等前一组完成
```

**Pipeline 同步模型**从「双缓冲的两份 buffer」推广到「N 级流水线」—— CUTLASS 的多级流水线就是这个思想的工程化。

**Hopper（SM90）换了机制**：`TMA`（Tensor Memory Accelerator）接管异步搬运与地址计算，`wgmma` 让操作数从共享内存直接进 Tensor Core。`wgmma` 的形状与执行单位见 [[01-GPU 硬件架构与存储层次]]。

### 台阶 8：SASS 级调优

到这一级只能看汇编：

| 手段 | 内容 |
| --- | --- |
| **寄存器 Bank Conflict** | 寄存器堆也分 Bank，操作数排布不当会有冲突 |
| **Register Reuse Cache** | 硬件可以复用最近读过的操作数，写法要配合 |
| **寄存器重映射** | 重新分配寄存器编号以减少冲突 |
| **指令交错** | 把访存与计算指令交错排布，提升 ILP |

> **前七级靠算法与结构，第八级靠指令排布。** 收益从 90% 到 95–99%，但**没有前七级做基础，手调 SASS 拿不到任何东西**。

## 五、推荐参数

| 参数 | 推荐值 | 说明 |
| --- | --- | --- |
| `BM` / `BN` | **128 / 128** | 过小则算术强度不足，过大则共享内存放不下 |
| `BK` | 8 | 影响共享内存用量与加载指令数 |
| `TM` / `TN` | **8 / 8** | 每线程负责 8×8，寄存器可承受 |
| Block Size | **256** | $= (128/8) \times (128/8) = 16 \times 16$ |
| Warp 形状 | **4 × 8** | 平衡 A/B 访问效率 |
| 寄存器/线程 | **120–128** | 用 `__launch_bounds__` 控制 |
| 共享内存/Block | **16–32 KB** | 含双缓冲 |

## 六、写法本身就是性能

有一处很值得单独记：**看似等价的代码写法在 SASS 层面差异巨大 —— 仅靠写法优化，性能就能从约 60% 提到约 93% cuBLAS。**

```cpp
// 慢：整数除法/取模在 GPU 上编译成多条指令（除数非 2 的幂时更甚）
int row = threadIdx.x / BN_TILE;
int col = threadIdx.x % BN_TILE;

// 快：BN_TILE 是 2 的幂时，变成单条位移/与指令
int row = threadIdx.x >> 3;
int col = threadIdx.x & 7;
```

> **这解释了为什么 Tile 尺寸偏爱 2 的幂**：除了对齐，更直接的原因是**除法取模能退化成位运算**。这是一条「改一行代码、性能翻倍」的优化，且不影响任何算法正确性。

## 七、调优时该看哪个 stall 原因

| Nsight Compute 的 stall 原因 | 含义 |
| --- | --- |
| `Stall Long Scoreboard` | **全局内存延迟未隐藏** → 上双缓冲 / `cp.async` |
| `Stall Short Scoreboard` | 共享内存延迟 → 检查 Tile 结构与复用 |
| `Stall MIO Throttle` | **共享内存指令压力过大** → 检查 Bank Conflict 与指令数 |
| `Stall Barrier` | 同步等待 → 检查 `__syncthreads()` 的必要性与位置 |

**这张表是「症状 → 下一步动作」的映射** —— 比盲目改参数有效得多。工具链的完整用法在源教程里只有大纲，本库暂未展开（见 [[00-CUDA 与算子优化专栏导览]] 的待建清单）。

## 八、工程结论

> **生产环境直接用 cuBLAS 或 CUTLASS。** 手写 GEMM 的价值在于理解优化阶梯，不在于替代它们 —— 这条判据与 [[05-Reduce 算子优化]] 结尾的 CUB 结论一致。
>
> 但**台阶的顺序本身就是知识**：知道「Tiling 之后瓶颈会转到共享内存」「向量化会改变冲突形态」「不做双缓冲就永远卡在 87%」，这些正是读 profiler 时能定位问题的前提。

## 相关

- [[03-CUDA 内存模型与访存优化]] —— 合并访问、Bank Conflict、向量化的机制
- [[04-Occupancy、同步与原子操作]] —— 128 寄存器与 25% Occupancy 的取舍
- [[05-Reduce 算子优化]] —— 同一套优化循环在更简单算子上的完整演示
- [[03-线性代数与 GEMM]] —— 分块矩阵乘法的数学等价性与算术强度估算
- [[01-GPU 硬件架构与存储层次]] —— Tensor Core 的形状与 `wgmma`

## 参考

- **AIInfraGuide 4.1 CUDA GEMM 算子性能优化**（性能指标、Roofline 与带宽视角、Wave 模型与 L2 命中率、八个优化台阶与逐级收益、Thread Tiling 的外积分解与共享内存访问量推导、float4 与 Bank Conflict 的耦合、两级双缓冲、`cp.async` 与 CUTLASS 多级流水线、SASS 级手段、推荐参数表、写法优化）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/41-cuda-gemm%E7%AE%97%E5%AD%90%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96
- **How to Optimize a CUDA Matmul Kernel for cuBLAS-like Performance**（该优化序列的经典出处，Simon Boehm）：https://siboehm.com/articles/22/CUDA-MMM
- **CUTLASS**（生产级 GEMM 模板库）：https://github.com/NVIDIA/cutlass
- **cuBLAS Documentation**：https://docs.nvidia.com/cuda/cublas/
- **NVIDIA Nsight Compute — Stall Reasons**：https://docs.nvidia.com/nsight-compute/
