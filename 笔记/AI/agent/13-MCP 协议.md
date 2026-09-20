---
tags:
  - AI/agent/协议
---

# MCP 协议

**MCP（Model Context Protocol）** 由 Anthropic 提出，把智能体与外部工具/资源的通信方式标准化。类比是「智能体的 USB-C」：不管用 Claude、GPT 还是别的模型，只要支持 MCP 就能访问同一批工具。

本篇按官方规范（2025-06-18 版）逐层拆到报文级。报文全部来自规范原文。

## 一、它建立在模型的哪个能力上

MCP 不是凭空出现的，它**建立在 Function Calling 之上**。

Function Calling 给了模型两件事：能读懂一段 JSON Schema 描述的工具定义，以及在对话中输出结构化的调用意图。MCP 要标准化的正是这条链路的两端——**工具定义怎么描述、调用怎么送达执行方**。

MCP 之前这两端都锁在 SDK 里：OpenAI 用 `parameters`、Claude 用 `input_schema`，每接一个模型平台就重写一遍工具定义（见 [[09-工具系统与 Function Calling]]）。MCP 把工具定义和执行收进 Server，用一套协议暴露出去，模型侧只要支持 MCP 就能复用。

**所以 MCP 的定位不是「更强的工具调用」，而是「工具调用的可移植层」。**

## 二、架构与四条设计原则

MCP 是 **client-host-server** 架构，一个 host 可以跑多个 client 实例。

| 角色 | 职责 |
| --- | --- |
| **Host** | 容器与协调者：创建并管理多个 client 实例、控制连接权限与生命周期、执行安全策略与同意要求、处理用户授权、协调 LLM 集成与采样、聚合跨 client 的上下文 |
| **Client** | 由 host 创建，**与某个 server 维持 1:1 的隔离连接**：建立有状态会话、处理协议协商与能力交换、双向路由消息、管理订阅与通知、维持 server 之间的安全边界 |
| **Server** | 通过原语暴露 resources / tools / prompts、独立运行且职责聚焦、**通过 client 接口请求采样**、必须遵守安全约束、可以是本地进程也可以是远程服务 |

四条设计原则决定了后面所有细节：

1. **Server 应该极容易构建**——复杂编排归 host，server 只管自己那块能力
2. **Server 应该高度可组合**——每个 server 独立提供聚焦功能，多个 server 可无缝组合
3. **Server 不能读取整段对话，也「看不进」其他 server**——server 只拿到必要的上下文，完整对话历史留在 host；连接之间相互隔离，跨 server 交互由 host 控制
4. **特性可以渐进添加**——核心协议只给最小必需功能，其余靠协商，双方独立演进，保持向后兼容

第 3 条是整个安全模型的基石：**隔离发生在协议层，不是靠约定**。

## 三、报文层：JSON-RPC 2.0

MCP 的报文**就是 JSON-RPC 2.0**，必须 UTF-8 编码。三类消息：

| 消息类型 | 特征 | 用途 |
| --- | --- | --- |
| Request | 有 `id`，期待响应 | `initialize`、`tools/list`、`tools/call` |
| Response | 有 `id`，含 `result` **或** `error` | 应答 |
| Notification | **无 `id`**，不需要响应 | `notifications/initialized`、`notifications/tools/list_changed` |

`jsonrpc: "2.0"` 与 `id` 是所有报文的骨架。**`id` 是请求与响应的配对依据**——这也是超时后要主动发取消通知、而不是干等的原因。

Notification 的「fire and forget」性质贯穿全篇：取消、进度、列表变化全靠它，都不需要应答。

## 四、生命周期与握手

生命周期分三段：**初始化 → 运行 → 关闭**。初始化**必须**是客户端和服务器的第一次交互。

### 第一步：客户端发 `initialize`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {},
      "elicitation": {}
    },
    "clientInfo": {
      "name": "ExampleClient",
      "title": "Example Client Display Name",
      "version": "1.0.0"
    }
  }
}
```

### 第二步：服务器回自己的能力和信息

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true }
    },
    "serverInfo": {
      "name": "ExampleServer",
      "title": "Example Server Display Name",
      "version": "1.0.0"
    },
    "instructions": "Optional instructions for the client"
  }
}
```

### 第三步：客户端发 `initialized` 通知

```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

没有 `id`，不期待响应。它标志着握手完成。

### 握手期间的硬约束

- 服务器响应 `initialize` **之前**，客户端除了 ping 不该发任何请求
- 收到 `initialized` **之前**，服务器除了 ping 和 logging 不该发任何请求

这两条把「谁先动」卡死了，避免双方在能力尚未确定时就开始调用。

### 版本协商

三条规则，顺序不能错：

1. 客户端**必须**在 `initialize` 里带上它支持的最新协议版本
2. 服务器支持这个版本 → **必须**回同一个版本；不支持 → **必须**回一个它自己支持的版本
3. 客户端收到后如果不支持服务器回的版本 → **应该**断开

HTTP 传输下，客户端**必须**在后续所有请求上带 `MCP-Protocol-Version: <version>` 头。兼容兜底：服务器收不到这个头且无法从别处推断时**应该假定 `2025-03-26`**；收到无效值**必须**返回 `400 Bad Request`。

### 能力协商

握手交换的 `capabilities` 决定这个会话里哪些功能可用。**能力是开关：声明了才能用，没声明就不能调。**

| 方向 | 能力 | 含义 |
| --- | --- | --- |
| Client | `roots` | 能提供文件系统根目录 |
| Client | `sampling` | 支持服务器发起的 LLM 采样请求 |
| Client | `elicitation` | 支持服务器向用户追问 |
| Client | `experimental` | 支持非标准实验特性 |
| Server | `prompts` | 提供提示模板 |
| Server | `resources` | 提供可读资源 |
| Server | `tools` | 暴露可调用工具 |
| Server | `logging` | 输出结构化日志 |
| Server | `completions` | 支持参数自动补全 |
| Server | `experimental` | 支持非标准实验特性 |

子能力两个：**`listChanged`**（列表变化时会不会发通知，prompts / resources / tools 都有）和 **`subscribe`**（能否订阅单个条目的变更，只有 resources 有）。两者都是可选的，可以都不支持、只支持其一、或都支持：

```json
{ "capabilities": { "resources": {} } }                     // 都不支持
{ "capabilities": { "resources": { "subscribe": true } } }  // 只支持订阅
```

**注意方向**：`sampling` 和 `elicitation` 是**服务器向客户端**发请求——MCP 是双向的，不是单纯的 client→server 调用。第五、六节会展开。

### 关闭与超时

协议**没有定义专门的关闭消息**，靠底层传输机制表达：

- **stdio**：客户端先关闭子进程的输入流，等服务器退出；不退则 `SIGTERM`，再退不出则 `SIGKILL`。服务器也可以关掉自己的输出流并退出
- **HTTP**：关闭对应的 HTTP 连接

超时方面：所有请求**应该**有超时，超时后**应该**发取消通知并停止等待。收到对应请求的进度通知时**可以**重置超时时钟（说明工作确实在推进），但**应该**始终存在一个最大超时。

## 五、服务端的三个原语

MCP 官方把三者按**控制方**区分，这是理解它们的钥匙：

| 原语 | 控制方 | 定位 |
| --- | --- | --- |
| **Tools** | **model-controlled** | 模型自主发现并调用 |
| **Resources** | **application-driven** | 由宿主应用决定怎么纳入上下文（如做成资源选择器、搜索、或按启发式自动纳入）|
| **Prompts** | **user-controlled** | 用户显式选择，典型形态是斜杠命令 |

### 5.1 Tools

服务器**必须**声明 `tools` 能力：

```json
{ "capabilities": { "tools": { "listChanged": true } } }
```

**列出工具**（支持分页）：

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list",
  "params": { "cursor": "optional-cursor-value" } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": { "type": "string", "description": "City name or zip code" }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

工具定义字段：`name`、`title`（可选，给人看的显示名）、`description`、`inputSchema`（JSON Schema）、`outputSchema`（可选）、`annotations`（描述工具行为的附加属性）。

**调用工具**：

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "get_weather", "arguments": { "location": "New York" } } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      { "type": "text",
        "text": "Current weather in New York:\nTemperature: 72°F\nConditions: Partly cloudy" }
    ],
    "isError": false
  }
}
```

`result.content` 是**数组**，元素按 `type` 区分——**一次调用可以同时返回多种类型**：

```json
{ "type": "text", "text": "Tool result text" }

{ "type": "image", "data": "base64-encoded-data", "mimeType": "image/png" }

{ "type": "audio", "data": "base64-encoded-audio-data", "mimeType": "audio/wav" }

{ "type": "resource_link", "uri": "file:///project/src/main.rs", "name": "main.rs",
  "description": "Primary application entry point", "mimeType": "text/x-rust" }

{ "type": "resource", "resource": { "uri": "file:///project/src/main.rs",
  "mimeType": "text/x-rust", "text": "fn main() { ... }" } }
```

**结构化输出**：除了 `content`，还可以带 `structuredContent`。如果工具声明了 `outputSchema`，服务器**必须**返回符合该 schema 的结构化结果，客户端**应该**校验；为向后兼容，**应该**同时在 `content` 里放一份序列化 JSON。

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [{ "type": "text",
      "text": "{\"temperature\": 22.5, \"conditions\": \"Partly cloudy\", \"humidity\": 65}" }],
    "structuredContent": { "temperature": 22.5, "conditions": "Partly cloudy", "humidity": 65 }
  }
}
```

> [!warning] `structuredContent` 是**服务器产出的结果数据**，与 LLM 领域的「结构化输出（schema 约束的模型生成）」不是一回事。

列表变化通知：

```json
{ "jsonrpc": "2.0", "method": "notifications/tools/list_changed" }
```

### 5.2 Resources

Resources 由 **URI 唯一标识**（RFC 3986）。服务器**必须**声明 `resources` 能力。

**列出资源**（支持分页）：

```json
{ "jsonrpc": "2.0", "id": 1, "method": "resources/list",
  "params": { "cursor": "optional-cursor-value" } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resources": [
      {
        "uri": "file:///project/src/main.rs",
        "name": "main.rs",
        "title": "Rust Software Application Main File",
        "description": "Primary application entry point",
        "mimeType": "text/x-rust"
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

资源定义字段：`uri`、`name`、`title`（可选）、`description`（可选）、`mimeType`（可选）、`size`（可选，字节数）。

**读取资源**：

```json
{ "jsonrpc": "2.0", "id": 2, "method": "resources/read",
  "params": { "uri": "file:///project/src/main.rs" } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      { "uri": "file:///project/src/main.rs", "mimeType": "text/x-rust",
        "text": "fn main() {\n    println!(\"Hello world!\");\n}" }
    ]
  }
}
```

**内容二选一**：文本用 `text`，二进制用 `blob`（base64）。

```json
{ "uri": "file:///example.txt", "mimeType": "text/plain", "text": "Resource content" }
{ "uri": "file:///example.png", "mimeType": "image/png", "blob": "base64-encoded-data" }
```

**资源模板**（RFC 6570 URI 模板，参数可通过补全 API 自动补全）：

```json
{ "jsonrpc": "2.0", "id": 3, "method": "resources/templates/list",
  "params": { "cursor": "optional-cursor-value" } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      { "uriTemplate": "file:///{path}", "name": "Project Files",
        "title": "📁 Project Files", "description": "Access files in the project directory",
        "mimeType": "application/octet-stream" }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

**订阅**（需要 `subscribe` 能力）：

```json
// 订阅
{ "jsonrpc": "2.0", "id": 4, "method": "resources/subscribe",
  "params": { "uri": "file:///project/src/main.rs" } }

// 变化通知
{ "jsonrpc": "2.0", "method": "notifications/resources/updated",
  "params": { "uri": "file:///project/src/main.rs" } }

// 列表变化通知
{ "jsonrpc": "2.0", "method": "notifications/resources/list_changed" }
```

**Annotations** 三个字段，资源、资源模板、内容块都支持：

| 字段 | 含义 |
| --- | --- |
| `audience` | 给谁看，取值 `"user"` / `"assistant"` |
| `priority` | 0.0–1.0，1 是「最重要（相当于必需）」，0 是「完全可选」|
| `lastModified` | ISO 8601 时间戳 |

```json
{
  "uri": "file:///project/README.md",
  "name": "README.md",
  "title": "Project Documentation",
  "mimeType": "text/markdown",
  "annotations": { "audience": ["user"], "priority": 0.8,
                   "lastModified": "2025-01-12T15:00:58Z" }
}
```

**常见 URI scheme**：`https://`（**只在客户端能自己直接抓取时用**，否则应该换 scheme 或自定义）、`file://`（不一定映射真实文件系统，非普通文件可用 XDG MIME type 如 `inode/directory`）、`git://`。自定义 scheme **必须**符合 RFC 3986。

**错误码**：资源不存在 `-32002`，内部错误 `-32603`。

```json
{ "jsonrpc": "2.0", "id": 5,
  "error": { "code": -32002, "message": "Resource not found",
             "data": { "uri": "file:///nonexistent.txt" } } }
```

### 5.3 Prompts

Prompts 由 **user-controlled**——典型形态是斜杠命令，用户显式选一个模板。服务器**必须**声明 `prompts` 能力。

**列出提示**（支持分页）：

```json
{ "jsonrpc": "2.0", "id": 1, "method": "prompts/list",
  "params": { "cursor": "optional-cursor-value" } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "title": "Request Code Review",
        "description": "Asks the LLM to analyze code quality and suggest improvements",
        "arguments": [
          { "name": "code", "description": "The code to review", "required": true }
        ]
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

**获取提示内容**——注意它返回的是**一条可直接喂给模型的 messages 数组**，而不只是模板字符串：

```json
{ "jsonrpc": "2.0", "id": 2, "method": "prompts/get",
  "params": { "name": "code_review",
              "arguments": { "code": "def hello():\n    print('world')" } } }
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "Code review prompt",
    "messages": [
      {
        "role": "user",
        "content": { "type": "text",
                     "text": "Please review this Python code:\ndef hello():\n    print('world')" }
      }
    ]
  }
}
```

`messages[].content` 支持的类型与 tools 一致：`text` / `image`（base64 + MIME）/ `audio` / `resource`（嵌入资源，需带 uri、mimeType，内容为 `text` 或 `blob`）。

列表变化通知：

```json
{ "jsonrpc": "2.0", "method": "notifications/prompts/list_changed" }
```

**错误码**：提示名无效 `-32602`，缺必需参数 `-32602`，内部错误 `-32603`。

## 六、客户端的反向能力

这一节是 MCP 最容易被忽略的部分——**服务器也能向客户端发请求**。

### Sampling：服务器请求 LLM 生成

采样让服务器能实现 agentic 行为：LLM 调用可以**嵌套在 server 的功能内部**。关键收益是**服务器不需要任何 API key**——模型访问、选择、权限都留在客户端手里。

客户端**必须**声明 `sampling` 能力。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      { "role": "user", "content": { "type": "text", "text": "What is the capital of France?" } }
    ],
    "modelPreferences": {
      "hints": [{ "name": "claude-3-sonnet" }],
      "intelligencePriority": 0.8,
      "speedPriority": 0.5
    },
    "systemPrompt": "You are a helpful assistant.",
    "maxTokens": 100
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "role": "assistant",
    "content": { "type": "text", "text": "The capital of France is Paris." },
    "model": "claude-3-sonnet-20240307",
    "stopReason": "endTurn"
  }
}
```

**`modelPreferences` 是这一节的设计核心。** 服务器和客户端可能用不同的模型供应商，服务器没法直接点名某个模型——客户端可能根本没有，或者更愿意用等价模型。所以 MCP 用「抽象优先级 + 可选提示」两层解决：

| 字段 | 含义 |
| --- | --- |
| `costPriority` | 0–1，值越高越倾向便宜模型 |
| `speedPriority` | 0–1，值越高越倾向低延迟模型 |
| `intelligencePriority` | 0–1，值越高越倾向能力更强的模型 |
| `hints` | 模型名或族名的**子串匹配**，按顺序优先；**只是建议，最终选择权在客户端** |

```json
{
  "hints": [ { "name": "claude-3-sonnet" }, { "name": "claude" } ],
  "costPriority": 0.3, "speedPriority": 0.8, "intelligencePriority": 0.5
}
```

规范给的例子：客户端没有 Claude 但有 Gemini 时，**可以**把 `sonnet` 这个 hint 映射到能力相近的 `gemini-1.5-pro`。所以 hint 是「意图」不是「指令」。

采样请求也可以被拒绝：

```json
{ "jsonrpc": "2.0", "id": 1,
  "error": { "code": -1, "message": "User rejected sampling request" } }
```

安全上要求**人始终在环**：客户端应该提供方便审核采样请求的 UI、允许用户在发送前查看和编辑提示、在交付前把生成结果呈现出来让人过目。

### Roots：客户端告诉服务器该看哪些目录

`roots` 能力让客户端把「文件系统根目录」告诉服务器，从而界定服务器的工作范围。客户端声明 `roots.listChanged` 后，根目录变化时发通知。

### Elicitation：服务器向用户追问

服务器在需要补充信息时，可以通过客户端向用户提问。与 sampling 一样，这是**服务器发起、用户在场**的交互。

## 七、通用机制

### 分页

`tools/list`、`resources/list`、`resources/templates/list`、`prompts/list` 四个列表操作都支持分页，模式统一：

```
请求带 params.cursor → 响应返回 nextCursor
nextCursor 存在就继续带它请求，不存在就是最后一页
```

**分页过程中列表不应变化。** 实现里最容易漏掉的是「没有 `nextCursor` 就收工」这个终止判断——漏了就永远翻不到头，或者只拿到第一页。

### 取消

任一方都可以取消进行中的请求，发 `notifications/cancelled`：

```json
{ "jsonrpc": "2.0", "method": "notifications/cancelled",
  "params": { "requestId": "123", "reason": "User requested cancellation" } }
```

行为约束里有几条必须记住：

1. 取消通知**只能**针对**同方向**先前发出、且**认为仍在进行中**的请求
2. **客户端不得取消 `initialize` 请求**
3. 接收方**应该**：停止处理、释放资源、**不为被取消的请求发响应**
4. 接收方**可以**忽略取消（请求未知 / 已完成 / 本身不可取消）
5. 发出取消的一方**应该**忽略之后才到达的响应

因为网络延迟，取消通知可能在处理完成后、甚至响应已发出后才到达——**双方都必须优雅处理这个竞态**。取消不保证「已经停下」。

### 进度

请求方可以带进度令牌，接收方回进度通知。规范允许实现收到进度通知时**重置超时时钟**（说明确实在推进），但必须仍有一个最大超时兜底。

### ping 与 logging

`ping` 是握手期间唯一被允许的请求——说明它是保活用的。`logging` 是服务器声明后向客户端输出结构化日志的能力。

### 补全

`completions` 能力用于参数自动补全，被 `resources/templates/list` 的模板参数和 `prompts/get` 的参数复用。

## 八、错误码一览

| 场景 | 码 | 载体 |
| --- | --- | --- |
| 参数无效（含未知工具名、提示名无效、缺必需参数）| `-32602` | 顶层 `error` |
| 资源不存在 | `-32002` | 顶层 `error` |
| 内部错误 | `-32603` | 顶层 `error` |
| 用户拒绝采样 | `-1`（实现自定义）| 顶层 `error` |
| 工具执行失败 | —— | `result.isError: true` |

**最后一个必须单独拎出来。** 工具错误不走 JSON-RPC 错误通道，而是走 `result` 里的 `isError`：

```json
// 协议错误：调用本身不成立
{ "jsonrpc": "2.0", "id": 3,
  "error": { "code": -32602, "message": "Unknown tool: invalid_tool_name" } }

// 工具执行错误：调用成立，业务失败
{ "jsonrpc": "2.0", "id": 4,
  "result": {
    "content": [{ "type": "text", "text": "Failed to fetch weather data: API rate limit exceeded" }],
    "isError": true
  } }
```

区别的意义在于：前者是客户端该修的 bug，后者**要交给模型去理解和重试**——所以它必须以正常结果的形式回到对话里，才能成为模型的输入。

## 九、完整交互时序

```
客户端                                        服务器
  │  ── initialize (protocolVersion + capabilities + clientInfo) ──▶
  │  ◀── result (protocolVersion + capabilities + serverInfo) ──────
  │  ── notifications/initialized ────────────────────────────────▶
  │
  │  ── tools/list {cursor} ───────────────────────────────────────▶
  │  ◀── result {tools: [...], nextCursor} ────────────────────────
  │      （有 nextCursor 就继续翻页）
  │
  │      （客户端把 tools[] 的 name/description/inputSchema 转成
  │        模型能读的工具清单，注入系统提示词；模型决定调用）
  │
  │  ── tools/call {name:"get_weather", arguments:{location}} ────▶
  │  ◀── result {content: [...], isError: false} ──────────────────
  │
  │      （result 作为工具结果回填给模型，模型生成最终回答）
  │
  │  ── 关闭 stdin / 关闭 HTTP 连接 ───────────────────────────────▶
```

**第 2 步和第 5 步之间发生的事情不在协议里**：把工具清单喂给模型、模型决定调用、把结果回填——这些是客户端（Host）的职责。MCP 只负责把 Server 的工具描述取回来、把调用送达。

反方向的一条链路（服务器请求采样）：

```
服务器                                       客户端/Host
  │  ── sampling/createMessage {messages, modelPreferences, maxTokens} ──▶
  │      （客户端选模型、向用户展示提示、等用户放行）
  │  ◀── result {role, content, model, stopReason} ──────────────────────
```

## 十、传输层

规范定义了两种标准传输。协议**与传输无关**——可以按需实现自定义传输，但**必须保持 JSON-RPC 报文格式和生命周期要求不变**。

### stdio

- 客户端**以子进程方式启动** MCP Server
- Server 从 `stdin` 读报文，往 `stdout` 写报文
- **报文之间用换行分隔，消息内部不能含内嵌换行**
- Server **可以**往 `stderr` 写 UTF-8 日志，客户端可以捕获、转发或忽略
- **Server 不得向 `stdout` 写任何非 MCP 报文**；客户端**不得**向 Server 的 `stdin` 写任何非 MCP 报文

那条 stdout 约束是最常见的实现事故来源：Server 代码里留一个 `print()`，或某个依赖往 stdout 打日志，报文流就被污染，客户端解析直接失败。**日志一律走 stderr。**

客户端**应该**尽可能支持 stdio——它是本地集成成本最低的一种。

### Streamable HTTP

Server 作为独立进程运行，可服务多个客户端连接。**必须提供单一 HTTP endpoint 路径同时支持 POST 和 GET**（例如 `https://example.com/mcp`）。

**POST 是发消息的唯一方式**，每次一个 JSON-RPC 报文：

1. **必须**带 `Accept` 头，同时列出 `application/json` 和 `text/event-stream`
2. Body 是单个 JSON-RPC request / notification / response
3. Body 是 **response 或 notification** 时：服务器接受则返回 `202 Accepted` 且无 body；不接受则返回 HTTP 错误码（如 400），body 里可以放一个没有 `id` 的 JSON-RPC 错误响应
4. Body 是 **request** 时：服务器**必须**二选一返回——`Content-Type: text/event-stream`（开 SSE 流）或 `Content-Type: application/json`（返回单个 JSON 对象）。**客户端必须两种都支持**
5. 走 SSE 流时：流里**应该**最终包含该请求的响应；响应之前**可以**先发其他请求和通知；响应发出后**应该**关闭流
6. **断连不等于取消**——客户端要取消得显式发 `CancelledNotification`

**GET 用于让服务器主动推消息**：客户端**必须**带 `Accept: text/event-stream`；服务器**必须**返回 SSE 内容类型，或返回 `405 Method Not Allowed` 表示这个 endpoint 不提供流。GET 开的流上，服务器**不得**发 JSON-RPC 响应（除非在续传）。

**多连接**：客户端**可以**同时连多个 SSE 流；服务器**必须**只把每条消息发在其中一条流上，**不得**广播。

**会话管理**：

| 环节 | 规则 |
| --- | --- |
| 建立 | 服务器**可以**在携带 `InitializeResult` 的响应里通过 `Mcp-Session-Id` 头分配会话 ID；ID **应该**全局唯一且密码学安全，且**只能**含可见 ASCII（0x21–0x7E）|
| 后续 | 客户端**必须**在之后所有请求上带 `Mcp-Session-Id` 头；要求会话的服务器对缺失该头的非初始化请求**应该**回 `400` |
| 终止 | 服务器**可以**随时终止会话，之后对该 ID 的请求**必须**回 `404` |
| 恢复 | 客户端收到 `404` **必须**不带会话 ID 重新发 `InitializeRequest` 开新会话 |
| 主动结束 | 客户端不再需要会话时**应该**带 `Mcp-Session-Id` 发 HTTP DELETE；服务器**可以**回 `405` |

**断线续传**：服务器**可以**给 SSE 事件加 `id`（会话内全局唯一）；客户端重连时带 `Last-Event-ID` 头，服务器**可以**重放该 ID 之后、**当初那条流上**的消息。**不得**重放属于其他流的消息——事件 ID 是**每条流各自的游标**。

**安全要求（不能省）**：

1. 服务器**必须**校验所有入站连接的 `Origin` 头，防 DNS rebinding
2. 本地运行时**应该**只绑 `127.0.0.1`
3. **应该**为所有连接实现认证

少这几条，攻击者可以从任意网页通过 DNS rebinding 访问到本地 MCP Server。

### 与旧传输的关系

Streamable HTTP 替代了 2024-11-05 版的 HTTP+SSE 传输。兼容探测流程：

```
POST InitializeRequest 到 server URL（带上面的 Accept 头）
  ├─ 成功        → 判断为新版 Streamable HTTP
  └─ HTTP 4xx   → GET 该 URL，期待首个事件是 endpoint 事件
                  收到即为旧版 HTTP+SSE，后续全程用它
```

服务端要兼容老客户端，则需同时保留新旧两套 endpoint。

## 十一、MCP 网关

MCP 规范刻意**没有**规定认证、授权、日志、路由这些生产部署必需的东西——它只管传输机制，不管应用层策略（类似 HTTP 不规定业务逻辑）。

规模一上来就出问题：**M 个客户端 × N 个服务器 = O(M×N) 的配置与治理开销**。每台开发者机器上放着各自的凭据、没有统一审计、也没法按人控制能用哪些 server。

**MCP 网关就是在客户端与服务器之间加的那一层中心代理**，把这个乘积压下来，成为认证、策略、可观测性的唯一执行点。它和 API 网关在微服务里的位置是同构的。

### 它具体做什么

| 能力 | 内容 |
| --- | --- |
| **聚合与路由** | 多个 MCP Server 合并成单一 endpoint；工具按后端加命名空间前缀（如 `github__issue_read`）以正确路由 |
| **工具过滤** | 按 include / includeRegex 之类规则决定哪些工具暴露给客户端——**也是控制工具数量、避免把模型上下文塞满的手段** |
| **认证授权** | 客户端到网关走 OAuth 2.0 / SSO；网关持有各上游凭据（API key、header 注入），**终端用户永远看不到上游密钥**。支持按 JWT claims、scope 做工具级细粒度授权 |
| **策略** | 按用户/角色允许或拒绝工具；对标记为破坏性的工具要求人工批准；限流防失控 agent 循环 |
| **可观测** | 记录每次 `tools/list` 和 `tools/call` 的身份、参数、延迟、结果大小，导出 trace 与指标（OpenTelemetry、Prometheus）|
| **协议翻译** | 让 ChatGPT Custom Actions 这类非 MCP 客户端也能接入 |
| **供应链防护** | 用 hash 固定工具描述，描述一变就触发人工审核 |

### 关键区分：MCP 网关 ≠ LLM 网关

| | LLM 网关 | MCP 网关 |
| --- | --- | --- |
| 代理对象 | 模型供应商 | 工具（MCP Server）|
| 管的是 | 模型调用的路由与计量 | 从 agent 到工具调用的治理 |

两者解决相邻的问题，成熟方案通常同时部署。

### 代表实现

Docker MCP Gateway（把 Server 跑在隔离容器里，管生命周期与凭据注入）、IBM MCP Context Forge、agentgateway（Linux Foundation 项目，同时代理 A2A）、Envoy AI Gateway，以及 Kong / Apache APISIX 这类 API 网关的 MCP 支持。

### 什么时候不该上

**一个开发者加两个本地 stdio server 的场景不要上**——网关的延迟和运维开销超过收益。判据大致是这几条满足两条以上：超过一个团队或 agent、凭据不能放在开发者机器上、策略要按角色区分、需要工具调用的审计日志。

### 代价与坑

- **成为单点故障与高价值攻击目标**
- **每次工具调用都加一层延迟**
- **聚合会重新引入跨服务器的信任问题**：把低信任服务器的工具描述和高信任的合并进同一张工具表，前者的描述就能遮蔽后者的。正确做法是按信任级拆成独立的虚拟服务器
- **把审计日志变成密钥库**：记录参数和结果但不脱敏，日志里就存满了秘密
- **原样透传上游工具描述** → 工具描述投毒会直接流到客户端

### 要防的三类攻击

威胁面比想象的宽，已记录的类别包括：**不可信的工具定义（tool poisoning）**、**伪造或冒充的 MCP Server**、**被攻破的第三方连接器带来的供应链攻击**。

规范只给了原则性要求，分两侧：

- **Server 必须**：校验所有工具输入、实现访问控制、对工具调用限流、清洗工具输出
- **Client 应该**：对敏感操作请求用户确认、调用前把工具输入展示给用户、把结果传给 LLM 前先校验、给工具调用设超时、记录工具使用日志备审
- **Resources 侧**：服务器**必须**校验所有资源 URI、**必须**正确编码二进制数据、**应该**对敏感资源做访问控制、**应该**在操作前检查权限
- **Prompts 侧**：**必须**仔细校验所有提示的输入输出，防注入与越权访问

工具注解（`annotations`）**必须被视为不可信**，除非来自受信任的服务器——这是规范原文的要求。

## 版本说明

上面所有报文与字段按 **2025-06-18 版规范**整理，来源：

- Lifecycle：https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle
- Architecture：https://modelcontextprotocol.io/specification/2025-06-18/architecture
- Tools：https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- Resources：https://modelcontextprotocol.io/specification/2025-06-18/server/resources
- Prompts：https://modelcontextprotocol.io/specification/2025-06-18/server/prompts
- Sampling：https://modelcontextprotocol.io/specification/2025-06-18/client/sampling
- Transports：https://modelcontextprotocol.io/specification/2025-06-18/basic/transports
- Cancellation：https://modelcontextprotocol.io/specification/2025-06-18/basic/utilities/cancellation

规范站提示当前已有更新的 **2026-07-28** 版，本文未逐条比对两版差异。报文骨架（JSON-RPC 2.0、`initialize` 握手、三个原语的 list/get/call）在两版之间是延续的，但能力项可能有增删。

## 相关

- [[09-工具系统与 Function Calling]] —— MCP 建立在它之上
- [[14-A2A 与 ANP]] —— 另一条轴：智能体之间的协议

## 参考

- MCP 规范（见上「版本说明」的逐页链接）
- MCP Gateway（Docker）：https://docs.docker.com/ai/mcp-catalog-and-toolkit/mcp-gateway/
- MCP Gateway 术语与代价分析：https://ossaihub.com/glossary/mcp-gateway
- 企业侧网关需求与威胁面：https://credal.ai/blog/why-organizations-need-an-mcp-gateway
- 来源：《Hello-Agents》第十章 §10.1–§10.2
