---
tags:
  - 分布式/存储
---

# GFS

Google File System（SOSP 2003）是一套**从假设反推设计**的系统 —— 先把负载特征和硬件现实摆出来，再让每个设计决策都能对回某条假设。这让整套设计很好把握：**大部分设计选择都能用一条假设解释**。

它先摆出这些假设：

- **由大量廉价商品组件构成，组件经常故障** —— 所以系统必须持续自我监控，并把检测、容忍、恢复组件故障当作**日常**工作；
- **存少量大文件**：预期几百万个文件，**每个通常 100 MB 以上**，多 GB 是常见情形；小文件要支持但不做优化；
- **负载以顺序读写大文件为主**，小随机读次之；
- 规模锚点：**最大的集群有 1000+ 存储节点、300+ TB 磁盘**，被数百个客户端持续重度访问。

## 架构：单 master + chunkserver

一个 GFS 集群包含**一个 master**、**多个 chunkserver** 和**多个 client**，三者都是跑在商品 Linux 机器上的用户态服务进程。

- **文件被切成固定大小的 chunk**（见下一节）。每个 chunk 有一个**不可变、全局唯一的 64 位 chunk handle**，由 master 在创建 chunk 时分配。
- chunkserver 把 chunk 当**普通 Linux 文件**存在本地盘上，按 chunk handle + 字节范围读写。
- **默认 3 副本**，用户可以对命名空间的**不同区域**指定不同的副本级别。

**master 持有全部元数据**：命名空间、访问控制信息、文件到 chunk 的映射、每个 chunk 副本的当前位置。它还控制系统级活动：**chunk 租约管理、孤儿 chunk 的垃圾回收、chunkserver 之间的 chunk 迁移**。master 与每个 chunkserver 通过 **HeartBeat** 消息定期通信（下指令 + 收集状态）。

**一条重要的分工**：client 与 master 只做**元数据操作**，**所有携带数据的通信都直接走 chunkserver**。客户端不经过 master 转发数据 —— 这是单 master 不会成为数据面瓶颈的前提。

另外两个刻意的"不做"：

- **不提供 POSIX API**，所以不需要挂进 Linux 的 vnode 层；
- **client 和 chunkserver 都不缓存文件数据**。client 侧的理由是多数应用**流式读大文件**，或者工作集**大到缓存不下**，缓存收益小；不缓存还消除了缓存一致性问题（**client 会缓存元数据**，这是两回事）。chunkserver 侧根本不需要缓存 —— chunk 就是本地文件，**Linux 的 buffer cache 已经把热数据留在内存里了**。

## 为什么 chunk 是 64 MB

**chunk 大小是"关键设计参数"之一，取值 64 MB**，比典型文件系统块大得多。每个 chunk 副本是一个普通 Linux 文件，**按需扩展（lazy space allocation）**，因此不会因为内部碎片浪费空间 —— 这是对大 chunk 最常见的反对意见，被这条实现细节化解了。

三条收益，每条都对应一条假设：

| 收益 | 机制 |
| --- | --- |
| **减少 client 与 master 的交互** | 同一 chunk 上的读写只需**一次**初始请求拿位置信息。对"顺序读写大文件"的负载效果显著；即使是小随机读，client 也能**轻松缓存多 TB 工作集的全部 chunk 位置** |
| **减少网络开销** | chunk 大意味着 client 更可能对同一 chunk 做多次操作，于是可以**保持长期 TCP 连接** |
| **减小 master 上的元数据规模** | 直接结果是元数据能**放进内存**，而这又带来内存元数据的一整套好处 |

**代价是热点**：小文件只占少量 chunk（可能就 1 个），很多 client 同时访问同一文件时，存这些 chunk 的 chunkserver 会被压垮。这个问题的处理很具体：

> 实践中热点不是主要问题（应用主要顺序读大文件），**但它确实发生过** —— GFS 最初被一个批处理队列系统使用时，一个可执行文件被作为**单 chunk 文件**写入 GFS，然后在**数百台机器上同时启动**，少数 chunkserver 被数百个并发请求压垮。

修法是两条：给这类可执行文件**更高的副本因子**，以及让批处理系统**错开应用启动时间**。长期方案是：允许 client 在这种情况下**从其他 client 读数据**。

## 元数据全放内存，但 chunk 位置不落盘

master 的元数据分三类：**文件与 chunk 命名空间**、**文件到 chunk 的映射**、**每个 chunk 副本的位置**。

**三类都在内存里**，但**只有前两类持久化** —— 通过**操作日志（operation log）**，日志写在 master 本地盘并复制到远程机器。

**内存方案的成本可以算出来**：master 为每个 64 MB chunk 维护**少于 64 字节**的元数据；文件命名空间用**前缀压缩**，每个文件通常也少于 64 字节。"大多数 chunk 是满的"，因为一个文件通常含很多 chunk，只有最后一个可能是部分填充。

### 为什么 chunk 位置反而不落盘

**chunk 位置是三类元数据里唯一不持久化的**，这条反直觉的决定有专门解释，而且**最初确实尝试过持久化**：

- 启动时（以及 chunkserver 加入集群时）**直接向 chunkserver 询问**它们有哪些 chunk；
- 此后 master 能保持最新，因为**它控制所有 chunk 放置**，并用定期 HeartBeat 监控 chunkserver 状态。

**放弃持久化的理由**：这**消除了 master 与 chunkserver 保持同步的问题** —— 在数百台服务器的集群里，chunkserver 加入、离开、改名、故障、重启这些事件"发生得太频繁了"。一句很本质的总结：

> **chunkserver 对自己盘上有哪些 chunk 有最终话语权。**

在 master 上维持一份一致的视图没有意义：chunkserver 上的错误可能让 chunk **突然消失**（比如盘坏掉被禁用），运维人员也可能**给 chunkserver 改名**。

### 操作日志与 checkpoint

**操作日志不只是一份恢复用的记录，它同时是定义并发操作顺序的逻辑时间线** —— 文件、chunk 以及它们的版本，都由**创建时的逻辑时间**唯一且**永久**地标识。

正确性的要求是严格的：**必须在日志记录本地与远程都刷盘之后，才响应 client**。否则即使 chunk 数据还在，也会**丢掉整个文件系统或最近的客户端操作**。为了不让刷盘与复制拖垮吞吐，master 会**批量攒多条日志记录**再一次刷盘。

恢复靠**重放日志**，所以日志必须小 —— 日志超过一定大小时 master 做 **checkpoint**：一次完整的 checkpoint 是**紧凑的 B 树形式，可以直接映射进内存**用于命名空间查找、无需额外解析。实现上有两个细节：

- **做 checkpoint 不阻塞 incoming mutation** —— master 切换到新日志文件，在**单独线程**里建新 checkpoint；新 checkpoint 覆盖切换前的所有 mutation；
- 耗时量级：**几百万文件的集群大约一分钟**能建完；完成后同时写本地与远程。恢复只需要最新一个完整 checkpoint 加其后的日志；旧 checkpoint 与日志可以删，但会保留几个以防灾难。**checkpoint 期间失败不影响正确性** —— 恢复代码会检测并跳过不完整的 checkpoint。

## 一致性模型：defined / consistent / inconsistent

GFS 的模型是**放宽的**，这么做的理由是"大幅简化文件系统而不给应用加上沉重负担"。它只有三个状态词，关键是把"一致"与"已定义"分开：

| 术语 | 定义 |
| --- | --- |
| **一致（consistent）** | 所有 client **无论读哪个副本**，都看到相同的数据 |
| **已定义（defined）** | 在一致的基础上，client 会看到该 mutation **完整写出**的内容 |
| **不一致（inconsistent）** | 不同 client 在不同时刻可能看到不同的数据 |

三种情形落在这三档上：

- **串行成功**（没有并发写干扰）→ 受影响区域 **defined**（因而也 consistent）；
- **并发成功** → 区域 **consistent 但 undefined**：所有 client 看到相同的**同一份**数据，但它可能**不反映任何一个 mutation 写的内容** —— 通常是多个 mutation 的**混合碎片**；
- **失败** → 区域 **inconsistent**（因而也 undefined）。

**应用不需要区分不同种类的 undefined 区域**，只需要分清 defined 与 undefined。

**命名空间 mutation（例如创建文件）是原子的**，由 master 独占处理：命名空间锁保证原子性与正确性，而**master 的操作日志定义了这些操作的全局全序**。

### 两种数据 mutation，与 record append 的代价

- **write**：数据写在**应用指定的文件偏移**上；
- **record append**：数据（"记录"）**在存在并发 mutation 的情况下也原子地至少追加一次**，但**偏移由 GFS 自己选**。

注意一个容易混的点：**"普通 append"只是 client 认为的当前文件末尾偏移上的一次 write**，不是同一个东西。record append 返回的偏移**标记一段 defined 区域的起点**，这段区域里含该记录。

**它的代价被明确写出来了**：GFS 可能在记录之间插入 **padding 或记录副本**，这些区域被视为 **inconsistent**，但"通常比用户数据量小得多"。**这也解释了为什么 record append 是"至少一次"（at least once）而不是恰好一次。**

### 这套保证靠什么成立

两条机制：

1. **在所有副本上以相同顺序应用 mutation**（靠租约，见下一节）；
2. **用 chunk version number 检测陈旧副本** —— chunkserver 宕机期间会错过 mutation，副本据此变陈旧。

**陈旧副本**永远不会参与 mutation，也**不会被给到**向 master 询问 chunk 位置的 client，并且会被尽早垃圾回收。

**一个真实存在的一致性窗口**同样被点明：因为 client **缓存了 chunk 位置**，它可能在信息刷新前**读到陈旧副本**。这个窗口受限于**缓存条目的超时**与**文件下次 open**（open 会清掉该文件所有 chunk 信息的缓存）。一条降低实际影响的观察：**由于多数文件是 append-only，陈旧副本通常返回"chunk 提前结束"而不是过期数据**。

## 租约与 mutation 顺序

**mutation** 指改变 chunk 内容或元数据的操作（write、append）。**每个 mutation 都会在该 chunk 的所有副本上执行。**

要在副本之间维持一致的顺序，GFS 用的是**租约**：

1. **master 把 chunk 租约授予其中一个副本，称为 primary**；
2. **primary 为该 chunk 的所有 mutation 选一个串行顺序**；
3. **所有副本按这个顺序应用 mutation**。

于是**全局 mutation 顺序由两段定义**：

> 先由 **master 选定的租约授予顺序**决定，租约之内由 **primary 分配的序列号**决定。

**租约参数的目的是让 master 的管理开销最小**：

- 初始超时 **60 秒**；
- 但只要 chunk 正在被 mutation，primary 就可以请求延期，**而且通常能无限期地拿到**；
- 延期请求与授予**捎带在 HeartBeat 消息上**（master 与所有 chunkserver 之间本来就定期交换），不额外增加消息类型；
- master 有时会**在到期前主动撤销租约**（例如它想禁用某个正在被改名的文件的 mutation）；
- **即使 master 与 primary 失去通信，它也能安全地在旧租约到期后把新租约授予另一个副本**。

### 写流程：七步，以及"数据流与控制流解耦"

写流程按编号图走一遍：

1. client 向 master 询问**哪个 chunkserver 持有该 chunk 的当前租约**以及其他副本的位置（若无租约，master 授予一个）；
2. master 回复 **primary 的身份与所有 secondary 副本的位置**。client **缓存**这些信息，只在 primary 不可达、或 primary 回复自己已不持有租约时，才再次联系 master；
3. **client 把数据推给所有副本**（顺序任意）。每个 chunkserver 把数据存在**内部 LRU 缓冲**里，直到数据被使用或被老化掉；
4. client 通知 primary 可以开始写……
5. primary 分配连续的序列号，按序应用 mutation；
6. secondary 按 primary 指定的顺序应用；
7. secondary 回复 primary，primary 回复 client。

**第 3 步的位置是关键**：数据**先于**控制消息推出去，而且 client 推给谁、按什么顺序推都自由。这条叫**把数据流与控制流解耦**，它带来一个直接收益：

> 可以**按网络拓扑调度昂贵的数据流**，而**不受哪个 chunkserver 是 primary 的影响**。

也就是说，"哪台机器负责排序"（控制）与"数据走哪条链路"（吞吐）分开优化 —— 数据可以沿链式（pipeline）传给拓扑上最近的机器，绕开远的 primary。

## 判据速查

| 问题 | 答案 |
| --- | --- |
| 为什么用单 master | 大幅简化设计，且让 master 能做复杂的 chunk 放置与复制决策 |
| chunk 为什么是 64 MB | 减少 client↔master 交互、支持长期 TCP 连接、让元数据能放进内存 |
| 大 chunk 的代价 | 小文件造成**热点**；真实案例是单 chunk 可执行文件被数百台机器同时启动 |
| 元数据哪一类**不**持久化 | **chunk 位置** —— 启动时向 chunkserver 询问；理由是 chunkserver 对自己盘上的 chunk 有最终话语权 |
| 每个 chunk 的元数据开销 | **少于 64 字节** |
| 什么时候才响应 client | 日志记录在**本地与远程都刷盘之后** |
| checkpoint 阻塞写入吗 | **不阻塞** —— 切新日志 + 单独线程建 checkpoint；几百万文件约一分钟 |
| 三个一致性术语的关系 | **defined ⊂ consistent**；不满足 consistent 就叫 inconsistent |
| 并发成功的结果是什么 | consistent 但 **undefined** —— 所有 client 看到同一份数据，但它可能是多个 mutation 的**混合碎片** |
| record append 为什么是"至少一次" | GFS 可能插入 **padding 或记录副本**，那些区域算 inconsistent |
| 陈旧副本怎么被发现 | **chunk version number**；陈旧副本不参与 mutation，也不给 client |
| 全局 mutation 顺序谁定 | 两段：master 的**租约授予顺序** + primary 在租约内分配的**序列号** |
| 租约超时多久，能续吗 | 初始 **60 秒**；chunk 持续被写时可**无限延期**，延期捎带在 HeartBeat 上 |
| 数据流与控制流解耦为了什么 | 让**数据**可按网络拓扑调度，不受 primary 位置影响 |

## 相关

- [[02-Bigtable|Bigtable]] —— 直接建在 GFS 之上，把 GFS 当 SSTable 与日志的存储层
- [[03-一致性模型]] —— GFS 的 defined/consistent/inconsistent 是一套**面向追加负载的专用模型**，与通用一致性模型的谱系不同
- [[01-分布式事务|分布式事务]] —— 跨文件的多步操作不在 GFS 范围内，那是上一层的事

## 参考

- S. Ghemawat, H. Gobioff, S.-T. Leung. *The Google File System*. SOSP 2003.
- 本篇用的 PDF 抽取文本里，引入部分的两段（约 3200 字符，占全文 3.6%）因字体缺 ToUnicode 映射无法还原；上述内容全部来自可读部分。
