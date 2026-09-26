---
tags:
  - AI/infra/CUDA
  - AI/infra/存储层次
---

# CUDA 内存模型与访存优化

GPU 算力极强而带宽有限，**这个落差决定了绝大多数 kernel 的瓶颈不在计算而在访存**。这一篇讲清内存层次的分工，以及两个最大的杠杆：**合并访问**（全局内存）与 **Bank Conflict**（共享内存）。

## 为什么瓶颈永远是访存

以 A100 为例：FP16 Tensor Core 峰值 312 TFLOPS，显存带宽约 2 TB/s。

按最坏情形算 —— 每次浮点乘加都要读 2 个 float（8 Bytes）而产生 2 FLOP：

$$\frac{2 \times 10^{12}\ \text{Byte/s}}{8\ \text{Byte/FMA}} \times 2\ \text{FLOP/FMA} = 500\ \text{GFLOPS}$$

**带宽最多只能喂饱 500 GFLOPS，不到峰值算力的 0.2%。**

> 这不是说实际只能跑到 0.2% —— 实际 kernel 靠数据复用把算术强度提上去。但它精确说明了「**不复用的话算力完全是闲着的**」，以及为什么所有优化的主线都是**让同一份数据被算更多次**。与 [[01-GPU 硬件架构与存储层次]] 的 Roofline 平衡点是同一件事。

## 内存层次

| 类型 | 位置 | 容量 | 延迟 | 带宽 | 作用域 | 生命周期 |
| --- | --- | --- | --- | --- | --- | --- |
| **寄存器** | SM 片上 | 约 256 KB/SM | **1 cycle** | 最高 | 单线程 | 线程 |
| **共享内存** | SM 片上 | 48–228 KB/SM | 约 20 cycles | 约 19 TB/s | **Block 内** | Block |
| L1 Cache | SM 片上 | 128–256 KB/SM | 约 30 cycles | 约 19 TB/s | 自动 | 自动 |
| L2 Cache | GPU 片上 | 6–50 MB | 约 200 cycles | 约 5 TB/s | 所有 SM | 自动 |
| **全局内存（HBM）** | 芯片外 | 16–80 GB | **约 400 cycles** | 1–3.4 TB/s | 全局 | 应用 |
| 常量内存 | 芯片外 + 缓存 | 64 KB | 1–400 cycles | 有缓存时极高 | 全局只读 | 应用 |

**寄存器到 HBM 差两个数量级**，这就是「把数据往上搬一层」是永恒优化主题的原因。

## 全局内存：合并访问（Coalesced Access）

GPU **不以单个 float 为单位取数** —— 它以 **32B / 64B / 128B 的粒度成事务（transaction）**从全局内存取。所以关键是：**一个 Warp 内 32 个线程的访问地址是否连续**。

```cpp
// 合并访问：连续线程访问连续地址
float val = data[threadIdx.x + blockIdx.x * blockDim.x];

// 跨步访问：地址散开，效率极低
float val = data[(threadIdx.x + blockIdx.x * blockDim.x) * stride];
```

| 访问模式 | Warp 取 32 个 float 需要的事务 | 带宽利用率 |
| --- | --- | --- |
| **完美合并**（连续且对齐） | 1 次 128B | **100%** |
| 连续但未对齐 | 2 次 128B | **50%** |
| 随机散列 | 最多 32 次 32B | **约 3%** |

**「未对齐」就要付 50% 的代价** —— 这是最容易忽略、也最容易修的一类损失。

Warp 一次访存覆盖多少地址，决定要几个内存事务：

```
  合并（连续且对齐）：

    lane     0     1     2     3    …    31
    地址   0x00  0x04  0x08  0x0C  …  0x7C
            └───────── 一个 128B 事务全覆盖 ─────────┘
            ⇒ 1 次事务，带宽利用率 100%

  跨步（stride = 4，每个线程隔 16 字节）：

    lane     0     1     2     3    …    31
    地址   0x00  0x10  0x20  0x30  …  0x1F0
            ↓     ↓     ↓     ↓          ↓
          每个线程落在不同的事务粒度里
            ⇒ 最多 32 次事务，带宽利用率约 3%

  连续但对齐差一点（跨 128B 边界）⇒ 2 次事务，掉到 50%。
```

### SoA 优于 AoS

```cpp
// 反例 AoS：相邻线程访问跨 24 字节，无法合并
struct Particle { float x, y, z, vx, vy, vz; };
Particle particles[N];

// 正例 SoA：每个字段各自连续，天然合并
struct Particles { float *x, *y, *z, *vx, *vy, *vz; };
```

**数据结构怎么组织，直接决定能不能合并访问。** 这是「改数据结构比调 kernel 更有效」的典型场景 —— 而且这个决定通常在写 kernel 之前就做完了。

## 共享内存：Bank Conflict

### Bank 结构

共享内存被划分为 **32 个 Bank**（与 Warp 大小一致），**每个 Bank 宽 4 Bytes**，连续的字轮流分配到连续 Bank：

```
地址 0~3     → Bank 0
地址 4~7     → Bank 1
...
地址 124~127 → Bank 31
地址 128~131 → Bank 0     ← 循环
```

**每个时钟周期，每个 Bank 只能服务一次请求。** 若同一 Warp 中有多个线程访问**同一 Bank 的不同地址**，这些访问必须串行化。

| 情况 | 代价 |
| --- | --- |
| 32 线程访问 32 个不同 Bank | 1 个周期 |
| 2 个线程撞同一 Bank | 2 个周期 |
| N 个线程撞同一 Bank | **N 个周期** |
| **多个线程访问同一 Bank 的同一地址（广播）** | **1 个周期，免费** |

> [!warning] 广播是免费的，但只在「地址完全相同」时成立
> 硬件会把同一个值广播给所有请求线程。**只有「同一 Bank + 不同地址」才产生冲突。** 这条区分很重要 —— 很多时候代码看起来「有多个线程读同一 Bank」，实际是广播，没有代价。

### 经典冲突：矩阵转置

```cpp
__shared__ float tile[32][32];

tile[threadIdx.y][threadIdx.x] = input[gy * N + gx];   // 写：连续线程写同一行 → 无冲突
output[gx * N + gy] = tile[threadIdx.x][threadIdx.y];  // 读：连续线程读同一列 → 冲突！
```

逐线程算一下读时的 Bank：

```
threadIdx.x=0 → tile[0][ty]  地址 (0*32+ty)*4  → Bank = ty % 32
threadIdx.x=1 → tile[1][ty]  地址 (1*32+ty)*4  → Bank = (32+ty)%32 = ty % 32
threadIdx.x=2 → tile[2][ty]  地址 (2*32+ty)*4  → Bank = (64+ty)%32 = ty % 32
...
```

**32 个线程的 Bank 全是 `ty % 32` → 32-way Bank Conflict**，读一列要 32 个周期。原因是**行宽 32 恰好等于 Bank 数**，跨行读同列时地址步长正好是 Bank 数，全撞在一起。

### 方案一：Padding

```cpp
__shared__ float tile[32][32 + 1];     // 加一列
```

行宽变成 33 之后：

```
threadIdx.x=0 → (0*33+ty)%32 = ty%32
threadIdx.x=1 → (33+ty)%32   = (1+ty)%32
threadIdx.x=2 → (66+ty)%32   = (2+ty)%32
```

**每个线程落在不同 Bank，冲突消失。** 代价是每行多浪费 4 Bytes 共享内存，收益通常远超这点开销。

Bank 映射与矩阵转置的冲突（共享内存 32 个 Bank、每个 4 字节，地址轮流落 Bank）：

```
  地址 0–3 → Bank 0 │ 地址 4–7 → Bank 1 │ … │ 地址 124–127 → Bank 31
  地址 128–131 → Bank 0（循环回去）

  转置里读 tile[threadIdx.x][ty]，行宽 32：

    线程 x=0  → 地址 (0×32+ty)×4  → Bank = ty % 32
    线程 x=1  → 地址 (1×32+ty)×4  → Bank = (32+ty)%32 = ty % 32
    线程 x=2  → 地址 (2×32+ty)×4  → Bank = (64+ty)%32 = ty % 32
    …
    ⇒ 32 个线程全落在同一个 Bank ⇒ 32-way conflict，读一列要 32 个周期
       （根因：行宽 32 恰好等于 Bank 数）

  加一列 padding 后（行宽 33）：

    线程 x=0  → Bank = ty % 32
    线程 x=1  → Bank = (33+ty)%32 = (1+ty) % 32
    线程 x=2  → Bank = (66+ty)%32 = (2+ty) % 32
    ⇒ 每个线程落在不同 Bank，冲突消失；代价是每行多 4 字节
```

### 方案二：Swizzle

用 XOR 变换打乱 Bank 分布，不浪费空间：

```cpp
// 写入 tile[row][col ^ row]，读取 tile[col][row ^ col]
```

代价是索引计算更复杂。**高性能 GEMM 实现（如 CUTLASS）广泛使用 Swizzle** —— 见 `07-GEMM 性能优化`。

### 怎么检测

用 Nsight Compute 看这两个计数器：

```
l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_ld   # 加载冲突
l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_st   # 存储冲突
```

**不为零就说明有冲突。** 不要靠读代码猜 —— 冲突与线程映射、tile 尺寸强耦合，算错一步结论就反了。

## 向量化加载

一次访存指令取多个元素，**减少指令数并提高有效带宽**：

| 类型 | 一次取 | 宽度 |
| --- | --- | --- |
| `float2` | 2 个 float | 8 B |
| **`float4`** | **4 个 float** | **16 B** |
| `int4` / `uint4` | 4 个 int | 16 B |
| `double2` | 2 个 double | 16 B |

```cpp
// 标量：4 条加载指令
float a = in[i], b = in[i+1], c = in[i+2], d = in[i+3];

// 向量化：1 条加载指令
float4 v = reinterpret_cast<const float4*>(in)[i / 4];
```

> [!warning] 向量化有对齐前提
> 要求**首地址按向量宽度对齐**（`float4` 需 16 字节对齐）且**元素在内存中连续**。不满足时要么退化为多次标量访问，要么触发未对齐访问错误。所以向量化的前提往往在**数据结构设计阶段**就决定了。
>
> 用于归约时还要注意：向量化改变的是数据划分方式，**归约的数学与边界处理要跟着改** —— 尾块可能不再整除向量宽度。

标量加载与向量化加载（同样取 4 个 float）：

```
  标量：  LDG.E      [0x00] → r1     ┐
          LDG.E      [0x04] → r2     │  4 条载入指令、4 次访存事务
          LDG.E      [0x08] → r3     │
          LDG.E      [0x0C] → r4     ┘

  向量化：LDG.E.128  [0x00] → r1..r4      1 条指令、1 次 128B 事务

  前提两条（都在数据结构设计阶段就定了）：
    · 首地址按向量宽度对齐（float4 需 16 字节）
    · 4 个元素在内存里连续
  ⇒ 归约类算子用向量化时，数据划分与尾块边界处理要跟着改
    （尾块可能不再整除向量宽度）
```

## 缓存提示

对访问模式明确的场景，可以用缓存提示影响 L1/L2 的行为：

| 提示 | 语义 |
| --- | --- |
| `__ldg()` / `const __restrict__` | 声明只读，走纹理路径的只读缓存 |
| `cudaMemcpyAsync` + `cudaStream` | 让拷贝与计算重叠 |
| `cudaMemsetAsync` | 异步初始化 |

**判断依据是「这份数据会不会被复用」**：会被复用的走缓存，一次性的用流式提示（避免污染缓存）。

## 相关

- [[01-GPU 硬件架构与存储层次]] —— 存储层次金字塔与 Roofline 的硬件侧
- [[02-CUDA 编程模型与执行模型]] —— Block 内协作是共享内存存在的前提
- [[04-Occupancy、同步与原子操作]] —— 共享内存用量会限制 Occupancy
- `06-Reduce 算子优化` —— 共享内存与 Warp Shuffle 在归约里的对照
- `07-GEMM 性能优化` —— Bank Conflict 消除与向量化的完整实战

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/13-cuda%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B
- https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#memory-hierarchy
- https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#coalesced-access-to-global-memory
- https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#shared-memory
