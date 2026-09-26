---
tags:
  - AI/infra/推理引擎
  - AI/infra/缓存
---

# 前缀缓存：APC 与 RadixAttention

[[02-PagedAttention：KV Cache 的分页管理]] 用引用计数和写时复制让**同时并存**的请求共享物理块。但那只是空间维度的共享。真实业务里还有一类更普遍的重复：**同一段前缀在不同时间被不同请求反复用到**。

## 重复的前缀有多浪费

三类再常见不过的场景：

| 场景 | 重复的部分 |
| --- | --- |
| 共享 System Prompt | 每个请求前面都挂着几百 Token 的角色设定与规则说明，只有用户那句话不同 |
| Few-shot 提示 | 分类任务里每个请求都带着相同的示例，几千 Token 完全一致 |
| 多轮对话 | 第 5 轮的输入 = 前 4 轮全部历史 + 新问题，前 4 轮和第 4 轮请求高度重合 |

共同点是：**大量 Token 在不同请求间完全相同，且位置也相同（都在序列开头）**。而 Prefill 是 Compute Bound 的重活，把同一段 800 Token 的前缀对每个请求重新 Prefill 一遍，等于反复做同样的昂贵计算。

能复用的理由是 Attention 的因果性：

> **一个 Token 的 Key/Value 只取决于它自己和它前面的 Token。** 所以两个请求只要前缀完全一致，这段前缀每个 Token 算出来的 KV 就一模一样，完全可以只算一次、大家共用。

## 物理基础：块共享从空间延伸到时间

前缀能复用的前提，是显存管理支持「多个请求指向同一份 KV」。这正是 PagedAttention 打下的地基 —— 块表间接引用 + 引用计数 + CoW。

> **PagedAttention 的块共享是空间维度（并存请求共享），Prefix Cache 把它延伸到时间维度（先后请求复用）。** 前一个请求算完的前缀块先不急着扔，留在缓存里；后来的请求前缀匹配就直接复用这些块，跳过对应的 Prefill。

块共享的两个维度：

```
  空间维度（PagedAttention）—— 同时并存的请求之间共享

     请求 A 的块表 ──┐
                     ├──▶ 同一批物理块
     请求 B 的块表 ──┘

  时间维度（Prefix Cache）—— 先后到达的请求之间复用

     请求 1（t₀）算完前缀 ──▶ 块保留在缓存里，哈希登记
     请求 2（t₁）前缀相同 ──▶ 查表命中，直接复用，跳过这段 Prefill

  ⇒ Prefix Cache 把「块共享」从空间延伸到了时间
  ⇒ 能复用的理由是 Attention 的因果性：一个 Token 的 K/V 只取决于它自己
     和它前面的 Token —— 前缀一致，算出来的 KV 就一模一样
```

## vLLM 的 APC：块哈希

vLLM 叫 **Automatic Prefix Caching（APC）**。「Automatic」是它的特点 —— **不需要手动声明哪段是共享前缀，引擎自动检测**，靠的是对 KV 块做哈希。

每个**填满的块**的哈希由三部分组成：

$$\text{block\_hash} = \text{Hash}(\text{parent\_hash},\ \text{block\_tokens},\ \text{extra})$$

| 分量 | 作用 |
| --- | --- |
| **父块的哈希** | 它前面所有前缀块的哈希 —— **这是精髓所在** |
| 本块内的 Token 序列 | 降低哈希碰撞概率 |
| 额外标识 | LoRA ID、多模态输入哈希、`cache_salt` 等 |

把「父块哈希」纳入计算保证了**只有从序列开头到当前块的整条前缀都完全一致，哈希才会相同**。第 3 个块的哈希相同，意味着前 3 个块的全部 Token 都相同 —— 前缀匹配被压缩成一次哈希查表，$O(1)$ 判定。

哈希算法：vLLM 自 **v0.11 起默认 `sha256`**，可通过 `--prefix-caching-hash-algo` 切换为 `sha256_cbor`（跨语言可复现）、`xxhash`、`xxhash_cbor` 等。**非加密算法更快但碰撞风险略高。**

> [!warning] **只有填满的块才参与哈希**。源码里不满一个 `block_size` 的尾块直接跳过。这也解释了为什么前缀命中总是**块对齐**的 —— 不足一块的零头无法复用。

### 匹配与复用流程

```
新请求前缀分块 → 逐块计算哈希 → 缓存表查命中？
    命中   → 块表指向已有物理块，跳过这部分 Prefill
    未命中 → 正常 Prefill，算完把新块哈希登记进缓存
```

链式计算的核心逻辑：

```python
NONE_HASH = <进程启动时确定的常量>

def hash_request_tokens(token_ids, block_size, extra):
    block_hashes = []
    parent_hash = NONE_HASH                       # 第一个块没有父块
    for blk_tokens in chunk(token_ids, block_size):
        if len(blk_tokens) < block_size:
            break                                 # 只对"填满的块"算哈希
        # 关键：把父块哈希一起塞进去 → 天然编码了"从头到此的整条前缀"
        curr_hash = hash((parent_hash, tuple(blk_tokens), extra))
        block_hashes.append(curr_hash)
        parent_hash = curr_hash                   # 链式推进
    return block_hashes
```

命中前缀缓存**直接砍掉那部分 Prefill 的计算**，最直接的收益是**大幅降低 TTFT**，同时省下的算力能服务更多请求。共享前缀越长、命中率越高，收益越大。

块哈希的链式计算：只有「从头到此的整条前缀都一致」，哈希才相同

```
  请求 1：  [ System Prompt 800 token ][ 用户问题 A ]
             ├──── 块 0 ────┤├──── 块 1 ────┤ …
              hash₀ = H(NONE,  tokens₀, extra)
              hash₁ = H(hash₀, tokens₁, extra)     ← 把父块哈希一起塞进去
              hash₂ = H(hash₁, tokens₂, extra)

  请求 2 的前两个块与之完全一致
             ⇒ hash₀、hash₁ 相同 ⇒ 查表命中 ⇒ 跳过这两块的 Prefill
             从块 2 开始分叉 ⇒ hash₂ 不同 ⇒ 只对分叉之后的内容做计算

  ⇒ 把「父块哈希」纳入计算，使前缀匹配压缩成一次哈希查表，O(1) 判定
  ⇒ 只有填满的块才参与哈希 ⇒ 命中总是块对齐的（不足一块的零头无法复用）
```

### V1 的「零开销」设计

早期版本（V0）里前缀缓存因为命中率低时 CPU 记账开销不划算，**默认关闭**。V1 重写了这块：所有块在初始化时预分配成一个块池，用嵌入块内的双向链表指针做 $O(1)$ 的移动与淘汰，把记账开销压到极低。

官方口径是**在 0% 缓存命中率下，吞吐下降小于 1%**，因此 V1 里**前缀缓存默认开启**。

> 这个数字比「几乎免费」这种说法有用得多 —— 它说明的是**最坏情况下的代价上限**，而不是平均值。一个优化敢不敢默认开，取决于它的最坏情况有多坏。

## 淘汰：LRU + 引用计数

缓存空间有限，物理块用完要淘汰。vLLM 的策略结合两者：

- **正在被使用的块不能淘汰**：引用计数 > 0 的块受保护
- **空闲块按 LRU 淘汰**：请求结束后块引用计数归零、进入空闲队列，但**内容和哈希先保留着**。显存吃紧时才真正回收最久没被用到的空闲块

一个有意思的细节：vLLM 释放块时把请求的块**按逆序**放进空闲队列尾部。原因是一个请求的**最后一个块哈希了最多的 Token**（前缀最长），最不容易被别的请求命中，所以让它**优先被淘汰**；靠前的块（如共享 System Prompt 的开头）更可能被复用，就更晚淘汰。

> 这个设计体现了一个朴素直觉：**越靠近序列开头的前缀越通用、越值得留；越靠后的前缀越个性化，留着也难命中。**

### V1 的四张表

`BlockPool` 初始化时预建四套结构协同工作：

| 结构 | 作用 |
| --- | --- |
| Block Pool（块列表） | 所有物理块的池子，启动时一次性预分配，避免运行时创建 Python 对象的开销 |
| Free Block Queue（空闲双向链表） | 管理空闲块，支持 $O(1)$ 从中间摘除（前缀命中时「抢救」块） |
| Cache（哈希 → 块 ID 映射） | 前缀查表的核心 |
| Request（请求 → 块 ID 映射） | 记录每个请求当前持有哪些块 |

复用发生在调度器为新请求调用 `get_computed_blocks()` 时：Prompt 逐块哈希、查 Cache 表命中；命中的块被 `touch()`（引用计数 +1，并从空闲链表摘出以防淘汰），随后 `allocate_slots()` 只为未命中的部分申请新块。

> [!warning] **V1 的块表是只追加（append-only）的** —— 同哈希的重复满块可能短暂共存（因为块表不能改写），这些冗余会在请求释放时被消除。这是 V1 为调度简洁性做的务实取舍，读源码时看到重复块不必惊讶。

## SGLang 的 RadixAttention

vLLM 的哈希方案开销低，但有个隐含限制：**匹配是「块对齐」且「线性前缀」的**，按块边界比对一条从头开始的前缀。而多轮对话、树状采样等场景里，共享关系是**树状**的：一个共同前缀分叉出多个不同的后续。

SGLang 的 **RadixAttention** 用一棵 **Radix Tree（压缩前缀树）**管理 KV 复用：

- 每条边代表一段 Token 序列，从根到某节点的路径就是一段前缀，对应缓存的 KV
- 新请求沿树做**最长前缀匹配**，匹配到的路径部分直接复用 KV，只对分叉之后的新内容做计算
- 分叉天然对应「共同前缀 + 不同后续」，契合多轮对话（同一段历史派生多轮）与并行采样（同一 Prompt 采多个回答）

演进过程：2024-01 的 LMSYS 博客给出原始方案，同年发在 **NeurIPS 2024**（Lianmin Zheng 等）。公布的收益口径有两条，值不同：

| 来源 | 报告收益 |
| --- | --- |
| 2024-01 LMSYS 博客（A10G，vLLM v0.2.5 基线） | 在 agent control / MMLU / JSON decoding 工作负载上最高 **5×** 吞吐 |
| NeurIPS 2024 论文 | 最高 **6.4×** |

> [!warning] 博客那组数字的基线是 **vLLM v0.2.5**，而 vLLM 早已把前缀缓存做到默认开启且近乎零开销，**这个 5× 不能直接拿来对比今天的 vLLM**。方向性信号成立，倍数会随基线演进而失效。

线性前缀匹配与树状前缀匹配：

```
  vLLM APC：块哈希，一条从头开始的线性前缀

     根 ──▶ 块0 ──▶ 块1 ──▶ 块2        只能沿这一条比对

  SGLang RadixAttention：压缩前缀树，最长前缀匹配

                  ┌── 后续 A
     根 ── 共同前缀 ──┼── 后续 B
                  └── 后续 C

     ⇒ 分叉天然对应「共同前缀 + 不同后续」——正合多轮对话
       （同一段历史派生多轮）与并行采样（同一 Prompt 采多个回答）

  ⇒ 两者不对立：哈希方案开销低、通用场景已够用；树方案在高度树状
     复用的负载下才有额外优势。选型看负载形态
```

### 两者不是对立关系

| | vLLM Automatic Prefix Caching | SGLang RadixAttention |
| --- | --- | --- |
| 数据结构 | Hash 表（块哈希 → 块 ID） | Radix Tree（压缩前缀树） |
| 匹配方式 | 块对齐的前缀哈希匹配 | 树上最长前缀匹配 |
| 擅长场景 | 通用共享前缀（System Prompt、Few-shot） | 树状共享（多轮对话、树采样） |
| 淘汰策略 | LRU + 引用计数 | 基于树的 LRU |

> vLLM 的方案在绝大多数场景已经够用，且随 V1 做到近乎零开销；RadixAttention 是在**高度树状复用**的负载下展现额外优势。选型看负载形态，而不是简单认为「树一定比哈希好」。

2026-01 一份第三方 H100 SXM5 基准（Llama-3.3-70B-Instruct FP8、50 并发）给出的差距是 **SGLang 约 1920 tok/s vs vLLM 约 1850 tok/s（约 4%）** —— 在通用混合流量上两者很接近，共享前缀越重差距越大。

## 什么情况下吃不到收益

**收益大的场景**：

- 长 System Prompt / 长 Few-shot 且被大量请求共享
- 多轮对话（历史不断累积复用）
- 同一 Prompt 并行采样多个输出（`n > 1`）
- RAG 中固定的指令模板 + 变化的检索内容（**固定部分放前面**才能命中）

**收益有限甚至无收益**：

- 每个请求前缀都不同（无模板的自由问答）
- **共享部分放在 Prompt 中间或末尾** —— 前缀缓存只能复用**从头开始连续相同**的部分，共享内容一旦不在开头就无法命中

> 想吃到红利，Prompt 工程上要有意识地**把公共内容前置**：固定不变的指令、示例、上下文放最前面，变化的用户输入放最后。这是一条几乎零成本却常被忽视的优化。

**这条和 [[18-Context Engineering]] 里「把静态内容放在提示开头」是同一件事在两个层次上的收益**：那一层省的是按 token 计价的输入成本（Prompt Cache），这一层省的是 Prefill 的计算。**一条规则同时吃到两层。**

> [!warning] **前缀缓存不改变模型输出**（复用的是数学上完全等价的 KV），可以放心开。但多租户环境要留意隔离 —— vLLM 提供 `cache_salt` 等机制，避免不同租户/权限的请求意外命中彼此的缓存造成信息泄露。

## 相关

- [[02-PagedAttention：KV Cache 的分页管理]] —— 块共享、引用计数与 CoW 的来源
- [[03-推理调度：Continuous Batching 与 Chunked Prefill]] —— 前缀命中后剩余部分如何与 Decode 混跑
- [[18-Context Engineering]] —— 「静态内容放开头」在成本侧的同一条规则
- [[10-KV Cache 与推理优化]] —— 该监控的缓存命中率指标

## 参考

- https://docs.vllm.ai/en/latest/design/prefix_caching.html
- https://blog.vllm.ai/2025/01/27/v1-alpha-release.html
- https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_utils.py
- Zheng et al., *SGLang: Efficient Execution of Structured Language Model Programs*, NeurIPS 2024，https://arxiv.org/abs/2312.07104
- https://news.creeta.com/en/llm-inference-engine-benchmarks-2026-vllm-sglang-tensorrt
- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E5%9B%9B-%E6%8E%A8%E7%90%86%E4%BC%98%E5%8C%96/%E7%AC%AC2%E7%AB%A0-%E6%8E%A8%E7%90%86%E5%BC%95%E6%93%8E%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF/23-prefix-cache-%E4%B8%8E-radixattention
