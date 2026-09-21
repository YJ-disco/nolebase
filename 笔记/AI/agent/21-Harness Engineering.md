---
tags:
  - AI/agent/Harness
---

# Harness Engineering

围绕 AI Agent 设计约束、工具、验证机制与反馈回路，让它在生产环境里可靠完成任务的工程实践。管理的是**模型之外的一切**。

一句话概括：`Agent = Model + Harness`。

Harness 本意是马具，指套在马身上、用来传递力量和控制方向的那套装备。换成工程语言，Harness 是 Agent 的运行外壳——它决定 Agent 记住什么、看到什么上下文、能调用什么工具、被允许做什么、出错之后发生什么。

> **一句话定调**：「今天的模型能做的」和「你看到它们做的」之间的差距，**大部分是 harness 差距**，不是模型差距。

## 时间线

| 时间 | 事件 |
| --- | --- |
| 2026-02-05 | Mitchell Hashimoto（HashiCorp 联合创始人，Terraform 作者）在博客《My AI Adoption Journey》中给这件事命名 |
| 2026-02-11 | OpenAI 工程师 Ryan Lopopolo 发布《Harness engineering: leveraging Codex in an agent-first world》 |
| 2026-02-17 | Birgitta Böckeler（Thoughtworks Distinguished Engineer）在 martinfowler.com 首发备忘录 |
| 2026-04-02 | 同一作者发表完整文章《Harness Engineering: Guides and Sensors for Coding Agents》 |
| 2026-09-02 | Google 团队在 DEV Community 发文，给出基于 Google ADK 2.0 的最小实现 |

Hashimoto 的定义值得逐字记住：

> anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again.

也就是：Agent 每犯一次错，就动一次环境，让这类错误在结构上不再可能发生。把「我希望它这样做」变成「它不可能那样做」。

Lopopolo 公开的内部实验数据：一个 3 人团队从 2025 年 8 月底的空仓库开始，5 个月内由 Codex 生成约 **100 万行**代码、合并约 **1500 个 PR**，应用逻辑、测试、CI 配置、文档、可观测性全部零手写。人在这套流程里做的是设计环境、定义意图、搭反馈回路。

## 三个同心圆：harness 是哪一层

「harness」这个词在不同尺度上被使用，不分开就会打架。Böckeler 用同心圆把边界画清了：

```
用户 harness（你建的）
  AGENTS.md · skills · hooks · 自定义 linter · review agent · CI sensors
    ↓ 包住
Builder harness（agent 厂商建的）
  system prompt · 工具集 · 检索机制
    ↓ 包住
Model（被驾驭的东西）
```

- **Model** 是内核
- 厂商围绕它建**内层 harness**（system prompt、工具集、代码检索机制）
- **你**围绕内层建**外层 harness**，为自己的代码库定制

**Harness Engineering 讲的基本都是最外那一圈。** 建得好的外层 harness 做两件事：**提高 agent 第一次就做对的概率**，以及**提供一个自我修正的反馈回路，让很多问题在到达人之前就被处理掉**——减少审查负担、提高质量、少烧 token。

## 两条正交的轴：Guides 与 Sensors

Böckeler 的核心贡献是一套**从控制论借来的词汇**。harness 像一个**调速器（governor）**，用两类控制把代码库调向期望状态：

| 类型 | 时机 | 作用 | 例子 |
| --- | --- | --- | --- |
| **Guides（前馈控制）** | agent **行动之前** | 预测它的行为并**引导**它，提高一次做对的概率 | 编码约定文档、`AGENTS.md`、skills、参考文档、how-to 指南、codemod |
| **Sensors（反馈控制）** | agent **行动之后** | 观测结果并帮它**自我修正** | 类型检查、linter、测试、静态分析、AI review agent |

**两者缺一不可**，原因很直白：

- **只有 guides 没有 sensors** —— harness「编码了规则，却永远不知道规则是否奏效」
- **只有 sensors 没有 guides** —— agent「一直重复同样的错误再修正」

### 每种控制又有两种执行形态

| 形态 | 特性 | 成本 | 例子 |
| --- | --- | --- | --- |
| **Computational（确定性）** | 快、便宜、可预测、CPU 跑 | 低（毫秒到秒级）| linter、结构测试、格式检查、类型检查 |
| **Inferential（推理性）** | 语义理解强、较慢、**非确定性** | 高 | `LLM-as-a-judge`、风格审查、需求符合度检查 |

**这一栏要按需选，不是越强越好**——确定性检查放在最前面（快且可靠），推理性检查放在后面（贵但能做 linter 做不到的判断）。

### 一个精妙的设计：让 sensor 说模型能听懂的话

**最好的 sensors 产出的反馈是「为模型消费优化」的。**

具体做法：一条自定义 linter 消息**不只说「error」，而是包含怎么修的指令**。Böckeler 称这个叫「**一种正向的 prompt injection**」——sensor 用 agent 能行动的语言对它说话。

这条经验在其他地方也出现过：[[03-ReAct]] 里解析失败要把错误信息作为 Observation 塞回历史；[[09-工具系统与 Function Calling|Function Calling]] 里校验失败要返回**描述性错误**而不是抛异常。**同一个原理：错误信息是给模型的输入，所以要按输入的规格来写。**

## 三类调控目标：难度差了一个数量级

| 类型 | 管什么 | 成熟度 |
| --- | --- | --- |
| **可维护性 harness** | 内部代码质量 | **最容易**——工具成熟（linter、复杂度分析、测试覆盖）|
| **架构适配 harness** | 性能、可观测性等架构特性（Fowler 的 fitness functions）| 中 |
| **行为 harness** | **应用是否真的做了它该做的事** | **最难——「房间里的大象」，基本未解** |

第三类为什么难：**AI 生成的测试套件目前还不够可靠**（见 [[15-AI Coding 工作流|AI Coding 工作流]] 里那条「覆盖率 90% 但只测了 happy path」）。**用模型验证模型，会撞上同源盲区**——这是 [[24-LLM Evaluation 与反馈闭环|LLM Evaluation 与反馈闭环]] 要单独处理的问题。

### Shift left

控制要**尽可能地往生命周期的前段放**：

```
commit 前          → linter、单元测试（快、便宜）
集成前            → 类型检查、结构测试
流水线            → 变异测试、架构审查（贵）
持续监控          → 代码漂移、生产指标
```

**越早发现越便宜修**——这是 CI 的老直觉，现在用在 agent 的产出上。

## Ashby 定律与 harnessability

**Ashby 的必要多样性定律**：**调控器必须至少有被调控系统的多样性。**

因为 LLM 几乎能产出任何东西（多样性极高），**承诺一个受约束的架构就是一个降多样性的动作**——固定拓扑、可预测结构，能让一个完整的 harness 变得可达。

由此推出 **harnessability（可驾驭性）**：**不是所有代码库都同样可被驾驭**。

| 更好驾驭 | 更难驾驭 |
| --- | --- |
| 强类型语言 | 动态类型 |
| 抽象框架（如 Spring）| 手写胶水代码 |
| 清晰的模块边界 | 边界模糊 |
| —— | **债台高筑的遗留代码** |

**最后那一格有个讽刺**：遗留系统**最难被驾驭，但它最需要**。

配套概念还有两个：**ambient affordances**（Ned Letcher）——环境里那些让 agent 读得懂、走得通的结构性属性；**harness templates**——把现有的服务模板演进成「每种拓扑一套 guides + sensors」的捆绑（dashboard / CRUD / event processor 这几类是多数企业的主力拓扑）。

## Harness 的结构

按执行、控制、状态切三层：

**工具与执行层。** 读写文件、调用 Bash、访问 API、操作浏览器——工具的定义和授权。工具表面越清晰，Agent 行为越可预期。

**控制与验证层。** 这是 Harness 区别于 Prompt 工程的核心。对照一下：让 Agent「请遵守代码规范」，依赖的是概率性合规；接一个违反规范就阻断 PR 的 Linter，属于确定性结构约束。测试套件、类型检查、权限边界、审批门控接进来之后，这类错误从概率上的减少变成结构上的关闭——它不再有机会进入主干。

**持久化与状态层。** 长任务的瓶颈是上下文窗口一满，之前的工作就丢了。用文件系统和 Git 把状态落盘，Agent 才可能从中断处续跑。

另一套流传较广的划分是三个支柱：上下文工程（在正确的时间给出正确的信息，包括 `AGENTS.md`、架构规范、测试结果）、架构约束（代码规范检查器、自动化测试强制边界）、熵管理（定期清理 AI 生成代码里积累的过时文档、命名偏差、死代码）。

> [!warning] 「三大支柱」这套说法来自中文技术自媒体，与 Böckeler 的 guides / sensors 框架并非同一套划分，未见官方对应，标为待验证。

## 具体的实现组件

LangChain 公开的 harness 组件里有两个值得记的：

| 组件 | 解决什么 |
| --- | --- |
| `LocalContextMiddleware` | 把**环境信息在起始阶段就注入**——这是 Guides 一侧的典型做法 |
| `LoopDetectionMiddleware` | **agent 会卡在重复循环里**——有时对同一个文件做 10 次以上微小修改，每次都是同样的失败路径。这个 middleware 追踪**每个文件的编辑次数**，超过阈值就注入提示，让 agent 重新思考策略 |

**`LoopDetectionMiddleware` 是 [[02-Agent Loop|Agent Loop]] 里「重复动作检测」的一个具体实现**——区别在于它检测的粒度是「同一文件被反复改」，而不是「同一工具被反复调」。后者看不出来的循环，前者能看出来。

**一组值得记的对照数据**：LangChain 团队用同一个模型（Claude Opus 4.6）测试，在**早期版本的 harness** 下跑出 **59.6%**——有竞争力但不如 Codex。**原因是那个 harness 还没跑过同等轮次的迭代改善循环，而不是模型差。**

OpenAI 那边的做法是：**自定义 linter + 结构测试 + 「垃圾回收」**（后者的目标对应上面说的「熵管理」）。

## 规则的分层

常见的分层方式是：

```text
Global Rules        全局，跨项目的个人/团队偏好
Project Rules       项目级约束
Repository Rules    仓库级约束
Task Rules          单次任务约束
```

载体通常落在 Agent 的全局配置和仓库根目录，`AGENTS.md` 是目前跨工具通用性最好的一个（Codex、Claude Code、WorkBuddy 等都原生识别）。`AGENTS.md` 的内容属于 Context 层，而它约束的东西能不能被强制执行，取决于 Harness 层有没有对应的检查器。

## 一条不能忘的边界

**harness 不能可靠地捕获高层问题**——误诊（诊断错了病因）、过度工程、误解指令，这三类都拦不住。

**人类经验仍然是一个不可替代的「隐式 harness」。**

> 目标是**把人的注意力导向最重要的地方**，而不是消灭人——这也和 [[24-LLM Evaluation 与反馈闭环|LLM Evaluation 与反馈闭环]] 里「减少人需要 review 的数据量」是同一个思路。

**Harness Engineering 是一个持续的工程实践，不是一次性的配置。** 它和 [[18-Context Engineering|Context Engineering]] 的关系是：**harness engineering 是应用于编码 agent 的一种特定形式的上下文工程。**

## 与 Prompt / Context Engineering 的层次关系

三个概念叠在一起，不是互相替代：

| 维度 | [[17-Prompt Engineering\|Prompt Engineering]] | [[18-Context Engineering\|Context Engineering]] | Harness Engineering |
| --- | --- | --- | --- |
| 核心问题 | 这句话怎么措辞 | 模型看到哪些信息 | 整个系统怎么运转 |
| 作用范围 | 单条提示 | 一次调用的全部输入 | 工具 + 权限 + 验证 + 状态 + 可观测性 |
| 作用时间 | 编写时一次性确定 | 运行时逐轮组装 | 跨全部轮次持续生效 |
| 失效信号 | 输出含糊、格式错乱 | 缺背景、答非所问 | 多步崩溃、死循环、结果不可预测 |
| 修复手段 | 改措辞 | 重组输入 | 重设计系统架构 |

这个划分能当诊断工具用：单步任务输出含糊 → 改提示；答错方向、缺关键背景 → 调输入组装；多步任务中途崩溃或反复循环 → 提示词改多少遍都没用，问题是缺验证回路或状态管理坏了。

三层按出现时间排序：Prompt Engineering 最早，Context Engineering 2025 年中成型，Harness Engineering 2026 年初才被命名。前两层各自的技术内容在链接里，这一篇只讲它们在体系里的位置。

## 相关

- [[15-AI Coding 工作流|AI Coding 工作流]] —— Harness 之上的工作流程层
- [[22-Rule 与 LLM 的边界|Rule 与 LLM 的边界]] —— 确定性约束与概率性合规的分工
- [[24-LLM Evaluation 与反馈闭环|LLM Evaluation 与反馈闭环]]
- [[19-上下文工程的工程实践|上下文工程的工程实践]] —— Context 这一层的实现

## 参考

- Mitchell Hashimoto 命名与定义、OpenAI Lopopolo 文章与百万行实验数据、Thoughtworks 时间线：https://blog.csdn.net/aidoudoulong/article/details/164332156
- **Böckeler 的完整框架（三个同心圆、guides/sensors 两轴、computational/inferential 两形态、三类调控目标、Ashby 定律、harnessability、harness templates、高层问题拦不住、与上下文工程的关系）**：https://www.thekb.eu/en/fiches/boeckeler-harness-engineering-coding-agents-2026-04-02
- **同心圆的表述、guides/sensors 的互补性、正向 prompt injection、shift left、Ashby 定律的应用**：https://dev.to/raminjafary/the-rise-of-agentic-engineering-part-5-harness-engineering-emerges-2d9o
- **harness 的四个层级（generic / project / domain / delivery）、self-correction loop、项目 harness 是改造起点**：https://www.analytical-software.de/?p=17686/
- **LangChain 的 LocalContextMiddleware 与 LoopDetectionMiddleware、59.6% 的对照、Osmani 的「harness 差距」表述、OpenAI 的 linter + 结构测试 + 垃圾回收**：https://tenten.co/learning/harness-engineering/
- 三层结构与「三大支柱」说法：https://ima.qq.com/wiki/ 分享的《从手动喂 Prompt 到 Harness 工程》一文
- Harness 官方博客对 agent harness「模型之外的一切」的表述：https://www.harness.io/blog/ai-writes-the-code-who-delivers-it-safely
