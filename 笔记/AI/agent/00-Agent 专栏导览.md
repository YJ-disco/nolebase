---
tags:
  - AI/agent
---

# Agent 专栏导览

本专栏收录 AI Agent 的工程实践：把模型当成一个不完全可靠的组件，用规格、规则、评测、评审和反馈回路，把它接进可靠的软件流程。

模型自身的原理在 `../llm/`（架构、分词、位置编码、采样），经典机器学习在 `../ml/`。

## 目录

**基础**

- 01 · [[01-智能体核心概念|智能体核心概念]] —— 四要素定义、架构与程序、理性智能体、传统智能体的五级阶梯、三个分类维度、PEAS 与环境分类七维、Workflow 与 Agent 的边界、协作架构的三种范式
- 02 · [[02-Agent Loop|Agent Loop]] —— 感知-思考-行动-观察循环、每一轮重新组装输入、三类停止条件（三道硬上限各抓什么）、五个失败模式、上下文增长与错误累积

**经典范式**

- 03 · [[03-ReAct|ReAct]] —— 动作空间 `A ∪ L`、工具定义三要素、提示词模板、输出解析正则、官方实验数据、失败模式分布（search error 23% / hallucination 6%）、人可以改轨迹
- 04 · [[04-Plan-and-Solve|Plan-and-Solve]] —— 论文的零样本 PS Prompting 与工程上的 PS Agent 是两个层次；PS/PS+ 提示模板、错误类型分布、计划存在率
- 05 · [[05-Reflection|Reflection]] —— 语言强化与语义梯度、三个组件、Evaluator 的三种实现、记忆两层（k=3）、HumanEval 91%、消融实验与三条局限

**框架**

- 06 · [[06-智能体框架的编排模型|智能体框架的编排模型]] —— 框架抽象掉的四件事，四种编排模型（AutoGen 对话轮询 / CAMEL 角色扮演与 Inception Prompting / AgentScope 消息驱动 / LangGraph 状态图）
- 07 · [[07-Agent 框架的抽象设计|Agent 框架的抽象设计]] —— 四个设计理念、「万物皆为工具」的取舍、core/agents/tools 三层与依赖方向、Message / Config / Agent 基类、五种范式收敛到同一个 `run()`
- 08 · [[08-LLM 接入层与多提供商|LLM 接入层与多提供商]] —— 「OpenAI 兼容」的三个层次、继承式扩展、静态适配表、流式响应的三个必处理细节、provider 自动推断的四级优先级与它的脆弱性
- 09 · [[09-工具系统与 Function Calling|工具系统与 Function Calling]] —— Tool 基类的自描述能力、ToolRegistry 两种注册、Function Calling 的完整报文与字段拆解、两轮往返的三条不变量、安全与成本

**记忆与检索**

- 10 · [[10-智能体记忆系统|智能体记忆系统]] —— LLM 的两个根本局限、认知科学的映射（三层记忆 + 形成五阶段）、四层架构、MemoryTool 九个操作、四种记忆类型与三种评分公式、三条跨类型规律
- 11 · [[11-RAG 检索增强|RAG 检索增强]] —— 六阶段链路、分块五策略与实测参数区间、嵌入链路四步、RRF 融合、两阶段重排、MQE 与 HyDE、上下文组装、失败的两种来源、评测与成本
- 12 · [[12-向量检索与 ANN 索引|向量检索与 ANN 索引]] —— 检索侧的四种方法、距离度量三选、为什么 B+ 树不行、HNSW / IVF / PQ / DiskANN / ScaNN、选型表、top-k 怎么取、一条查询的完整旅程

**通信协议**

- 13 · [[13-MCP 协议|MCP 协议]] —— JSON-RPC 2.0 报文层、生命周期握手、三个服务端原语（Tools / Resources / Prompts）、客户端反向能力（Sampling / Roots / Elicitation）、通用机制、传输层、MCP 网关
- 14 · [[14-A2A 与 ANP|A2A 与 ANP]] —— A2A 的对等通信、ANP 的服务发现三挑战与 DID 身份验证、三个协议的边界与选型

**工作流与规格**

- 15 · [[15-AI Coding 工作流|AI Coding 工作流]] —— Vibe Coding 的失效边界，Spec-Driven Development 的四阶段流程
- 16 · [[16-Spec 与需求理解|Spec 与需求理解]] —— 输入输出契约怎么定、需求文档互相矛盾时怎么处理、Spec 粒度怎么控制

**约束与边界**

前两组按出现时间排：Prompt Engineering 最早，Context Engineering 2025 年中成型，Harness Engineering 2026 年初才被命名，后两层都叠在前一层之上。

- 17 · [[17-Prompt Engineering|Prompt Engineering]] —— 单条提示怎么写：零/单/少样本、指令调优改变了什么、角色扮演、思维链
- 18 · [[18-Context Engineering|Context Engineering]] —— 一次调用的全部输入怎么组装：四个核心操作、六个信息源的 token 争夺、compaction 与结构化笔记、context rot
- 19 · [[19-上下文工程的工程实践|上下文工程的工程实践]] —— 系统提示/工具/示例三个组件的工程做法、JIT 上下文与渐进式披露、长时程三手段、GSSC 流水线
- 20 · [[20-上下文工具：NoteTool 与 TerminalTool|上下文工具：NoteTool 与 TerminalTool]] —— 结构化笔记的四种类型与落地、只读命令白名单、两种工具的设计对照
- 21 · [[21-Harness Engineering|Harness Engineering]] —— 模型之外的运行环境：工具与执行、控制与验证、持久化与状态三层
- 22 · [[22-Rule 与 LLM 的边界|Rule 与 LLM 的边界]] —— 确定性交给规则、开放语义交给模型、高风险交给人

**可信度**

- 23 · [[23-置信度分层与 Human-in-the-loop|置信度分层与 Human-in-the-loop]] —— 校准与区分度、Selective Prediction、降级路由
- 24 · [[24-LLM Evaluation 与反馈闭环|LLM Evaluation 与反馈闭环]] —— 裁判偏差与缓解、从线上 Bad Case 到回归测试
- 25 · [[25-AI 生成代码的质量保障|AI 生成代码的质量保障]] —— 同源盲区、确定性约束优先、质量门禁链路

## 阅读顺序

01 → 02 铺基础；03 → 05 是三个经典范式的实现（ReAct 走一步看一步、Plan-and-Solve 先谋后动、Reflection 事后校正，前两个都可以当第三个的初稿环节）；06 → 14 是能力层（06 → 09 框架：06 横向看别人怎么做编排，07 → 09 是自建一套要处理的三件事——抽象、模型接入、工具系统；10 → 12 记忆与检索：从「窗口之外维持状态」到「六阶段检索链路」再到「检索侧的算法层」；13 → 14 通信协议：怎么把自己接出去）；15 → 16 是流程侧；17 → 22 是约束侧；23 → 25 是可信度侧。

七段相互依赖：没有 23～25 的评测闭环，17～22 定的规则就只是拍脑袋。
