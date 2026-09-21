---
tags:
  - AI/agent/框架
---

# LLM 接入层与多提供商

框架的 LLM 客户端不能绑死在某一家供应商上。这一层要解决三个问题：**多提供商切换**、**本地模型接入**、**配置自动推断**。

## 依赖的是「OpenAI 兼容」这个事实标准

这一层的设计前提是：**`chat.completions` 这个接口形态已经成为事实标准**，几乎所有主流 LLM 提供商都在兼容它。

但「兼容」不是同一个东西，实际分三个层次，混起来会造成很难查的 bug：

| 层次 | 内容 | 可靠性 |
| --- | --- | --- |
| **核心参数** | `model` / `messages` / `temperature` / `max_tokens` / `stream` | 各家一致，可以放心用 |
| **扩展功能** | `tools` / `tool_choice`、`response_format`、流式下的 tool call 分片、`logprobs` | **各家实现有差异**，接新供应商时是第一个要验的地方 |
| **私有扩展** | 各家自己的参数（推理强度、思考模式之类的开关）| 不通用，跨供应商时必然要处理 |

这也是 [[09-工具系统与 Function Calling]] 里 `model_info` 必须显式声明 `function_calling` / `json_output` 的原因——**框架不能假设「兼容 OpenAI」就等于「支持这些扩展」**。

## 多提供商：用继承扩展，不改库源码

直接改已安装库的源码是不该做的——它会让后续升级变得困难。正确做法是继承现有客户端，只拦截新增的 provider：

```python
class MyLLM(HelloAgentsLLM):
    def __init__(self, model=None, api_key=None, base_url=None,
                 provider="auto", **kwargs):
        if provider == "modelscope":
            self.provider = "modelscope"
            self.api_key = api_key or os.getenv("MODELSCOPE_API_KEY")
            self.base_url = base_url or "https://api-inference.modelscope.cn/v1/"
            if not self.api_key:
                raise ValueError("ModelScope API key not found.")
            self.model = model or os.getenv("LLM_MODEL_ID") or "Qwen/Qwen2.5-VL-72B-Instruct"
            self.timeout = kwargs.get('timeout', 60)
            self._client = OpenAI(api_key=self.api_key,
                                  base_url=self.base_url, timeout=self.timeout)
        else:
            super().__init__(model=model, api_key=api_key,
                             base_url=base_url, provider=provider, **kwargs)
```

关键在 `else` 分支：**只拦截自己想处理的 provider，其余全部交还父类**。这样父类的所有既有能力和未来新增能力都保留，`think` 之类的方法也不用重写。

**注意参数优先级**：每个字段都是 `显式传参 or 环境变量 or 默认值` 的三级链，且**缺失凭证要显式抛错**（`raise ValueError`），而不是带着空 key 往下走——后者的失败会推迟到第一次请求才出现，错误信息也指向不了根因。

## 静态适配表

按原书实现整理的各 provider 配置：

| provider | 环境变量 | 默认 `base_url` |
| --- | --- | --- |
| `openai` | `OPENAI_API_KEY` → `LLM_API_KEY` | `https://api.openai.com/v1` |
| `modelscope` | `MODELSCOPE_API_KEY` → `LLM_API_KEY` | `https://api-inference.modelscope.cn/v1/` |
| `zhipu` | `ZHIPU_API_KEY` → `LLM_API_KEY` | 域名特征 `open.bigmodel.cn` |
| `vllm` | 任意非空串 | `http://localhost:8000/v1` |
| `ollama` | 任意非空串 | `http://localhost:11434/v1` |
| `local` | 任意非空串 | 其他本地端口 |

注意每一行都是**至少两个来源的备选链**：专用变量优先，通用 `LLM_API_KEY` 兜底。这样既能同时配多家、也能只配通用的那一个。

## 流式响应：三个必须处理的细节

`think()` 默认开 `stream=True`。流式返回的每个 chunk 是增量，而不是完整的响应：

```python
def think(self, messages, temperature=0):
    response = self.client.chat.completions.create(
        model=self.model, messages=messages,
        temperature=temperature, stream=True)

    collected_content = []
    for chunk in response:
        if not chunk.choices:          # ① 有些 chunk 的 choices 是空数组
            continue
        content = chunk.choices[0].delta.content or ""   # ② content 可能是 None
        print(content, end="", flush=True)
        collected_content.append(content)

    return "".join(collected_content)  # ③ 用列表收集再 join
```

三个细节都是踩过才知道的：

1. **`chunk.choices` 可能是空数组**。除了正常的增量块，流里还会夹带用量统计之类的**非内容块**，直接取 `chunk.choices[0]` 会 `IndexError`。
2. **`delta.content` 可能是 `None`**。工具调用之类的块只有 `delta.tool_calls`，没有 content，必须用 `or ""` 兜住。
3. **用 list 收集再 `join`，不要字符串累加**。CPython 里字符串是 immutable，循环内 `+=` 在长响应上会退化成 O(n²) 的拷贝。

这三点在非流式调用里都不存在——这也是为什么「本地能跑、接到长响应就出问题」这类故障常见于流式路径。

## 本地模型：VLLM 与 Ollama

第三章用 Hugging Face Transformers 直接跑模型适合入门验证，但底层实现在高并发下性能有限，不是生产首选。生产级方案是 **VLLM** 和 **Ollama**——它们靠连续批处理、PagedAttention 等技术提升吞吐，并把模型封装成**兼容 OpenAI 标准的 API 服务**。

| | VLLM | Ollama |
| --- | --- | --- |
| 定位 | 高性能推理库 | 模型管理 + 部署封装 |
| 启动 | `python -m vllm.entrypoints.openai.api_server --model <id> --host 0.0.0.0 --port 8000` | `ollama run llama3`（首次自动下载） |
| 默认地址 | `http://localhost:8000/v1` | `http://localhost:11434/v1` |
| 适用 | 需要压榨吞吐量的服务端 | 快速上手 |

```python
llm_client = HelloAgentsLLM(
    provider="vllm",
    model="Qwen/Qwen1.5-0.5B-Chat",   # 必须与服务启动时指定的模型一致
    base_url="http://localhost:8000/v1",
    api_key="vllm",                    # 本地服务不需要真 Key，填任意非空串
)
```

**注意那个 `model` 必须与服务启动时指定的模型一致**——本地服务的模型不是由请求参数决定的，请求里写错名字会被拒或静默路由到已加载的那个。

**统一接口的价值在这里兑现**：核心代码一行不改，就能在云端 API 和本地模型之间切换。成本控制、数据隐私、离线运行三个需求都由这一层吸收掉了。

## 自动检测：三级优先级

遵循「约定优于配置」，两个方法协同工作：`_auto_detect_provider` 推断服务商，`_resolve_credentials` 按推断结果补齐参数。

推断按固定优先级，从最可靠到最弱：

| 优先级 | 依据 | 做法 |
| --- | --- | --- |
| 1（最高） | 特定服务商的环境变量 | 依次查 `MODELSCOPE_API_KEY` / `OPENAI_API_KEY` / `ZHIPU_API_KEY`，命中即定 |
| 2 | `LLM_BASE_URL` | **域名匹配**（`api-inference.modelscope.cn`、`open.bigmodel.cn`）与**端口匹配**（`:11434` → Ollama，`:8000` → VLLM） |
| 3（辅助） | API Key 格式 | 如 `ms-` 前缀 → ModelScope。**多家的密钥格式可能相似，所以只作辅助** |
| 4（兜底） | 都没有 | 返回 `"auto"`，走通用配置 |

```python
if actual_base_url:
    u = actual_base_url.lower()
    if "api-inference.modelscope.cn" in u: return "modelscope"
    if "open.bigmodel.cn" in u: return "zhipu"
    if "localhost" in u or "127.0.0.1" in u:
        if ":11434" in u: return "ollama"
        if ":8000" in u: return "vllm"
        return "local"          # 其他本地端口
```

**顺序为什么不能乱**：第 1 级是显式配置，最能表达意图；第 2 级要从 URL 里「猜」，而猜错的代价是请求打到错误的服务；第 3 级连猜测依据都是模糊的，所以只做辅助。

检测出 provider 后，`_resolve_credentials` 按 provider 查对应环境变量并给默认 `base_url`，每个分支结构一致：

```python
if self.provider == "openai":
    api_key = api_key or os.getenv("OPENAI_API_KEY") or os.getenv("LLM_API_KEY")
    base_url = base_url or os.getenv("LLM_BASE_URL") or "https://api.openai.com/v1"
    return api_key, base_url

elif self.provider == "modelscope":
    api_key = api_key or os.getenv("MODELSCOPE_API_KEY") or os.getenv("LLM_API_KEY")
    base_url = base_url or os.getenv("LLM_BASE_URL") or "https://api-inference.modelscope.cn/v1/"
    return api_key, base_url
```

于是用户想用本地 Ollama，只需要在 `.env` 里写：

```bash
LLM_BASE_URL="http://localhost:11434/v1"
LLM_MODEL_ID="llama3"
```

**不用配 `LLM_API_KEY`，也不用在代码里指定 `provider`**——`HelloAgentsLLM()` 直接实例化即可。

## 端口约定是可依赖的事实

`:11434` 和 `:8000` 之所以能当判据，是因为它们分别是 Ollama 和 VLLM 的默认端口。这条推断链的可靠性建立在**生态层面的约定**上：只要服务按默认端口启动、且对外暴露 OpenAI 兼容接口，就能被零配置识别。

**反过来说，这条推断也是脆的**：任何人把服务起在非默认端口，推断就退到 `"local"` 兜底；写 `127.0.0.1` 而不是 `localhost` 时靠 `in` 匹配也能过，但如果 URL 里带的是域名或反代路径，就完全失效。所以**显式指定 `provider` 永远是更可靠的做法**，自动检测是省事不是可靠。

## 相关

- [[07-Agent 框架的抽象设计]] —— 这一层产出的 LLM 实例注入给谁
- [[08-模型选型]] —— 选哪家、选云还是选本地的判据
- [[09-工具系统与 Function Calling]] —— `model_info` 里那几个能力标记的用途

## 参考

- 来源：《Hello-Agents》第四章 §4.1.3、第七章 §7.2
- VLLM 官方文档：https://docs.vllm.ai/en/latest/getting_started/installation.html
- Ollama：https://ollama.com
