---
tags:
  - AI/infra/CUDA
---

# CUDA 编程模型与执行模型

[[01-CUDA 开发环境与第一个 Kernel]] 走通了一条链路。这一篇讲**线程是怎么被组织和调度的** —— 三级线程层次是软件侧的组织，Warp 是硬件侧的执行单位，**两者的错位正是大部分性能问题的来源**。

## 异构计算与函数修饰符

CUDA 程序跑在异构系统上：**CPU（Host）管控制逻辑与串行代码，GPU（Device）管大规模并行**。

执行流程固定为：初始化 → `cudaMemcpy` 传到 GPU → `kernel<<<grid, block>>>()` → 数千线程并行 → `cudaMemcpy` 取回。

三个修饰符决定函数在哪执行、由谁调用：

| 修饰符 | 执行位置 | 调用方 |
| --- | --- | --- |
| `__global__` | GPU | CPU（或 GPU 动态并行） |
| `__device__` | GPU | GPU |
| `__host__` | CPU | CPU（默认） |

```cpp
__host__ __device__ float square(float x) { return x * x; }   // 两边都编译
```

## 三级线程层次

| 层 | 类比 | 关键性质 |
| --- | --- | --- |
| **Grid** | 整个学校 | 一次 Kernel 启动的所有线程 |
| **Block** | 一个班级 | **Block 内线程可协作**（共享内存 + 同步） |
| **Thread** | 一个学生 | 最小执行单位 |

**「Block 内可协作、跨 Block 不可」是这套设计的核心约束** —— 它决定了后面所有算子的分块方式（见 `06-Reduce 算子优化`、`07-GEMM 性能优化`）。

Grid 与 Block 都可以是 1D / 2D / 3D：

```cpp
dim3 grid(16, 16);    // 2D：16×16 = 256 个 Block
dim3 block(16, 16);   // 每个 Block 16×16 = 256 个线程
```

三级层次与硬件侧的对应关系：

```
  Grid：一次 Kernel 启动的全部线程
   ├── Block 0                 ├── Block 1                 ├── …
   │    ├── Warp 0             │    ├── Warp 0
   │    │    └── 32 个 Thread  │    │    └── 32 个 Thread   ← 硬件实际按 Warp 调度
   │    ├── Warp 1             │    ├── Warp 1
   │    └── …                  │    └── …
   └── …

  软件侧给的组织方式与硬件侧的执行单位在这里错位：
    · 软件侧承诺的是「Block 内线程可协作」——共享内存 + __syncthreads()
    · 硬件实际发射的是一条 Warp 指令，32 个线程锁步跟随
    · Block 的线程数不是 32 的倍数 ⇒ 最后一个 Warp 里有永久空置的 lane
```

### 硬件限制（Compute Capability ≥ 8.0）

| 限制项 | 最大值 |
| --- | --- |
| Block 每维线程数 | x:1024, y:1024, z:64 |
| **Block 内线程总数** | **1024** |
| Grid 每维 Block 数 | x:$2^{31}-1$, y:65535, z:65535 |
| 每 SM 最大活跃线程数 | **2048（sm_80）/ 1536（sm_89）** |
| 每 SM 最大活跃 Block 数 | 16–32（随架构） |

> [!warning] 1024 是硬上限
> `blockDim.x * blockDim.y * blockDim.z ≤ 1024`。这是**每个 Block** 的限制，不是每个 Grid。而每 SM 的活跃线程数上限（2048 / 1536）又是另一条约束 —— **它决定了同一时刻一个 SM 上能驻留几个 Block**，直接连到 Occupancy，见 [[04-Occupancy、同步与原子操作]]。

## 索引计算

四个内置变量：

| 变量 | 含义 |
| --- | --- |
| `threadIdx` | 线程在所属 Block 内的索引 |
| `blockIdx` | Block 在 Grid 内的索引 |
| `blockDim` | 每个 Block 的维度 |
| `gridDim` | Grid 的维度 |

```cpp
// 一维
int globalIdx = blockIdx.x * blockDim.x + threadIdx.x;

// 二维
int col = blockIdx.x * blockDim.x + threadIdx.x;
int row = blockIdx.y * blockDim.y + threadIdx.y;
int linearIdx = row * width + col;             // 行主序转一维

// 三维：把前两维的跨度乘进去
int idx = (blockIdx.z * gridDim.y * gridDim.x
         + blockIdx.y * gridDim.x
         + blockIdx.x) * blockDim.x + threadIdx.x;
```

### Grid-stride Loop

数据量大于线程总数时的标准做法：

```cpp
__global__ void processLargeArray(float* data, int N) {
    int idx    = blockIdx.x * blockDim.x + threadIdx.x;
    int stride = blockDim.x * gridDim.x;        // 总线程数
    for (int i = idx; i < N; i += stride) {
        data[i] *= 2.0f;
    }
}
```

> **Grid-stride loop 是 CUDA 的最佳实践之一。** 它同时解决两个问题：**任意规模的数据都能正确覆盖**，以及**可以在不改代码的前提下调整 Grid 大小来试性能**。同一个 kernel 在 1 万和 1 亿元素上都能用。

## Block 大小怎么选

### 必须是 32 的倍数

**因为 Warp 是 32 个线程。** 若 `blockDim = 48`，硬件会把它拆成「32 + 16」，第二个 Warp 里有一半 lane 永久空置 —— 相当于白白浪费 25% 的线程槽位。**Block 大小不是 32 的倍数，等于每次启动都自带浪费。**

### 用 API 自动算

```cpp
int minGridSize, blockSize;
cudaOccupancyMaxPotentialBlockSize(&minGridSize, &blockSize, myKernel, 0, 0);
myKernel<<<minGridSize, blockSize>>>(...);
```

这是把「让 Occupancy 最大」这件事交给运行时，比手调 `256` 更靠谱。

### 实用起点

| Block 大小 | 适用 |
| --- | --- |
| 128 | 寄存器压力大、共享内存用得多的 kernel |
| **256** | **通用起点** —— 与多种 SM 架构兼容性好 |
| 512 / 1024 | 访存密集、寄存器压力小的 kernel |

## Warp：硬件真正的执行单位

**32 个线程组成一个 Warp，锁步执行同一条指令。**

### SIMT 不是 SIMD

| 维度 | SIMD（如 AVX-512） | **SIMT（CUDA Warp）** |
| --- | --- | --- |
| 编程视角 | 程序员显式操作向量寄存器 | **每个线程有独立 PC 和栈**（逻辑上） |
| 分支处理 | 用掩码跳过 lane | **硬件自动串行化分支路径** |
| 寻址 | 所有 lane 通常访问连续数据 | **每个线程可独立寻址** |
| 编程难度 | 需手动向量化 | **标量代码自动在 32 线程上并行** |

> **SIMT 的核心理念是：程序员写单线程的标量代码，硬件把相同指令广播到 32 个线程。** 它降低了并行编程门槛，代价就是下面这个陷阱。

### Warp Divergence

Warp 内的线程遇到 `if/else`、`switch` 或循环次数不同时，**硬件在同一时刻只能发射一条指令，于是把所有分支路径串行执行**，不走当前路径的线程被掩码（masked off）。

```cpp
if (tid % 2 == 0) data[tid] = expf(data[tid]);   // 偶数线程
else              data[tid] = logf(data[tid]);   // 奇数线程
```

一个 Warp 的 32 个线程一半走 `if`、一半走 `else`：硬件先执行路径 A（16 活跃 / 16 空闲），再执行路径 B（16 活跃 / 16 空闲）—— **总耗时约为无分支的 2 倍**。

判据是「**分支的粒度与 Warp 的对齐程度**」：

| 分支条件 | 代价 |
| --- | --- |
| 按 Warp 边界对齐（前 16 个 Warp 走 A，后 16 个走 B） | **无分歧** —— 每个 Warp 内 32 个线程走同一路径 |
| 按 lane 交替（`tid % 2`） | 每个 Warp 内部对半，**2×** |
| 32 个线程各走不同路径 | **最坏 32×** |

度量指标是 **Branch Efficiency**：

$$\text{Branch Efficiency} = \frac{\text{Non-Divergent Branches}}{\text{Total Branches}} \times 100\%$$

Nsight Compute 能看到每条分支指令的 Warp 活跃线程占比 —— **理想值是每个分支都 32/32**。

> [!warning] 规避策略的优先顺序
> 1. **让分支条件按 Warp 对齐** —— 最根本，且通常免费
> 2. **用算术或查表替换分支** —— 例如 `fmaxf`/`fminf` 代替 `if`
> 3. **重构循环让各线程迭代次数一致** —— 尾块用掩码统一处理，而不是让部分线程提前退出
> 4. 真无法避免时，接受它 —— **关键在于要能量化代价，而不是「猜测这里有分歧」**

一个 Warp 内 32 个线程遇到 `if/else` 时硬件怎么走：

```
  源程序：  if (tid % 2 == 0) A();  else B();

  硬件执行（同一时刻只能发射一条指令，于是两条路径串行）：
    ├── 第 1 段：执行 A() —— 16 个线程活跃，另外 16 个被掩码
    └── 第 2 段：执行 B() —— 另 16 个活跃，前 16 个被掩码

  总耗时约为无分支时的 2 倍。

  反过来，只要分支落在 Warp 边界上就免费：
    Warp 0–15 全走 A、Warp 16–31 全走 B ⇒ 每个 Warp 内部 32/32 同路
```

### Independent Thread Scheduling（Volta+）

Volta 起每个线程有独立的 PC，硬件可以在 lane 之间做更细粒度的调度。**但它不消除分歧的代价** —— 只是让「Warp 内线程互相等待（如 `__syncwarp` 之前的隐式同步）」的行为更宽松。**性能模型仍然是「同一时刻一条指令」**，所以规避策略依旧适用。

## Warp Shuffle：寄存器级的数据交换

### 为什么需要它

传统线程间交换要走共享内存：

```
线程A写入共享内存 → __syncthreads() → 线程B读取
```

两个代价：**占用共享内存**，以及**同步屏障的延迟**。

**Warp Shuffle 让同一 Warp 内的线程直接读彼此的寄存器值**，不经共享内存、不需显式同步。

### 四条指令

```cpp
T __shfl_sync     (unsigned mask, T var, int srcLane,      int width = 32);
T __shfl_up_sync  (unsigned mask, T var, unsigned delta,   int width = 32);
T __shfl_down_sync(unsigned mask, T var, unsigned delta,   int width = 32);
T __shfl_xor_sync (unsigned mask, T var, int laneMask,     int width = 32);
```

| 指令 | 源 lane |
| --- | --- |
| `__shfl_sync` | `srcLane` |
| `__shfl_up_sync` | `laneId - delta` |
| `__shfl_down_sync` | `laneId + delta` |
| `__shfl_xor_sync` | `laneId ^ laneMask`（蝶形配对） |

参数：`mask` 是参与线程掩码（全参与写 `0xFFFFFFFF`）；`width` 是逻辑子 Warp 宽度（2/4/8/16/32），**把 Warp 切成更小的交换域**，这在归约的最后几步很有用。

### Warp 内归约：5 步搞定

```cpp
__device__ float warp_reduce_sum(float val) {
    for (int offset = 16; offset > 0; offset >>= 1)
        val += __shfl_down_sync(0xFFFFFFFF, val, offset);
    return val;                     // 只有 lane 0 持有正确结果
}
```

蝶形归约的过程（以 8 线程演示）：

```
初始:      [1] [2] [3] [4] [5] [6] [7] [8]
offset=4:  [1+5=6] [2+6=8] [3+7=10] [4+8=12] ...
offset=2:  [6+10=16] [8+12=20] ...
offset=1:  [16+20=36]
```

**$\log_2 32 = 5$ 步完成 32 个元素的归约，全程零共享内存、零 `__syncthreads()`。** 这是后面 Reduce 优化的最后一块（见 `06-Reduce 算子优化`）。

`__shfl_down_sync(val, offset)` 的蝶形归约（8 线程示意，offset 依次 4 → 2 → 1）：

```
  lane:     0    1    2    3    4    5    6    7
  初始:     1    2    3    4    5    6    7    8
            │    │    │    │    │    │    │    │
  off=4:    6    8   10   12    ·    ·    ·    ·      lane i 加上 lane i+4
            │    │    │    │
  off=2:   16   20    ·    ·                          lane i 加上 lane i+2
            │    │
  off=1:   36    ·                                    lane i 加上 lane i+1

  ⇒ 5 步（$\log_2 32$）完成 32 元素归约，全程零共享内存、零 __syncthreads()
  ⇒ 结束时只有 lane 0 持有正确结果
```

### Warp 级原语

| 类别 | 指令 | 作用 |
| --- | --- | --- |
| **Vote** | `__all_sync` / `__any_sync` / `__ballot_sync` | 全体判定 / 任一判定 / 得到 32 位掩码 |
| 掩码 | `__activemask()` | 当前活跃 lane 的掩码 |
| 同步 | `__syncwarp()` | Warp 内显式同步 |
| **Match**（Volta+） | `__match_any_sync` / `__match_all_sync` | 找出值相同的 lane 集合 |

`__ballot_sync` 在稀疏算子与流压缩里很常用 —— 它能一次拿到「哪些 lane 满足条件」的位图。

## 相关

- [[01-CUDA 开发环境与第一个 Kernel]] —— 五步流程与索引公式的入门版
- [[03-CUDA 内存模型与访存优化]] —— Block 内可协作用的共享内存
- [[04-Occupancy、同步与原子操作]] —— Block 大小与每 SM 活跃线程数的关系
- `06-Reduce 算子优化` —— Warp Shuffle 在真实算子里的用法
- [[01-GPU 硬件架构与存储层次]] —— SM 与 Warp Scheduler 的硬件侧

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/12-cuda%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B
- https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#programming-model
- https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#warp-shuffle-functions
- https://images.nvidia.com/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf
