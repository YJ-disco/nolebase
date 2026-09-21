---
tags:
  - AI/infra/推理引擎
  - AI/infra/显存管理
---

# PagedAttention：KV Cache 的分页管理

[[10-KV Cache 与推理优化]] 算过一笔账：7B 量级模型在 32K 上下文下单请求的 KV Cache 约 21 GB。但**算出它有多大，和把它管起来是两道题**。真正的困难在于事先不知道一个请求会生成多长：

- 按最大长度预留 → 大量空间被锁着不用
- 按实际用量预留 → 长度预估错了就得整块搬移

PagedAttention 是 vLLM 用来解这道题的机制，来自 `arXiv:2309.06180`。它的做法**原样搬了操作系统的虚拟内存分页**。

## 一、连续分配的两宗罪

PagedAttention 之前的推理框架给每个请求分配**一整块连续显存**，并按「这个请求可能生成的最大长度」预留。这带来两类碎片：

| 类型 | 成因 |
| --- | --- |
| **内部碎片** | 请求最大可能生成 2048 Token 就先占好 2048 格，实际生成了 100 个就遇 EOS，剩下 1948 格被锁着却没用上 |
| **外部碎片** | 不同请求预留的大块之间留下大小不一的空隙，加起来可能不少，但每块都不够放下一个完整请求 |

论文的实测口径很扎眼：

| 系统 | KV Cache 利用率 | 浪费 |
| --- | --- | --- |
| FasterTransformer / Orca | **20.4% – 38.2%** | **60% – 80%** |
| vLLM（PagedAttention） | **> 96%** | **< 4%** |

> 碎片的根源是**连续 + 预留**这两个约束叠加。打破「必须连续」和「必须按最大长度预留」，问题就解开了。

## 二、分页：切块、映射、按需分配

三个动作：

1. **切块**：KV Cache 切成固定大小的 **KV Block**，每块存固定数量 Token 的 K 和 V。块大小由 `block_size` 决定，vLLM 常见取值为 **16**。
2. **映射**：用一张 **Block Table** 记录「这个请求的第几个逻辑块，对应物理显存里的哪一个物理块」。请求看到的是连续的逻辑块序列，物理上可以散落在任意位置。
3. **按需分配**：填满一块才申请下一块，**用多少申请多少，绝不提前预留**。

`block_size = 4` 的示意（真实常用 16，取 4 是为了好画）。一个请求 Prefill 了 9 个 Token，需要 $\lceil 9/4 \rceil = 3$ 个逻辑块：

| 逻辑块 | 物理块 | 状态 |
| --- | --- | --- |
| 0 | #7 | 已满（4/4） |
| 1 | #2 | 已满（4/4） |
| 2 | #5 | 部分填充（1/4） |

物理块编号可以完全不连续。进入 Decode 后每生成一个 Token 就往当前逻辑块空位里填，逻辑块 2 还有 3 格；**填满后才按需申请新物理块挂到逻辑块 3**。

> [!warning] vLLM 官方文档特别提醒：这里的 KV Block 是 vLLM 自己的显存管理单位，和 CUDA 的 **thread block（线程块）**完全是两回事，不要混淆。

### 分页如何消灭两类碎片

| | 传统连续分配 | PagedAttention |
| --- | --- | --- |
| 显存布局 | 每请求一整块连续显存 | 固定大小块，物理上分散 |
| 分配时机 | 提前按最大长度预留 | 按需逐块分配 |
| 内部碎片 | 严重（预留远超实际） | **最多不到 1 个块**（`block_size=16` 时最多浪费 15 个 Token 的位置） |
| 外部碎片 | 存在（空隙大小不一） | **消除**（所有块大小相同，任何空闲块都能被任何请求用） |

省下的显存直接变成吞吐：Decode 是 Memory Bound（见 [[01-推理性能指标与瓶颈定位]]），显存利用率上去了，同一张卡能同时塞下更多并发请求。

## 三、Kernel 视角：分散的块怎么被读出来

到这里还留着一个疑问：KV 被打散到显存各处，Attention Kernel 怎么还能把它们凑齐？看懂这一层才会明白，PagedAttention 属于**算子与内存布局的协同设计**，把它当成纯显存管理技巧会漏掉一半。

传统 Attention 假设一个序列的 K、V 在显存里连续排列，Kernel 用基地址加偏移顺序扫过去。PagedAttention 打破了连续性，Kernel 必须先查块表，把「逻辑位置」翻译成「物理地址」：

| 量 | 作用 |
| --- | --- |
| `physical_block_number` | 决定去哪个物理块取数，乘以块的跨度（block stride）得到块的基址 |
| `physical_block_offset` | 定位这个 Token 在块内的第几格 |

Key 指针的地址形式大致是（vLLM 官方 Kernel 文档的简化示意）：

```cpp
const scalar_t* k_ptr = k_cache
                      + physical_block_number * kv_block_stride   // 定位到物理块
                      + kv_head_idx           * kv_head_stride    // 定位到对应 KV 头
                      + physical_block_offset * x;                // 定位块内 token
```

一个 Warp 在外层循环里逐块推进，每次迭代通过块号跳到下一个物理块，`k_ptr` 随之指向不同块里的 Key。**物理上东一块西一块的存储，被 Kernel 当成一段逻辑连续的上下文来访问。**

> [!warning] vLLM 官方那篇 Kernel 走读文档标注为**基于原始论文的历史文档，不再反映当前代码**。上面的指针公式应当作理解原理的示意，不是当前源码。V1 引擎里这层逻辑已交给可插拔的 Attention 后端接管（见 [[05-Attention 后端与图优化]]）。

## 四、附带能力：块共享与写时复制

分页顺带解锁了连续分配做不到的事：**多个请求共享同一份物理块**。既然 KV 以块为单位、通过块表间接引用，多个请求的块表就可以指向同一个物理块，用**引用计数（`ref_cnt`）**记录有几个请求在用。

分叉时靠 **写时复制（Copy-on-Write, CoW）**：某个请求要往一个被共享的块里写新内容时，先把块复制一份成为私有块，在副本上写，再更新自己的块表。其他请求不受影响。

论文实测的共享节省：

| 场景 | 显存节省 |
| --- | --- |
| 并行采样（parallel sampling） | **6.1% – 9.8%** |
| Beam search（width 6） | **37.6% – 55.2%** |
| 共享系统前缀 | 某些配置下最高 **30%** |

> 这套「引用计数 + 写时复制」和 Linux `fork()` 之后父子进程共享内存页是同一个机制。理解了操作系统的分页与 CoW，PagedAttention 几乎没有新东西。

块共享也是 [[04-前缀缓存：APC 与 RadixAttention]] 的物理基础 —— 前者把共享限定在**同时并存的请求**之间，后者把它延伸到**先后到达的请求**之间。

## 五、显存被抽干时：抢占

并发太多、物理块池被抽干，新来的 Decode 步骤申请不到块怎么办？vLLM 的答案是**抢占（Preemption）**：临时踢出一部分请求、释放它们的块，等有空间再恢复。

被踢出的请求，已经算好的 KV 有两种处理方式：

| 恢复方式 | 做法 | 优点 | 代价 |
| --- | --- | --- | --- |
| **换出（Swapping）** | KV 从 GPU 拷到 CPU 内存暂存，恢复时拷回 | 不用重算，省算力 | 占 CPU 内存，来回拷贝有 PCIe 开销 |
| **重计算（Recomputation）** | 直接丢弃 KV，恢复时把已生成的 Token 当新 Prompt 重新 Prefill | 不占 CPU 内存，实现简单 | 重跑一次 Prefill，浪费算力 |

论文给出的量化判据：

- **重计算的开销从不超过换出的 20%**（在所有测试配置下）
- **块大小 ≤ 64 Token 时，重计算通常比换出往返更快**

**vLLM V1 的默认是重计算而非换出** —— 项目在简化后的架构里实测重算开销更低。

抢占是「保证不 OOM、请求不失败」的兜底机制。日志里频繁出现 preemption 告警说明并发压得太满：调低并发、调高 `gpu_memory_utilization` 给 KV 更多空间，或者加卡。**它是系统在硬扛的信号，不是常态。**

## 六、想调什么，调哪几个参数

作为使用者能影响 PagedAttention 行为的旋钮：

| 参数 | 含义 | 取值与取舍 |
| --- | --- | --- |
| `gpu_memory_utilization` | 模型权重之外，剩余显存的这个比例划给 KV Cache 块池 | 默认约 **0.9**。调高能容纳更多并发、更少触发抢占，但留给激活值和其它开销的余量变小，太激进会 OOM |
| `block_size` | 每块存多少个 Token 的 KV，即分页粒度 | 太小 → 块表变长、管理开销上升；太大 → 尾块内部碎片变多。vLLM 常用 **16** 这样中等值 |
| `enable_prefix_caching` | 相同前缀的请求复用物理块 | 有共享前缀就开，见 [[04-前缀缓存：APC 与 RadixAttention]] |
| `max_num_seqs` | 单次迭代最多处理多少条序列 | 与调度器共同决定并发上限，见 [[03-推理调度：Continuous Batching 与 Chunked Prefill]] |

> 绝大多数场景**不需要手动改 `block_size`**，默认值已在多数模型上调优过。真正常动的是 `gpu_memory_utilization`（压榨显存）和 `enable_prefix_caching`。

## 七、源码骨架：V1 的四个咬合件

V1 引擎把管理逻辑集中到两个类：

```
Scheduler ──allocate_slots / free──▶ KVCacheManager（请求视角：管每个请求的块）
                                        │
                        get_new_blocks / free_blocks
                                        ▼
                                     BlockPool（全局视角：管所有物理块）
                                        ├─ FreeKVCacheBlockQueue（空闲块双向链表）
                                        └─ BlockHashToBlockMap（前缀缓存哈希表）
```

**`KVCacheManager`**（`vllm/v1/core/kv_cache_manager.py`）三个核心方法：

| 方法 | 做什么 |
| --- | --- |
| `get_computed_blocks` | 前缀缓存查询 —— 返回该请求能命中的、已经算好的块及命中 Token 数 |
| `allocate_slots` | 为新增 Token 申请槽位/块，检查空闲块是否够用；不够返回 `None`（交调度器触发抢占） |
| `free` | 请求结束时**逆序**释放块 |

`free` 的逆序是个容易被忽略的设计：一个请求的**最后一个块哈希了最多的 Token**（前缀最长），最不容易被别的请求命中复用，所以让它**优先被淘汰**；靠前的块（如共享 System Prompt 的开头）更可能被复用，留得更久。

**空闲块为什么用双向链表**：`BlockPool` 里所有块存在一个普通列表 `self.blocks`（按 block id 索引），但空闲块单独用 `FreeKVCacheBlockQueue` 管理，内部是**双向链表**而非栈或 `deque`。原因是要同时满足两种操作：

- 分配时从头部快速取走 N 个（`popleft_n`）
- **前缀缓存命中时，一个原本空闲但仍缓存着有效内容的块要被重新激活 —— 需要从链表中间任意位置把它摘出来**（`remove`）

普通队列没法 $O(1)$ 地从中间删除。这个设计服务于「空闲块也可能携带可复用缓存」这一前提。

```python
def get_new_blocks(self, num_blocks):
    if num_blocks > self.get_num_free_blocks():
        raise ValueError("空闲块不足")          # 触发上层抢占
    blocks = self.free_block_queue.popleft_n(num_blocks)
    for blk in blocks:
        assert blk.ref_cnt == 0                 # 新分配的块必须无人引用
        blk.ref_cnt += 1
    return blocks

def free_blocks(self, ordered_blocks):
    for blk in ordered_blocks:
        blk.ref_cnt -= 1
        if blk.ref_cnt == 0:                    # 没人用了才真正回收
            self.free_block_queue.append(blk)   # 放回空闲链表尾部

def touch(self, blocks):                        # 前缀缓存命中时调用
    for blk in blocks:
        if blk.ref_cnt == 0:                    # 从空闲链表里"抢救"回来
            self.free_block_queue.remove(blk)
        blk.ref_cnt += 1
```

两个要点：

1. **释放不等于清空**：`ref_cnt` 归零的块只是回到空闲链表，**内容还在**。后续请求的前缀哈希命中它时，`touch()` 直接复活复用。
2. **带哈希的块留得更久**：释放时，没缓存价值的块放链表头（优先被覆盖），带哈希、有复用潜力的块放链表尾。

## 八、代价

PagedAttention 不是免费的抽象：

| 代价 | 量级 |
| --- | --- |
| **Kernel 间址开销** | 块表查找 + 非连续访存，比连续张量读慢。论文口径约高 **20% – 26%** |
| 块表管理开销 | 块表本身占空间，块越小表越长 |
| 实现复杂度 | 需要专门的 CUDA Kernel，`block_size` 需在管理开销与尾块碎片间权衡 |

> 但单 Kernel 略慢换来的是**能同时跑 2–4 倍多的请求**。系统层面看的是吞吐，不是单个 Kernel 的延迟 —— 这是「局部变慢换全局变快」的典型。
>
> 一条更硬的证据：论文对比了 **Orca-Oracle**（给 Orca 完美的输出长度预知，从而完全消除超额预留），vLLM 仍快 **1.7× – 2.7×**。可见收益**并不限于「不预留」** —— 分页分配与跨请求共享各自贡献了一部分。

在 H100 上对比厂商库与 HuggingFace Transformers 时，后者差距更大（单 completion 14–24×），但那个倍数反映的是基线本身的低效，不应归功于 PagedAttention 单独一项。

## 相关

- [[10-KV Cache 与推理优化]] —— KV Cache 的显存账本与 MHA/GQA/MQA 布局
- [[03-推理调度：Continuous Batching 与 Chunked Prefill]] —— 分页提供细粒度可回收显存，调度才能动态补位
- [[04-前缀缓存：APC 与 RadixAttention]] —— 把块共享从「并存请求」延伸到「先后请求」
- [[05-Attention 后端与图优化]] —— 接管块间接寻址的后端层

## 参考

- **PagedAttention 原论文**（碎片与利用率数据、2–4× 吞吐、Orca-Oracle 对比、共享节省、抢占恢复的 20% 判据、块 ≤64 时重算更快、Kernel 开销 20–26%）：Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*, SOSP 2023，https://arxiv.org/abs/2309.06180
- **vLLM 官方博客**（PagedAttention 的动机与 vs HuggingFace 的对比）：https://blog.vllm.ai/2023/06/20/vllm.html
- **vLLM Design — PagedAttention Kernel**（physical_block_number / physical_block_offset 的寻址示意；官方已标注为历史文档）：https://docs.vllm.ai/en/latest/design/paged_attention.html
- **vLLM Engine Arguments**（`gpu_memory_utilization`、`block_size`、`enable_prefix_caching`、`max_num_seqs` 的默认值口径）：https://docs.vllm.ai/en/latest/configuration/engine_args.html
- **vLLM V1 源码**（`vllm/v1/core`：KVCacheManager、BlockPool、FreeKVCacheBlockQueue）：https://github.com/vllm-project/vllm/tree/main/vllm/v1/core
- **AIInfraGuide 2.1 PagedAttention**（Block Table 示例、Kernel 视角、工程参数、V1 源码骨架）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E5%9B%9B-%E6%8E%A8%E7%90%86%E4%BC%98%E5%8C%96/%E7%AC%AC2%E7%AB%A0-%E6%8E%A8%E7%90%86%E5%BC%95%E6%93%8E%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF/21-pagedattention
