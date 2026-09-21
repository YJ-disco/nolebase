---
tags:
  - AI/infra/推理引擎
  - AI/infra/编译优化
---

# Attention 后端与图优化

[[01-推理性能指标与瓶颈定位]] 说 Decode 卡在「等数据」上（Memory Bound）。但在**小模型、小 Batch** 的场合，还有第二个瓶颈会冒出来：**等指令**。

## 一、CPU 拖了 GPU 的后腿

GPU 自己不会主动干活。它每做一个操作（一次矩阵乘、一次 LayerNorm、一次激活）都要由 CPU 通过一次「启动（launch）」命令告诉它。跑一遍 Transformer 有成百上千个这样的小操作，对应成百上千次 CPU→GPU 启动调用。每次启动本身有固定的 CPU 开销（构造参数、驱动调度、提交队列），量级在微秒级。

平时这点开销不算什么 —— Prefill 一步处理成千 Token，每个 Kernel 一跑就几百微秒甚至毫秒，CPU 那点启动开销被淹没。但 **Decode 每步只处理 1 个（或很少几个）Token**，每个 Kernel 的实际计算可能就几微秒，结果**CPU 发射 Kernel 的时间比 GPU 执行它的时间还长**。GPU 干几微秒就停下来等 CPU 发下一条命令。

```
CPU 发射 Kernel1 → GPU 算（极快）→ CPU 发射 Kernel2 → GPU 算（极快）→ ...
                                      ↑ GPU 大量时间在等 CPU 发指令
```

> Decode 阶段有**两重等待**：等数据（Memory Bound，靠 Batching 与量化缓解）和等指令（CPU launch 开销，靠图优化缓解）。**小模型、小 Batch 时后者尤其突出** —— 这也是为什么图优化对 Decode 加速格外有效。

## 二、可插拔的 Attention 后端

Attention 是 Transformer 里最重、最讲究的算子，它的 Kernel 实现直接决定推理速度。vLLM 的设计是**不绑死一种实现**，提供可插拔后端：

| 后端 | 特点 | 典型适用 |
| --- | --- | --- |
| **FlashAttention** | IO 感知的经典高性能实现，支持到 FlashAttention 3 | 主流 NVIDIA GPU 的通用首选 |
| **FlashInfer** | 面向推理优化，KV Cache 的布局可配置，支持多种数据类型 | 高吞吐服务、需要自定义 KV 布局 |
| **FlashMLA** | 针对 MLA（Multi-head Latent Attention）优化 | DeepSeek 等采用 MLA 结构的模型 |
| **Triton** | 用 Triton 语言实现，可移植性好 | 非 NVIDIA 硬件或需自定义时的兜底 |

> 这些后端是**同一个数学操作的不同工程实现**。结果在数学上等价，区别在访存模式、并行策略、对特定硬件指令和模型结构的适配程度。**选对后端，同样的卡能跑出明显不同的吞吐。**

多数情况下不用手动指定，vLLM 会根据 GPU 架构、模型类型和数据精度自动挑选。只有在深度性能调优、或跑特殊模型结构（如 MLA）时才需要显式干预：

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct --attention-backend FLASH_ATTN
vllm serve Qwen/Qwen2.5-7B-Instruct --attention-backend FLASHINFER
```

后端支持矩阵随版本演进较快，某张卡/某个模型用哪个后端，以所用 vLLM 版本的官方文档为准。

## 三、为什么后端必须支持混合批次

这里有一条和 [[03-推理调度：Continuous Batching 与 Chunked Prefill]] 的关键联系。V1 的统一调度器会把 **Prefill 的 Chunk 与 Decode 请求塞进同一步**，那这一步的 Attention 怎么算？两部分的计算形态不同：

| 部分 | 要算什么 |
| --- | --- |
| Prefill chunk | 一整段新 Token 之间的注意力 **+** 对历史的注意力 |
| Decode | `1 个新 Token` 对全部历史的注意力 |

这就要求后端**原生支持混合批次（mixed prefill/decode batch）**：一次 Kernel 调用里同时正确处理批中既有 Prefill 又有 Decode 的请求。FlashAttention 3 等现代后端正是为此设计的 —— 接受变长的、混合阶段的输入，用统一的 Kernel 算完，不需要拆成两次 Kernel 调用。

> **「统一调度器」（软件层抹平 Prefill/Decode）和「支持混合批次的 Attention 后端」（Kernel 层抹平）是配套的。** 没有后者，V1 那套 Token 预算调度落不了地 —— 这也是 V1 对后端能力有要求、早期并非所有后端都支持的原因。

前端到后端的这套设计出自 FlashAttention 的 IO 感知思路（`arXiv:2205.14135`）。

## 四、CUDA Graph：把一串启动打包成一次

回到第一节的 CPU launch 开销。**CUDA Graph** 是 NVIDIA 提供的解药：把一连串固定的 GPU 操作**录制（capture）**成一张图，之后只需一条命令**重放（replay）**整张图，GPU 依次执行录好的所有 Kernel —— **成百上千次 CPU 启动被压缩成一次**。

前提是**录制的操作序列和张量形状必须固定**。这对 Decode 是天作之合：每步都是「处理固定 Batch 个 Token」，形状稳定，可以针对不同 Batch Size 分别录制，运行时按当前 Batch Size 选对应图重放。

> [!warning] 正因为要求形状固定，vLLM **只为预设的一组 Batch Size 捕获 CUDA Graph**（`cudagraph_capture_sizes`）。实际 Batch Size 命中列表才走重放，否则回退到逐 Kernel 启动。这也意味着 CUDA Graph 的收益主要体现在 Decode（形状规整），**多变的 Prefill 较难直接套用**。

## 五、Piecewise CUDA Graph：在 Attention 处切一刀

CUDA Graph 解决了启动开销，还有一层空间：**能不能把很多小 Kernel 本身合并成更少更大的 Kernel？** 这是 `torch.compile` 干的事 —— 把前向编译成优化后的图，通过算子融合减少 Kernel 数量。vLLM V1 中 `torch.compile` 默认开启，且**所有编译在开始服务前完成**，避免请求过程中触发编译导致延迟尖刺。

矛盾来了：CUDA Graph 要求整段操作形状固定，而 **Attention 恰恰最难被兼容** —— 它涉及变长 KV、复杂 Mask，形状不规整。因为 Attention 不好处理就放弃整张图的 CUDA Graph，又太可惜。

V1 的解法是 **Piecewise CUDA Graph（分段）**：

1. 把整个前向图**在 Attention 算子处切开**（Attention 被包装成一个「不透明」算子，编译器不去动它）
2. **Attention 之外的部分**（LayerNorm、各种 GEMM、激活、残差 —— 都是规整的 token-wise 操作）用 CUDA Graph 捕获、重放
3. **Attention 本身保持 eager（即时）执行**，保留处理变长 KV 与复杂 Mask 的能力

```
[非 Attention 层: CUDA Graph 重放] → [Attention: eager] → [非 Attention 层: CUDA Graph 重放] → ...
```

> 对规整的部分用图优化榨干 CPU 开销，对 Attention 网开一面。**CPU launch 开销的大头（那成百上千个小算子）被消除，而 Attention 处理变长 KV 的能力不受损。** 这是 V1 相比 V0 在 Decode 上提速的关键工程之一。

vLLM 也支持在兼容后端下做 **Full CUDA Graph**（把 Attention 也纳入捕获），官方文档指出这在「小模型或 MoE 的 Decode」等场景能进一步提速。Piecewise 是通用默认，Full 是进阶选项。

## 六、源码骨架：怎么切图与捕获

### Attention 被包成不透明算子

切图的第一步是让编译器「看不进」Attention。vLLM 把 Attention 注册成一个自定义算子 `torch.ops.vllm.unified_attention_with_output`。PyTorch 的图追踪器（Dynamo）遇到它时会当成**黑盒** —— 不追踪其内部、不试图优化，但仍能围绕它捕获出完整的计算图。

> **把 Attention 包成不透明算子是「切图」能成立的前提。**

### 一张图切成三种子图

Dynamo 追踪 `forward` 后，按 `splitting_ops`（通常就是 Attention）切图。由于 Transformer 是同构层的堆叠，切完只会得到**三种不同的子图**：

| 子图 | 内容 | 编译次数 |
| --- | --- | --- |
| **首层** | 进入第一个 Attention 之前的部分 | 1 次 |
| **中间重复层** | 夹在相邻两个 Attention 之间的部分 | 1 次（**所有中间层复用**） |
| **末层** | 最后一个 Attention 之后到输出 | 1 次 |

正因为中间层同构、子图可复用，编译成本被摊得很低 —— **几十层的模型也只编译三种子图**。这是 Transformer 结构规整性带给工程实现的红利。

### 按 Batch Size 捕获与重放

```python
from vllm import LLM
from vllm.config import CompilationConfig

llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    compilation_config=CompilationConfig(
        # 只为这些 batch size 捕获 CUDA Graph，命中才走重放
        cudagraph_capture_sizes=[1, 2, 4, 8, 16, 32],
        # 可选：为特定静态形状做 Inductor 自动调优（首次较慢，默认关闭）
        compile_sizes=[1, 2, 4, 8],
    ),
)
```

- `cudagraph_capture_sizes`：为哪些 Batch Size 捕获图。命中才走重放，否则回退逐 Kernel 启动
- `compile_sizes`：让 Inductor 为指定静态形状做自动调优，找更快的 Triton 配置。收益是运行更快，代价是首次编译更慢，**所以默认关闭**

编译产物写入**编译缓存**（缓存键包含所有相关配置与被追踪的源文件）。缓存目录可在部署间**直接拷贝**以省掉重复编译时间；想强制重编可用 `VLLM_DISABLE_COMPILE_CACHE=1`。

## 七、默认开着，但要清楚它付了什么

这些优化对使用者大多是**默认开启、自动生效**的：

| | 内容 |
| --- | --- |
| 收益 | Decode 阶段 CPU 开销大幅下降，小模型/小 Batch 下吞吐与延迟明显改善；`torch.compile` 的算子融合进一步减少 Kernel 数量 |
| 代价 | **启动变慢**（首次要做编译与 CUDA Graph 捕获）；**显存占用**（捕获多个 Batch Size 的图需额外显存）；**调试复杂度**（编译后的图不如 eager 好调试） |

> [!warning] 遇到疑难的数值异常、或想确认某个 Kernel 行为时，可以临时用 eager 模式（关闭 `torch.compile` / CUDA Graph）对比，定位完再打开。**生产环境务必保持开启**，否则白白损失 Decode 性能。

## 相关

- [[01-推理性能指标与瓶颈定位]] —— Decode 的两重等待：等数据与等指令
- [[03-推理调度：Continuous Batching 与 Chunked Prefill]] —— 混合批次的来源，与后端能力配套
- [[02-PagedAttention：KV Cache 的分页管理]] —— 块间接寻址由哪一层接管
- [[10-KV Cache 与推理优化]] —— Prefill/Decode 的性格差异

## 参考

- **FlashAttention**（IO 感知 Attention 的原始论文，后端设计的理论来源）：Dao et al., https://arxiv.org/abs/2205.14135
- **vLLM Design — torch.compile Integration**（`splitting_ops`、三种子图、Piecewise CUDA Graph）：https://docs.vllm.ai/en/latest/design/torch_compile.html
- **vLLM V1 发布说明**（Piecewise CUDA Graph、FlashAttention 3）：https://blog.vllm.ai/2025/01/27/v1-alpha-release.html
- **NVIDIA Developer Blog — Getting Started with CUDA Graphs**（录制/重放机制）：https://developer.nvidia.com/blog/cuda-graphs/
- **AIInfraGuide 2.5 Attention 后端与图优化**（CPU launch 瓶颈、不透明算子、三种子图、编译缓存）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E5%9B%9B-%E6%8E%A8%E7%90%86%E4%BC%98%E5%8C%96/%E7%AC%AC2%E7%AB%A0-%E6%8E%A8%E7%90%86%E5%BC%95%E6%93%8E%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF/25-attention-%E5%90%8E%E7%AB%AF%E4%B8%8E%E5%9B%BE%E4%BC%98%E5%8C%96
