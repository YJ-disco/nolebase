---
tags:
  - 分布式/存储
---

# Windows Azure Storage

WAS（SOSP 2011）是微软的云存储，2008 年 11 月起在生产环境运行。它在这条线上的位置很特殊：**前面几篇里"高可用"和"强一致"基本是二选一**（[Dynamo](03-Dynamo.md) 选 AP、[Spanner](04-Spanner 与 F1.md) 用时钟设施把强一致买回来），而 WAS 的标题就直接写着 "**with Strong Consistency**" —— 它靠的是**把强一致限定在一个层次里做，把可用性交给另一层**。

论文给了一个真实负载作为规模锚点：Windows Azure 上的**摄入引擎**（为 Facebook 与 Twitter 做近实时搜索，是 Bing 管道的一部分，**在用户发帖后 15 秒内让内容可被公开搜索到**）—— 它在 WAS 里存**约 350 TB 数据（复制前）**，**峰值约 40,000 事务/秒**，**每天 20 到 30 亿次事务**。内容走 Blobs、工作流走 Queues、处理结果与状态走 Tables —— 论文把这个组合称为"我们看到的常见用法模式"。

## 两个层级：stamp 与位置服务

**Storage Stamp** 是一个集群：

- **N 个机架的存储节点**，每个机架作为**独立故障域**建设（冗余网络与电力）；
- **典型 10 到 20 个机架，每机架 18 个磁盘密集型节点**；
- **第一代 stamp 每个约 2 PB 原始存储，下一代最多 30 PB**。

**利用率目标值得单独记**：论文要把 stamp 维持在**约 70% 利用率**（容量、事务、带宽三个维度），**避免超过 80%**，因为要留 20% 余量给两件事：

- **磁盘短行程（short stroking）** —— 只用盘的外圈轨道以获得更好的寻道时间与更高吞吐；
- **机架故障时继续提供容量与可用性**。

stamp 达到 70% 时，**位置服务用跨 stamp 复制把账号迁到别的 stamp**。

**Location Service（LS）** 管所有 stamp，也管跨 stamp 的**账号命名空间**：

- 把账号分配到 stamp，并跨 stamp 管理它们以做**灾难恢复与负载均衡**；
- **LS 自己分布在两个地理位置**做自身的灾难恢复；
- WAS 在**北美、欧洲、亚洲**三个地理区域提供存储；每个位置是一个数据中心（含一栋或多栋建筑），每个位置有多个 stamp；
- 扩容的方式：在目标位置部署新 stamp 并加入 LS → LS 把新账号分给新 stamp，也能把**已有账号从旧 stamp 迁到新 stamp**；
- 申请新账号时应用**指定位置亲和性**（如 US North），LS 用启发式（考虑各 stamp 的满载程度、网络与事务利用率）选一个作为该账号的 primary stamp，把账号元数据存进去，然后**更新 DNS** 让 `https://AccountName.service.core.windows.net/` 路由到该 stamp 的 VIP。

## 一个 stamp 内的三层

这三层的分工方式是全篇的核心设计，**每层只解决一类问题**：

| 层 | 负责 | 不负责 |
| --- | --- | --- |
| **Stream Layer** | 存盘上的 bit；把数据分布与复制到许多服务器以在 stamp 内保持持久 | **不理解更高层的对象构造或语义** |
| **Partition Layer** | 理解 Blob / Table / Queue 三种抽象、提供可扩展命名空间、**提供事务排序与强一致性**、把数据存在流层之上、缓存数据 | 不直接管 bit 的落盘与复制 |
| **Front-End (FE) Layer** | 一组**无状态服务器**：查 AccountName、认证授权、按 PartitionName 路由 | 不持有状态 |

**关键分工的一句话**：**数据存在流层，但从分区层访问** —— **分区服务器与流服务器在同一存储节点上 co-located**。

**Partition Layer 的规模手段**：**所有对象都有 PartitionName**，按 PartitionName 值切成**不相交的范围**，由不同分区服务器服务；这一层管**哪个分区服务器服务哪些范围**，并提供**跨分区服务器的自动负载均衡**。

**FE 的规模手段**：**无状态** → 可以随意加。它靠系统维护的 **Partition Map** 知道 PartitionName 范围与分区服务器的对应关系。

## 流层：append-only 的文件系统

流层只被分区层使用，提供类似文件系统的命名空间，**但所有写都是 append-only**。四个数据概念：

| 概念 | 定义与参数 |
| --- | --- |
| **Block** | **读写的最小单位**，最大 N 字节（**例如 4 MB**）；**不要求等大**，由客户端控制大小；**读时必须读整个 block** —— 因为**校验和是 block 级的、每块一个**；此外**系统里所有 block 每隔几天会做一次校验和验证** |
| **Extent** | **流层复制的单位**，**默认在一个 stamp 内保留三份副本**；存在一个 **NTFS 文件**里，由一串 block 组成；**分区层使用的目标大小是 1 GB** |
| **Stream** | **extent 指针的有序列表**，由 Stream Manager 维护；对分区层来说像一个大文件；**可追加、可随机读**；**只有最后一个 extent 可追加，之前的全部不可变** |
| **Seal** | extent 填到目标大小后**在一个 block 边界上被封**，之后不可追加。**对冷 extent 会做纠删码编码** |

**大小对象的两种处理**（这一条说明了 1 GB 这个目标值为什么不能一刀切）：

- **小对象**：分区层把多个追加到**同一个 extent、甚至同一个 block**；
- **TB 级大对象（Blob）**：分区层把它**拆到许多 extent 上**。

分区层**在自己的索引里记录对象存在哪些 stream、extent 与 extent 内的字节偏移**。

**一个很快的操作**：**用拼接现有 stream 的 extent 来构造新 stream** —— 因为只是**更新一张指针列表**。

### Stream Manager 与那些"不做"的事

Stream Manager（SM）**本身是一个标准 Paxos 集群，且在客户端请求的关键路径之外**。它维护 stream 命名空间、extent 状态与 extent 在 Extent Node（EN）上的分配，职责六条：监控 EN 健康、创建并分配 extent、**惰性再复制**丢失的副本、**垃圾回收**不再被引用的 extent、按策略**调度纠删码编码**。

论文特意交代了 SM 的边界，这些"不做"正是它能扩展的原因：

- **SM 不知道 block，只知道 stream 与 extent**；
- **它不跟踪每一次 block 追加** —— 因为 block 总数可能极大，**SM 无法扩展到跟踪它们**；
- 它**周期性轮询（sync）EN 的状态**；发现某 extent 的副本数少于期望时，**惰性地重建再复制**；
- **副本放置靠随机**：SM 在**不同故障域**里随机选 EN，使副本不会因电力、网络或同机架而相关失效。

### 复制流程：为什么这里不需要 lease

这是与前面几篇一个明显的对照：

1. 创建 stream 时，**SM 为第一个 extent 分配三个副本（一 primary、两 secondary）到三个 EN**，节点由 SM **随机**选以分散故障域与升级域（同时考虑 EN 使用率做负载均衡）；
2. **SM 还决定哪个副本是 primary**；
3. **写总是从客户端到 primary EN，primary EN 负责协调写往两个 secondary EN**；
4. **extent 在被追加期间（未 seal 时），primary 与三个副本的位置永不改变**。

由第 4 点直接推出一条结论：

> **不需要用 lease 来表示 extent 的 primary —— 因为 extent 未 seal 时 primary 总是固定的。**

对比一下 [GFS](01-GFS.md)：那里必须用租约来选 primary，因为**同一个 chunk 会被反复修改**；而这里 extent 是 append-only 的，**"谁是 primary"在 extent 生成时就定死了，直到它被 seal**。**把可变性从数据结构里去掉，就省掉了一整套租约机制。**

**多块追加（multi-block append）的契约**值得单独记：它允许**把一次大量顺序数据作为单个原子操作写入**，而**最小读单位是单个 block** —— 两者合起来就实现了"一次性写大量顺序数据、之后做小读"。代价是一份契约：

> **若客户端因故障没收到回复，应当重试请求（或 seal 该 extent）。** 这**意味着客户端必须预期同一个 block 可能被追加多次，并正确处理重复记录**。

**分区层处理重复记录的两种方式**（按数据类型分）：

- **元数据与 commit log stream**：所有写入的事务都有**序列号**，重复记录的序列号相同；
- **行数据与 blob 数据 stream**：重复写时**只有最后一次写会被 RangePartition 的数据结构指向**，之前的重复写没有引用，会被后续垃圾回收。

### 流层给分区层的两条保证

这是三层设计能成立的枢纽 —— **流层与分区层是共同设计的（co-designed）**：

> 分区层提供强一致的**正确性建立在流层这两条保证之上**：
> 1. **一旦一条记录被追加并向客户端确认，任何后续从任何副本的读都会看到相同数据（数据不可变）**；
> 2. **一旦 extent 被 seal，从任何 sealed 副本的任何读都总是看到该 extent 相同的内容。**

**注意这两条都是"不可变性"保证，不是"最新性"保证。** 流层只承诺"写下去的东西不会变"，**谁是最新的、写入顺序怎么定，全部交给分区层**。这就是"把强一致限定在一层里做"的确切含义。

论文还划了威胁边界：**恶意对手由数据中心、Fabric Controller 与 WAS 的安全机制负责，流复制不处理这类威胁**；而它处理的故障范围是**从磁盘与节点错误到断电、网络问题、位翻转、随机硬件故障，以及软件 bug** —— 这些都会造成数据损坏，**用校验和检测**。

## 分区层：Object Table 与 RangePartition

**Object Table（OT）** 是分区层的内部数据结构，**可以增长到数 PB**，按**流量负载动态切分成 RangePartition** 散布到各个分区服务器上。

> **一个 RangePartition 是 OT 中从 low-key 到 high-key 的一段连续行。同一 OT 的所有 RangePartition 互不重叠，且每一行都在某个 RangePartition 里。**

六张 OT 各有明确职责：

| OT | 存什么 |
| --- | --- |
| **Account Table** | 分配到该 stamp 的每个存储账号的元数据与配置 |
| **Blob Table** | 该 stamp 里所有账号的所有 blob 对象 |
| **Entity Table** | 所有账号的所有 entity 行（公开的 Windows Azure Table 抽象） |
| **Message Table** | 所有账号队列的所有消息 |
| **Schema Table** | 所有 OT 的 schema |
| **Partition Map Table** | 所有 OT 当前的 RangePartition 与对应的分区服务器 —— **FE 靠它路由请求** |

**主键设计**：Blob / Entity / Message 表的主键都是三个属性 —— **AccountName、PartitionName、ObjectName**，它们同时提供索引与排序顺序。

## 一个顺带发现的性能数字

论文在讲 commit log 时给了一组很说明问题的对比：**不带 journaling 的 commit log stream 平均端到端 append 延迟 30 ms；带 journaling 时平均 append 延迟 6 ms，而且延迟方差显著下降。**

**加一层日志反而快了 5 倍** —— 这条数据本身就是"为什么所有存储系统最后都会在写路径上加一层缓冲日志"的最好注脚，值得和 Aurora 那篇的"日志即数据库"并读。

## 判据速查

| 问题 | 答案 |
| --- | --- |
| WAS 相对 Dynamo / Spanner 的位置 | 不靠时钟设施也不放弃强一致，靠**把强一致限定在分区层、把持久性交给流层** |
| stamp 的规模与构成 | **10–20 个机架、每机架 18 个磁盘节点**；第一代约 **2 PB**，下一代最多 **30 PB** |
| 为什么利用率目标定 70%、不超 80% | 留 20% 给**磁盘短行程**（用外圈轨道）与**机架故障时继续服务** |
| 为什么 stamp 满了要迁移账号 | 用**跨 stamp 复制**把账号迁到新 stamp —— 这是 LS 的负载均衡手段 |
| LS 自己怎么容错 | **分布在两个地理位置** |
| 三层分工的实质 | **流层管"写下的不会变"，分区层管"谁是最新的与写入顺序"，FE 无状态** |
| 流层的 API 特征 | **所有写都是 append-only** |
| block 的读写与校验 | 读写最小单位；**读必须读整块**（校验和是 block 级的）；所有 block **每隔几天**做一次校验和验证 |
| extent 的参数 | **复制单位**，默认 **3 副本**，存在**一个 NTFS 文件**里，分区层目标大小 **1 GB** |
| 小对象 / TB 级对象分别怎么放 | 小的多个挤同一 extent 甚至同一 block；大的**拆到许多 extent** |
| 什么操作很快 | **用拼接现有 extent 构造新 stream**（只更新指针列表） |
| SM 是单点吗 | 不是 —— 它是**标准 Paxos 集群**，而且**在客户端关键路径之外** |
| SM 为什么不管 block | **block 总数可能极大，SM 无法扩展到跟踪它们** |
| 副本放置靠什么 | **在不同故障域里随机选 EN**，避免电力/网络/同机架的**相关失效** |
| 为什么这里不用 lease | **extent 未 seal 时 primary 与副本位置永不改变** —— 把可变性去掉就省掉了租约 |
| 多块追加的契约代价 | **客户端必须预期同一个 block 被追加多次并正确处理重复记录** |
| 重复记录怎么处理 | commit log 用**序列号**；行/blob 数据靠"**只有最后一次写被引用**"、旧的被回收 |
| 流层给分区层的两条保证 | ① 追加并确认后**任何副本读到相同数据**；② extent seal 后**任何 sealed 副本读到相同内容** |
| 这两条保证是"最新性"吗 | **不是** —— 全是**不可变性**。最新性与顺序由分区层负责 |
| 流层处理恶意对手吗 | **不处理**（交给数据中心 / Fabric Controller / WAS 的安全机制）；它处理的是位翻转、断电、软件 bug 等**导致数据损坏**的故障，用校验和检测 |
| Object Table 是什么 | 可增长到**数 PB** 的内部表，按流量切分成 RangePartition |
| RangePartition 的定义 | OT 中**从 low-key 到 high-key 的一段连续行**；同 OT 内互不重叠、覆盖每一行 |
| Blob/Entity/Message 表的主键 | **AccountName + PartitionName + ObjectName** |
| FE 靠什么路由 | **Partition Map Table** |
| 加 journaling 的效果 | commit log 的 append 延迟 **30 ms → 6 ms**，且方差显著下降 |

## 相关

- [[01-GFS|GFS]] —— 对照：**GFS 必须用租约选 primary**（chunk 会被反复修改），**WAS 不用**（extent 只追加，primary 固定）—— 可变性决定了要不要租约
- [[03-Dynamo|Dynamo]] / [[04-Spanner 与 F1|Spanner 与 F1]] —— 强一致与高可用的另外两条路线（放弃 / 用时钟设施买）
- [[02-Bigtable|Bigtable]] —— 同构的分层：那里是 tablet 建在 GFS 上，这里是 RangePartition 建在 stream/extent 上
- [[06-Aurora|Aurora]] —— 也提到"加一层日志"的收益；那篇的日志即数据库与本篇的 journaling 数字可以并读

## 参考

- Brad Calder, Ju Wang, Aaron Ogus, Niranjan Nilakantan, Arild Skjolsvold, Sam McKelvie, Yikang Xu, Shashwat Srivastava, Jiesheng Wu, Huseyin Simitci, Jaidev Haridas, Chakravarthy Uddaraju, Hemal Khatri, Andrew Edwards, Vaman Bedekar, Shane Mainali, Rafay Abbasi, Arpit Agarwal, Mian Fahim ul Haq, Muhammad Ikram ul Haq, Deepali Bhardwaj, Sowmya Dayanand, Anitha Adusumilli, Marvin McNett, Sriram Sankaran, Kavitha Manivannan, Leonidas Rigas. *Windows Azure Storage: A Highly Available Cloud Storage Service with Strong Consistency*. SOSP 2011.（stamp 构成与利用率目标、LS 与账号迁移、三层分工、流层的 block/extent/stream/seal 与 SM 的职责边界、复制流程与"为什么不用 lease"、multi-block append 的契约、流层给分区层的两条不可变性保证、Object Table 与 RangePartition、journaling 的延迟对比）
