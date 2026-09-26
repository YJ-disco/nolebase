---
tags:
  - AI/infra/推理引擎
  - AI/infra/调度
---

# 推理调度：Continuous Batching 与 Chunked Prefill

[[01-推理性能指标与瓶颈定位]] 给出的结论是 Decode 卡在搬权重上。这句话里藏着一个机会：**搬一次权重本来只服务 1 个请求的 1 个 Token，太亏了**。让这一次搬运同时服务多个请求的 Token，算术强度就成倍上升。

这就是批处理的价值。但真正难的是**怎么组织这个批** —— 请求的生成长度天差地别。这一篇讲的就是调度器怎么处理这件事。

## Static Batching 的致命短板

最朴素的做法：攒够一批请求，一起送进模型，**等这一批全部生成完**再开下一批。

问题出在长度差异上。同一批里请求 A 可能 20 个 Token 就结束，请求 B 要生成 500 个。在 Static Batching 下：

- A 在第 20 步就完事了，但必须**待在批次里干等**到 B 跑完 500 步，整批才释放
- A 完成后的那 480 步里，它占的槽位纯粹在空转，GPU 为一个已经没有意义的位置做无用计算

论文与工程实测给出的口径：

| 观察 | 数值 |
| --- | --- |
| Static Batching 下的 GPU 利用率 | 常只有 **约 30%** |
| 32 条序列、平均长度 100、最长 2000 时的计算浪费 | **约 95%**（大量时间花在 padding 和已完成的请求上） |

请求长度差异越大，浪费越严重。而聊天负载的输出长度本来就能差一个数量级。

## Continuous Batching：把调度粒度降到一个迭代步

破局点是把调度粒度从「一整批」细化到「一个迭代步（iteration）」。

**Iteration-level Scheduling**：每一步（生成一个 Token）结束后立刻重新组批 ——

- 哪些请求生成了 EOS 或达到长度上限？**立即移出批次、释放显存**，不等别人
- 显存和批次容量腾出空间了？**立即从等待队列拉新请求进来**，加入下一步计算

批次构成于是变成动态流动的：请求随到随补、完成即退，GPU 几乎每一步都在为「当前真正还在生成」的请求干活。

```
步 t   : A B C D 在批中
         A 完成 → 退出，拉入 E
步 t+1 : B C D E 在批中
         C 完成 → 退出，拉入 F
步 t+2 : B D E F 在批中
```

> **Continuous Batching 把 GPU 利用率从约 30% 拉到 80% 以上。** 收益分两块：完成的请求立刻退出，不再空转；新请求立刻补位，不必等整批结束，排队延迟也大幅下降。

术语归属值得记清楚：**这个机制最早是 Orca 在 OSDI 2022 提出的**，这被称为 iteration-level scheduling；Orca 报告在同等延迟下相对 FasterTransformer 最高 **36.9×** 吞吐。「Continuous Batching」是这套细粒度调度哲学在工程界的通行叫法，TensorRT-LLM 文档里叫 in-flight batching，TGI 从 0.9（2023）起也实现了同一模型。

两种批处理在时间轴上的差别：

```
  Static Batching：等整批全部生成完才释放

    步:       1 … 20 …                                500
    请求 A    ████████ 完成 ⇒ 之后 480 步占着槽位空转 ─────┐
    请求 B    █████████████████████████████████████████████┘ 整批才释放
    请求 C    ████████████████ 完成 ⇒ 同样空转到 500
              └─ GPU 利用率常只有约 30%；长度差异大时浪费可达约 95%

  Continuous Batching：每一步重新组批，完成即退、新请求立即补位

    步:       1      2      3      4       5
    批       ABCD → BCDE → BDEF → DEFG → EFGH …
              │      │      │
              A 完成  C 完成  D 完成 ⇒ 立即移出，并从等待队列拉新请求进来
              └─ GPU 利用率从约 30% 拉到 80% 以上
```

## 一次调度循环里发生什么

vLLM 每一个 step 的循环：

```
收新请求 → 调度 → 前向一步 → 处理产出 → 回到开头
```

1. **收新请求**：新到达的请求进 Waiting 队列
2. **调度决策**：决定这一步谁上车。要同时兼顾两拨 —— 正在生成的 Running 队列（做 Decode），和等待队列里能塞进来的新请求（做 Prefill）。是否有空间取决于显存里还有多少空闲 KV Block（这正是 [[02-PagedAttention：KV Cache 的分页管理]] 提供的信息）
3. **前向计算一步**：对选中的批次跑一次模型前向，Decode 请求各产出 1 个新 Token，Prefill 请求产出首 Token
4. **处理产出**：新 Token 追加到各请求的 KV Cache；检查 EOS 或长度上限，完成的请求释放 KV Block 归还空闲池
5. **循环**

## Continuous Batching 必须和 PagedAttention 配套

这一层关系是本模块反复出现的：

- Continuous Batching 要做到「随退随补」，前提是**能随时、以细粒度分配和回收显存**。传统连续大块分配下，一个请求退出留下的空隙未必恰好塞得下新请求，动态补位无从谈起。
- PagedAttention 的**固定大小块 + 按需分配 + 即时回收**恰好提供了这种弹性。

| 技术 | 解决的问题 | 提供的能力 |
| --- | --- | --- |
| PagedAttention | 显存碎片、利用率低 | 细粒度、可即时回收的显存分配 |
| Continuous Batching | GPU 空转、排队延迟 | 迭代级调度、动态批次构成 |

> **分页管「显存怎么放」，调度管「请求怎么排」，二者相乘才有高吞吐。** 单有其一都不够。

## Prefill 干扰 Decode

迭代级调度带来一个新问题。批是**每一步重新组**的，如果某一步里调度器把一个 4000 Token 的长 Prompt 的 Prefill 和一批 Decode 请求塞进同一个 step：

- Prefill 是 Compute Bound，处理整段 Prompt 的矩阵运算量大
- 这一步的耗时会被它**整体拉长**，同批所有 Decode 请求的这个 Token 都得等它算完

用户端的表现就是：**出字很流畅，然后突然卡顿一下，再恢复流畅** —— TPOT 出现尖刺，P99 抖动。源材料给出的量级是 Decode 的 **P95 TPOT 可被拖慢 3–5 倍**（该数字出自教程第7章大纲，未回溯一手报告，标为待验证）。

> 问题的性质是**一步里塞了太多 Compute Bound 的活** —— 这与「要不要批处理」无关。

**Chunked Prefill** 的思路直接：把长 Prompt 切成若干固定大小的 Chunk，分散到多个 step 里逐块处理。一个 4000 Token 的 Prompt 按 512 一块切成 8 个 Chunk，每步只处理一块，剩下的算力留给同批 Decode 正常出字。

| | 不分块 | Chunked Prefill |
| --- | --- | --- |
| 长 Prefill 步的耗时 | 被拉长到整步 | 控制在可预期范围 |
| 同批 Decode 的 TPOT | 出现尖刺 | 平稳推进 |
| 该长请求的 TTFT | 一次 Prefill 完，较早 | **略微变大**（要跨多步） |

**这是一个刻意的权衡：牺牲少量单请求 TTFT，换整批 Decode 的 TPOT 稳定。** 机制出自 SARATHI（`arXiv:2308.16369`）。

长 Prefill 与 Decode 塞进同一步会怎样：

```
  不分块：

    步 t   [ 4000 Token 的 Prefill ][ 一批 Decode ]   ← 整步被它拉长
           ⇒ 同批 Decode 的这个 Token 都得等它算完
           ⇒ 用户看到「出字流畅 → 突然卡一下 → 又流畅」，P99 抖动

  Chunked Prefill（4000 Token 按 512 一块，切成 8 块）：

    步      │ 这一步的批次
    t       │ [Chunk 1][ Decode ]
    t+1     │ [Chunk 2][ Decode ]
    t+2     │ [Chunk 3][ Decode ]
      …     │ （每步只处理一块，剩下的算力留给 Decode 正常出字）
           ⇒ 每步耗时可控，Decode 的 TPOT 平稳
           ⇒ 代价：这个长请求的 TTFT 略微变大（要跨多步才 Prefill 完）

  底下是统一的度量衡 —— Token Budget：

    Σ Decode 请求（各 1 token） ＋ Σ Prefill Chunk（各 chunk_size token） ≤ token_budget
```

## Token Budget：调度的统一货币

Chunked Prefill 要落地，调度器需要一个统一度量衡：**Token Budget**。

洞察在于**Prefill 和 Decode 对 GPU 来说都是「处理若干 Token 的前向计算」**：

- 一个 Decode 请求这一步贡献 **1 个 Token**
- 一个 Prefill 请求（或它的一个 Chunk）这一步贡献 **N 个 Token**

于是每一步设一个总 Token 预算，先保证 Decode 请求（每个占 1 个额度），剩余预算再分给 Prefill Chunk，预算用完就发车：

$$\underbrace{\sum \text{Decode 请求}}_{\text{每个 1 token}} + \underbrace{\sum \text{Prefill Chunk}}_{\text{每个 chunk\_size token}} \leq \text{token\_budget}$$

> 用 Token 预算这把统一的尺子，「Chunk 该切多大」「一步能塞几个 Decode」「长 Prefill 和 Decode 怎么共处一步」都归结成同一个装箱问题。

## V1 的统一调度器

早期 vLLM（V0）的调度器把 Prefill 和 Decode 当**两类阶段**分别处理，代码复杂且在切换时容易产生空隙。

V1 引擎做了个简化：**取消 Prefill/Decode 的二分**。调度器眼里只有「这一步每个请求要处理多少个 Token」，决策被表达成一个字典：

```python
{
    "req_A": 1,      # 正在 Decode，处理 1 个新 token
    "req_B": 1,      # 正在 Decode，处理 1 个新 token
    "req_C": 512,    # 长 Prompt 的一个 Prefill chunk
    "req_D": 128,    # 短 Prompt，一步就能 Prefill 完
}
# 约束：所有 value 之和 ≤ token_budget
```

> `{request_id: num_tokens}` 这一个统一表示，同时涵盖了 Chunked Prefill、Prefix Cache 和 Speculative Decoding —— 它们本质都是「某个请求这一步要处理多少 Token」的不同取值：Prefill chunk 是较大的值，Decode 是 1，投机解码的验证是若干个候选 Token。

这也是 Chunked Prefill 在 V1 里能做成**默认行为**的原因：既然调度天然以 Token 为单位、不区分阶段，长 Prompt 自然被预算约束切成 Chunk，不需要任何特殊逻辑。

### 源码骨架：调度器的三个咬合件

`vllm/v1/core/sched/scheduler.py` 的 `Scheduler` 维护两组状态：`self.running`（列表，正在计算的请求）与 `self.waiting`（优先级队列，策略有 `FCFS` 和 `PRIORITY`，另有 `self.skipped_waiting` 装因异步依赖被暂时跳过的请求）。

两个主方法正好对应上面那个循环：`schedule()` 产出调度决策，`update_from_output()` 消费模型输出、追加 Token、检查停止条件、释放资源。

```python
def schedule(self):
    token_budget = self.max_num_batched_tokens
    scheduled = {}

    # 第一优先级：喂饱正在运行的请求
    for req in self.running:
        if token_budget <= 0: break
        # 这个请求还差多少 token 才算完（prefill 未完成部分 或 decode 的 1 个）
        need = req.num_tokens_with_spec - req.num_computed_tokens
        # 关键：被剩余预算截断 → 长 prefill 在这里被"切"出一个 chunk
        num_tokens = min(need, token_budget)
        blocks = self.kv_cache_manager.allocate_slots(req, num_tokens)
        if blocks is None:                          # 显存不够 → 触发抢占
            victim = self.running.pop()             # FCFS 踢最后一个（最晚来的）
            self._preempt_request(victim)           # 释放块，打回 waiting 队头
        else:
            scheduled[req.id] = num_tokens
            token_budget -= num_tokens

    # 第二优先级：预算有富余且未抢占，才拉新请求
    if not preempted and token_budget > 0:
        while self.waiting and token_budget > 0:
            req = self.waiting.pop_front()
            if not self._can_admit(req): break      # 检查 running 上限 / LoRA 等
            need = req.num_prompt_tokens
            num_tokens = min(need, token_budget)    # 长 prompt 首个 chunk 也在此截断
            scheduled[req.id] = num_tokens
            token_budget -= num_tokens
            self.running.append(req)

    return SchedulerOutput(scheduled)
```

三个要点：

1. **在途请求优先于新请求** —— 先保证已在跑的请求推进，剩余预算才接纳新请求。避免「新请求不断插队、老请求迟迟跑不完」的饥饿。
2. **`min(need, token_budget)` 就是切块的本质** —— 一个 4000 Token 的 Prompt，若这一步只剩 512 预算，就只 Prefill 512，剩下 3488 留到后续步。**没有任何专门的「分块器」。**
3. **续块靠 `num_computed_tokens` 记账** —— 「已计算到第几个 Token」被持久记录，下一步从断点继续，直到追上 `num_prompt_tokens`（Prefill 完成）后自然转入每步 +1 的 Decode。

**抢占的代码行为**：`_preempt_request()` 里释放 KV 块和编码器缓存 → 状态置为 `PREEMPTED` → **`num_computed_tokens` 归零** → 重新塞回 `waiting` 队头等资源。归零这一步说明 V1 默认的抢占恢复策略是**重计算**（见 [[02-PagedAttention：KV Cache 的分页管理]] 的换出 vs 重算对照）。

> 读这个调度器的精髓就一句话：**`num_computed_tokens` 一路推向 `num_tokens_with_spec`，每步推进多少由 `min(剩余需求, 剩余预算)` 决定。**

## 调度器面对的三组冲突

| 冲突 | 表现 |
| --- | --- |
| **吞吐 vs 延迟** | 一步塞进的请求越多系统吞吐越高，但每步前向耗时也越长，单请求 TPOT 变差 |
| **Prefill vs Decode 争抢** | 长 Prompt 的 Prefill 拖慢同批 Decode 的出字速度。由 Chunked Prefill 缓解 |
| **公平 vs 效率** | 先来的请求是否优先？长请求会不会被短请求反复插队饿死？ |

调优旋钮主要是 `max_num_batched_tokens`（即 Token Budget）：

| 取向 | 做法 | 代价 |
| --- | --- | --- |
| 偏吞吐 | 调大预算，一步塞更多 Token，GPU 装得更满 | 长 Prefill 占比上升，Decode 的 TPOT 稳定性下降 |
| 偏延迟稳定 | 调小预算，每步计算量更可控 | 单步能干的活变少，峰值吞吐受限 |

没有普适最优值。正确做法是**在目标负载（真实 Prompt 长度分布、并发量）下压测**，看 TTFT、TPOT 的 P95/P99 与吞吐三条曲线，找满足 SLO 的最大吞吐点。

> [!warning] 由于批次构成随负载变化，**Continuous Batching 让延迟指标高度负载敏感**。容量规划必须在目标并发下压测。

## 和投机解码的耦合

投机解码（Draft 模型先猜多个 Token、Target 模型并行验证）也在同一套调度框架里落脚 —— 在 `{request_id: num_tokens}` 里，投机解码的验证就是「这一步处理若干个候选 Token」。两者叠加会抬高调度复杂度：**一步内不同请求推进的 Token 数不再统一**，KV 块的增长也不再是每步一块。

它与量化的叠加还会放大精度风险。这两条约束是 [[01-推理性能指标与瓶颈定位]] 里「技术叠加不等于效果叠加」的具体来源。投机解码本身需要独立成篇（见 [[00-AI Infra 专栏导览]] 的待建清单），本库暂不展开。

## 相关

- [[02-PagedAttention：KV Cache 的分页管理]] —— 调度依赖的细粒度显存弹性从哪来
- [[01-推理性能指标与瓶颈定位]] —— Memory Bound 结论与 TTFT/TPOT/Goodput 的定义
- [[04-前缀缓存：APC 与 RadixAttention]] —— 同一套调度框架里的第三种 Token 取值来源
- [[10-KV Cache 与推理优化]] —— batched decode 为什么必须做

## 参考

- Yu et al., *Orca: A Distributed Serving System for Transformer-Based Generative Models*, OSDI 2022，https://www.usenix.org/conference/osdi22/presentation/yu
- Agrawal et al., *SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills*，https://arxiv.org/abs/2308.16369
- https://blog.vllm.ai/2025/01/27/v1-alpha-release.html
- https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py
- https://docs.vllm.ai/en/latest/configuration/optimization.html
- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E5%9B%9B-%E6%8E%A8%E7%90%86%E4%BC%98%E5%8C%96/%E7%AC%AC2%E7%AB%A0-%E6%8E%A8%E7%90%86%E5%BC%95%E6%93%8E%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF/22-continuous-batching
