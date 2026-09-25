---
tags:
  - 分布式/存储
---

# Spanner 与 F1

Spanner（OSDI 2012 / TOCS 2013）是 Google 的**全球分布数据库**。前面几篇里，[GFS](01-GFS.md) 与 [Bigtable](02-Bigtable.md) 放弃了跨行的强一致，[Dynamo](03-Dynamo.md) 干脆把一致性做成可配置项 —— Spanner 走的是第三条路：**先把"一台机器能做到的事"当成目标，再用一个物理设施把它在全局范围内重新做出来。**

这个定位的表述是：Spanner 是**第一个在全球规模上提供这些保证的系统**，而关键使能者是 **TrueTime API 及其实现**。

F1 是 Spanner 的落地案例 —— Google 广告后端的重写，从**手工分片多份的 MySQL** 迁过来。

## 一个物理前提：TrueTime

传统分布式系统里"时间不可靠"是公理，所有设计都围绕**绕开时钟**展开（逻辑时钟、向量时钟、不依赖时序的安全性）。Spanner 反过来做：**它把时钟不确定性变成一个可以被代码读到的数值，然后为这个数值付代价。**

### API 的形状

TrueTime 把时间显式表示为 **TTinterval** —— 一个**带界时间不确定性**的区间，这与"返回一个时间点、不告诉你不确定度"的标准时间接口是根本区别。区间的端点类型是 **TTstamp**。

| 方法 | 语义 |
| --- | --- |
| `TT.now()` | 返回一个 TTinterval，**保证包含 `TT.now()` 被调用期间的真实绝对时间** |
| `TT.after(t)` / `TT.before(t)` | 前者的便捷包装 |

形式化保证是：对一次调用 $tt = \text{TT.now}()$，有

$$tt.\text{earliest} \le t_{abs}(e_{now}) \le tt.\text{latest}$$

其中 $e_{now}$ 是这次调用事件。定义**瞬时误差界 $\epsilon$** 为区间宽度的一半，**平均误差界 $\bar{\epsilon}$**。时间纪元类比 UNIX 时间，但用**闰秒涂抹（leap-second smearing）**。

### 实现：两类时间源与它们的失效模式

**为什么同时用 GPS 和原子钟** —— 理由是**它们的失效模式不同**：

| 时间源 | 失效模式 |
| --- | --- |
| **GPS** | 天线与接收机故障、本地无线电干扰、**相关故障**（闰秒处理错误这类设计缺陷、以及**欺骗 spoofing**）、GPS 系统级中断 |
| **原子钟** | 失效方式**与 GPS 不相关、彼此也不相关**；但**长期会因频率误差显著漂移** |

两类一起用，才能让"同时坏掉"成为不可能。

具体实现是**每个数据中心一组 time master 机器 + 每台机器一个 timeslave daemon**：

- **多数 master 带 GPS 接收机与专用天线**，这些 master **在物理上分散**，以降低天线故障、无线电干扰与欺骗的影响；
- **其余 master 装原子钟，称 "Armageddon masters"** —— 成本说明也很实在：**原子钟并不那么贵，一个 Armageddon master 的成本与一个 GPS master 同一量级**；
- **所有 master 的时间参考定期互相比对**；每个 master 也**核对自己的参考推进时间的速率与本地时钟**，**出现显著分歧就自我驱逐**；
- 每个 daemon **轮询多个 master**（附近数据中心与远处数据中心的 GPS master，加一些 Armageddon master），用 **Marzullo 算法的一个变体**检测并**剔除说谎者（liars）**，再把本地时钟同步到非说谎者；
- **同步之间，Armageddon master 公布一个缓慢增长的不确定性**（由保守施加的最坏情况时钟漂移推出）；**GPS master 公布的不确定性通常接近 0**；
- 为防止本地时钟坏掉，**频率偏移超过"由组件规格与运行环境推出的最坏情况界"的机器会被驱逐**。

### ε 的实际数字：一条锯齿

这是全篇最有用的一组具体数字：

- **生产环境中 $\epsilon$ 是时间的锯齿函数，在每个轮询周期内从约 1 ms 变到 7 ms；$\bar{\epsilon}$ 大部分时间是 4 ms**；
- 分解：**daemon 的轮询间隔目前是 30 秒，当前使用的漂移率设为 200 微秒/秒** —— 这两者一起解释了**锯齿的 0 到 6 ms 边界**；**剩下的 1 ms 来自到 time master 的通信延迟**；
- 故障时会超出锯齿：**偶发的 time master 不可用会造成数据中心范围的 $\epsilon$ 增大**，过载的机器与网络链路会造成**偶发的局部尖峰**。

**这里有一句决定性的边界说明**：

> **方差不影响正确性，因为 Spanner 可以等掉不确定性；但 $\epsilon$ 变得太大时性能会退化。**

这就把 TrueTime 的两条设计约束讲清楚了：**保守地报告不确定性对正确性是必要的，把不确定性边界保持得小对性能是必要的。**

### 实测数据里的两个真实事件

一组**跨距离达 2200 km 的数据中心、几千台 spanserver** 的实测给出了 $\epsilon$ 的 90/99/99.9 分位。采样方式是**在 timeslave daemon 刚轮询完 time master 后**，因此**略去了本地时钟不确定性造成的锯齿**，测的是 **time master 不确定性（一般是 0）+ 到 time master 的通信延迟**。

结论是这两个因素**在决定 $\epsilon$ 的基值时一般不是问题**，但**尾部延迟会造成更高的 $\epsilon$**。两个真实事件：

- 3 月 30 日开始的**尾部延迟下降**，原因是**网络改进减少了瞬时的链路拥塞**；
- 4 月 13 日 $\epsilon$ 增大（**持续约一小时**），原因是**一个数据中心的两台 time master 为例行维护而关闭**。

## 外部一致性：两条规则与四步证明

这是整篇的核心。Spanner 要保证的是**外部一致性**（等价于线性一致性的分布式版本），它可以化归成两条规则加一个传递性证明。

先定义事件：设写事务 $T_i$ 的提交请求到达其协调者 leader 的事件为 $e^i_{server}$，该事务的提交事件为 $e^i_{commit}$，事务 $T_i$ 分配的提交时间戳为 $s_i$。

> **Start 规则**：写 $T_i$ 的协调者 leader 分配的提交时间戳 $s_i$ **不小于"在 $e^i_{server}$ 之后计算的 $\text{TT.now}().\text{latest}$"**。
>
> **Commit Wait 规则**：协调者 leader 确保**客户端在看到 $T_i$ 提交的任何数据之前，$\text{TT.after}(s_i)$ 已为真**。这保证 $s_i$ 小于 $T_i$ 的绝对提交时间：$s_i < t_{abs}(e^i_{commit})$。

要维持的不变式是：若 $t_{abs}(e^1_{commit}) < t_{abs}(e^2_{start})$，则 $s_1 < s_2$ —— 也就是"真实时间上先完成的事务，时间戳必须更小"。

**证明只有四步不等式链：**

$$
\begin{aligned}
s_1 &< t_{abs}(e^1_{commit}) && \text{（Commit Wait）} \\
t_{abs}(e^1_{commit}) &< t_{abs}(e^2_{start}) && \text{（前提假设）} \\
t_{abs}(e^2_{start}) &\le t_{abs}(e^2_{server}) && \text{（因果性）} \\
t_{abs}(e^2_{server}) &\le s_2 && \text{（Start 规则）} \\
\hline
s_1 &< s_2 && \text{（传递性）}
\end{aligned}
$$

**每一步都对应一个具体机制**：Commit Wait 把"时间戳"压到"提交时刻"之前；因果性保证请求到达不早于客户端发起；Start 规则把"请求到达时刻"压到"时间戳"之下。三段夹逼把两个时间戳的相对顺序和两个真实事件的相对顺序锁在一起。

**代价值得单独记住**：commit wait 里，协调者 leader 因为 $s$ 是基于 `TT.now().latest` 选的，而现在要等到那个时间戳保证成为过去，**所以期望的等待至少是 $2\epsilon$**。**这个等待通常与 Paxos 通信重叠** —— 这是它能被接受的原因。

## safe time：副本怎么判断自己够不够新

外部一致性给了"时间戳有意义"的保证，接下来要回答"某个副本能不能满足一次指定时间戳的读"。Spanner 的答案是 **safe time**：

> 每个副本跟踪 **$t_{safe}$ = 它已经是最新的最大时间戳**。**副本可以满足时间戳 $t$ 的读，当且仅当 $t \le t_{safe}$。**

$t_{safe}$ 取两者的较小值：

$$t_{safe} = \min(t^{Paxos}_{safe},\ t^{TM}_{safe})$$

- **$t^{Paxos}_{safe}$** 简单：**已应用的最高 Paxos 写的时间戳**。因为时间戳单调递增、写按序应用，所以在这个时间戳及以下不会再出现新写。
- **$t^{TM}_{safe}$** 处理"已 prepare 但未提交"这个不确定区间：若无此类事务，它是 $\infty$；若有，那些事务影响的状态是**不确定的** —— 参与者的副本还不知道它们会不会提交。
  公式是

$$t^{TM}_{safe} = \min_i(s^{prepare}_{i,g}) - 1$$

  （对所有在 group $g$ 上 prepare 的事务取最小的 prepare 时间戳再减 1。）**之所以能减 1，是因为协调者保证提交时间戳 $s_i \ge s^{prepare}_{i,g}$。** 这条不等式把"我不知道它会不会提交"转成了"它提交的话时间戳至少是这个值"。

## 事务的三种形态

### 读写事务：客户端驱动的两阶段提交 + commit-wait

- 与 Bigtable 一样，**事务内的写缓冲在客户端直到提交**；因此**事务内的读看不到本事务写的效果**（读返回所读数据的时间戳，而未提交的写还没有时间戳）。这个设计在 Spanner 里成立是因为**时间戳是读的一部分**；
- 事务内的读用 **wound-wait** 避免死锁；
- 客户端向相应 group 的 leader replica 发读，leader 获取读锁后读最新数据；事务开着期间客户端**发 keepalive** 防止参与者 leader 判它超时；
- **由客户端驱动两阶段提交** —— 理由很具体：**避免数据跨广域网链路传送两次**；
- **非协调者参与者 leader**：① 先获取写锁；② 选一个 **prepare 时间戳**，要求它**大于它此前为任何事务分配过的所有时间戳**（保持单调性）；③ 通过 Paxos 记录 prepare 记录；
- **协调者 leader**：也先获取写锁，但**跳过 prepare 阶段**；在听到所有其他参与者 leader 的回复后为**整个事务**选时间戳 $s$，三个约束是
  $$s \ge \text{所有 prepare 时间戳},\quad s > \text{收到 commit 消息时的 } \text{TT.now}().\text{latest},\quad s > \text{该 leader 此前分配过的所有时间戳}$$
- **只有 Paxos leader 获取锁**；锁状态**只在 prepare 时记日志**。若 prepare 前锁已丢失（死锁避免、超时、Paxos leader 变更），参与者中止；**leader 变更时新 leader 先恢复已 prepare 但未提交事务的锁状态，才接受新事务**；
- 最后：**协调者等到 `TT.after(s)`**（就是 Commit Wait），然后才允许任何协调者副本应用提交记录；之后把 $s$ 发给客户端与其他参与者 leader，各参与者通过 Paxos 记录结果，**所有参与者以相同时间戳应用，然后释放锁**。

### 快照事务（只读）：先做 scope 推断，再挑最小可用时间戳

- 分配时间戳需要涉及读的所有 Paxos group 之间**协商**，所以 Spanner 要求每个快照事务带一个 **scope 表达式** —— **概括整个事务将读的键**。对独立查询，Spanner **自动推断 scope**；
- 若 scope 的值由**单个** Paxos group 服务，客户端就把快照事务发给该 group 的 leader（**当前实现只在 Paxos leader 上为快照事务选时间戳**）；
- **单点读有一个比 `TT.now().latest` 更好的选择**：定义 $\text{LastTS}()$ 为**某 Paxos group 最后一次已提交写的时间戳**。**若没有 prepared 事务，则赋值 $s_{read} = \text{LastTS}()$ 显然满足外部一致性** —— 事务会看到最后一次写的结果，因此排在它之后。**这样做还避免了读被 $t_{safe}$ 卡住**（而选 `TT.now().latest` 则可能需要阻塞等待 $t_{safe}$ 前进）；
- scope 跨多个 group 时，最复杂的方案是与所有 leader 协商 $s_{read}$；**Spanner 当前实现了一个更简单的选择**（避免协商轮次）。

## 目录：复制与数据移动的单位

- **universe** = 一个 Spanner 部署（全局只有少数几个）；
- **zone** ≈ 一个 Bigtable 服务器部署，是**管理部署的单位**、**数据可被复制到的位置集合**、也是**物理隔离的单位**（一个数据中心里可能有多个 zone）；
- 一个 zone 有 **1 个 zonemaster** 和 **100 到几千个 spanserver**；**per-zone location proxy** 让 client 定位到服务其数据的 spanserver；
- **universe master** 是单例，主要是显示状态的控制台；**placement driver** 也是单例，负责**分钟级**的数据跨 zone 自动移动；
- **每个 spanserver 负责 100 到 1000 个 tablet**，tablet 实现 `(key:string, timestamp:int64) → string`。**与 Bigtable 不同，Spanner 给数据分配时间戳** —— 这是它更像**多版本数据库**而不是 KV 的地方；
- tablet 状态存在**一组 B 树状文件 + 一个预写日志**里，都放在 **Colossus**（GFS 的继任者）上；
- **每个 spanserver 在每个 tablet 上实现一个 Paxos 状态机**。有一条演进：**早期 Spanner 支持一个 tablet 多个 Paxos 状态机**（可以让复制配置有更多变化），**但因为复杂而放弃了**。

## F1：落地案例

- Spanner 从 **2011 年初**开始在**生产负载**下被实验性评估，作为 Google **广告后端 F1** 重写的一部分；
- 这个后端**原本基于手工分片多份的 MySQL**；
- **未压缩数据集有数十 TB** —— "与许多 NoSQL 实例相比不大，但**大到足以让分片 MySQL 出现困难**"。

这条对照对理解 Spanner 的定位有用：它的目标**是在一个中等规模、但需要强事务语义的数据集上，把手工分片的运维负担消掉**，而不是"比 NoSQL 更能装"。

## 判据速查

| 问题 | 答案 |
| --- | --- |
| TrueTime 与传统时间接口的根本区别 | 它**显式返回一个带界不确定性区间**，而不是一个时间点 |
| 为什么同时用 GPS 和原子钟 | **两者失效模式不相关**：GPS 有欺骗/干扰/闰秒处理错误，原子钟会长期漂移 |
| Armageddon master 是什么 | 装原子钟的 time master；成本与 GPS master **同一量级** |
| daemon 怎么防说谎的 master | 轮询多个 master，用 **Marzullo 算法的变体**剔除说谎者 |
| $\epsilon$ 的实际值 | 锯齿，**每个轮询周期 1–7 ms**，$\bar{\epsilon}$ 大部分时间 **4 ms** |
| 锯齿怎么来的 | **30 秒轮询 + 200 µs/s 漂移率 → 0–6 ms；通信延迟贡献 1 ms** |
| 不确定性大了会怎样 | **正确性不受影响**（可以等），**性能退化** |
| 外部一致性靠哪两条规则 | **Start**（时间戳 ≥ 请求到达后算的 `TT.now().latest`）+ **Commit Wait**（客户端看到数据前 `TT.after(s)` 已为真） |
| Commit Wait 的证明作用 | 它给出 $s_1 < t_{abs}(e^1_{commit})$，与 Start 的 $t_{abs}(e^2_{server}) \le s_2$ 夹逼出 $s_1 < s_2$ |
| Commit Wait 的代价 | **期望等待至少 $2\epsilon$**，但**通常与 Paxos 通信重叠** |
| 副本怎么判断能否满足某个时间戳的读 | $t \le t_{safe}$，而 $t_{safe} = \min(t^{Paxos}_{safe}, t^{TM}_{safe})$ |
| $t^{TM}_{safe}$ 为什么减 1 | 因为协调者保证 $s_i \ge s^{prepare}_{i,g}$，所以 pending 事务的时间戳至少是 prepare 时间戳 |
| 谁驱动两阶段提交 | **客户端** —— 避免数据跨广域网链路传送两次 |
| 协调者 leader 为什么可以跳过 prepare | 它是协调者，只需在听到所有参与者回复后为整个事务选时间戳 |
| 事务内的读能看到本事务的写吗 | **不能** —— 写缓存在客户端，未提交的写还没有时间戳 |
| 快照事务为什么要 scope 表达式 | 分配时间戳需要跨 Paxos group 协商，scope **概括将读的键**，从而知道要找哪些 leader |
| 单点读为什么用 `LastTS()` 而不用 `TT.now().latest` | 前者**显然满足外部一致性**，且**避免因 $t_{safe}$ 未前进而阻塞** |
| Spanner 与 Bigtable 的关键差别 | **Spanner 给数据分配时间戳**，因此更像多版本数据库 |
| 复制单位是什么 | **每个 tablet 上一个 Paxos 状态机**；早期一个 tablet 多个 Paxos 状态机**因复杂被放弃** |
| F1 的原始架构与规模 | **手工分片多份的 MySQL**；未压缩数据**数十 TB** |

## 相关

- [[01-GFS|GFS]] / [[02-Bigtable|Bigtable]] —— Spanner 的 tablet 状态存在 **Colossus**（GFS 继任者）上，Paxos 状态机叠在 Bigtable 式的 tablet 抽象上；本层补上了那两层缺的跨行事务
- [[03-Dynamo|Dynamo]] —— 同一问题的另一条路线：Dynamo 用可配置弱一致换可用性，Spanner 用**物理时钟设施**把强一致拿回来
- [[01-不可能性结果：FLP 与部分同步|FLP 与部分同步]] —— TrueTime 本质上把系统**放进了部分同步模型**（有界时钟误差 = 有界不确定性），于是 FLP 的不可能性不再适用；代价是依赖 GPS 与原子钟这套物理设施
- [[04-Zab 与 ZooKeeper|Zab 与 ZooKeeper]] —— 另一种"用时间换一致"的思路：Zab 用 epoch 做逻辑时钟，Spanner 用 TrueTime 做物理时钟

## 参考

- J. C. Corbett, J. Dean, M. Epstein, A. Fikes, C. Frost, J. J. Furman, S. Ghemawat, A. Gubarev, C. Heiser, P. Hochschild, W. Hsieh, S. Kanthak, E. Kogan, H. Li, A. Lloyd, S. Melnik, D. Mwaura, D. Nagle, S. Quinlan, R. Rao, L. Rolig, Y. Saito, M. Szymaniak, C. Taylor, R. Wang, D. Woodford. *Spanner: Google's Globally-Distributed Database*. OSDI 2012（TOCS 31(3), 2013）.
- J. Shute, M. Oancea, S. Ellner, B. Handy, E. Rollins, B. Samwel, R. Vingralek, C. Whipkey, X. Chen, B. Jegerlehner, K. Littlefield, P. Tong. *F1: A Distributed SQL Database That Scales*. VLDB 2012.
