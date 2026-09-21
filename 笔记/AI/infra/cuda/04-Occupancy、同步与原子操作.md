---
tags:
  - AI/infra/CUDA
---

# Occupancy、同步与原子操作

前三篇讲的是「怎么让数据搬得少」和「怎么让线程映射对齐硬件」。这一篇讲**资源与并发控制** —— 一个 SM 能塞多少 Block、线程之间怎么协调、竞争同一地址时代价有多大。

## 一、Occupancy 是什么

$$\text{Occupancy} = \frac{\text{每 SM 上活跃 Warp 数}}{\text{每 SM 支持的最大 Warp 数}}$$

A100 上每 SM 最多 64 个 Warp（2048 线程）。若资源限制只能驻留 32 个，Occupancy = 50%。

### 它为什么重要：延迟隐藏

GPU 靠 **Warp 切换**掩盖访存延迟：

```
Warp 0: [计算] [等内存 ... 400 cycles ...] [计算]
Warp 1:        [计算] [等内存 ... 400 cycles ...] [计算]
Warp 2:               [计算] [等内存 ... 400 cycles ...]
```

**活跃 Warp 足够多，调度器就总能找到就绪的 Warp 填满等待期。**

需要多少 Warp 才够？粗算：

$$\text{所需 Warp 数} \ge \frac{\text{内存延迟}}{\text{每条指令的执行周期}} = \frac{400}{4} = 100$$

**但一个 SM 最多只有 64 个 Warp** —— 所以实际中**永远无法完全隐藏访存延迟**。这条不等式说明了为什么 Occupancy 值得追求，同时也预告了下一节：它不可能是唯一目标。

## 二、SM 的资源池

| 资源 | Ampere (A100) | Ada (RTX 4090) | Hopper (H100) |
| --- | --- | --- | --- |
| 每 SM 最大线程数 | **2048** | **1536** | 2048 |
| 每 SM 最大 Warp 数 | **64** | **48** | 64 |
| 每 SM 最大 Block 数 | 32 | 16 | 32 |
| 每 SM 寄存器总数 | 65536 | 65536 | 65536 |
| 每线程最大寄存器数 | 255 | 255 | 255 |
| 每 SM 共享内存上限 | 164 KB | 100 KB | **228 KB** |
| 每 Block 共享内存上限 | 163 KB | 99 KB | 227 KB |
| 每 Block 最大线程数 | 1024 | 1024 | 1024 |

> [!warning] 资源以 Block 为单位分配
> **一个 Block 要么完整进入 SM，要么完全不进入** —— 不存在半个 Block 驻留。所以如果某个 Block 占用的资源刚好超过 SM 剩余空间的一半多一点，**整个剩余空间就浪费了**。这是「Block 大小与资源用量要一起调」的原因，也是 Occupancy 常常不是整数百分比的原因。

## 三、三个限制因素

### 寄存器

最常见的限制因素。每个 Warp 的寄存器分配有**粒度约束：以 256 个为一组**：

$$\text{每 Warp 分配} = \left\lceil \frac{\text{每线程寄存器数} \times 32}{256} \right\rceil \times 256$$

A100（65536 寄存器/SM）上的手算：

| 每线程寄存器 | 每 Warp 实际分配 | SM 可容纳 Warp | Occupancy |
| --- | --- | --- | --- |
| 32 | 1024 | 64 | **100%** |
| 48 | 1536 | 42 | 约 65% |
| 64 | 2048 | 32 | 50% |
| 128 | 4096 | 16 | 25% |
| 255 | 8160 → **对齐到 8192** | 8 | **12.5%** |

**看一眼最后一行**：`255 × 32 = 8160`，但因为 256 粒度对齐，实际分配 8192 —— **每线程 255 个寄存器时，有 32 个被白白对齐掉了**。

两种限制手段：

```cpp
// 方式一：__launch_bounds__(maxThreadsPerBlock, minBlocksPerMultiprocessor)
__global__ void __launch_bounds__(256, 2) my_kernel(...);

// 方式二：编译选项
//   nvcc -maxrregcount=32
```

取值方向取决于 kernel 的性质 —— 这是后面那张「Occupancy 不是越高越好」的表的直接应用：

| kernel 性质 | `__launch_bounds__` 第二参数 | 目的 |
| --- | --- | --- |
| 访存密集 / 带宽瓶颈 | **取大值** | 提升 occupancy，换取更多延迟隐藏 |
| 计算密集 / FLOP 瓶颈 | **取 1** | 保留寄存器，提升数据复用与 ILP |

`--ptxas-options=-v` 能打印出每个 kernel 实际用了多少寄存器 —— **调 Occupancy 的第一步是看这个数，不是猜**。

### 共享内存

每 Block 的共享内存用量直接决定一个 SM 能驻留几个 Block。**申请了就要占，哪怕实际只用了其中一部分**。Hopper 的 228 KB 是四代里最大的，这也是它能做更大 tile 的原因。

### Block 大小

Block 越大，寄存器与共享内存的需求也越大，可能反而降低 Occupancy。而 Block 太小则受「每 SM 最大 Block 数」（16–32）限制 —— **Block 数上限是另一条独立的约束**：即使线程数远远没用满，Block 数到顶了也塞不进去更多。

## 四、Occupancy 不是越高越好

**这是 CUDA 优化里最重要的一个反直觉结论。**

| 版本 | 寄存器/线程 | 共享内存/Block | Occupancy | 实测 GFLOPS |
| --- | --- | --- | --- | --- |
| Version A | 32 | 8 KB | **100%** | 800 |
| Version B | 96 | 48 KB | **33%** | **1200** |

**Version B 的 Occupancy 只有 A 的三分之一，性能却高 50%。** 原因：它用更多寄存器和共享内存换来**更高的数据复用率** —— 每次从全局内存加载的数据被反复使用多次，总的全局访存量反而更少。

### 盲目追求高 Occupancy 的三种代价

| 代价 | 机制 |
| --- | --- |
| **寄存器溢出（Spilling）** | 用 `maxrregcount` 强行压低寄存器，编译器把变量溢出到 Local Memory（**实质是全局内存 + L1 缓存**），速度慢几十倍 |
| **共享内存复用不足** | 共享内存给得少 → tile 更小 → 复用率降低 → **全局访存量反而增加** |
| **缓存抖动（Cache Thrashing）** | 太多活跃 Warp 争抢有限的 L1/L2，cache miss 率上升 |

### 什么时候该在意

| 该在意（低复用、纯靠延迟隐藏） | 可以接受低 Occupancy（高复用，资源换效率） |
| --- | --- |
| Elementwise 操作（向量加法、激活函数） | **GEMM** —— 大量数据复用 |
| 归约操作（每个元素只读一次） | **FlashAttention** —— 用更多共享内存减少 HBM 访问 |
| 简单 Stencil | 计算密集型 kernel |

> **黄金法则**：不要设固定的 Occupancy 目标（比如「必须到 75%」）。性能是 Occupancy、**ILP（指令级并行）**、**数据复用率**、访存效率的函数 —— 降低 Occupancy 但提升复用或 ILP，整体可能更好。
>
> 判据是**用 profiler 测吞吐**，不是看 Occupancy 百分比。这条与 [[03-CUDA 内存模型与访存优化]] 「不要在代码里猜 Bank Conflict」是同一条纪律。

## 五、同步的三个层级

| 层级 | 手段 | 范围 |
| --- | --- | --- |
| Warp | `__syncwarp()` / Shuffle 的隐式同步 | 32 线程 |
| **Block** | `__syncthreads()` | Block 内所有线程 |
| Grid | Cooperative Groups 的 `grid.sync()` | 全部线程（需特殊启动） |

### `__syncthreads()`

语义是「**Block 内所有线程都到达这一点，才继续往下**」，同时兼具**内存屏障**作用 —— 保证之前对共享内存的写在之后对所有线程可见。

```cpp
__shared__ float tile[256];
tile[threadIdx.x] = input[idx];
__syncthreads();                    // 必须：否则线程可能读到别人还没写的旧值
float v = tile[255 - threadIdx.x];
```

> [!warning] `__syncthreads()` 的三个陷阱
> 1. **必须在所有线程都执行的代码路径上。** 若写在 `if (threadIdx.x < 32)` 里，**其余线程永远到不了这个屏障，整个 kernel 死锁**（表现为挂起而不是报错）
> 2. **它是 Block 级，跨 Block 无效。** 多 Block 之间的依赖要靠 kernel 拆分或原子/栅栏，不能指望它
> 3. **有实在的代价。** 它迫使 warp 等待最慢的 warp，**分歧严重的 kernel 上同步点会成为瓶颈**

### Warp 级同步

- **Pre-Volta**：Warp 内是隐式同步的（lockstep），不需要显式同步
- **Volta+**：Independent Thread Scheduling 让每个线程有独立 PC，**隐式同步不再有保证**，需要时显式调用 `__syncwarp()`

**这是一个常见的移植坑** —— 在 Volta 之前能跑对的代码（依赖隐式同步），到新架构上可能出现竞态。

### Memory Fence

`__syncthreads()` 管 Block 内。跨 Block 的可见性需要内存栅栏，按作用范围分三级：

| 栅栏 | 作用范围 |
| --- | --- |
| `__threadfence_block()` | Block 内 |
| `__threadfence()` | 设备内（全局内存可见） |
| `__threadfence_system()` | 含主机内存 |

经典用例是**跨 Block 归约的标志位**：最后完成的那个 Block 负责把局部结果汇总，它需要先 `__threadfence()` 保证自己的结果对其他 Block 可见，再用 `atomicInc` 争抢「最后一个」的角色。

## 六、原子操作

原子操作是**不可分割的读-改-写**。非原子版本会丢更新：

```
线程A：读 counter=5, 加1, 写回6
线程B：读 counter=5, 加1, 写回6    ← A 的更新丢了
```

### 清单

| 操作 | 函数 | 支持类型 |
| --- | --- | --- |
| 加 / 减 | `atomicAdd` / `atomicSub` | int, unsigned, **float, double** |
| 最小 / 最大 | `atomicMin` / `atomicMax` | int, unsigned |
| 交换 | `atomicExch` | int, unsigned, float |
| **CAS** | `atomicCAS(addr, compare, val)` | int, unsigned, ull |
| 位运算 | `atomicAnd` / `atomicOr` / `atomicXor` | int, unsigned |
| 递增 / 递减 | `atomicInc` / `atomicDec` | unsigned |

### 代价完全取决于冲突程度

| 场景 | 代价 |
| --- | --- |
| 各线程操作**不同地址** | 接近非原子操作 |
| 同 Warp 内多线程竞争**同一地址** | 串行化，最差 **32×** |
| 跨 Block 大量线程竞争同一地址 | 极慢，可能成为全局瓶颈 |

底层的延迟差别值得记：

| 位置 | 延迟 |
| --- | --- |
| **全局内存原子** | **约 400–600 cycles**（要走到 L2 或 DRAM） |
| **共享内存原子** | **约 20–100 cycles**（在 SM 内部完成） |

**差一个数量级** —— 这就是所有优化策略都围绕「把原子操作往共享内存搬、往少冲突的方向分层」的原因。

### atomicCAS 是万能原语

```cpp
// 语义：old = *addr; if (old == compare) *addr = val; return old;
__device__ double atomicAddDouble(double* addr, double val) {
    unsigned long long* a = (unsigned long long*)addr;
    unsigned long long old = *a, assumed;
    do {
        assumed = old;
        old = atomicCAS(a, assumed,
                __double_as_longlong(__longlong_as_double(assumed) + val));
    } while (assumed != old);        // CAS 失败 = 别人改过，重试
    return __longlong_as_double(old);
}
```

**所有原子操作都可以用 CAS 实现。** 但注意那个 `while` —— **CAS 循环在高竞争下会反复重试，性能急剧下降**。这从机制上说明了为什么「减少原子冲突」比「换更快的原子指令」更有效。

### 四种减少冲突的策略

| 策略 | 做法 |
| --- | --- |
| **分层归约** | 不让所有线程直接 `atomicAdd` 到同一地址 —— 先在 Block 内归约成一个值，再由每 Block 一个线程去全局原子累加。**冲突从 N 降到 Block 数** |
| **共享内存原子** | 把全局内存的原子操作搬到共享内存，利用 20–100 cycles 的低延迟 |
| **私有化（Privatization）** | 每个 Block（或每个 warp）维护一份私有的部分和，最后才合并 —— 把「读改写同一地址」变成「各地独立累加」 |
| **Warp 聚合原子**（CUDA 9+） | 硬件自动把同一 Warp 内对同一地址的原子操作聚合成一次 —— 但**要求这些访问在同一个 Warp 内连续**，需要写法配合 |

「分层归约」与「私有化」是同一个思路的两个尺度：**把 O(N) 的冲突压成 O(块数)**。这是后面 Reduce 算子优化的核心动机（见 [[05-Reduce 算子优化]]）。

## 七、Cooperative Groups

`__syncthreads()` 只能同步 Block。**Cooperative Groups** 提供了更一般的线程组抽象，其中 `grid_group::sync()` 能做 **Grid 级同步**。

代价是必须用 `cudaLaunchCooperativeKernel` 启动，且 **Grid 大小受「一次能同时驻留多少 Block」限制** —— 也就是说，只有全部 Block 能同时在 GPU 上驻留时，Grid 同步才有意义。这使它的可用规模受限，不能替代「拆成多个 kernel」的常规做法。

| 组类型 | 范围 |
| --- | --- |
| `thread_block` | 一个 Block |
| `thread_block_tile<N>` | Block 内一个 N 线程的 tile |
| `coalesced_group` | Block 内当前活跃的线程 |
| `grid_group` | **整个 Grid** |

## 相关

- [[02-CUDA 编程模型与执行模型]] —— Block 大小、Warp Shuffle、Divergence
- [[03-CUDA 内存模型与访存优化]] —— 共享内存用量是 Occupancy 的第二大限制
- [[01-GPU 硬件架构与存储层次]] —— 内存延迟 400 与指令周期的量级来源
- [[01-数值计算与精度]] —— 原子累加的顺序不确定与浮点非结合性

## 参考

- **AIInfraGuide 2.3 Occupancy 与资源分配 / 2.4 同步与原子操作**（Occupancy 定义与延迟隐藏的不等式、三代架构资源对照、寄存器的 256 粒度与手算表、`__launch_bounds__` 的取值方向、Occupancy 反例与三种代价、同步三级与 `__syncthreads` 陷阱、Memory Fence 三级、原子清单与代价、CAS 万能原语、四种降冲突策略、Cooperative Groups）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/23-occupancy%E4%B8%8E%E8%B5%84%E6%BA%90%E5%88%86%E9%85%8D
- **CUDA C++ Programming Guide — Synchronization Functions / Atomics**：https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#atomics
- **CUDA C++ Best Practices Guide — Occupancy**：https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#occupancy
- **CUDA C++ Programming Guide — Cooperative Groups**：https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#cooperative-groups
- **CUDA Occupancy Calculator**（API 与表格）：https://docs.nvidia.com/cuda/cuda-occupancy-calculator/index.html
