---
tags:
  - AI/infra/推理引擎
  - AI/infra/部署
---

# vLLM 部署、参数与服务特性

前面四篇讲的是机制，这一篇落到具体引擎上 —— [[02-PagedAttention：KV Cache 的分页管理]]、[[03-推理调度：Continuous Batching 与 Chunked Prefill]]、[[04-前缀缓存：APC 与 RadixAttention]]、[[05-Attention 后端与图优化]]里的东西在 vLLM 里怎么开、怎么调、会踩什么。

vLLM 由 UC Berkeley Sky Computing Lab 开发。截至教程撰写时的口径（v0.19.0，2026-04）GitHub 星标 76,000+、贡献者 2,000+。

## 一、它解决什么

把模型搬上线最头疼的是**显存不够用、吞吐上不去**。vLLM 的两大支点是：

| 特性 | 内容 |
| --- | --- |
| **PagedAttention** | 分页管理 KV Cache，显存利用率从 20–38% 提到 96% 以上 |
| **Continuous Batching** | 迭代级调度，GPU 利用率从约 30% 提到 80% 以上 |
| OpenAI 兼容 API | 改 `base_url` 即可迁移，业务代码不动 |
| 模型覆盖 | Llama、Qwen、Mistral、DeepSeek 等主流架构，含 MoE 与多模态 |
| 多硬件 | NVIDIA / AMD GPU、Google TPU、Intel Gaudi、CPU |
| 量化生态 | FP8、INT8、INT4、GPTQ、AWQ、GGUF |

## 二、安装

系统要求：Linux（推荐 Ubuntu 20.04+）、Python 3.10–3.13、CUDA 12.x、GPU Compute Capability 7.0+（Volta 及以上）。

官方推荐用 `uv` 安装，它能自动检测 CUDA 版本并选匹配的 PyTorch：

```bash
uv venv --python 3.12 --seed
source .venv/bin/activate
uv pip install vllm --torch-backend=auto     # 也可显式指定 --torch-backend=cu126
```

AMD ROCm 环境：

```bash
uv pip install vllm --extra-index-url https://wheels.vllm.ai/rocm/
```

ROCm 支持 Python 3.12、ROCm 7.0，要求 `glibc >= 2.35`。

## 三、离线批量推理

适合不需要实时响应的场景（批量生成、数据标注、评测跑分）。

```python
from vllm import LLM, SamplingParams

sampling_params = SamplingParams(temperature=0.8, top_p=0.95, max_tokens=256)
llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")

outputs = llm.generate(
    ["请用一句话解释什么是 Transformer：", "Python 和 C++ 的主要区别是什么？"],
    sampling_params,
)
for output in outputs:
    print(output.outputs[0].text)
```

`LLM` 构造时会**一次性把模型加载到 GPU 显存**，默认使用 90% 可用显存（权重 + KV Cache），由 `gpu_memory_utilization` 调整。

### Chat 模型必须用 chat template

这是最容易踩的一个坑，`llm.generate` **不会自动应用 chat template**：

```python
# 推荐：llm.chat 直接吃 messages
outputs = llm.chat(messages_list, sampling_params)

# 或者手动应用
texts = tokenizer.apply_chat_template(messages_list, tokenize=False, add_generation_prompt=True)
outputs = llm.generate(texts, ...)
```

> [!warning] 直接传裸文本给 Chat 模型，输出质量可能很差甚至乱码。**同类问题在 [[10-KV Cache 与推理优化]] 里还有第二个后果** —— chat template 的 token 没进缓存前缀时，前缀缓存也跟着失效。

### generation_config 会被覆盖

vLLM 默认读取模型仓库里的 `generation_config.json` 并用其中的参数覆盖 vLLM 默认值。想用 vLLM 自身默认采样配置需显式声明：

```python
llm = LLM(model="...", generation_config="vllm")
```

## 四、在线服务

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct --host 0.0.0.0 --port 8080 --tensor-parallel-size 2
```

默认监听 `http://localhost:8000`，暴露 OpenAI 兼容端点：

| 端点 | 说明 |
| --- | --- |
| `GET /v1/models` | 列出可用模型 |
| `POST /v1/completions` | 文本补全 |
| `POST /v1/chat/completions` | 对话补全 |
| `GET /health` | 健康检查 |

客户端只需改 `base_url`：

```python
from openai import OpenAI
client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")
response = client.chat.completions.create(model="Qwen/Qwen2.5-7B-Instruct", messages=[...])
```

> [!warning] **vLLM 默认不做认证**，`api_key="EMPTY"` 就能调。生产环境必须开启：

```bash
vllm serve model_name --api-key my-secret-key      # 或 export VLLM_API_KEY=...
```

把不带认证的 OpenAI 兼容服务直接暴露到公网，等于开放算力与数据。**认证要在网关层做，加 API Key、TLS 与网络隔离。**

## 五、关键参数

### 采样参数（`SamplingParams`）

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `temperature` | 随机性 | 1.0 |
| `top_p` | 核采样 | 1.0 |
| `top_k` | 保留概率最高的 k 个 | -1（不限制） |
| `max_tokens` | 最大生成 Token 数 | 16 |
| `repetition_penalty` | >1 减少重复 | 1.0 |
| `stop` | 遇到指定字符串停止 | None |
| `n` | 每 prompt 生成的序列数 | 1 |
| `seed` | 随机种子，设置后可复现 | None |

采样参数的完整语义（三种切池方式、logits 调整的操作顺序）见 [[04-采样参数]]。生产常用起点是 `temperature=0.7, top_p=0.9`；**工具调用 / JSON 提取这类任务该用 `temperature=0`**。

### 引擎参数

| 参数 | 说明 | 默认/推荐 |
| --- | --- | --- |
| `--tensor-parallel-size` | 张量并行 GPU 数 | 按模型大小定 |
| `--gpu-memory-utilization` | GPU 显存使用比例 | **0.9** |
| `--max-model-len` | 最大序列长度 | 按需设置 |
| `--dtype` | 权重精度 | auto |
| `--quantization` | 量化方式 | None |
| `--max-num-seqs` | 最大并发序列数 | 256 |
| `--max-num-batched-tokens` | 一步 Token 预算 | 见 [[03-推理调度：Continuous Batching 与 Chunked Prefill]] |
| `--enable-prefix-caching` | 前缀缓存 | V1 中默认开启 |
| `--block-size` | KV 块大小 | 16 |
| `--attention-backend` | Attention 后端 | 自动选择 |
| `--enforce-eager` | 禁用 CUDA Graph | 仅调试时用 |

### 显存怎么分

```
总 GPU 显存 × gpu_memory_utilization = 模型权重 + KV Cache + 临时缓冲区
```

**KV Cache 可用空间 = 总配额 − 模型权重 − 固定开销。** KV Cache 越大，能同时处理的请求越多，吞吐越高。

```bash
vllm serve model_name --gpu-memory-utilization 0.85   # 显存紧张
vllm serve model_name --max-model-len 4096            # 限制序列长度省 KV 空间
```

启动报 OOM 时通常就这么几条路：降低 `gpu-memory-utilization`、缩短 `max-model-len`、增加 `tensor-parallel-size` 分摊到多卡、或用量化模型。

## 六、多 GPU 与并行

vLLM 支持五种并行策略：

| 策略 | 做法 |
| --- | --- |
| **张量并行（TP）** | 各层权重矩阵按行/列切分到多卡 |
| 流水线并行（PP） | 不同层分配到不同 GPU |
| 数据并行（DP） | 多个副本独立处理不同请求 |
| 专家并行（EP） | MoE 的不同专家分布到不同 GPU |
| 上下文并行（CP） | 长序列的上下文分散到多卡 |

**推理场景下 TP 是首选** —— 配置最简单，且在单次前向内部通过 AllReduce 聚合，直接降低单卡显存与单请求延迟。代价是通信量随并行度上升，依赖 NVLink 带宽，**所以 TP 通常不跨节点**。

FP16 下的显存需求对照：

| 模型规模 | 权重大小 (FP16) | 推荐最小配置 |
| --- | --- | --- |
| 7B | ~14 GB | 1× A100 80G / 1× L40S 48G |
| 13B | ~26 GB | 1× A100 80G |
| 34B | ~68 GB | 1× A100 80G（需降低 `max_model_len`）或 2× A100 |
| 70B | ~140 GB | 2× A100 80G |
| 405B | ~810 GB | 16× A100 80G 或 8× H100 80G |

**实际所需显存 = 权重 + KV Cache + 运行时开销。** 这张表只是权重的下界 —— 见 [[10-KV Cache 与推理优化]]，长上下文下 KV 能占 50%–80%，是更常见的那道瓶颈。AWQ INT4 能把权重缩到约 1/4。

## 七、V1 引擎

V1 是 vLLM 近年来最重要的一次重构，也是目前**唯一的引擎**（V0 代码已被完全移除）。版本时间线：

| 时间 | 事件 |
| --- | --- |
| 2025-01-27 | V1 alpha 发布 |
| 2025-03（v0.8.0） | V1 成为默认引擎 |
| 2025-07（v0.10.0） | 开始删除 V0 代码 |
| 2025-10-02（v0.11.0） | 完成移除 `AsyncLLMEngine` / `LLMEngine` / `MQLLMEngine` 及全部 V0 Attention 后端 |

**核心变化**：

- **多进程隔离** —— `EngineCore` 专注模型执行，Tokenize / Detokenize / server 各自独立进程，CPU 任务与 GPU 核心循环重叠，通过 ZeroMQ 通信
- **统一 Token 预算调度器** —— 抹平 Prefill/Decode 边界，见 [[03-推理调度：Continuous Batching 与 Chunked Prefill]]
- **零开销前缀缓存** —— 0% 命中率下吞吐下降 <1%，因此默认开启，见 [[04-前缀缓存：APC 与 RadixAttention]]
- **Persistent Batch** —— input tensor 通过 NumPy 缓存并增量更新，不再每次用 Python 重建
- **Piecewise CUDA Graph** —— 见 [[05-Attention 后端与图优化]]

### 1.7× 的来源要看清

官方口径是 V1 相比 V0 吞吐**最高提升 1.7×**（测试模型 Llama 3.1 8B 与 Llama 3.3 70B）。关键在于**提升来自削减 scheduler 与编排循环的 Python/CPU 开销，不是新的 GPU Kernel**。

这个机制决定了两件事：

1. **无需改 API** —— 现有服务代码和请求 schema 照常工作
2. **收益随 CPU-bound 程度缩放** —— **8B 及以下的小模型获益更大**（它们在每步 CPU 调度开销上占比更高）；70B+ 的模型 GPU 已是主要成本，CPU 节省的占比就小

> 所以「V1 快 1.7 倍」不能直接外推到自己的场景。**要在实际模型与流量组合上压测。** 这条和 [[01-推理性能指标与瓶颈定位]] 里「先测清自己的负载」是同一条纪律。

### 迁移不是可选的

V1 已成为唯一引擎，升级到近期版本就是强制迁移。**升级前应当 pin 版本并做一轮完整测试。**

冷启动参考（含服务类引擎横向）：vLLM 约 **62 秒**，SGLang 约 58 秒，TensorRT-LLM 因需要按模型编译约 **28 分钟**。这是 vLLM 「从 git clone 到活端点」摩擦最小的原因之一。

## 八、服务特性

这些不改变推理速度的上限，却决定模型能不能接进真实产品。

| 特性 | 内容 |
| --- | --- |
| **结构化输出** | 通过 `xgrammar` / `guidance` 后端，用 JSON Schema、正则或语法约束强制只生成合法 Token（受约束解码 / Constrained Decoding，`cFSM` 加速） |
| **Tool Calling 与 Reasoning Parser** | 解析模型输出中的工具调用请求与推理过程（思维链），对接 OpenAI 的 `tools` / `tool_choice` 协议 |
| **Multi-LoRA** | 同一引擎内动态加载/切换多个 LoRA Adapter，一套底座权重服务多业务，单卡多租户 |
| **多模态（VLM）** | 图文/音视频输入的预处理、Encoder Cache、图像 Hash 前缀缓存；多模态预处理已从 GPU 分离 |
| **采样算法** | Temperature / Top-p / Top-k、Logprobs、并行采样（`n`）、Beam Search 的工程实现 |

结构化输出、Tool Calling 与 [[09-工具系统与 Function Calling]] 里那套协议是同一套东西的两端：**服务端负责让输出合法，调用侧负责解析与回填。** vLLM 的 Tool Calling 解析器产出的正是 `tool_calls` 结构。

**量化**在源材料里只有大纲（见 [[00-AI Infra 专栏导览]] 的待建清单），但 vLLM 侧的启用方式是可查证的：`--quantization` 支持 `compressed-tensors`、GPTQ、AWQ、FP8、NVFP4 等格式。选型口径按优先级是：精度优先 → W8A8（SmoothQuant 解决 Activation Outlier）；省显存 → INT4；长上下文 → KV Cache 量化；Hopper → FP8；Blackwell → NVFP4/MXFP4。

> [!warning] **量化不等于必然加速**：硬件没有对应低精度 Kernel、或计算图频繁执行量化与反量化时，延迟可能不降反升。评估量化必须同时报模型大小、延迟、内存与精度变化。Weight-only 量化对 **Decode** 加速明显（因为 Decode 的瓶颈就是搬权重），这条在 [[01-推理性能指标与瓶颈定位]] 里已经推出。

## 九、生产部署与运维

**容器化与 K8s**：GPU 资源请求、就绪/存活探针（**模型加载耗时长，探针阈值要放宽**，否则 Pod 会在加载完之前被判死）、镜像与权重的拉取策略。可用 vLLM Production Stack、KServe 这类现成方案。

**可观测性**：vLLM 暴露 Prometheus 指标端点，关键指标包括 TTFT、TPOT、running/waiting 请求数、**KV Cache 利用率**、**抢占次数**。配 Grafana 看板与结构化日志。

> 抢占次数是「并发压得太满」的直接信号（见 [[02-PagedAttention：KV Cache 的分页管理]]）；KV Cache 利用率反映的是离显存天花板还有多远。

**扩缩容与负载均衡**：可基于 QPS、队列长度或 KV Cache 利用率做 HPA。**前缀感知（Prefix-aware）路由**值得单独记 —— 把命中同一 System Prompt 的请求导向同一副本，提升前缀缓存命中率。这条是 [[04-前缀缓存：APC 与 RadixAttention]] 在集群层的延伸：**缓存命中率不只取决于引擎，还取决于路由器把请求送到了哪。**

**容量规划**：从 SLO 与峰值 QPS 反推 GPU 数量 —— 结合单副本压测得到的吞吐上限、显存约束和冗余系数。**单副本吞吐上限必须在目标并发下压**，空载数据会给出过于乐观的结论。

## 十、排错

| 症状 | 处理 |
| --- | --- |
| 启动 OOM | 降 `gpu-memory-utilization` / 缩短 `max-model-len` / 加 TP / 用**量化**模型 |
| Chat 模型输出乱码或不符合预期 | 改用 `llm.chat` 或手动 `apply_chat_template` |
| 国内下载 HF 模型慢 | `export VLLM_USE_MODELSCOPE=True` |
| 模型需要执行自定义代码 | `--trust-remote-code` |
| 性能不达预期 | 先确认 Attention 后端、CUDA Graph 是否真的启用（别用 `--enforce-eager`） |

## 十一、框架选型

| 引擎 | 强项 | 代价 |
| --- | --- | --- |
| **vLLM** | 通用、模型覆盖与文档最全、生态最广、冷启动约 62 s | 专精负载上不占优 |
| **SGLang** | RadixAttention 的树状前缀复用、结构化输出、Agent 场景；共享前缀重时领先 | 生态与模型覆盖不如 vLLM |
| **TensorRT-LLM** | NVIDIA 深度优化、编译后极限吞吐与延迟 | **构建需约 28 分钟**、架构支持面窄 |

2026-01 第三方 H100 SXM5 基准（Llama-3.3-70B-Instruct FP8、50 并发）下，SGLang 约 1920 tok/s、vLLM 约 1850 tok/s（约 4% 差距）。**在原始 H100 吞吐上三者已经足够接近，决定因素移到别处**：是否需要长时间编译、模型与架构覆盖面、文档与社区、以及负载形态是否偏重共享前缀。

> 选型该看「引擎的取舍与我的服务形态是否匹配」，而不是单看某个基准的头名。

## 相关

- [[02-PagedAttention：KV Cache 的分页管理]] —— `gpu-memory-utilization` / `block_size` / `enable_prefix_caching` 在管什么
- [[03-推理调度：Continuous Batching 与 Chunked Prefill]] —— `max-num-batched-tokens` / `max-num-seqs` / V1 统一调度器
- [[04-前缀缓存：APC 与 RadixAttention]] —— 前缀感知路由与 `cache_salt`
- [[05-Attention 后端与图优化]] —— `--attention-backend` / CUDA Graph / 编译缓存
- [[01-推理性能指标与瓶颈定位]] —— 该采集哪些指标、在哪一层定位瓶颈
- [[04-采样参数]] —— `SamplingParams` 各参数的语义与任务对应
- [[08-模型选型]] —— 什么时候该用 API 而不是自建

## 参考

- https://docs.vllm.ai/en/stable/getting_started/quickstart.html
- https://docs.vllm.ai/en/latest/configuration/engine_args.html
- https://blog.vllm.ai/2025/01/27/v1-alpha-release.html
- https://github.com/vllm-project/vllm/blob/main/docs/design/arch_overview.md
- https://news.creeta.com/en/llm-inference-engine-benchmarks-2026-vllm-sglang-tensorrt
- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E5%9B%9B-%E6%8E%A8%E7%90%86%E4%BC%98%E5%8C%96/%E7%AC%AC3%E7%AB%A0-%E6%B7%B1%E5%85%A5vllm/vllm%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8
