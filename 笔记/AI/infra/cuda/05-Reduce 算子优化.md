---
tags:
  - AI/infra/CUDA
  - AI/infra/算子优化
---

# Reduce 算子优化

Reduce 是「多输入 → 单输出」的一类操作（Sum / Max / Min / Dot Product）。它是**学习 CUDA 优化的最佳入门案例** —— 从朴素实现到接近带宽上限，中间每一步都针对一个**可定位、可量化**的瓶颈。

这一篇按八个版本走完这个循环：**分析瓶颈 → 针对优化 → 量化收益**。

## 先定位瓶颈

Sum Reduce 的算术强度：

- 每个元素读一次（1 次 load，4 Byte）
- 每次读取做一次加法（1 次 FLOP）

$$\text{算术强度} = \frac{1\ \text{FLOP}}{4\ \text{Byte}} = 0.25\ \text{FLOP/Byte}$$

对比 H100 的平衡点约 295 FLOP/Byte（见 [[01-GPU 硬件架构与存储层次]]）—— **低了三个数量级**。

> **结论：Reduce 是典型的内存受限操作。优化的核心是提升带宽利用率，不是减少计算。** 整篇的每一个版本都在回答「哪一部分带宽被浪费了」。

本文的测试基准：A100 80GB SXM4，理论带宽 **2,039 GB/s**，$N = 2^{27}$（128M 个 float，共 **512 MB**）。

## 八个版本的演进

| 版本 | 核心优化点 | 带宽利用率 | 相对速度 |
| --- | --- | --- | --- |
| **V0** 朴素树形 | 无 | **约 15%** | 1.0× |
| **V1** 交错寻址 | 减少 Warp Divergence | 约 32% | 2.1× |
| **V2** 步长反转 | 同时消除 Bank Conflict 与 Divergence | 约 40% | 2.7× |
| **V3** 双元素处理 | 减少空闲线程，Block 数减半 | 约 45% | 3.0× |
| **V4** 展开最后 Warp | 省去 Warp 内多余的 `__syncthreads()` | 约 52% | 3.5× |
| **V5** 完全循环展开 | 模板参数，编译期消除所有循环 | 约 62% | 4.1× |
| **V6** Warp Shuffle | 寄存器直通，绕过共享内存 | 约 72% | **4.8×** |
| **V7** 向量化 + Grid Stride | 提升访存效率 + 完整覆盖 GPU | **约 85%** | **5.7×** |

**V7 达到约 1733 GB/s** —— 已经接近实际可达上限（受 ECC、时钟波动影响，**Reduce 的合理目标是 85%–90%**）。

## 每一步在修什么

### V0 → V1：Warp Divergence

朴素树形规约：

```cpp
for (int step = 1; step < blockDim.x; step *= 2) {
    if (tid % (2 * step) == 0) {          // ← 性能杀手
        smem[tid] += smem[tid + step];
    }
    __syncthreads();
}
```

`tid % (2*step) == 0` 让**同一 Warp 内的线程走不同分支** —— 这是 V0 只有 15% 的直接原因：

| step | 一个 Warp 内工作线程 | 利用率 |
| --- | --- | --- |
| 1 | 16 / 32（只有偶数） | 50% |
| 2 | 8 / 32（每 4 个 1 个） | 25% |
| ... | 越来越差 | — |

**修法：把判断改成 strided index，让整个 Warp 一起进或一起跳。**

```cpp
int index = threadIdx.x * 2 * s;         // 相邻线程负责相邻步长
if (index < blockDim.x) { ... }
```

`step=1` 时 `tid 0~127` 的 index 都 < 256、`tid 128~255` 都 ≥ 256 —— **前 4 个 Warp 整体进入，后 4 个整体跳过，Warp 内零分化**。随着 step 增大，活跃区间收窄到更小的 tid 范围，直到 `step=8` 时活跃线程不足 32 个才开始分化。**从「每轮都分化」变成「只有最后 5 轮、且都集中在 Warp 0 内部」。**

V0 的树形规约为什么一开始就只跑出 15%：

```
  blockDim = 256，只看 Warp 0 的 tid 0–31，条件 tid % (2*step) == 0

    step = 1：
      tid:    0    1    2    3   …   15   16   17   …   31
      工作:   ●    ·    ●    ·        ●    ·    ●        ·
              └── 32 个线程里只有 16 个活跃 ⇒ 利用率 50%

    step = 2：
      工作:   ●    ·    ·    ·    ●    ·    ·    ·    ●  …
              └── 每 4 个线程里 1 个活跃 ⇒ 利用率 25%
      （step 继续翻倍，利用率继续对半）

  交错寻址 V1 的修法：index = threadIdx.x * 2 * s

    step = 1 时：tid 0–127 的 index 都 < 256 → 整体进入
                 tid 128–255 的 index 都 ≥ 256 → 整体跳过
    ⇒ Warp 内零分化；分化只剩最后几轮，且都关在 Warp 0 内部
```

### V1 → V2：Bank Conflict

V1 看似解决了 Divergence，但共享内存访问有严重冲突。以 `blockDim=256`、Warp 0（tid 0–31）为例：

| 轮次 | 活跃线程访问的地址 | 冲突 |
| --- | --- | --- |
| s=1 | tid 0 → smem[0], smem[1]；tid 16 → smem[32], smem[33] | **2 路**（0 与 32 都落 Bank 0）|
| s=2 | tid 0/8/16/24 → smem[0/32/64/96] | **4 路** |
| s=4 | — | **8 路** |

根因是**步长从小往大翻倍**，跨步间隔正好是 32 的倍数，全撞同一个 Bank。

**修法：把规约方向反过来 —— 从 `blockDim.x/2` 开始逐步减半，且低编号线程始终活跃。**

```
step=128：tid 0→smem[0,128]、tid 1→smem[1,129]、…、tid 31→smem[31,159]
          → 32 个线程正好覆盖 Bank 0~31，无冲突
```

**这一步同时拿到两个收益**：地址间隔变成 1（无 Bank Conflict），且活跃线程始终是连续的低编号线程（**天然无 Warp Divergence**）。这就是为什么 V2 的增益比 V1 更大。

> **一个改动修两个瓶颈** —— 这是优化循环里最理想的情形，也说明「先想清楚瓶颈的根因」比「逐个打补丁」有效。

同一次改动为什么能同时修掉两个瓶颈：

```
   用 Bank 分布看这次修法：

     Bank 号          0      1      2    …    31
     V1（步长递增）    0/32   1/33   2/34  …   31/63   ← 多个地址挤进同一个 Bank
     V2（步长递减）    0      1      2    …    31     ← 32 个线程正好铺满一组
                       ▲
                       └─ 步长变成 1 之后，地址间隔与 Bank 数错开，冲突消失

  V1 的问题：步长从小往大翻倍，跨步间隔正好是 32 的倍数，全撞同一个 Bank

    step = 1：tid 0 → smem[0]、smem[1]；tid 16 → smem[32]、smem[33]
              ⇒ 地址 0 与 32 都落 Bank 0 ⇒ 2 路冲突
    step = 2：tid 0/8/16/24 → smem[0/32/64/96] ⇒ 4 路
    step = 4：⇒ 8 路

  V2 的修法：把方向反过来，从 blockDim.x/2 开始逐步减半，低编号线程始终活跃

    step = 128：tid 0  → smem[0]、smem[128]
                tid 1  → smem[1]、smem[129]
                …
                tid 31 → smem[31]、smem[159]
                ⇒ 32 个线程正好覆盖 Bank 0–31，无冲突
                ⇒ 活跃线程始终是连续的低编号线程，也天然无分化

  一次改动拿到两个收益，这就是 V2 的增益比 V1 更大的原因。
```

### V2 → V3：别让一半线程当搬运工

V0–V2 每个线程只负责 1 个元素 → 每个 Block 处理 `blockDim.x` 个数据 → $N = 2^{27}$ 需要 **524288 个 Block**。

而规约第一轮（`step = blockDim.x/2`）就有一半线程闲置 —— **它们的唯一贡献是把数据从全局搬到共享内存**。

**修法：每个线程在加载阶段就处理 2 个元素并预先求和。**

```cpp
int gid = blockIdx.x * (blockDim.x * 2) + threadIdx.x;
float val = 0.0f;
if (gid < n)              val += input[gid];
if (gid + blockDim.x < n) val += input[gid + blockDim.x];
smem[tid] = val;
```

同样 256 个线程现在处理 512 个数据 —— **Block 数减半（262144 个），而规约阶段的工作量不变**。收益来自两处：**Grid 更小、调度开销降低**；以及每个线程在加载阶段就做了有用功。

### V3 → V4：最后 32 个线程不需要屏障

`step <= 32` 时只剩 1 个 Warp（tid 0–31）在工作 —— **此时循环里的 `__syncthreads()` 完全是多余的**：同一 Warp 内的线程执行相同指令路径时本身就同步。

**修法：把最后 5 轮展开，去掉屏障。**

```cpp
__device__ void warpReduce(volatile float* smem, int tid) {
    smem[tid] += smem[tid + 32];
    smem[tid] += smem[tid + 16];
    smem[tid] += smem[tid +  8];
    smem[tid] += smem[tid +  4];
    smem[tid] += smem[tid +  2];
    smem[tid] += smem[tid +  1];
}
```

`volatile` 是必需的 —— 它阻止编译器把这些共享内存访问优化到寄存器里，保证每次读的都是最新值。

### V4 → V5：循环本身也是开销

剩下的循环有**循环计数、条件判断、分支**。**修法是用模板参数把 `blockDim` 变成编译期常量**，让编译器完全展开 —— 循环开销归零，且地址计算可以常量折叠。

### V5 → V6：寄存器直通

共享内存往返本身就是开销，而**同一个 Warp 内的线程可以直接读彼此的寄存器**（`__shfl_down_sync`）：

```cpp
__device__ float warp_reduce_sum(float val) {
    for (int offset = 16; offset > 0; offset >>= 1)
        val += __shfl_down_sync(0xFFFFFFFF, val, offset);
    return val;
}
```

**5 步完成 32 个元素的归约，零共享内存、零 `__syncthreads()`**，也彻底没有 Bank Conflict 的可能。机制见 [[02-CUDA 编程模型与执行模型]]。

实际实现是**两级归约**：Warp 内用 Shuffle，Warp 之间用共享内存（每个 Warp 写一个部分和，再由第一个 Warp 归约）。

V6 起的归约是两级的：

```
  第一级（Warp 内，零共享内存）

    Warp 0 的 32 个线程 ──__shfl_down_sync──▶ 5 步 ⇒ 1 个部分和
    Warp 1 的 32 个线程 ──__shfl_down_sync──▶ 5 步 ⇒ 1 个部分和
    ⋮
    （每个 Warp 各出一个部分和）

  第二级（Warp 之间，走共享内存）

    [Warp 0 的部分和][Warp 1 的部分和][…]  ──▶  由第一个 Warp 再归约一次  ──▶  最终结果

  ⇒ 共享内存只写「每个 Warp 一个值」，往返量比逐元素方案小一个数量级
  ⇒ 全程不再有 Bank Conflict 的可能
```

### V6 → V7：让访存指令更少

两个改动叠加：

- **向量化加载** —— `float4` 一次取 4 个元素，访存指令数降到 1/4（对齐要求见 [[03-CUDA 内存模型与访存优化]]）
- **Grid-stride loop** —— 让每个线程以 `blockDim.x * gridDim.x` 为步长遍历，**用恰好能填满 GPU 的 Grid 覆盖任意规模的数据**（512 MB 的数组不必启动 50 万个 Block）

V7 达到 85%，**这一版才真正把「访存效率」和「GPU 利用率」两条一起解决**。

## 瓶颈与对策的对照

八版优化可以归成五类瓶颈：

| 瓶颈 | 对策 | 版本 |
| --- | --- | --- |
| **计算效率低**（Warp Divergence） | 交错寻址 → 步长反转 | V1、V2 |
| **共享内存冲突**（Bank Conflict） | 步长反转 | V2 |
| **同步开销大**（多余 `__syncthreads`） | 展开最后 Warp → 完全展开 | V4、V5 |
| **访存效率低**（逐元素加载） | Warp Shuffle → `float4` | V6、V7 |
| **GPU 利用不足**（线程 / Block 空闲） | 双元素处理 → Grid Stride Loop | V3、V7 |

**这张表才是本篇最有用的部分** —— 它给出的是「症状 → 对策」的映射，而不是「某个 kernel 该怎么写」。

## 工程上怎么选

| 场景 | 推荐 |
| --- | --- |
| 学习 / 教学 | V2 或 V4 —— 逻辑清晰 |
| 生产环境通用 | **V6 Warp Shuffle 版** |
| 超大数组（> 1 GB） | V7 Grid Stride Loop |
| 追求极致 | **NVIDIA CUB 的 `cub::DeviceReduce::Sum`** |

> **生产环境优先用 CUB。** 它在各代 GPU 架构上做了针对性优化，**通常能到 90% 以上带宽利用率**，且维护成本为零。
>
> 手写 kernel 的价值在于**理解瓶颈在哪**，不在于替代库 —— 这条判据在 GEMM、Attention 上同样成立（cuBLAS / CUTLASS、FlashAttention）。

## 相关

- [[02-CUDA 编程模型与执行模型]] —— Warp Shuffle 的指令语义与蝶形归约
- [[03-CUDA 内存模型与访存优化]] —— Bank Conflict 与向量化加载的机制
- [[04-Occupancy、同步与原子操作]] —— `__syncthreads()` 的代价与原子分层归约
- [[01-GPU 硬件架构与存储层次]] —— 算术强度与 Roofline
- [[01-数值计算与精度]] —— 归约顺序对浮点结果的影响

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/31-cuda-reduce%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96
- https://github.com/NVIDIA/cccl/tree/main/cub
- https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html
- https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf
