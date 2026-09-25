---
tags:
  - AI/agent/协议
---

# A2A 与 ANP

[[13-MCP 协议|MCP]] 解决「智能体怎么访问工具」，这两个协议解决的是更进一步的问题：**智能体之间怎么通信，以及在大规模网络里怎么找到对方**。

本篇按 A2A 官方规范（v0.3.0）拆到报文级；ANP 还没有成熟规范，按可查证的部分写，缺口明确标出。

## 一、A2A 是什么

**A2A（Agent2Agent Protocol）** 由 Google 提出。类比是「agent 之间的 HTTP」——**一个中立的线上协议，让两套由互不相识、不共享技术栈的团队构建的系统，都能同意说同一种话**。

它建模两个角色：

- **Client Agent**：想要某件事被完成
- **Remote Agent**：完成它，并**拥有结果**

底下就是 **JSON-RPC 2.0 加一层 SSE 做流式**，外加可选的 webhook 做推送通知。**线上没有任何异乎寻常的东西**——任何一条报文都能格式化出来用眼睛读。

### 三层结构

| 层 | 内容 |
| --- | --- |
| **数据模型** | 核心结构：`Task`、`Message`、`AgentCard`、`Part`、`Artifact` |
| **抽象操作** | 协议无关的能力：SendMessage、GetTask 等 |
| **协议绑定** | 具体实现：**JSON-RPC 2.0 / gRPC / HTTP+JSON(REST)** |

**第三层是可选的原因很重要**：A2A **不发明自己的线上格式**，实现可以任选一种绑定并在 Agent Card 里声明。这就是为什么一个 A2A agent 能直接落进已有的 service mesh 或 API gateway，不需要特殊处理。

## 二、AgentCard：发现与自描述

Agent Card 是 A2A 里承担最多工作的一个对象，**也是被引用得最不准的那个**——很多材料还在用旧路径。

```
标准路径：/.well-known/agent-card.json     ← 遵循 RFC 8615 的 well-known URI 约定
旧路径（已不是标准）：/.well-known/agent.json
```

```json
{
  "name": "my_agent",
  "description": "Agent description",
  "url": "http://localhost:8080/",
  "version": "1.0.0",
  "protocolVersion": "0.3.0",
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"],
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "extendedAgentCard": false
  },
  "skills": [
    {
      "id": "skill_id",
      "name": "Skill Name",
      "description": "What this skill does",
      "examples": ["Example query 1", "Example query 2"],
      "tags": []
    }
  ],
  "securitySchemes": {},
  "security": []
}
```

| 字段 | 声明什么 |
| --- | --- |
| `name` / `description` / `url` / `version` | 这个 agent 是谁、在哪里 |
| `skills` | 它提供什么能力 |
| `capabilities` | 支持哪些协议特性：`streaming`、`pushNotifications`、`extendedAgentCard` |
| `securitySchemes` / `security` | 接受哪些凭据——**用的是 OpenAPI 规范的同一种形状** |
| `defaultInputModes` / `defaultOutputModes` | 能接收和返回的媒体类型 |

### 两个刻意的设计选择

**第一，card 是摘要级的，不枚举远端 agent 的 tools。** 这正是 A2A 避开 [[13-MCP 协议|MCP]] 那种上下文窗口问题的原因——如果每个被发现的 agent 都把完整工具清单塞进来，光发现阶段就把模型的上下文撑爆了。

**第二，可以发两张卡。** 一张受限的公开 card，加一张更丰富、**需要有效凭据才能取**的 extended card：

```json
{ "jsonrpc": "2.0", "id": 1, "method": "agent/getAuthenticatedExtendedCard" }
```

这是协议对「我想被发现，但不想把能力面全公开在互联网上」的回答。

### 客户端怎么找到这张卡

| 方式 | 适用 |
| --- | --- |
| 直接取 well-known URI | 已知 agent 的地址 |
| 查 curated registry（按 skill 或 tag 索引）| 需要发现未知 agent |
| 直接配置（config / 环境变量 / 私有 API）| 固定关系 |

**registry 对企业来说最关键**——它是最自然的那个「决定哪些 agent 允许被发现」的位置。

## 三、五个核心对象

| 对象 | 说明 |
| --- | --- |
| **AgentCard** | 自描述清单（见上）|
| **Task** | 工作单元：`id` + `contextId`（把相关工作归到一组）+ `status` + `history` + `artifacts` |
| **Message** | 一轮对话。`role` 是 `user` 或 `agent`，内部携带 `parts`，还有 `messageId`、`contextId`、可选的 `referenceTaskIds` |
| **Part** | **原子内容容器**，三选一：`text` / `file`（`uri` + `mimeType`）/ `data`（结构化 JSON）|
| **Artifact** | 产出物：`artifactId` + `name` + `description` + `parts` |

**Part 就是整个内容模型**——三个类型就够让两个 agent 交换表格或图片，而不需要自己发明编码方式。

**Messages 与 Artifacts 的分工是一处容易看漏的设计**：

- **Message 是临时的对话轮次**
- **Artifact 是留存下来的交付物**

而且 **artifact 的名字跨修订保持稳定**——一个 agent 在精修草稿时，返回的是**同一个 artifact 的连续版本**，而不是一堆互不相关的 blob。下游因此可以按名字跟踪「这份东西改到第几版了」。

## 四、Task 状态机：九个状态，两个是暂停不是失败

```
submitted → working → completed
                   → canceled
                   → failed
                   → rejected
                   → input-required   ← 可恢复
                   → auth-required    ← 可恢复
                   → unknown
```

| 状态 | 含义 |
| --- | --- |
| `submitted` | 远端 agent 已接收，尚未开始 |
| `working` | 正在执行 |
| **`input-required`** | **暂停**——需要客户端提供东西才能继续 |
| **`auth-required`** | **暂停**——需要它没有的凭据 |
| `completed` | 完成，带 artifacts |
| `failed` | 尝试过但无法完成 |
| `canceled` | 客户端中止 |
| `rejected` | agent 直接拒绝 |
| `unknown` | 状态无法确定 |

终态是 `completed` / `failed` / `canceled` / `rejected`。

**`input-required` 和 `auth-required` 根本不是失败**——它们是**刻意的暂停，把控制权交回客户端**。这就是为什么一个 A2A 任务**是一次协商，而不是一次调用**。这也让它天然适配 [[23-置信度分层与 Human-in-the-loop|Human-in-the-loop]]：agent 不确定时不用猜，直接进 `input-required` 等回答。

## 五、JSON-RPC 方法

| 方法 | 作用 |
| --- | --- |
| `message/send` | 发消息，返回 **Task 或 Message** |
| `message/stream` | 同上，但响应是 SSE 事件流 |
| `tasks/get` | 按 id 取任务（可裁剪 history）|
| `tasks/list` | 列出任务，支持过滤与分页 |
| `tasks/cancel` | **协作式地**请求停止 |
| `tasks/subscribe` / `tasks/resubscribe` | 订阅（或重新订阅）已有任务的更新流 |
| `tasks/pushNotificationConfig/create` / `get` / `list` / `delete` | 注册 webhook，让长任务回调而不是被轮询 |
| `agent/getExtendedCard` / `agent/getAuthenticatedExtendedCard` | 取需认证的完整 card |

规范里方法不少，但**真正干活的是其中五个**：`message/send`、`message/stream`、`tasks/get`、`tasks/cancel`、`tasks/pushNotificationConfig/*`。其余属于生活质量改进。

> [!warning] 网上能看到另一组方法名——`agents/getCard`、`tasks/send`、`tasks/subscribe` 之类。那是早期草案的命名，**v0.3.0 起用的是上表这套**（`message/send`、`tasks/get`……）。查资料时注意版本。

## 六、报文原文

### 请求

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{ "kind": "text", "text": "tell me a joke" }],
      "messageId": "9229e770-767c-417b-a0b0-f0741243c589"
    },
    "metadata": {}
  }
}
```

### 响应一：返回 Task（有工作单元）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "id": "363422be-b0f9-4692-a24d-278670e7c7f1",
    "contextId": "c295ea44-7543-4f78-b524-7a38915ad6e4",
    "status": { "state": "completed" },
    "artifacts": [
      {
        "artifactId": "9b6934dd-37e3-4eb1-8766-962efaab63a1",
        "name": "joke",
        "parts": [
          { "kind": "text",
            "text": "Why did the chicken cross the road? To get to the other side!" }
        ]
      }
    ],
    "history": [
      {
        "role": "user",
        "parts": [{ "kind": "text", "text": "tell me a joke" }],
        "messageId": "9229e770-767c-417b-a0b0-f0741243c589",
        "taskId": "363422be-b0f9-4692-a24d-278670e7c7f1",
        "contextId": "c295ea44-7543-4f78-b524-7a38915ad6e4"
      }
    ],
    "kind": "task",
    "metadata": {}
  }
}
```

如果任务耗时更长，服务端可能先返回 `status.state: "working"`，客户端随后周期性调 `tasks/get` 直到进入终态。

### 响应二：直接返回 Message（没有工作单元）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "messageId": "363422be-b0f9-4692-a24d-278670e7c7f1",
    "contextId": "c295ea44-7543-4f78-b524-7a38915ad6e4",
    "parts": [
      { "kind": "text",
        "text": "Why did the chicken cross the road? To get to the other side!" }
    ],
    "kind": "message",
    "metadata": {}
  }
}
```

**`message/send` 能返回两种东西之一，这一点必须处理**：短问答直接回 `kind: "message"`，长任务回 `kind: "task"`。客户端如果只按一种写，遇到另一种就会崩。判别字段就是 `kind`。

### 一个版本差异要留意

**Part 的类型字段在官方 v0.3.0 报文里是 `kind`**（`{"kind": "text", ...}`），而部分早期材料和 SDK 文档写成 `type`。对接时以目标实现的规范版本为准，别照抄博客。

## 七、流式与推送

**流式（`message/stream`）** 的事件顺序是：先发 **Task 对象**，然后发 **`status-update`** 和 **`artifact-update`** 事件，任务到达终态时关闭流。

**推送（push notification）** 用于**以分钟或小时计的**工作——这时候流式不实用（连接要一直挂着），改用 webhook 回调替代轮询。注册走 `tasks/pushNotificationConfig/create`。

选哪条路取决于任务时长和客户端的部署形态：客户端在浏览器里、任务几十秒 → 流式；客户端是个服务、任务几小时 → webhook。

## 八、ANP：还没长成规范的那一条

**ANP（Agent Network Protocol）** 和上面两个不在同一个成熟度上。原文的定性是：**它是一个概念性的协议框架，目前由开源社区维护，还没有成熟的生态。**

它解决的是「**如何在大规模网络中发现和连接智能体**」，三个挑战：

| 挑战 | 问题 |
| --- | --- |
| **服务发现** | 新任务到达时，如何快速找到能处理它的智能体？ |
| **智能路由** | 多个智能体都能处理同一任务时，如何选最合适的（按负载、成本）并分派？ |
| **动态扩展** | 新加入网络的智能体如何被其他成员发现和调用？ |

### 三步流程（据原书整理）

**1. 服务的发现与匹配。** 智能体 A 通过公开的发现服务，基于语义或功能描述查询，定位到符合需求的智能体 B。发现服务**预先爬取各智能体对外暴露的标准端点（`.well-known/agent-descriptions`）建立索引**。

`.well-known/...` 这个路径约定沿用 Web 的既有惯例，意味着**智能体的可发现性被建模成一个 Web 问题**，而不是需要私有注册中心的问题。

**2. 基于 DID 的身份验证。** 交互开始时，A 用私钥对包含自身 DID 的请求签名；B 收到后解析该 DID 获取对应公钥，验证签名真实性与请求完整性。关键在于**没有中央签发机构**——把身份验证的信任根基从「某个平台发我个 token」换成了「我持有私钥」。

**3. 标准化的服务执行。** 双方依据预定义的标准接口和数据格式交换数据或调用服务。

### 一个必须说清的边界

ANP 的定位是「**用 DID 构建去中心化的信任根基，借标准化的描述协议实现服务的动态发现**，让智能体能在无需中央协调的前提下形成协作网络」。

> [!warning] **ANP 的 wire level 细节目前查不到稳定的公开规范。** 上面三步流程来自原书叙述，`.well-known/agent-descriptions` 这个端点路径与 DID 的用法**未在官方规范文档中逐字段核实**——原书明确说它「还是一个概念性的协议框架，没有成熟的生态」。要把它写到和 A2A 同等的深度，需要等它发布稳定规范，或者去读它的 GitHub 仓库源码。这一处是**已知缺口**，不是省略。

## 九、三个协议的边界

| 协议 | 连接的是 | 线上是什么 | 成熟度 |
| --- | --- | --- | --- |
| **MCP** | 智能体 ↔ 工具/资源 | JSON-RPC 2.0（stdio / Streamable HTTP）| 生态相对成熟 |
| **A2A** | 智能体 ↔ 智能体 | JSON-RPC 2.0 / gRPC / REST + SSE | 有正式规范（v0.3.0）|
| **ANP** | 智能体 ↔ 大规模网络 | 未稳定 | 概念框架，社区维护 |

一句串联：**MCP 解决「怎么访问工具」，A2A 解决「怎么和其他智能体对话」，ANP 解决「怎么在大规模网络中发现和连接智能体」。**

### 与框架内编排的分界

[[06-智能体框架的编排模型]] 里那四种编排模型（AutoGen / AgentScope / CAMEL / LangGraph）解决的是**同一个应用内部**多个智能体怎么协作——它们共享一个运行时，可以直接传对象。

**A2A 要解决的是跨应用、跨平台**——双方**没有共享运行时**，所以必须发明一套自描述（AgentCard）、一套任务生命周期（九状态机）、一套异步协商机制（两个可恢复暂停状态）。这就是协议必要性的来源。

### 三者的关系不是三选一

真实系统里可以叠加：智能体内部用 MCP 接工具，对外用 A2A 与其他智能体协作；网络规模大到需要动态发现时，再引入 ANP 那类机制。选型判据：

- 智能体要访问外部服务（文件、数据库、API）→ **MCP**
- 多个智能体相互协作完成任务 → **A2A**
- 构建大规模智能体生态 → **ANP**

## 十、A2A 的工程现实

`a2a-sdk` 已经把协议正确性的部分做完了：

| SDK 里已有 | 说明 |
| --- | --- |
| 所有 Pydantic 模型 | `AgentCard` / `Task` / `Message` / `Part` / `Artifact` / `TaskState` / `TaskArtifactUpdateEvent` |
| JSON-RPC 2.0 传输与请求路由 | —— |
| SSE 流式事件队列 | —— |
| `InMemoryTaskStore` | 默认任务存储 |
| `BasePushNotificationSender` | webhook 派发器 |
| `TaskUpdater` | 让 skill 直接说 `updater.complete()` / `updater.requires_input()`，不用手工构造状态事件 |
| `ClientFactory` / `AuthInterceptor` | 客户端侧 |

**要自己写的只有四样：AgentCard、AgentExecutor、你的 skills、认证中间件。** 一个能跑通七种场景（发现 / 普通发送 / 流式 / 多轮 input-required / 取消 / 推送 / 扩展 card）的示例应用，应用代码大约 **356 行**。

**这个规模对比值得记**：协议本身复杂，但**协议正确性已经被 SDK 吸收掉了**——这也是为什么前面说 A2A 的难点不在实现，而在把任务生命周期想清楚（什么时候该进 `input-required`、artifact 怎么命名、`contextId` 怎么分组）。

## 相关

- [[13-MCP 协议|MCP 协议]] —— 另一条轴：智能体与工具
- [[06-智能体框架的编排模型]] —— 应用内部的多智能体协作，与 A2A 的跨应用协作相对
- [[23-置信度分层与 Human-in-the-loop|置信度分层与 Human-in-the-loop]] —— `input-required` 是协议层对 HITL 的原生支持

## 参考

- https://a2a-protocol.org/v0.3.0/specification/
- https://eliteai.tools/agent-skills/a2a-protocol-1
- http://agen.co/learning-center/mcp-vs-a2a
- https://tuhidulhossain.com/blog/the-agent2agent-a2a-protocol-a-complete-guide-to-ai-agent-interoperability-20260419/
- 《Hello-Agents》第十章 §10.4
