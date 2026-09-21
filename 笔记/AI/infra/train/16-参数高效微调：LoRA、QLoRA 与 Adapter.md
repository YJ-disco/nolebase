---
tags:
  - AI/infra/微调
  - AI/infra/显存管理
---

# 参数高效微调：LoRA、QLoRA 与 Adapter

训练大模型有两条正交的省资源路线：**分布式**（多切几张卡，见 [[02-分布式训练总论与显存账本]]）与**只训一小部分参数**（PEFT）。

这一篇讲后者 —— 它的核心假设只有一句：**微调不需要更新所有参数。**

## 一、全量微调的显存账

| 组成部分 | BF16 下的显存 | 7B | 70B |
| --- | --- | --- | --- |
| 模型参数 | $2N$ | 14 GB | 140 GB |
| 梯度 | $2N$ | 14 GB | 140 GB |
| Adam 两个状态（$m$、$v$） | $8N$（FP32） | 56 GB | 560 GB |
| 激活值 | 随 batch / 序列长度 | 约 10–30 GB | 约 100–300 GB |
| **合计** | **约 $12N$ + 激活** | **约 100 GB** | **约 1 TB** |

> [!warning] 这张表少了一项
> 来源给的是 $12N$，**漏了 FP32 主权重**（$4N$）。完整账本是 **$16N$** —— 因为低精度权重上过小的更新会被直接舍掉，优化器必须保留一份 FP32 副本（推导见 [[05-优化器与显存开销]]）。
>
> **两处口径的差异就这一行**，引用时注意。这里保留来源的 $12N$ 而把完整账本指过去，不覆盖任何一方。

无论按哪个口径，结论一样：**一块 A100 80 GB 连全量微调 7B 都勉强，70B 要 16+ 块卡**。

## 二、PEFT 的核心假设

全量微调把参数从 $W_0$ 改成 $W_0 + \Delta W$。PEFT 的假设是：

> **$\Delta W$ 具有低维结构 —— 它可以用远少于 $N$ 个参数表示。**

不同的 PEFT 方法，本质上是**对 $\Delta W$ 的结构做了不同的假设**。这句话是理解整个 PEFT 领域的钥匙。

理论依据来自 Aghajanyan et al.（2020）：预训练模型微调时的**内在维度（Intrinsic Dimensionality）远低于参数空间的实际维度** —— 数百万参数的模型，微调的有效自由度可能只有几千。

## 三、LoRA

**核心假设：$\Delta W$ 低秩。**

$$\Delta W = BA, \qquad B \in \mathbb{R}^{d \times r},\ A \in \mathbb{R}^{r \times k},\ r \ll \min(d,k)$$

训练时 $W_0$ **冻结**，只训 $A$ 与 $B$：

$$h = W_0x + \Delta W x = W_0x + BAx$$

### 两个实现细节

| 细节 | 做法 | 为什么 |
| --- | --- | --- |
| **初始化** | $A$ 随机高斯，**$B$ 初始化为零** | 保证训练开始时 $\Delta W = BA = 0$，**模型行为与预训练完全一致**，不会因随机扰动破坏已有能力 |
| **缩放因子** | 实际用 $\frac{\alpha}{r}BA$，$\alpha$ 常取 $r$ 或 $2r$ | 控制 LoRA 更新的幅度。$\alpha = r$ 时缩放为 1 |

$$h = W_0x + \frac{\alpha}{r}BAx$$

### 参数量（$d = 4096$ 的线性层）

| 方法 | 可训练参数 | 占比 |
| --- | --- | --- |
| 全量微调 | 16.7 M | 100% |
| **LoRA（$r=8$）** | 65 K | **0.39%** |
| LoRA（$r=16$） | 131 K | 0.78% |
| LoRA（$r=64$） | 524 K | 3.1% |

### 用在哪几层

原始论文**只在 $W_Q$ 与 $W_V$ 上应用**（实验发现这就够了）。但后续实践表明：

> **在所有线性层上都应用（"full LoRA"）通常效果更好**，尤其在复杂任务上。LLaMA-Factory 等工具默认对所有线性层应用。

这里的「所有线性层」指 Q/K/V/O 投影 + SwiGLU FFN 的三个投影（$W_{gate}$、$W_{up}$、$W_{down}$）—— 见 [[12-FFN 与激活函数]]。

### 推理时合并：零额外开销

$$W = W_0 + \frac{\alpha}{r}BA$$

**合并后的模型与全量微调在推理时完全等价** —— 相同架构、相同计算量、相同速度。所以 LoRA 是一个「训练省显存、推理零代价」的方案。

再加一条工程优势：可以为不同任务训不同的 adapter，**按需加载与切换** —— 每个只要几十到几百 MB，不必存整份模型副本。

> **低秩约束不只是省参数，还起正则化作用。** 它限制了模型的更新自由度，降低小数据集上的过拟合风险 —— 这也是 LoRA 在**数据量少时往往优于全量微调**的原因之一。

### $r$ 怎么选

| 任务 | $r$ |
| --- | --- |
| 简单（情感分类、格式调整） | 8 |
| 中等（指令微调、对话） | 16 – 32 |
| 复杂（数学推理、代码生成） | 64 或更高 |

$r$ 太小（1–4）适配能力不足；太大（256+）参数量逼近全量微调，PEFT 的优势就没了。

## 四、QLoRA

**LoRA 的漏洞**：虽然只训少量参数，但**前向传播仍需完整模型** —— 70B 的 $W_0$ 在 BF16 下仍要 140 GB。

**QLoRA（Dettmers et al., 2023）把冻结的 $W_0$ 量化到 4-bit**，在 4-bit 模型上训 LoRA。

### 三个关键创新

| 创新 | 内容 |
| --- | --- |
| **NF4（NormalFloat 4-bit）** | 传统 4-bit 量化把值域均匀分 16 段；但预训练权重**近似正态分布**（大部分集中在零附近）。NF4 按**正态分布的分位数**定区间，使每段包含大致相同数量的权重，最小化量化误差 |
| **双重量化** | 每个量化块要存一个 FP32 缩放因子。块大小 64 时开销 $4/64 = 0.0625$ 字节/参数 —— 看着少，70B 上就是 **4.4 GB**。双重量化**把缩放因子本身也量化到 8-bit**，降到 0.0156 字节/参数 |
| **分页优化器** | 利用 NVIDIA Unified Memory，让优化器状态在 GPU 显存与 CPU 内存间**自动分页** —— 显存不足时换出，需要时换入 |

### 显存对比

| 配置 | 7B | 70B |
| --- | --- | --- |
| 全量微调（BF16） | 约 100 GB | 约 1 TB |
| LoRA（BF16 模型） | 约 15 GB | 约 145 GB |
| **QLoRA（NF4 + LoRA）** | **约 6 GB** | **约 48 GB** |

**QLoRA 让单卡 24 GB（如 RTX 4090）微调 7B、单卡 A100 80 GB 微调 70B 成为可能。**

### 效果与代价

**效果**：论文报告 **4-bit QLoRA 微调的效果与 16-bit 全量微调接近**。Guanaco（QLoRA 微调 LLaMA-65B）在 Vicuna 基准上达到 ChatGPT 的 **99.3%**。

机制解释：**量化损失被 LoRA 补偿了** —— LoRA 训练的参数是 BF16 精度的，它们能「修正」4-bit 量化引入的误差。

**代价**：

> [!warning] QLoRA 训练比 BF16 LoRA 慢 30–50%
> 4-bit 权重**每次前向都要反量化回 BF16** —— 这就是它省显存的代价。**显存够（比如多卡）时，BF16 LoRA 是更快的选择**；QLoRA 的价值是「显存极其有限时仍能训」。

## 五、其他路线

### Adapter（Houlsby et al., 2019）

在 Transformer 每个子层**之后插入**一个小瓶颈网络：

$$\text{Adapter}(h) = h + f(hW_{\text{down}})W_{\text{up}}$$

$W_{\text{down}} \in \mathbb{R}^{d \times r}$ 降维、$f$ 是非线性激活、$W_{\text{up}} \in \mathbb{R}^{r \times d}$ 升回。**残差连接确保不破坏原始信息流** —— 与 [[13-归一化与残差连接]] 里那条「恒等通道」是同一个设计。

**与 LoRA 的差别**：Adapter **改结构**（新增模块），LoRA **改权重**（在既有层上加增量）。前者推理时有额外计算，后者没有。

### Prompt-based：软提示

| 方法 | 做法 |
| --- | --- |
| **Prefix-Tuning**（Li & Liang, 2021） | 在每层 Self-Attention 里**前置一组可学习的虚拟 token**，扩 K/V：$K' = [K_{\text{prefix}}; K]$、$V' = [V_{\text{prefix}}; V]$。真实 token 可以「关注」这些前缀，前缀充当任务指示 |
| **P-Tuning v2**（Liu et al., 2022） | 在**每一层**都加可学习 prefix（而非只在输入层），并在更多任务类型上验证 |

**为什么式微**：

- **有效容量有限** —— prefix 通常只有 10–100 个 token，能编码的任务信息有限
- **占上下文窗口** —— 推理时减少可用的输入长度
- **效果不及 LoRA** —— 在大模型上几乎全面落后

### 综合对比

| 方法 | 可训练参数 | 推理额外开销 | 显存 | 效果 | 现状 |
| --- | --- | --- | --- | --- | --- |
| 全量微调 | 100% | 无 | 极高 | 最好（上限） | 有资源时首选 |
| **LoRA** | 0.1–3% | **零**（可合并） | 中 | 接近全量 | **当前主流** |
| **QLoRA** | 0.1–3% | **零**（合并后） | **极低** | 接近 LoRA | 显存受限时首选 |
| Adapter | 0.5–5% | 有（额外层） | 中 | 接近全量 | 已被 LoRA 取代 |
| Prefix-Tuning | < 0.1% | 有（增序列长） | 低 | 低于 LoRA | 已少用 |

**选型只看一个问题：显存够不够。**

```
显存充足（多卡 / A100 / H100） → 全量微调（效果上限）
显存有限但够放 BF16 权重      → LoRA（效果与速度的平衡）
单卡 ≤ 24 GB                 → QLoRA（NF4 + LoRA）
```

## 六、实践

### 超参数

| 参数 | 推荐 | 说明 |
| --- | --- | --- |
| `lora_rank` | 8 – 64 | 简单任务小、复杂任务大 |
| `lora_alpha` | $2r$ 或 $r$ | 控制更新缩放 |
| `lora_dropout` | 0.05 – 0.1 | 正则化 |
| `target_modules` | 所有线性层 | `q/k/v/o_proj` + `gate/up/down_proj` |
| `learning_rate` | $1\times10^{-4}$ – $5\times10^{-4}$ | **比全量微调高一个量级** |
| `epochs` | 2 – 5 | 数据少时可多跑几轮 |

### 变体

| 变体 | 改动 |
| --- | --- |
| **DoRA** | 把权重分解为**幅度 + 方向**，只对方向部分用 LoRA |
| **LoRA+** | 对 $A$ 与 $B$ 用**不同学习率**（$B$ 更高） |
| **rsLoRA** | 缩放因子从 $\alpha/r$ 改成 $\alpha/\sqrt{r}$ —— 让**不同 rank 的训练更稳定**，换 $r$ 时不必重调 $\alpha$ |
| **AdaLoRA** | **动态分配 rank** —— 重要的层给高 rank，不重要的给低 rank 甚至剪枝 |

### 多任务与组合

一个基础模型训多个 LoRA（中文对话 / 代码 / 医疗…），推理时按需加载 —— **基础模型只存一份**。

更进一步，多个 LoRA 可以**线性组合**：

$$W = W_0 + w_1\Delta W_1 + w_2\Delta W_2$$

**不重新训练就能组合多个能力。**

> 这条与 [[06-vLLM 部署、参数与服务特性]] 里的 **Multi-LoRA 服务**是同一件事的两端：训练侧产出多个 adapter，服务侧在同一个引擎里动态加载切换。**LoRA 的「可组合」是它区别于其他 PEFT 方法的工程价值。**

### 工具

| 工具 | 特点 |
| --- | --- |
| **PEFT**（Hugging Face） | LoRA / QLoRA / Prefix-Tuning，与 Transformers 深度集成 |
| **LLaMA-Factory** | 全流程（SFT / RLHF / DPO）+ Web UI |
| **Unsloth** | 自定义 Triton kernel，号称提速 2–5× |
| Axolotl | YAML 配置驱动，多数据格式 |
| **TRL**（Hugging Face） | RLHF / DPO，与 PEFT 结合 |

## 相关

- [[05-优化器与显存开销]] —— 全量微调 $16N$ 账本的完整推导
- [[12-FFN 与激活函数]] —— 低秩分解的数学基础与 SwiGLU 的三个投影
- [[08-量化]] —— NF4 / INT4 量化的机制与坑
- [[06-vLLM 部署、参数与服务特性]] —— 服务侧的 Multi-LoRA
- [[02-分布式训练总论与显存账本]] —— 另一条正交的省资源路线
- [[17-Agentic RL]] —— SFT / RL 的训练流程

## 参考

- **LoRA: Low-Rank Adaptation of Large Language Models**（Hu et al., 2021）：https://arxiv.org/abs/2106.09685
- **QLoRA: Efficient Finetuning of Quantized LLMs**（Dettmers et al., 2023，NF4 / 双重量化 / 分页优化器）：https://arxiv.org/abs/2305.14314
- **Parameter-Efficient Transfer Learning for NLP**（Adapter，Houlsby et al., 2019）：https://arxiv.org/abs/1902.00751
- **Prefix-Tuning: Optimizing Continuous Prompts for Generation**（Li & Liang, 2021）：https://arxiv.org/abs/2101.00190
- **P-Tuning v2**（Liu et al., 2022）：https://arxiv.org/abs/2110.07602
- **Intrinsic Dimensionality Explains the Effectiveness of Language Model Fine-Tuning**（低秩假设的实证依据）：https://arxiv.org/abs/2012.13255
- **DoRA**：https://arxiv.org/abs/2402.09353 ｜ **rsLoRA**：https://arxiv.org/abs/2312.03732
- **来源**：ting.is-a.dev「LLM 原理」专栏第 05 篇（`scripts/ting-llm-raw/md/`，2026-09-21 抓取）。**该篇的「$12N$」口径漏了 FP32 主权重一项，本笔记按 $16N$ 补全并保留了来源口径**；部分公式在原站转换中退化，已按原论文重建
