---
tags:
  - AI/agent/框架
---

# Agent 框架的抽象设计

这一篇讲自建一套 Agent 框架要处理的三层抽象：**框架整体怎么分层、核心接口怎么定、已有范式怎么收敛到统一入口**。

## 先问：为什么自建

成熟框架的功能很全，但代价是**概念太多**——完成一个简单任务往往要先理解 Chain、Agent、Tool、Memory、Retriever 十几个概念，每个概念都有自己的抽象层，学习曲线陡得没法起步。

自建一套的理由通常是这几条之一：

- **可读性要求**：需要能完整读懂框架工作原理，而不是在依赖关系里找答案
- **可调试性**：出问题时能直接定位到框架代码
- **教学透明性**：要让人看清每一步是怎么搭起来的，以及不同范式的差异在哪

## 四个设计理念

| 理念 | 具体做法 |
| --- | --- |
| **轻量级与教学友好的平衡** | 极简依赖——除 OpenAI 官方 SDK 和几个基础库外不引入重型依赖。任何有编程基础的开发者都应能在合理时间内读完全部核心代码 |
| **基于标准 API，不重新发明抽象** | OpenAI 的 API 已成为事实标准（几乎所有主流 LLM 提供商都在兼容它）。直接在它之上构建：迁移到别的框架时底层调用逻辑一致，也不必学一套新的概念模型 |
| **渐进式学习路径** | 每一章的学习代码保存为一个可 `pip` 下载的历史版本，每一步升级都是自然的，不产生概念跳跃 |
| **万物皆为工具** | 除核心 `Agent` 类之外，**一切都抽象为 Tool** |

### 「万物皆为工具」这个简化值得单独说

其他框架里需要独立学习的 Memory（记忆）、RAG（检索增强）、RL（强化学习）、MCP（协议）等模块，在这里**全部被统一抽象成一种「工具」**。

这个选择的收益是**消除抽象层**：学习者不必掌握五六套并行的概念体系，只需要沿「智能体调用工具」这一条主线理解下去——而这条主线正是 [[03-ReAct]] 的原始形态。

代价也要认：**统一抽象意味着丢掉各模块的专属语义**。记忆有短期/长期、容量/过期策略，RAG 有召回与重排，这些在「工具」这个统一接口里只能靠参数和描述去表达，框架层面拿不到类型级的区分。教学框架可以这么换，生产框架通常不敢——这也是 [[06-智能体框架的编排模型]] 里那些框架会保留 `MemoryLayer` 之类独立概念的原因。

## 目录分层

```
hello-agents/
└── hello_agents/
    ├── core/                   # 核心框架层
    │   ├── agent.py            # Agent 基类
    │   ├── llm.py              # LLM 统一接口
    │   ├── message.py          # 消息系统
    │   ├── config.py           # 配置管理
    │   └── exceptions.py       # 异常体系
    ├── agents/                 # Agent 实现层
    │   ├── simple_agent.py
    │   ├── react_agent.py
    │   ├── reflection_agent.py
    │   └── plan_solve_agent.py
    └── tools/                  # 工具系统层
        ├── base.py             # 工具基类
        ├── registry.py         # 工具注册机制
        ├── chain.py            # 工具链管理系统
        ├── async_executor.py   # 异步工具执行器
        └── builtin/            # 内置工具集
```

**分层方式本身说明了一件事**：`core` 不含任何具体范式，`agents` 只放范式的实现，`tools` 只放工具的定义与调度。三层的依赖方向是单向的——`agents` 依赖 `core`，`core` 不认识 `agents`。

## 核心接口：三个类撑起整个框架

### Message：对内丰富，对外兼容

```python
MessageRole = Literal["user", "assistant", "system", "tool"]

class Message(BaseModel):
    content: str
    role: MessageRole
    timestamp: datetime = None
    metadata: Optional[Dict[str, Any]] = None

    def to_dict(self) -> Dict[str, Any]:
        """转换为 OpenAI API 格式"""
        return {"role": self.role, "content": self.content}
```

三个设计点：

**`typing.Literal` 把 `role` 锁死成四种取值**，直接对应 OpenAI API 规范。这不是形式主义——非法角色在构造对象时就被挡住，而不是等发请求时才报错。

**在 `content` / `role` 之外加了 `timestamp` 和 `metadata`**，为日志记录和后续扩展（多模态、工具调用元信息）预留位置。[[03-ReAct]] 那种手写实现里，历史就是裸字符串列表，加不了这些。

**`to_dict()` 是整个类的核心出口**，把内部对象转成 OpenAI 兼容的字典。设计原则一句话：**对内丰富，对外兼容**——内部想存多少存多少，发出去时只留对方认识的字段。

继承 `pydantic.BaseModel` 让这些约束在赋值时就生效，而不是靠调用方自觉。

### Config：零配置可用，环境变量可覆盖

```python
class Config(BaseModel):
    default_model: str = "gpt-3.5-turbo"
    default_provider: str = "openai"
    temperature: float = 0.7
    max_tokens: Optional[int] = None
    debug: bool = False
    log_level: str = "INFO"
    max_history_length: int = 100

    @classmethod
    def from_env(cls) -> "Config":
        return cls(
            debug=os.getenv("DEBUG", "false").lower() == "true",
            log_level=os.getenv("LOG_LEVEL", "INFO"),
            temperature=float(os.getenv("TEMPERATURE", "0.7")),
            max_tokens=int(os.getenv("MAX_TOKENS")) if os.getenv("MAX_TOKENS") else None,
        )
```

两个原则：**每项都有合理默认值**，保证零配置也能跑起来；**`from_env()` 做覆盖**，换部署环境不用改代码。

`max_history_length` 是隐藏的关键项——它是 [[02-Agent Loop]] 里 `max_steps` 的另一种形态，防止历史无界增长。

### Agent 基类：强制统一入口

```python
class Agent(ABC):
    def __init__(self, name: str, llm: HelloAgentsLLM,
                 system_prompt: Optional[str] = None,
                 config: Optional[Config] = None):
        self.name = name
        self.llm = llm
        self.system_prompt = system_prompt
        self.config = config or Config()
        self._history: list[Message] = []

    @abstractmethod
    def run(self, input_text: str, **kwargs) -> str:
        """运行Agent"""
        pass

    def add_message(self, message: Message): self._history.append(message)
    def clear_history(self): self._history.clear()
    def get_history(self) -> list[Message]: return self._history.copy()
```

用 `abc.ABC` + `@abstractmethod` 的组合，效果有两条：

- **不能直接实例化**，必须实现 `run`——所有子类有统一执行入口，调用方不需要知道内部是 ReAct 还是 Reflection
- **构造函数把依赖固定成四项**：名称、LLM 实例、系统提示词、配置。依赖注入而非内部 new，意味着换模型、换配置都不用碰子类

`get_history()` 返回的是 `self._history.copy()` 而不是原列表——**外部拿不到内部状态的引用，改不了框架内部**。这三个历史方法在基类实现、子类复用，是抽象基类少见的「有具体内容」的部分。

## 范式的框架化：四种实现收敛到同一个 run()

这一层要回答的问题是：[[03-ReAct]]、[[04-Plan-and-Solve]]、[[05-Reflection]] 以及最基础的单轮对话，能不能用同一套骨架承载？

答案是能，前提是**把差异归结为「循环体里做什么」，而不是「有没有循环」**：

| 实现 | `run()` 内部的差异 |
| --- | --- |
| `SimpleAgent` | 一次 LLM 调用，没有循环 |
| `ReActAgent` | 循环体是「思考 → 选工具 → 执行 → 记录观察」 |
| `ReflectionAgent` | 循环体是「执行 → 反思 → 优化」，终止条件从「拿到答案」变成「反思无新问题」 |
| `PlanAndSolveAgent` | 循环分两段：先一次调用生成计划，再按计划逐步执行 |
| `FunctionCallAgent` | 循环体与 ReAct 同构，差别只在工具调用由模型原生输出而非正则解析 |

四种实现的公共部分——消息历史管理、LLM 调用、配置读取——全在基类；差异部分的代码量远小于公共部分。**这就是抽象该起的作用：把变化的部分缩到最小。**

反过来也有一条判据：**如果某个新范式无法用「循环体不同」来表达，而需要改基类的接口，那说明抽象选错了层次**——这时该改的是基类，而不是打补丁。

## 抽象带来的东西

对比手写实现，这套抽象的收益可以逐项对应：

| 手写时的问题 | 抽象后 |
| --- | --- |
| 历史是裸字符串列表，塞不进元信息 | `Message` 带 `timestamp` / `metadata` |
| 每个 Agent 各自 new 一个 LLM 客户端 | 构造时注入，可整体替换 |
| 换模型要改多处代码 | 改 `Config` 或环境变量 |
| 各范式的入口方法名不统一 | 统一到 `run()` |

## 相关

- [[06-智能体框架的编排模型]] —— 别人怎么做的
- [[08-LLM 接入层与多提供商]] —— 基类依赖的那个 LLM 实例怎么来的
- [[09-工具系统与 Function Calling]] —— 另一组需要统一的接口

## 参考

- 《Hello-Agents》第七章 §7.1、§7.3、§7.4
