---
tags:
  - AI/infra/架构
---

# NVIDIA GPU 架构演进：Volta 到 Blackwell

做 AI Infra 经常要回答这类问题：训练跑不动该升级什么卡？推理延迟高卡在哪里？FP8 和 BF16 到底选哪个？答案大多藏在架构本身的设计决策里 —— **每一代架构都在回应上一代暴露出来的瓶颈**。

## 一、五代架构全景

| 架构 | 年份 | 代表产品 | 核心突破 | 解决了什么问题 |
| --- | --- | --- | --- | --- |
| Volta | 2017 | V100 | **首次引入 Tensor Core** | CUDA Core 做矩阵乘太浪费 |
| Turing | 2018 | T4 | **INT8/INT4 Tensor Core**、RT Core | 推理负载用不上 FP16 精度，算力浪费 |
| Ampere | 2020 | A100 | 第三代 Tensor Core、**TF32**、**BF16**、**MIG** | FP16 会溢出；一卡一任务利用率低 |
| Hopper | 2022 | H100 | **FP8**、Transformer Engine、**NVLink 4.0**、TMA | LLM 时代的算力饥渴与通信瓶颈 |
| Blackwell | 2024 | B200 / GB200 | 第五代 Tensor Core、**FP4**、**NVLink 5.0**、双芯封装 | 单 die 算力增长追不上模型膨胀 |

三条贯穿全程的主线：**精度格式不断下探**（FP32 → FP16 → BF16/TF32 → FP8 → FP4）、**互联带宽持续翻倍**（NVLink 300 GB/s → 1,800 GB/s）、**软硬件协同加深**（从 Tensor Core 到 Transformer Engine 的自动精度调度）。

## 二、Volta（2017）：AI 加速的开端

**Volta 之前**，深度学习训练靠 CUDA Core 做通用浮点运算 —— 什么都能算，但没对矩阵乘法做专门优化。

**关键创新是 Tensor Core**。传统 CUDA Core 每周期做一次标量乘加（FMA）；一个 Volta Tensor Core 每周期完成一个 $4 \times 4 \times 4$ 的混合精度矩阵乘累加：

$$D = A \times B + C$$

其中 $A$、$B$ 是 FP16，$C$、$D$ 是 FP16 或 FP32。一次操作完成 64 次乘加。

| V100 规格 | 值 |
| --- | --- |
| SM 数 | 80 |
| Tensor Core 数 | **640**（80 SM × 8） |
| FP16 Tensor 算力 | 125 TFLOPS |
| FP32 CUDA 算力 | 15.7 TFLOPS |
| 显存 | 16 / 32 GB HBM2 |
| 显存带宽 | 900 GB/s |
| NVLink | 2.0，300 GB/s |

**实践意义**：Volta 开启了一个编程范式的转变 —— 想用满 Tensor Core，矩阵维度需要是 8 的倍数，数据类型要用混合精度（FP16 计算 + FP32 累加）。这是 NVIDIA AMP（Automatic Mixed Precision）的硬件基础，也是「loss scaling」这套机制的起点。

## 三、Turing（2018）：为推理加速

Volta 面向训练。到了部署阶段，推理对延迟与吞吐更苛刻，而**很多模型用 INT8 就能保持足够精度** —— Volta 的 Tensor Core 只支持 FP16，在推理场景下算力利用不充分。

**关键创新**：

- **INT8 / INT4 Tensor Core** —— 第二代 Tensor Core 新增整数精度，推理吞吐相对 FP16 可再翻倍
- **RT Core** —— 光追加速单元（AI Infra 很少涉及）
- **低功耗推理定位** —— T4 的 TDP 仅 **70W**

| T4 规格 | 值 |
| --- | --- |
| Tensor Core 数 | 320 |
| FP16 Tensor 算力 | 65 TFLOPS |
| INT8 Tensor 算力 | 130 TOPS |
| 显存 | 16 GB GDDR6 |
| 显存带宽 | 320 GB/s |
| TDP | 70W |

**实践意义**：Turing 推动了**量化**在工业界的普及 —— TensorRT 的 INT8 量化 pipeline 正是以 Turing 为目标设计的。T4 的绝对算力不如 V100，但低功耗 + 高 INT8 吞吐让它成为推理部署的经典选择，很多云厂商的推理实例至今仍在用。

## 四、Ampere（2020）：数据中心 AI 的分水岭

到 2020 年模型规模急剧膨胀（GPT-3 有 1750 亿参数），两个痛点突出：**精度格式不够用**（FP16 在某些模型上溢出，FP32 又太慢）、**单卡利用率不高**（一卡一任务，多租户场景浪费）。

### TF32：不改代码的加速

TF32 取 **FP32 的 8 位指数**（保持数值范围）+ **FP16 的 10 位尾数**，共 19 位：

| 格式 | 符号位 | 指数位 | 尾数位 | 总位宽 |
| --- | --- | --- | --- | --- |
| FP32 | 1 | 8 | 23 | 32 |
| **TF32** | 1 | 8 | **10** | 19 |
| FP16 | 1 | 5 | 10 | 16 |
| BF16 | 1 | 8 | **7** | 16 |

> TF32 在 Ampere 上**默认开启**。在 A100 上跑 FP32 的 `torch.matmul` 时，Tensor Core 会自动用 TF32 加速 —— 不需要改代码，代价是乘法精度降低（累加仍在 FP32）。

### BF16：LLM 训练的默认精度

BF16 与 FP32 共享 8 位指数，动态范围一致，因此**训练时不易溢出**，同时把显存与通信量减半。**大部分 LLM 训练从 Ampere 时代起以 BF16 为默认训练精度** —— 这条切换的影响远大于算力本身的提升，因为它同时省显存、省带宽、省通信量。

### MIG：一卡多用

**MIG（Multi-Instance GPU）** 把一块 A100 物理隔离为最多 **7 个**独立实例，每个实例有独立的显存、缓存和计算资源。

```bash
nvidia-smi mig -lgip          # 查看支持的 MIG 配置
sudo nvidia-smi mig -cgi 9,9 -C   # 创建一个 3g.20gb 实例
nvidia-smi mig -lgi           # 查看已创建实例
```

适用：多推理服务共享一卡、开发调试多人共用。**不适用**：需要完整 GPU 算力的大模型训练。

| A100 SXM 规格 | 值 |
| --- | --- |
| SM 数 | 108 |
| Tensor Core 数 | 432 |
| FP16 / BF16 Tensor 算力 | 312 TFLOPS |
| TF32 Tensor 算力 | 156 TFLOPS |
| FP32 CUDA 算力 | 19.5 TFLOPS |
| 显存 | 40 / 80 GB HBM2e |
| 显存带宽 | 2,039 GB/s（80GB 版） |
| NVLink | 3.0，600 GB/s |

第三代 Tensor Core 支持的精度矩阵大幅扩展：FP16、BF16、TF32、INT8、INT4、Binary（1-bit），覆盖训练到推理全场景。

## 五、Hopper（2022）：为大模型而生

LLM 时代带来双重压力：**算力饥渴**（千亿参数要成百上千卡）与**通信瓶颈**（AllReduce / All-to-All 成主要瓶颈）。Hopper 的重要特性几乎都在回应这两条。

### FP8 与 Transformer Engine

第四代 Tensor Core 支持 **FP8**（E4M3 / E5M2），训练精度从 BF16 的 16 位压到 8 位，理论算力翻倍。

| 格式 | 指数位 | 尾数位 | 适用 |
| --- | --- | --- | --- |
| **E4M3** | 4 | 3 | 前向（精度优先） |
| **E5M2** | 5 | 2 | 反向（范围优先） |

精度降低会不会伤模型质量？**Transformer Engine** 就是配套答案：它在每层计算前动态分析张量数值分布，自动决定这一层用 FP8 还是 BF16/FP16 —— 数值稳定的层挂高速档，数值敏感的层自动降档。

```python
import transformer_engine.pytorch as te

model.layer = te.Linear(hidden_size, hidden_size)

with te.fp8_autocast(enabled=True):
    output = model(input_data)
```

实践中 **FP8 + Transformer Engine 把训练吞吐提升 30%–60%**（取决于模型结构与数值分布）。

### NVLink 4.0 与新的执行模型

| 指标 | Ampere (A100) | Hopper (H100) | 变化 |
| --- | --- | --- | --- |
| NVLink 带宽（双向） | 600 GB/s | 900 GB/s | 1.5× |
| NVSwitch 范围 | 节点内 8 卡 | 节点内 8 卡 + 跨节点 NVLink Network | 新增 |

执行模型上加了两个东西，后续 CUDA 章节会用到：

- **Thread Block Cluster** —— 编程模型新增一层，允许多个 Thread Block 协作，利用 SM 间的分布式共享内存
- **TMA（Tensor Memory Accelerator）** —— 硬件异步拷贝单元，把数据搬运从计算流水线解耦，大幅减少地址计算占用的寄存器

### H100 两种封装，差距很大

| | H100 SXM5 | H100 PCIe |
| --- | --- | --- |
| SM 数 | **132** | 114 |
| Tensor Core 数 | **528** | 456 |
| FP16 Tensor 算力 | 989 TFLOPS | 约 800 TFLOPS |
| FP8 Tensor 算力 | 1,979 TFLOPS | 约 1,600 TFLOPS |
| TF32 Tensor 算力 | 495 TFLOPS | 378 TFLOPS |
| 显存带宽 | **3.35 TB/s** | **2 TB/s** |
| NVLink | 4.0，900 GB/s（8 卡全互联） | 4.0，仅 2 卡桥接 |

> [!warning] 选型务必确认封装
> SXM 与 PCIe 同名不同性能，**显存带宽差 1.7 倍，NVLink 组网能力差一个量级**。同一个型号报出来的价格差一大截，原因往往在这里。做容量规划时不能只看型号。

另有一条中国市场特供的事实值得记：H800 的 FP16 算力与 H100 相同（989 TFLOPS），**但 NVLink 带宽被压到 400 GB/s** —— 对张量并行的影响是直接的（见 [[03-多卡互联与集群网络]]）。

### H200：长序列推理的答案

| | H100 | H200 |
| --- | --- | --- |
| 显存 | 80 GB HBM3 | **141 GB HBM3e** |
| 显存带宽 | 3.35 TB/s | **4.8 TB/s** |
| 算力 | 相同 | 相同 |

**只是显存和带宽变大了** —— 这恰好对应 LLM 推理的瓶颈：KV Cache 吃显存（见 [[10-KV Cache 与推理优化]]），Decode 吃带宽。H200 是「不为算力买单，为上下文长度和并发买单」的定位。

## 六、Blackwell（2024）：万亿参数时代的基石

单 die 的算力增长已经跟不上模型膨胀。Blackwell 的思路是转向**芯片互联层面的突破**。

### 双芯封装

B200 由**两块 die 通过 10 TB/s 的片间互联**封装在同一块芯片上，对外表现为一块完整的 GPU。代价是编程模型上需要注意跨 die 的带宽与延迟特征 —— 这与「SM 内 vs SM 间」的差异是同一类问题，只是又加了一层。

### 其他关键变化

| 项 | 内容 |
| --- | --- |
| **第五代 Tensor Core** | 首次硬件支持 **FP4**；最大单 CTA MMA 原子 m128n256k16 |
| **累加器落到 TMEM** | Blackwell 的 UMMA 不再往线程寄存器累加，而是写进专用 Tensor Memory —— 因为 m128n256k16 的累加器有 32,768 个 FP32 值，按 warpgroup 分是每线程 256 个，**超过 255 的寄存器上限** |
| **NVLink 5.0** | 双向 1,800 GB/s，通过 NVLink Switch 可连 **72 块 GPU** |
| **HBM3e** | 单卡最高 192 GB，8 TB/s |
| **第二代 Transformer Engine** | 更细粒度的 FP8/FP4 动态调度 |
| **机密计算** | 硬件级 GPU TEE |

> NVLink 5.0 的意义不只是带宽翻倍：**72 卡的互联域意味着「一个 NVLink 域」的范围从 8 卡扩到 72 卡**。过去必须精心设计通信拓扑来规避的瓶颈，现在可以用更大范围的机内带宽直接碾过 —— 这会直接改变张量并行的可用规模（见 [[03-多卡互联与集群网络]]）。

### B200 / GB200

| 参数 | B200 | GB200（Grace-Blackwell） |
| --- | --- | --- |
| FP8 Tensor 算力 | 4,500 TFLOPS（Dense） | 同左（GPU 部分） |
| FP4 Tensor 算力 | 9,000 TFLOPS（Dense） | 同左（GPU 部分） |
| 显存 | 192 GB HBM3e | 192 GB HBM3e + 480 GB LPDDR5x（CPU） |
| 显存带宽 | 8 TB/s | 8 TB/s（GPU 部分） |
| NVLink | 5.0，1,800 GB/s | 5.0，1,800 GB/s |
| 设计 | 双 die 封装 | Grace CPU + Blackwell GPU 经 NVLink-C2C 紧耦合 |

GB200 把 NVIDIA 自研的 Grace ARM CPU 与 Blackwell GPU 通过 NVLink-C2C 封装在一起，消除了传统 PCIe 的 CPU-GPU 通信瓶颈。

> [!warning] B200 算力口径存疑
> 上表的 B200 数字按 Blackwell 架构口径（**Dense**）填写，可由硬件参数反推验证：148 个启用 SM（两块 die 各 80 个、启用 74 个）× 8,192 FLOP/clock/SM × 1.86 GHz ≈ **2.25 PFLOPS FP16 Dense**，与 FP8 = 2× FP16 的关系一致。
>
> 来源教程的表格给的是一半（FP16 约 1,125 TFLOPS、FP8 Dense 2,250 TFLOPS），**两者差 2 倍**。最可能的解释是教程把 Dense 与稀疏口径串了。本表按架构推导口径填写，**引用前建议以 NVIDIA 官方数据表复核**。

## 七、纵向对比

### Tensor Core 算力（Dense）

| 精度 | V100 | T4 | A100 | H100 | B200 |
| --- | --- | --- | --- | --- | --- |
| FP16 / BF16 | 125 | 65 | 312 | 989 | 2,250 |
| TF32 | — | — | 156 | 495 | 1,100 |
| FP8 | — | — | — | 1,979 | 4,500 |
| INT8 | — | 130 TOPS | 624 TOPS | 1,979 TOPS | 4,500 TOPS |
| FP4 | — | — | — | — | 9,000 TOPS |

单位 TFLOPS / TOPS。Ampere 起支持 2:4 结构化稀疏，开启后吞吐翻倍；V100 / T4 不支持。**厂商宣传常用稀疏数字，比较时必须对齐口径**。

### 显存与互联

| 指标 | V100 | A100 | H100 | B200 |
| --- | --- | --- | --- | --- |
| 显存容量 | 32 GB | 80 GB | 80 GB | 192 GB |
| 显存类型 | HBM2 | HBM2e | HBM3 | HBM3e |
| 显存带宽 | 900 GB/s | 2,039 GB/s | 3,350 GB/s | 8,000 GB/s |
| NVLink 带宽 | 300 GB/s | 600 GB/s | 900 GB/s | 1,800 GB/s |
| 单 NVLink 域 GPU 数 | 8 | 8 | 8（+ 跨节点网络） | **72** |

**一条明显的趋势：显存带宽的增长慢于算力增长。** H100 到 B200 算力涨了 2.3 倍，带宽只涨了 2.4 倍（看起来同步），但从 V100 到 B200 算力涨了 18 倍、带宽只涨了 9 倍。这正是 memory-bound 特性越来越突出的硬件原因。

## 八、选型

**训练**：

| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| < 10B | A100 80GB | 性价比与生态 |
| 10B – 100B | H100 SXM | FP8 + NVLink 4.0 提升多卡效率 |
| > 100B | B200 / GB200 | 192 GB 显存 + 1.8 TB/s NVLink，且单域 72 卡 |

**推理**：

| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| 低成本在线推理 | T4 | 70W，INT8 够用 |
| 中等规模 LLM | A100 / L40S | 显存与生态 |
| 大规模 LLM | H100 / H200 | FP8 吞吐高；H200 的 141 GB 与 4.8 TB/s 直接对口长上下文 |
| 极致性能 | B200 | FP4 与 192 GB |

> [!warning] 除了单卡性能，还要看集群网络拓扑（NVLink vs PCIe vs InfiniBand）、功耗散热、交付周期。**硬件选型从来不是纯技术问题。**

## 相关

- [[01-GPU 硬件架构与存储层次]] —— SM、Warp、存储层次与 Roofline，本篇的组件基础
- [[03-多卡互联与集群网络]] —— NVLink / NVSwitch / InfiniBand 的完整展开
- [[10-KV Cache 与推理优化]] —— H200 的定位为什么是显存而非算力
- [[01-推理性能指标与瓶颈定位]] —— 算力与带宽两条上限如何决定推理快慢

## 参考

- **NVIDIA Hopper 架构深入解析**（A100 / H100 SXM5 / H100 PCIe 的逐项规格对照，本笔记 SXM 与 PCIe 差异一节的来源）：https://blogs.nvidia.com.tw/blog/nvidia-hopper-architecture-in-depth/
- **NVIDIA Hopper Architecture Whitepaper**（FP8 / Transformer Engine / TMA / Thread Block Cluster）：https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper
- **NVIDIA Blackwell Architecture Technical Brief**（双芯封装、FP4、NVLink 5.0、TMEM）：https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/
- **NVIDIA A100 / V100 / V100 Whitepapers**：https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf ｜ https://images.nvidia.com/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf
- **NVIDIA Transformer Engine Documentation**：https://docs.nvidia.com/deeplearning/transformer-engine/user-guide/index.html
- **NVIDIA Multi-Instance GPU User Guide**：https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- **NVIDIA NVLink and NVSwitch**：https://www.nvidia.com/en-us/data-center/nvlink/
- **AIInfraGuide 5.1 NVIDIA GPU 架构演进**（来源教程；B200 算力口径一处已按架构推导修正）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/gpu/nvidia-gpu-evolution
