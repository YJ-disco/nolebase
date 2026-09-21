---
tags:
  - AI/infra/通信
---

# 集合通信与 NCCL

[[03-多卡互联与集群网络]] 讲的是「线有多宽」。这一篇讲的是**数据在这条线上怎么流** —— 通信原语、算法，以及把这两者落地的库。

分布式训练的本质是多张 GPU 协同完成一个大任务。GPU 之间的数据交换遵循一套标准化的**集合通信原语**。理解这些原语，就能理解各种并行策略里「数据到底怎么流动」。

## 一、点对点：最基础的积木

**Send / Recv** —— 一个进程发送，另一个接收。

```python
import torch.distributed as dist

# 假设已完成 dist.init_process_group(backend="nccl")
if dist.get_rank() == 0:
    tensor = torch.tensor([1.0, 2.0, 3.0]).cuda()
    dist.send(tensor, dst=1)
else:
    tensor = torch.zeros(3).cuda()
    dist.recv(tensor, src=0)
    print(f"rank 1 received: {tensor}")
```

点对点能覆盖的场景很广 —— 流水线并行的 micro-batch 激活值跨机传递就靠它。但当需要「多方同时协作」时，手动逐一调用 Send/Recv 既繁琐又低效，于是有了集合通信原语。

## 二、五大原语

| 原语 | 语义 | 典型用途 |
| --- | --- | --- |
| **Broadcast** | 一对多：root 的数据发给所有 rank | 分发初始模型参数 |
| **Reduce** | 多对一：所有 rank 的数据按元素聚合到 root | Loss 汇总 |
| **AllReduce** | 所有人聚合后所有人都拿到结果 | **数据并行同步梯度** |
| **AllGather** | 每人一片，最后人人拿全部 | ZeRO-3 前向前收集完整参数 |
| **ReduceScatter** | 每人拿到其中一个分片的聚合结果 | ZeRO 中分片存梯度 |

数据流示意（4 个 rank，每份数据 `[1,2,3]` 这类按元素求和）：

```
Broadcast（root=GPU0）
操作前                   操作后
GPU0: [ABCD]      →      GPU0: [ABCD]
GPU1: [    ]             GPU1: [ABCD]
GPU2: [    ]             GPU2: [ABCD]
GPU3: [    ]             GPU3: [ABCD]

AllReduce（sum）
操作前                   操作后
GPU0: [1,2,3]     →      GPU0: [10,20,30]
GPU1: [2,4,6]            GPU1: [10,20,30]
GPU2: [3,6,9]            GPU2: [10,20,30]
GPU3: [4,8,12]           GPU3: [10,20,30]

AllGather
操作前                   操作后
GPU0: [A]         →      GPU0: [A,B,C,D]
GPU1: [B]                GPU1: [A,B,C,D]
GPU2: [C]                GPU2: [A,B,C,D]
GPU3: [D]                GPU3: [A,B,C,D]

ReduceScatter（sum，各 rank 输入相同 [1,2,3,4]）
操作前                   操作后
GPU0: [1,2,3,4]   →      GPU0: [4]     ← 第 0 片的归约结果
GPU1: [1,2,3,4]          GPU1: [8]     ← 第 1 片
GPU2: [1,2,3,4]          GPU2: [12]    ← 第 2 片
GPU3: [1,2,3,4]          GPU3: [16]    ← 第 3 片
```

> **`AllReduce = ReduceScatter + AllGather`。** Ring AllReduce 的算法正是按这个分解实现的 —— 见下一节。

### 通信量对照

设总数据量 $M$、参与节点数 $N$：

| 原语 | 每 rank 发送量 | 每 rank 接收量 |
| --- | --- | --- |
| Broadcast | $M$（root）/ 0 | 0（root）/ $M$ |
| Reduce | 0（root）/ $M$ | $M$（root）/ 0 |
| **AllReduce** | 约 $2M$ | 约 $2M$ |
| AllGather | $(N-1)M/N$ | $(N-1)M/N$ |
| ReduceScatter | $(N-1)M/N$ | $(N-1)M/N$ |

AllReduce 那行的 $2M$ 值得单独记住 —— 它是「AllReduce = ReduceScatter + AllGather」的通信量来源，也是下面 Ring 公式的由来。

### 原语与并行策略的映射

| 并行策略 | 主要原语 | 通信频率 | 通信数据量 |
| --- | --- | --- | --- |
| 数据并行（DDP） | AllReduce | 每层反向后 | 梯度大小 |
| ZeRO-1/2 | ReduceScatter + AllGather | 反向后 + 参数更新后 | 梯度大小 |
| ZeRO-3 | AllGather + ReduceScatter | **每层前向 + 反向** | 参数大小 |
| 张量并行（TP） | AllReduce / AllGather | **每层前向 + 反向** | 激活大小 |
| 流水线并行（PP） | 点对点 Send / Recv | 每 micro-batch | 激活大小 |

> **TP 的通信频率最高**（每层都要），且通信量与激活大小相关 —— 这就是它必须待在 NVLink 范围内的原因，量化估算见 [[03-多卡互联与集群网络]] 第三节。

## 三、Ring AllReduce：带宽最优

### 朴素方案的瓶颈

最直觉的做法是「中心化归约」：所有 GPU 把梯度发给 rank 0，rank 0 求和后广播回去。节点数一多，**rank 0 成为瓶颈，其他链路全程空闲**，带宽利用率极低。

### 环形拓扑

把所有节点排成一个逻辑环，每个节点向下一个发送、从上一个接收，让**所有链路同时工作**：

```
rank0 → rank1 → rank2 → rank3 → rank0
```

整个过程分两阶段。以 4 个节点、每个持有 $[a,b,c,d]$ 四份数据为例：

1. **ReduceScatter（$N-1$ 轮）** —— 每个 rank 的数据均分为 $N$ 块。每轮把一个块传给下一个 rank，同时接收上一个 rank 的块并与本地对应块累加。$N-1$ 轮后，**每个 rank 持有完全归约后的 $1/N$ 数据块**。
2. **AllGather（$N-1$ 轮）** —— 再传 $N-1$ 轮，每个 rank 把自己那块已归约的数据沿环传出，最终所有 rank 都拥有完整结果。

### 通信量公式

每个 rank 在 ReduceScatter 阶段发送 $N-1$ 次、每次 $M/N$ 字节；AllGather 阶段同理：

$$\text{每 rank 通信量} = 2 \times (N-1) \times \frac{M}{N} = \frac{2(N-1)}{N} \times M$$

$N$ 较大时 $\frac{N-1}{N} \to 1$，通信量趋近 $2M$，**与节点数基本无关**。这就是 Ring AllReduce 是**带宽最优（bandwidth-optimal）**的原因 —— 线性扩展性极佳。

> [!warning] 带宽最优 ≠ 延迟最优
> 整个过程需要 $2(N-1)$ **轮**传递，**延迟随节点数线性增长**。在「节点很多但数据量很小」的场景（例如同步一个标量 loss）里，延迟而非带宽成为瓶颈。

## 四、Tree AllReduce：低延迟的另一条路

Tree AllReduce 以树形组织通信，把延迟降到 $O(\log N)$：

- **Reduce 阶段**（叶 → 根）：逐层归约，$\lceil \log_2 N \rceil$ 步
- **Broadcast 阶段**（根 → 叶）：逐层广播，同样 $\lceil \log_2 N \rceil$ 步

| | Ring AllReduce | Tree AllReduce |
| --- | --- | --- |
| 端到端延迟 | $O(N)$ | **$O(\log N)$** |
| 带宽利用率 | 接近 100%（大数据量） | 较低（根节点是瓶颈） |
| 最适场景 | 梯度同步（大张量） | 标量指标同步（loss、step） |

**NCCL 会根据数据大小自动在两者之间切换**，这个决策对上层框架完全透明。也可用 `NCCL_ALGO=Ring` 或 `NCCL_ALGO=Tree` 强制指定（调优对比时有用）。

## 五、通信与计算 Overlap

训练性能的理论上限是「计算和通信完全并行」—— 把通信延迟藏进计算时间里。核心思路：**在 GPU 反向传播计算后续层梯度的同时，并行传输已就绪的前面层梯度**。

```
反向传播：第 L 层梯度就绪
   → 异步触发第 L 层 AllReduce
反向传播：第 L-1 层梯度就绪        ← 与上面的 AllReduce 并行
   → 异步触发第 L-1 层 AllReduce
反向传播：第 L-2 层梯度就绪
...
```

PyTorch DDP 通过 **bucket 机制**自动实现：模型参数被分组为若干 bucket，一个 bucket 内所有参数的梯度算完就立即异步触发 AllReduce，不必等整个模型的梯度就绪。

```python
from torch.nn.parallel import DistributedDataParallel as DDP

model = DDP(model, device_ids=[local_rank], bucket_cap_mb=25)
```

> [!warning] `bucket_cap_mb` 是一个双向权衡：**太大** → 要积累更多梯度才触发，等待时间长，overlap 效果差；**太小** → 通信次数多，每次通信的固定启动开销占比上升。默认 **25 MB** 是较好的起点，可按模型层大小调整。

> 这条「用更多次小通信换重叠机会」的思路，与 ZeRO 分片梯度/参数是同一个动机的两种实现 —— 一个在时间维度切，一个在数据维度切。

## 六、NCCL

**NCCL（NVIDIA Collective Communications Library）** 是 NVIDIA 的 GPU 集合通信库。PyTorch、DeepSpeed、Megatron-LM 等所有主流框架底层的多卡通信都由它驱动。

它的角色是「智能调度中心」：你只说「帮我做个 AllReduce」，NCCL 负责探测硬件拓扑、选择最优算法与传输路径。

核心能力：**自动拓扑感知**（检测 NVLink / PCIe / IB 连接，选最优路径）、全套集合通信原语、单机多卡与多机多卡透明支持、与 CUDA Stream 配合做通信-计算重叠。

### 传输后端

| 传输方式 | 场景 | 说明 |
| --- | --- | --- |
| NVLink / NVSwitch | 单机卡间 | 最高带宽，优先使用 |
| PCIe P2P | 单机卡间（无 NVLink） | 带宽较低 |
| InfiniBand Verbs | 多机 | RDMA，低延迟高带宽 |
| RoCE | 多机 | 以太网上的 RDMA |
| Socket (TCP) | 回退 | 性能最差，仅兼容 |

### PyTorch 中的用法

```python
import torch.distributed as dist

# AllReduce：所有卡求和，结果每张卡都拿到
tensor = torch.ones(1024, 1024, device=f"cuda:{dist.get_rank()}")
dist.all_reduce(tensor, op=dist.ReduceOp.SUM)

# AllGather：收集所有卡的数据拼成完整张量
world_size = dist.get_world_size()
local_chunk = torch.randn(256, device=f"cuda:{dist.get_rank()}")
gathered = [torch.zeros(256, device=f"cuda:{dist.get_rank()}") for _ in range(world_size)]
dist.all_gather(gathered, local_chunk)

# ReduceScatter：先归约再分片
chunk_size = 1024 // world_size
output = torch.zeros(chunk_size, 1024, device=f"cuda:{dist.get_rank()}")
dist.reduce_scatter(output, list(tensor.chunk(world_size)), op=dist.ReduceOp.SUM)
```

### 关键环境变量

**调试**：

| 变量 | 说明 | 常用值 |
| --- | --- | --- |
| `NCCL_DEBUG` | 日志级别 | `VERSION` / `WARN` / `INFO` / `TRACE` |
| `NCCL_DEBUG_SUBSYS` | 过滤子系统 | `ALL` / `NET` / `INIT` |
| `NCCL_TOPO_DUMP_FILE` | 导出检测到的拓扑为 XML | `/tmp/nccl_topo.xml` |

**算法与协议**：

| 变量 | 说明 | 可选值 |
| --- | --- | --- |
| `NCCL_ALGO` | 强制通信算法 | `Ring` / `Tree` |
| `NCCL_PROTO` | 强制传输协议 | `LL` / `LL128` / `Simple` |

**网络**：

| 变量 | 说明 | 示例 |
| --- | --- | --- |
| `NCCL_SOCKET_IFNAME` | 指定 TCP/IP 接口 | `eth0` / `^lo` |
| `NCCL_IB_HCA` | 指定 IB HCA 设备 | `mlx5_0` |
| `NCCL_NET_GDR_LEVEL` | GPUDirect RDMA 拓扑级别 | 0（禁用）– 5 |
| `NCCL_P2P_DISABLE` | 禁用 P2P（NVLink / PCIe） | `1` |
| `NCCL_SHM_DISABLE` | 禁用共享内存传输 | `1` |
| `NCCL_BUFFSIZE` | 通信缓冲区大小（默认 4 MB） | `16777216`（16 MB） |

```bash
NCCL_DEBUG=INFO python train.py                    # 看拓扑识别与算法选择
NCCL_IB_HCA=mlx5_0 NCCL_NET_GDR_LEVEL=2 python train.py
NCCL_ALGO=Ring NCCL_DEBUG=INFO python train.py     # 强制算法，对比测试
```

`NCCL_IB_HCA` 与 `NCCL_SOCKET_IFNAME` 是多机训练最常见的两个必设项 —— 机器上有多张网卡时，NCCL 选错网卡会让性能掉到原理论值的一个零头。

### 用 nccl-tests 量实际带宽

```bash
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
make MPI=1 MPI_HOME=/usr/local/mpi CUDA_HOME=/usr/local/cuda NCCL_HOME=/usr/local/nccl

./build/all_reduce_perf -b 8 -e 256M -f 2 -g 8          # 单机 8 卡
mpirun -np 16 --hostfile hosts -x NCCL_IB_HCA=mlx5_0 \
  ./build/all_reduce_perf -b 8 -e 256M -f 2 -g 8        # 多机
```

输出：

```
#  size(B)    count   type    redop    time(us)  algbw(GB/s)  busbw(GB/s)
   8388608  2097152   float     sum      285.4       29.4         51.5
  67108864 16777216   float     sum     1823.0       36.8         64.4
 268435456 67108864   float     sum     5210.0       51.5         90.1
```

两个指标必须分清：

| 指标 | 定义 | 用来看什么 |
| --- | --- | --- |
| **algbw**（算法带宽） | 数据量 / 时间 | 应用层看到的吞吐 |
| **busbw**（总线带宽） | $\text{algbw} \times \frac{2(N-1)}{N}$ | **硬件链路的实际利用率** |

> **对照硬件理论带宽要看 `busbw`。** 那个 $\frac{2(N-1)}{N}$ 修正系数正是 Ring AllReduce 的通信量倍数 —— 用不用它，判断「带宽是否正常」的结论会差近一倍。

**经验参考**：8× H100 SXM 单机 AllReduce 的 busbw 应接近 **约 850 GB/s**（NVLink 4.0 理论 900 GB/s 的约 95%）。实测远低于此值说明 NVLink 拓扑或 NCCL 配置有问题。

## 相关

- [[03-多卡互联与集群网络]] —— 本篇依赖的硬件层：NVLink、InfiniBand 与拓扑判据
- [[01-GPU 硬件架构与存储层次]] —— 互联带宽在四个核心指标里的位置
- [[01-数值计算与精度]] —— 梯度归约顺序为什么会影响结果

## 参考

- **NVIDIA NCCL Documentation**（原语 API、传输后端、环境变量）：https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/
- **NCCL Collective Operations API**：https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/api/colls.html
- **NCCL Tests**（`algbw` / `busbw` 的定义与测法）：https://github.com/NVIDIA/nccl-tests
- **Bringing HPC Techniques to Deep Learning**（Ring AllReduce 的原始介绍）：https://andrew.gibiansky.com/blog/machine-learning/baidu-allreduce/
- **PyTorch Distributed Overview**：https://pytorch.org/docs/stable/distributed.html
- **AIInfraGuide 集群通信网络与 NCCL**（来源教程；本笔记取其第 3–9 节的通信库与算法部分，互联硬件部分拆入 [[03-多卡互联与集群网络]]）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/communication/collective-communication-primer
