---
tags:
  - AI/agent/范式
---

# Plan-and-Solve

**Plan-and-Solve** 把任务处理明确切成两个阶段：**先规划（Plan），后执行（Solve）**。

## 先分清两件同名的事

这个名词在两个层面被使用，混在一起讲会失真：

| | 原论文的 PS Prompting | 本文后半讲的 PS Agent |
| --- | --- | --- |
| 是什么 | **零样本 prompt 技巧**——在提示里加两句话 | **多阶段智能体架构**——Planner 生成计划、Executor 逐步执行 |
| 实现载体 | 一段提示文本 | 两个类 + 多次 LLM 调用 |
| 论文/出处 | Wang et al., 2023, arXiv:2305.04091 | 《Hello-Agents》第四章的工程实现 |
| 解决的问题 | 零样本 CoT 的**漏步错误** | 复杂任务中「执行到中途忘了目标」 |

两者共享同一个直觉（先谋后动），但一个改提示、一个改架构。下面先讲论文，再讲工程实现。

## 论文部分：PS 是对零样本 CoT 的改进

`Wang, L., et al. Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models. arXiv:2305.04091, 2023`

### Zero-shot-CoT 的三个缺陷

论文指出「Let's think step by step」有三类失败：

| 缺陷 | 表现 |
| --- | --- |
| **计算错误**（calculation errors）| 中间步骤算错 |
| **漏步错误**（missing-step errors）| 跳过了关键的中间计算步骤 |
| **语义误解错误**（semantic misunderstanding）| 理解错了题意 |

**PS 主要针对「漏步错误」**——先要求模型把问题拆成子任务，再按计划执行，这样难以跳过步骤。

### PS 与 PS+ 的提示原文

PS 只是在提示里加两句话：

```
Q: [问题]
Let's first understand the problem and devise a plan to solve it.
Then, let's carry out the plan and solve the problem step by step.
```

PS+ 在此基础上再加三条指令：

```
Q: [问题]
Let's first understand the problem and devise a plan to solve it.
Then, let's carry out the plan to solve the problem step by step.
- Extract relevant variables and their corresponding numerals.
- Calculate intermediate results (pay attention to calculation errors).
- Check your answer for reasonableness.
```

三条附加指令各自对应一类错误：第一条**把变量和数值显式拎出来**（对抗语义误解），第二条对计算错误，第三条做合理性自检。

### 性能（text-davinci-003，全部零样本）

| 方法 | MultiArith | GSM8K | AddSub | AQuA | SingleEq | SVAMP | 平均 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Zero-Shot CoT | 83.8 | 56.4 | 85.3 | 38.9 | 88.1 | 69.9 | 70.4 |
| PS | 87.2 | 58.2 | 88.1 | 42.5 | 89.2 | 72.0 | 72.9 |
| **PS+** | **91.8** | **59.3** | **92.2** | 46.0 | **94.7** | 75.7 | **76.7** |
| Manual-CoT（**8-shot**）| 93.6 | 58.4 | 91.6 | 48.4 | 93.5 | 80.3 | 77.6 |

**关键结论**：PS+ 用**零样本**就逼近 8 个手工示例的 CoT（平均 76.7 vs 77.6），并在 MultiArith、AddSub、SingleEq 上反超。GSM8K 上也略高（59.3 vs 58.4）。

代价是：**8-shot 手工示例仍然在更难的数据集上更稳**（AQuA 48.4 vs 46.0、SVAMP 80.3 vs 75.7）。

### 错误类型分布：改进发生在本来的三个缺陷上，但只改了两个

论文从 GSM8K 里抽样 100 道各方法都答错的题，人工分型：

| 方法 | 计算错误 | 漏步错误 | 语义误解 |
| --- | --- | --- | --- |
| Zero-Shot CoT | 7% | 12% | 27% |
| Zero-shot PS | 7% | 10% | 26% |
| **Zero-shot PS+** | **5%** | **7%** | 27% |

**这张表比准确率表更能说明 PS 的边界**：PS+ 把计算错误压到 5%、漏步错误压到 7%（相对 Zero-Shot CoT 分别降了约 29% 和 42%），但**语义理解错误一点没改善**（27%）。

> **PS 治的是「步骤走偏」，治不了「题意理解错」。** 题读错了，计划再完整也没用——这条判据决定了什么时候不该指望 PS。

配套的相关性分析印证了这一点：**变量定义的存在与计划的存在，都与计算错误、漏步错误呈负相关**。另外，随机抽 100 例检查，**90 例的预测里确实出现了计划**——说明当代模型（论文用的是 GPT-3.5/GPT-4 代）已经具备被提示出来的规划能力。

### 两条使用边界

- **计划粒度是权衡**：计划太粗没有收益（等于没拆），太细会增加错误面
- **不是普适优势**：在常识推理等任务上 PS+ 的增益明显变小甚至消失（不同模型上结论不一致），数学与符号推理才是它的主场

## 工程部分：把它做成智能体

原论文只改提示，[[03-ReAct]] 那种「走一步看一步」的节奏在长任务里会漂移。把 PS 落成架构，就是把**规划**和**执行**拆成两个组件。

两者的性格差异可以这么记：ReAct 像侦探，根据现场蛛丝马迹一步步推理、随时调整方向；Plan-and-Solve 像建筑师，动工前先把蓝图（Plan）画完，然后严格照图施工（Solve）。

### 两阶段

| 阶段 | 做什么 |
| --- | --- |
| **规划阶段** | 接收用户完整问题。第一个任务**不是**解决问题或调工具，而是把问题分解、制定出清晰分步骤的行动计划。计划本身就是一次 LLM 调用的产物 |
| **执行阶段** | 拿到完整计划后，**严格按计划步骤逐一执行**。每步可能是一次独立的 LLM 调用，也可能是对上一步结果的加工，直到所有步骤完成 |

形式化：

$$P = \pi_{\text{plan}}(q)$$

$$s_i = \pi_{\text{solve}}\big(q, P, (s_1,\dots,s_{i-1})\big)$$

规划模型 $\pi_{\text{plan}}$ 由问题 $q$ 生成含 $n$ 步的计划 $P$；执行时第 $i$ 步的解 $s_i$ 同时依赖原始问题、完整计划和之前所有步骤的结果。最终答案是最后一步的结果 $s_n$。

注意 $s_i$ 的三个依赖项——**原始问题一直在场**，这是避免「执行到第三步忘了要干什么」的机制。

### 规划阶段：把计划强制成可解析的格式

规划器的提示词核心是**格式约束**：强制输出 Python 列表，让解析从自然语言处理变成 `ast.literal_eval`。

```python
PLANNER_PROMPT_TEMPLATE = """
你是一个顶级的AI规划专家。你的任务是将用户提出的复杂问题分解成一个由多个简单步骤组成的行动计划。
请确保计划中的每个步骤都是一个独立的、可执行的子任务，并且严格按照逻辑顺序排列。
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。

问题: {question}

请严格按照以下格式输出你的计划,```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...]
```
"""
```

解析侧：

```python
plan_str = response_text.split("```python")[1].split("```")[0].strip()
plan = ast.literal_eval(plan_str)          # 不用 eval
return plan if isinstance(plan, list) else []
```

两个工程点：用 `ast.literal_eval` 而不是 `eval`（只解析字面量，不执行代码）；解析失败时返回空列表并在 `run()` 里终止流程，而不是带着残缺计划往下跑。

### 执行阶段：状态管理是执行器的真正职责

执行器不只是「调 LLM」，它承担**状态管理**——记录每一步结果，作为上下文喂给后续步骤。它的提示词要包含四样东西：

| 内容 | 作用 |
| --- | --- |
| 原始问题 | 保证模型始终知道最终目标 |
| 完整计划 | 让模型知道当前步骤在整个任务里的位置 |
| 历史步骤与结果 | 当前步骤的直接输入 |
| 当前步骤 | 明确现在要解决哪一个 |

```python
for i, step in enumerate(plan):
    prompt = EXECUTOR_PROMPT_TEMPLATE.format(
        question=question, plan=plan,
        history=history if history else "无",
        current_step=step)
    response_text = self.llm_client.think(messages=[{"role": "user", "content": prompt}]) or ""
    history += f"步骤 {i+1}: {step}\n结果: {response_text}\n\n"
final_answer = response_text
```

提示词里明确要求「仅输出该步骤的最终答案，不要输出任何额外的解释或对话」——**执行阶段不需要模型的推理过程，只需要结果**，否则历史会被废话撑爆。

### 一个实例的观察

问题：周一卖 15 个苹果，周二卖周一的两倍，周三比周二少 5 个，三天共多少？

规划器输出的计划（4 步）：

```python
["计算周一卖出的苹果数量： 15个",
 "计算周二卖出的苹果数量： 周一数量 × 2 = 15 × 2 = 30个",
 "计算周三卖出的苹果数量： 周二数量 - 5 = 30 - 5 = 25个",
 "计算三天总销量： 周一 + 周二 + 周三 = 15 + 30 + 25 = 70个"]
```

值得注意的一点：**计划里已经把答案算出来了**（每一步都带着结果）。执行阶段只是把每步的数值单独再输出一遍（15 / 30 / 25 / 70）。这说明规划模型的能力足够时，执行阶段更像是在做校验与格式化，而不是真正的计算——**这与论文的结论一致：PS 治步骤不治理解**。

## 与 ReAct 的选择

| | ReAct | Plan-and-Solve |
| --- | --- | --- |
| 决策节奏 | 走一步看一步 | 先谋后动 |
| 对环境的依赖 | 强，每步靠 Observation 修正 | 弱，计划在执行前已固定 |
| 目标一致性 | 可能漂移 | 高，计划全程在场 |
| 适合 | 探索性、需要外部工具输入的任务 | 逻辑路径确定、内部推理密集的任务 |

适合 Plan-and-Solve 的典型任务：多步数学应用题、需要整合多个信息源的报告撰写、代码生成（先构思函数/类/模块结构，再逐一实现）。

## 相关

- [[03-ReAct]] —— 另一种决策节奏
- [[17-Prompt Engineering]] —— PS / PS+ 本身是提示技巧，属于那一层
- [[05-Reflection]] —— 两个范式都可以作为它的「初稿」环节

## 参考

- 来源：《Hello-Agents》第四章 §4.3
- Wang, L., et al. Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models. arXiv:2305.04091, 2023.
- **论文原文（错误类型分布、相关性分析、计划存在率、提示模板）**：https://ar5iv.labs.arxiv.org/html/2305.04091
- 准确率表（六个数学数据集）：https://tomesphere.com/paper/2305.04091
- PS / PS+ 提示模板与三条附加指令：https://spaceservices.org/learn/chain-of-thought-reasoning
- 粒度权衡与不同模型上增益不一致：https://www.emergentmind.com/topics/plan-and-solve-prompting
