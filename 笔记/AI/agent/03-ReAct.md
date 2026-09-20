---
tags:
  - AI/agent/范式
---

# ReAct

**ReAct（Reason + Act）** 由 Shunyu Yao 等人于 2022 年提出（`arXiv:2210.03629`，ICLR 2023），把**推理**与**行动**显式结合起来，形成一个「思考-行动-观察」的循环。它是 [[02-Agent Loop]] 的第一个具体实现，也是后来所有工具调用型智能体的结构原型。

## 它解决了什么问题

ReAct 之前的方法分两类，各缺一半：

| 类型 | 代表 | 缺什么 |
| --- | --- | --- |
| 纯思考 | 思维链（CoT） | 无法与外部世界交互，**只能从参数化记忆里回忆，因而会幻觉** |
| 纯行动 | 直接输出动作 | 没有工作记忆来规划、追踪进度或从错误里恢复 |

ReAct 的出发点是：**思考与行动相辅相成**——思考指导行动（诱导/追踪/调整计划），行动又反过来校正思考（带回外部事实）。原论文的说法是 `reason to act` 与 `act to reason`。

## 一个容易被忽略的设计：动作空间被扩展了

原论文把智能体的决策形式化为在 **`A ∪ L`** 中选择——`A` 是环境动作集合，`L` 是语言空间。也就是说：

> **产生一条 Thought 本身也是一次 action。**

这个扩充是 ReAct 的结构核心。因为 thought 和 action 走的是同一个 token 流、同一种「输出」，模型不需要在两种模式间切换接口，才可能在一次生成里交替输出两者。

每个时间步 $t$，策略（即大语言模型 $\pi$）根据初始问题 $q$ 和之前所有步骤的「行动-观察」轨迹生成当前的思考与行动：

$$(th_t,a_t)=\pi\big(q,(a_1,o_1),\dots,(a_{t-1},o_{t-1})\big)$$

随后工具 $T$ 执行行动 $a_t$，返回新观察 $o_t$：

$$o_t = T(a_t)$$

新的 $(a_t,o_t)$ 对追加进历史，循环直到模型在 $th_t$ 中判断任务完成。三个输出字段：`Thought` 是「内心独白」（分析情况、分解任务、反思上一步），`Action` 是具体动作（如 `Search['华为最新款手机']`），`Observation` 是工具返回的结果。

## 它不是强化学习

这点常被误解。原论文的方法跑在**冻结的 PaLM-540B** 上：

```
没有梯度更新
没有奖励模型
没有策略优化
```

它是纯 prompting 方法（外加一个轻量的 bootstrap 监督微调实验：生成约 3000 条答案正确的 ReAct 轨迹，再拿去微调小模型——仍是**监督**微调，不是 RL）。

ReAct 的价值是**结构性的而非算法性的**：它定义的 `reason → act → observe` 循环，正是后续 **agentic RL** 所训练的交互结构——每个 Action 是 episode 的一步，每个 Observation 是环境反馈，任务完成或可验证的奖励挂在最后。多轮工具调用 RL、RLVR 之类的工作都假设交互轨迹长得像 ReAct 轨迹。

## 工具怎么定义

工具的三个核心要素，其中**描述最关键**——大语言模型完全依赖这段描述判断何时该用哪个工具：

| 要素 | 说明 |
| --- | --- |
| 名称（Name） | 简洁、唯一的标识符，供 `Action` 调用 |
| 描述（Description） | 一段清晰的自然语言，说明用途。**机制中最关键的部分** |
| 执行逻辑 | 真正执行任务的函数 |

多个工具时需要统一的管理器（`ToolExecutor`），职责是注册、按名取函数、导出格式化描述串：

```python
class ToolExecutor:
    def __init__(self):
        self.tools: Dict[str, Dict[str, Any]] = {}

    def registerTool(self, name: str, description: str, func: callable):
        if name in self.tools:
            print(f"警告:工具 '{name}' 已存在，将被覆盖。")
        self.tools[name] = {"description": description, "func": func}

    def getTool(self, name: str) -> callable:
        return self.tools.get(name, {}).get("func")

    def getAvailableTools(self) -> str:
        return "\n".join([f"- {name}: {info['description']}"
                          for name, info in self.tools.items()])
```

`getAvailableTools()` 的输出直接拼进提示词的 `{tools}` 占位符——**描述写得好不好，决定工具会不会被正确调用**。

## 提示词模板

```python
REACT_PROMPT_TEMPLATE = """
请注意，你是一个有能力调用外部工具的智能助手。

可用工具如下:
{tools}

请严格按照以下格式进行回应:

Thought: 你的思考过程，用于分析问题、拆解任务和规划下一步行动。
Action: 你决定采取的行动，必须是以下格式之一:
- `{{tool_name}}[{{tool_input}}]`:调用一个可用工具。
- `Finish[最终答案]`:当你认为已经获得最终答案时。
- 当你收集到足够的信息，能够回答用户的最终问题时，你必须在Action:字段后使用 Finish[最终答案] 来输出最终答案。

现在，请开始解决以下问题:
Question: {question}
History: {history}
"""
```

四个组成部分各司其职：角色定义、工具清单、格式规约（最重要——它强制输出结构化，代码才能精确解析意图）、动态上下文（原始问题 + 累积历史）。

**原论文用的是 few-shot 而非 zero-shot**：HotpotQA 配 6 个示例、FEVER 配 3 个、ALFWorld 配 2 个、WebShop 配 1 个。这解释了它那句限制——「方法依赖语言模型仅从少量示例中就学会推理格式和领域动作空间」。

## 输出解析

LLM 返回的是纯文本，靠正则把 `Thought` 和 `Action` 拆出来：

```python
def _parse_output(self, text: str):
    # Thought: 匹配到 Action: 或文本末尾
    thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
    # Action: 匹配到文本末尾
    action_match = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
    thought = thought_match.group(1).strip() if thought_match else None
    action = action_match.group(1).strip() if action_match else None
    return thought, action

def _parse_action(self, action_text: str):
    match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
    if match:
        return match.group(1), match.group(2)
    return None, None
```

`_parse_output` 负责拆分两个字段，`_parse_action` 把 `Search[华为最新手机]` 进一步拆成工具名 `Search` 与输入 `华为最新手机`。

**`Finish` 的判定要在工具查找之前**：`action.startswith("Finish")` 直接提取最终答案并 `return`，不能等到查工具表时才发现没有名为 `Finish` 的工具。

## 循环安全阀

`max_steps`（示例取 5）是防止无限循环耗尽资源的关键参数。循环结束后返回 `None`，表示未在预算内得出结论。

`run()` 每次调用都重置 `history = []`——历史是**单次任务内**的上下文，跨任务复用会把无关信息带进下一次决策。

## 实验数据

`PaLM-540B`，纯 prompting（数据来自 Google Research 官方博客）：

| 方法 | HotpotQA（EM，6-shot）| FEVER（acc，3-shot）| ALFWorld（成功率，2-shot）| WebShop（成功率，1-shot）|
| --- | --- | --- | --- | --- |
| Standard prompting | 28.7 | 57.1 | —— | —— |
| CoT（纯推理）| 29.4 | 56.3 | —— | —— |
| Act-only（纯行动）| 25.7 | 58.9 | 45 | 30.1 |
| **ReAct** | **27.4** | **60.9** | **71** | **40** |
| 最佳 ReAct + CoT | 35.1 | 64.6 | —— | —— |
| 有监督 SoTA | 67.5（用 ~140k 样本）| 89.5（用 ~90k 样本）| —— | —— |
| 模仿学习基线 | —— | —— | 37（用 ~100k 样本）| 29.1（用 ~90k 样本）|

**三件事必须一起看**：

1. **单独用 ReAct 在 HotpotQA 上略输 CoT（27.4 vs 29.4）。** 原论文的解释是：受限的结构提高了推理错误率，而无信息量的搜索会把推理带偏。但在 FEVER 上 ReAct 反超 CoT（60.9 vs 56.3）。
2. **最好的结果来自 ReAct + CoT 的结合（35.1 / 64.6）**——在内部知识和外部获取的信息之间来回。
3. **交互式任务上的差距最大**：ALFWorld 上 ReAct 用 **2 个示例**达到 71%，而模仿学习基线用约 10 万条样本只有 37%——**绝对提升 34 个百分点**。WebShop 上提升 10 个百分点（40 vs 29.1）。ReAct 不宣称打败全监督基线（HotpotQA/ FEVER 上差得远），它证明的是：**prompting + 几个示例就补上了大部分需要 GPU-months 训练才能达到的差距，而在交互任务上直接反超。**

> [!warning] 查证时发现二手来源的 HotpotQA 数字互相矛盾（有的写 ReAct 35.1，有的写 71.2）。上表按 **Google Research 官方博客**的口径取数——那是论文作者自己的图表。引用时请以官方为准。

## 失败模式分布

原论文对 HotpotQA 上的 ReAct 轨迹做了人工分型，这是理解 ReAct 边界最硬的证据：

| 失败类型 | 占比 | 含义 |
| --- | --- | --- |
| **Search error** | **23%** | 搜索本身没找到正确页面 |
| **Reasoning error** | 13% | 页面找对了，但 Thought 里的归因 / 计数 / 时序推理错了 |
| **Hallucination** | **6%** | 手里拿着 Observation，Thought 却和它矛盾 |
| Label ambiguity | 5% | 数据集标注本身有歧义 |
| Other | 3% | 解析失败、API 错误等 |

**和 CoT 对照着看才有意义**：CoT 的主导失败模式是幻觉，占**56%**；ReAct 把幻觉压到 **6%**，几乎清零——但暴露出新瓶颈：**23% 的失败源于外部工具本身的质量**。

这条结论直接推出了后续一批工作：**与其优化推理，不如优化检索器**（Toolformer、WebGPT 那一类「优化 retriever」的路线）。

## 一条被忽视的实用特性：人可以改轨迹

原论文还做了一组实验：允许人类检查者**直接编辑 ReAct 的推理轨迹**（把一句幻觉换成人写的提示），ReAct 随后就能按编辑后的方向继续，成功完成任务。

在一个 ALFWorld 的例子里，轨迹因为第 17 步的幻觉而失败；人类编辑了第 17 和第 23 步两条推理后，ReAct 产生了正确行为。

**这意味着行为纠正不需要改模型参数，改几行 thought 就行。** 这是 CoT 做不到的——CoT 的推理链是黑箱，没有可插入的观察点。这条是 ReAct 在可解释性之外的第二个实用价值。

## 四个特点与四个局限

| 特点 | 局限 |
| --- | --- |
| **高可解释性**：`Thought` 链暴露每一步的心路历程，也让人可以中途介入 | **强依赖 LLM 能力**：推理、指令遵循或格式化输出任一不足，流程就断在 Thought 或 Action |
| **动态规划与纠错**：走一步看一步，搜索不理想可在下一步改搜索词 | **执行效率低**：串行多次调用 LLM，步数与 token 都显著高于 CoT |
| **工具协同**：LLM 负责运筹帷幄，工具负责具体执行 | **提示词脆弱**：模板里微小用词差异都会影响行为，且并非所有模型都能稳定遵循格式 |
| 突破单一 LLM 在知识时效性、计算准确性上的固有局限 | **可能陷入局部最优**：步进决策缺乏全局长远规划，可能在原地打转 |

原论文自己也列了三条方法层面的限制：**长时程任务会超出上下文预算**；推理轨迹**不保证忠实反映模型真实的计算过程**（thought 是「说出来的理由」，不一定是「实际的理由」）；性能对**搜索质量、解码策略、提示示例、动作空间设计**都敏感。

## 调试的五条入手点

1. **打印完整提示词**——每次调用前把格式化后的完整提示（含全部历史）打出来，这是追溯决策源头最直接的办法
2. **打印原始输出**——解析失败时先看模型到底返回了什么，才能分辨是模型没遵循格式还是解析逻辑有误
3. **验证工具输入输出**——确认生成的 `tool_input` 是工具期望的格式，同时确认工具返回的 `observation` 是模型能理解的格式
4. **加少样本示例**——频繁出错时在提示词里塞一两个完整的 `Thought-Action-Observation` 成功案例。原论文本身就是这么用的
5. **换模型或调参数**——`temperature` 通常设为 0 以保证输出确定性

## 适用与不适用

适合：需要外部知识的任务（实时信息、专业领域检索）、需要精确计算的任务（交给计算器，避开 LLM 的计算错误）、需要与 API 交互的任务。**动作空间大、需要探索的交互式任务收益最大**（ALFWorld / WebShop 那类）。

不适合：逻辑路径完全确定、不需要外部反馈的任务——那些更适合 [[04-Plan-and-Solve]]。

## 相关

- [[02-Agent Loop]] —— ReAct 是它的第一个具体实现
- [[05-Reflection]] —— 在 ReAct 之上再加一层事后校正
- [[09-工具系统与 Function Calling]] —— 论文之后，工具调用从「prompt 约束 + 解析」演进到模型原生函数调用
- [[07-模型幻觉]] —— ReAct 把幻觉从 56% 压到 6% 的那条数据在这里的语境下更有意义

## 参考

- 来源：《Hello-Agents》第四章 §4.1–§4.2
- Yao, S., et al. ReAct: Synergizing Reasoning and Acting in Language Models. arXiv:2210.03629, ICLR 2023.
- **官方实验数据与人机编辑实验**：https://research.google/blog/react-synergizing-reasoning-and-acting-in-language-models/
- 失败模式分布（search error 23% / reasoning error 13% / hallucination 6% / label ambiguity 5% / other 3%）与 CoT 幻觉率 56%：https://awesome.papernotes.org/en/era4_foundation_models/2022_react
- 方法定位（A ∪ L 动作空间、与 agentic RL 的关系、无 RL）：https://huggingface.co/datasets/rl-llm-wiki/knowledge-base/discussions/188/files
- ALFWorld / WebShop 对比数字与论文自陈的限制：https://paperswelove.org/papers/react-synergizing-reasoning-and-acting-in-language-24dd3c33
