---
tags:
  - AI/llm
---

# 文本分词与子词算法：BPE、WordPiece 与 Unigram

计算机只理解数字，喂给模型之前必须把文本转成数字序列，这个过程叫**分词（Tokenization）**。**分词器（Tokenizer）** 定义一套规则，把原始文本切成最小单元，称为**词元（Token）**。

## 先把三层概念分开

「XX 模型用 BPE 还是 SentencePiece」这类问题之所以容易吵，是因为三个不同层的东西被混在一句话里：

| 层 | 回答什么问题 | 取值 |
| --- | --- | --- |
| **算法** | 怎么建词表 | BPE / WordPiece / Unigram |
| **实现（工具 / 框架）** | 谁来跑这个算法 | `tiktoken`（OpenAI）/ `SentencePiece`（Google）/ HF `tokenizers` |
| **粒度** | 在什么单位上切 | character-level / **byte-level** |

**`SentencePiece` 不是算法，是工具**——它同时实现了 BPE 和 Unigram 两种（`--model_type=bpe` / `unigram`）。所以「Llama 用 SentencePiece」和「Llama 用 BPE」**两句话都对**：一个说的是框架，一个说的是算法。这是这一块最常见的一处混淆。

`SentencePiece` 还有一个关键设计：它把输入当成**原始 Unicode 字符流**，连空格也当普通符号（用 `▁`，U+2581 表示）。标准 BPE / WordPiece 会先把文本按空格切开、假定语言是用空格分词的——那样在中文、日文、泰文上就不好使。把空格编码进 token 里还带来一个附加好处：**分词完全可逆**。

## 现状：谁在用哪个

| 模型 | 算法 | 粒度 | 实现 |
| --- | --- | --- | --- |
| GPT-2 / 3 / 4 / GPT-5 系列 | BPE | **byte-level** | `tiktoken` |
| Llama 1 / 2 / 3 | BPE | byte-level | `SentencePiece` |
| Qwen 2 及之后 | BPE | byte-level | `tiktoken` 风格 |
| Mistral / Gemma | BPE | byte-level | `SentencePiece` |
| BERT / DistilBERT / ELECTRA | **WordPiece** | character-level（`##` 前缀标非词首）| HF `tokenizers` |
| T5 / mT5 / mBART / ALBERT / XLNet | **Unigram** | — | `SentencePiece` |

**所以「BPE 是不是主流」的答案是：是，而且是压倒性的。** GPT 系列、Llama、Qwen、Mistral、Gemma 都在 BPE 这一支上；WordPiece 基本只剩 BERT 家族在用，Unigram 主要在 T5 系。

而且在 2026 年**说「BPE」基本就等于说「byte-level BPE」**——下面会讲为什么。

跨算法速查：

| 算法 | 建表方向 | 核心判据 | 词表规模 |
| --- | --- | --- | --- |
| BPE | 自底向上合并 | **频率** | 几万 – 200K |
| WordPiece | 自底向上合并 | **最大化语料似然** | ~几万 |
| Unigram | **自顶向下裁剪** | 对语料似然的**贡献** | 几万 – 200K+ |

**Unigram 与 BPE 的方向是反的**：它从一个很大的候选词表出发，迭代**剪掉**那些移除后对语料总似然影响最小的 token，直到降到目标规模。由此带来一个 BPE 没有的性质——**同一字符串可以有多种合法切分**（BPE 是确定性的）。这个概率性可以被利用来做数据增强（subword regularization，训练时采样不同切分提升鲁棒性）。

WordPiece 则只是在 BPE 上换了合并准则：不看原始频率，看**哪个合并能让语料概率提升最多**。实际效果与 BPE 差别通常很小。

## 为什么不能按词或按字符切

| 策略 | 做法 | 问题 |
| --- | --- | --- |
| 按词（Word-based） | 用空格或标点切分成单词 | **词表爆炸**；**未登录词（OOV）**——词表外的词（如 `DatawhaleAgent`）无法处理；**语义关联缺失**——`look` / `looks` / `looking` 被当成三个无关词元，低频词语义学不好 |
| 按字符（Character-based） | 切成单个字符 | 词表很小、无 OOV；但单字符大多不具备独立语义，模型要额外花精力学「怎么把字符组成词」，学习效率低 |

现代大模型普遍用**子词分词（Subword Tokenization）** 折中：常见词（`agent`）保留为完整词元，不常见的词（`Tokenization`）拆成有意义的子词片段（`Token` + `ization`）。既控制词表大小，又让模型能通过组合子词理解和生成新词。

**OOV 那一条是分词的原始动机**：词表 5 万，用户输入 `untokenizable` 返回 `[UNK]`，模型拿不到这个词的任何信号。子词分词让罕见词**分解成已知片段**（`un` + `token` + `izable`），既保住信息又不必扩词表。

## BPE 算法

**字节对编码（Byte-Pair Encoding, BPE）** 是最主流的子词算法之一（`Gage, A new algorithm for data compression, C Users Journal 1994`），GPT 系列采用。核心是一个「贪心」的合并过程：

```
1. 初始化   → 词表 = 语料库中出现过的所有基本字符
2. 迭代合并 → 统计所有相邻词元对的频率，把频率最高的一对合并成新词元，加入词表
3. 重复     → 重复第 2 步，直到词表大小达到预设阈值
```

**案例**：迷你语料集 `{"hug": 1, "pug": 1, "pun": 1, "bun": 1}`，目标词表大小 10。合并顺序是 `u+g → ug`、`ug+</w> → ug</w>`、`u+n → un`、`un+</w> → un</w>`。

训练完成后，对没见过的词 `bug` 的分词过程：查 `bug` 不在词表 → 查 `bu` 不在词表 → 查 `b` 和 `ug` 都在 → 切成 `['b', 'ug']`。

```python
import re, collections

def get_stats(vocab):
    """统计词元对频率"""
    pairs = collections.defaultdict(int)
    for word, freq in vocab.items():
        symbols = word.split()
        for i in range(len(symbols) - 1):
            pairs[symbols[i], symbols[i + 1]] += freq
    return pairs

def merge_vocab(pair, v_in):
    """合并词元对"""
    v_out = {}
    bigram = re.escape(' '.join(pair))
    p = re.compile(r'(?<!\S)' + bigram + r'(?!\S)')
    for word in v_in:
        w_out = p.sub(''.join(pair), word)
        v_out[w_out] = v_in[word]
    return v_out

# 每个词末尾加 </w> 表示结束，并切分好字符
vocab = {'h u g </w>': 1, 'p u g </w>': 1, 'p u n </w>': 1, 'b u n </w>': 1}

for i in range(4):
    pairs = get_stats(vocab)
    if not pairs:
        break
    best = max(pairs, key=pairs.get)
    vocab = merge_vocab(best, vocab)
```

注意 `</w>` 的作用：标记词边界。没有它，`ug` 可以在词中和词尾无差别合并，分词结果就不可逆了。

### byte-level BPE：现在说的「BPE」默认指它

原始的 BPE 从**字符**起步，字符集依赖语言（英文和中文的词表起点完全不同）。**byte-level BPE 从 256 个字节值起步**，算法一样，只是基本单位换成了字节。

这一步换来一个关键性质：**任何 UTF-8 文本都能被编码，永远不会有 `[UNK]`**。任何语言、emoji、罕见符号，最终都能拆成字节——所以词表里不需要「未知」这个类别。这就是 2026 年几乎所有生成式 LLM 都用它的原因。

代价是**非拉丁文字可能被切得更碎**：一个字符在 UTF-8 下占多个字节，每个字节都要先成为 token 再逐步合并，所以同样长度下非英语内容往往消耗更多 token。

**GPT-2 的词表构成可以精确拆开**：

```
50,257 = 256（字节）+ 50,000（合并次数）+ 1（特殊 token）
```

`tiktoken` 就是 OpenAI 的 byte-level BPE 实现（Rust 核心 + Python 绑定）。它的词表在代际之间是变的：

| 编码名 | 用在 |
| --- | --- |
| `gpt2` | GPT-2 |
| `p50k_base` | GPT-3 / Codex |
| `cl100k_base` | GPT-3.5 / GPT-4 |
| `o200k_base` | GPT-4o 及更新 |

**所以不要记死映射，按模型名取**（`tiktoken.encoding_for_model(...)`）。词表从 100K 扩到 200K 主要是为了改善多语言与代码覆盖。

### 两个后续优化

| 算法 | 提出方 | 与 BPE 的差别 |
| --- | --- | --- |
| **WordPiece** | Google，BERT 采用（`Schuster & Nakajima, 2012`） | 合并标准是「能最大化提升语料库的语言模型概率」，而不是单纯的「最高频率」——优先合并那些让整个语料库通顺度提升最大的词元对 |
| **SentencePiece** | Google 开源工具（`Kudo & Richardson, 2018`），Llama 系列采用 | 把空格也视作普通字符（通常用下划线 `_` 表示）。分词与解码完全可逆，且不依赖特定语言——不需要知道中文不用空格分词 |

## 对开发者实际影响的三件事

理解算法细节不是目的，但这三件事直接影响智能体的性能、成本和稳定性：

**一、上下文窗口限制。** 窗口（8K、128K）是按 **Token 数量**算的，不是字符数或单词数。同样一段话，在不同语言（中英文）或不同分词器下，Token 数量可能相差巨大。精确管理输入长度是构建长时记忆智能体的基础。

**二、API 成本。** 多数模型 API 按 Token 计费，知道自己文本会被怎么分词是预估成本的前提。

**三、模型表现的异常。** 有些奇怪表现的根源就在分词：

- 模型可能很擅长算 `2 + 2`，但对 `2+2`（无空格）就出错——后者可能被切成一个不常见的独立词元
- 同一个词因首字母大小写不同，可能被切成完全不同的 Token 序列
- 设计提示词和解析模型输出时把这些「陷阱」考虑进去，能提升智能体鲁棒性

## 编解码的工程接口

用 Hugging Face `transformers` 时，文本与 token id 的互转走这一套：

```python
# 文本 → id（含对话模板）
text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
model_inputs = tokenizer([text], return_tensors="pt").to(device)

# 生成
generated_ids = model.generate(model_inputs.input_ids, max_new_tokens=512)

# 只取新生成的部分再解码，否则会把输入原样吐回来
generated_ids = [out[len(inp):] for inp, out in zip(model_inputs.input_ids, generated_ids)]
response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
```

`apply_chat_template` 负责把 `{"role": ..., "content": ...}` 的消息列表按模型自己的对话格式拼成字符串——不同模型的模板不一样，这一步不能手写。解码时**必须先截掉输入部分**，否则输出里会包含原始 prompt。`skip_special_tokens=True` 用来过滤 `<|im_start|>` 这类控制符。

## 生产里的五个坑

分词在训练时就固定了——**换 tokenizer 是类别级的改动，模型必须重训**。所以这些问题只能在选型与集成阶段处理，上线后改不了。

| 坑 | 表现 | 处理 |
| --- | --- | --- |
| **训练/服务的 tokenizer 漂移** | 同一家族的 tokenizer 不同版本会微妙不同 | **必须把 tokenizer 版本与模型一起锁死** |
| **特殊 token 不一致** | 不同 chat 模型对 system / user / assistant 轮次期望不同的特殊 token 格式，发错就质量退化 | **用模型自己的 chat template，不要自己拼字符串** |
| **多语言成本不对称** | 英文约 4 字符/token，日文可到 1 字符/token。1000 字符的日文文档可能比同长度英文贵 **10 倍** | 按语言分别估预算；必要时按字符或句数而非固定 token 数切块 |
| **数字切分方式不同** | 逐位切数字（对算术友好）vs 多位数当一个 token（算术差）| 需要算术能力的场景要确认模型是逐位切的（LLaMA 与现代模型多为逐位）|
| **Unicode 规范化差异** | 「同一个字符串」的合成式与分解式会产生不同的 token 序列 | 统一走 NFC 规范化 |

**还有一个容易低估的量级问题**：同一个字符串在不同 tokenizer 下的 token 数能差 **10–30%**（`cl100k` 对比 LLaMA tokenizer）。多语言密集或代码密集的内容差距更大。所以「我的 prompt 大概多少 token」这种估算，**必须用目标模型自己的 tokenizer 去数**。

> [!warning] 有些厂商（如 Anthropic、Google）不公开 tokenizer 细节，第三方实现的计数只是估算。对精度敏感的计费或截断逻辑，应该用厂商 API 自己的 token 计数端点。

## 相关

- [[03-Decoder-Only 与自回归]] —— 自回归生成的单位是 token
- [[04-采样参数]] —— 采样的对象是分词器切出来的 token 分布
- [[09-位置编码]] —— token 序列进入编码器后，位置信息怎么注入
- [[11-RAG 检索增强]] —— 嵌入链路的第 ① 步就是过 tokenizer，且**嵌入模型的 tokenizer 与生成模型的不是同一套**

## 参考

- 《Hello-Agents》第三章 §3.2.2、§3.2.3
- Gage, P. A new algorithm for data compression. C Users Journal, 1994.
- Schuster, M., & Nakajima, K. Japanese and korean voice search. ICASSP, 2012.
- Kudo, T., & Richardson, J. SentencePiece. arXiv:1808.06226, 2018.
