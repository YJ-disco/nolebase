---
tags:
  - 分布式/存储
---

# Bigtable

Bigtable（OSDI 2006）是 Google 的**结构化数据存储**，建在 [[01-GFS|GFS]] 之上。它被定位成"能可靠扩展到 PB 级数据与数千台机器"的系统，上线时已被 **60 多个 Google 产品**使用（Analytics、Finance、Orkut、Personalized Search、Writely、Google Earth），集群规模从几台到数千台服务器、存到几百 TB。

它在设计上有一个很明确的态度：**不追求完整的关系模型**。论文的说法是——Bigtable 提供的是一个**简单的数据模型**，它支持**对数据布局与格式的动态控制**，并让客户端能**推理数据的局部性**。

## 数据模型：一个三维有序 map

Bigtable 的定义只有一句：

> 它是一个**稀疏的、分布式的、持久的多维有序 map**，索引是 **(row key, column key, timestamp)**，每个值是**未解释的字节数组**。

$$(\text{row:string},\ \text{column:string},\ \text{time:int64}) \to \text{string}$$

论文给的例子最能说明意图：存网页的表，**row name 是反转的 URL**（`com.google.www` 这种，好让同一域名下的页面聚在一起），`contents` column family 存页面内容，`anchor` column family 存引用该页面的锚文本。同一行的不同 column 由冒号分隔的 `family:qualifier` 指定。

三个设计后果值得单独指出：

- **不支持完整的关系数据模型**，也没有跨行事务（只有**单行事务**）；
- **数据当未解释的字符串处理** —— 客户端自己把结构化/半结构化数据序列化进去，Bigtable 不管语义；
- **客户端可以通过 schema 的精心选择控制数据局部性**，schema 参数还能**动态控制数据是从内存服务还是从磁盘服务**。

## 两个构件：SSTable 与 Chubby

Bigtable 的实现只依赖两个外部构件。

### SSTable

**SSTable** 是 Google 内部的存储文件格式：一个**持久的、有序的、不可变的** key→value map，key 与 value 都是任意字节串。支持两个操作：查指定 key 的值、在指定 key 范围内迭代。

结构上：

- 内含**一系列 block**，**典型每块 64 KB**（可配置）；
- **block index 存在文件末尾**，打开 SSTable 时加载进内存；
- 因此**一次查寻只需一次磁盘寻道** —— 先在内存索引里二分定位 block，再从盘读那一块；
- **可选地把整个 SSTable 映射进内存**，于是查寻与扫描完全不碰盘。

### Chubby

Chubby 是**高可用、持久的分布式锁服务**（5 个活跃副本，一个选为 master 主动服务；**多数副本运行且能互通时服务才活着**；**Chubby 内部用 Paxos 保持副本一致**）。它提供目录与小文件的命名空间，**每个文件或目录都能当锁用**，读写原子；客户端与服务之间维持**会话**，会话靠租约续期，**续不上就过期并丢失所有锁与打开的句柄**；客户端还能在文件/目录上注册**变更或会话过期的回调**。

**Bigtable 用 Chubby 做了五件事**，这五件事解释了它为什么离不开 Chubby：

1. 确保**至多一个活跃 master**；
2. 存 Bigtable 数据的**引导位置（bootstrap location）**；
3. **发现 tablet server，并判定 tablet server 的死亡**；
4. 存 **schema 信息**（每张表的 column family 信息）；
5. 存**访问控制列表**。

**代价被量化了**：Chubby 长时间不可用会让 Bigtable 不可用。论文在跨 11 个 Chubby 实例的 14 个 Bigtable 集群上测量，**因 Chubby 不可用导致数据不可用的平均时间占比是 0.0047%**；受影响最大的单集群是 **0.0326%**。

## 组件与职责划分

三个组件：**链接进每个 client 的库**、**一个 master**、**多个 tablet server**。

| 角色 | 职责 |
| --- | --- |
| **master** | 分配 tablet 给 tablet server、**检测 tablet server 的加入与过期**、平衡负载、**垃圾回收 GFS 里的文件**、处理 schema 变更（建表 / 建 column family） |
| **tablet server** | 管理一组 tablet（**典型 10 到 1000 个**）、处理读写、**切分过大的 tablet** |
| **client 库** | 直接与 tablet server 通信 |

一条与 GFS 同构的分工：**client 数据不经过 master**，client 直接与 tablet server 读写。**因为 client 不依赖 master 获取 tablet 位置信息，多数 client 从不与 master 通信** —— 论文的对应用词是"master 在实践中负载很轻"。

## 一个 tablet 内部：commit log + memtable + SSTables

tablet 的状态由**三层**构成：

- **commit log**：存 redo 记录，更新先提交到这里；
- **memtable**：**最近提交**的更新存在内存里的一个**有序缓冲**；
- **一系列 SSTable**：较旧的更新。

**恢复路径**：tablet server 从 METADATA 表读出该 tablet 的元数据（含**组成它的 SSTable 列表**与一组 **redo points** —— 指向可能含该 tablet 数据的 commit log 的指针）；把 SSTable 的索引读进内存，然后**通过重放自 redo points 以来提交的所有更新来重建 memtable**。

**写路径**三步：① 检查格式良好、发送者有权限（权限从 Chubby 文件读允许写者列表，**几乎总是 Chubby 客户端缓存命中**）；② 合法 mutation 写入 commit log，**用 group commit 提升大量小 mutation 的吞吐**；③ 提交后内容插入 memtable。

**读路径**：同样先做格式与权限检查，然后在 **SSTable 序列与 memtable 的合并视图**上执行 —— 两者都是字典序有序结构，所以合并视图可以用一次归并得到。

**读写可以在 tablet 切分与合并进行时继续。**

## 三档 compaction

memtable 会一直增长，所以有一整套后台整理机制：

| 档 | 触发与动作 | 目标 |
| --- | --- | --- |
| **minor compaction** | memtable 达到阈值 → **冻结**它、建新 memtable、把冻结的转成 SSTable 写入 GFS | ① 降低 tablet server 内存占用；② **减少 server 挂掉时从 commit log 恢复要读的数据量** |
| **merging compaction** | 后台定期把若干 SSTable 与 memtable 合成一个新 SSTable，输入完成后即可丢弃 | **限制 SSTable 的数量** —— 否则读操作可能要在任意多个 SSTable 上合并更新 |
| **major compaction** | 把所有 SSTable 重写成**恰好一个** | 回收已删数据占用的资源，并保证**已删数据及时消失** |

**后两档的关键区别在于删除**：非 major compaction 产出的 SSTable 里可能含**特殊删除条目**，用来抑制仍存活的旧 SSTable 里的已删数据；而 **major compaction 产出的 SSTable 不含任何删除信息或已删数据**。Bigtable **轮转所有 tablet 并定期对它们做 major compaction** —— 论文特意指出这条对**存敏感数据的服务**很重要。

## 五处性能改进

论文的"refinements"一节是整篇最实用的部分。

### 局部性组（locality group）

客户端可以把多个 column family **分组成一个 locality group**，**每个 tablet 为每个 locality group 生成一个独立的 SSTable**。把通常**不一起访问**的 family 隔开，读就更省 —— 论文的例子：Webtable 里页面**元数据**（语言、校验和）一组，**页面内容**另一组，只想读元数据的应用**不必读穿全部页面内容**。

locality group 还能声明为 **in-memory**：它的 SSTable 被**惰性加载**进 tablet server 内存，加载后读该组的列**完全不碰盘**。论文说这个特性用于"小而频繁访问的数据"，**内部拿它放 METADATA 表的 location column family**。

### 压缩：两遍方案与实测压缩率

压缩是**按 SSTable block 为单位**做的（block 大小也由 locality group 参数控制）。**代价是牺牲一点空间，收益是可以只解压一小块而不必解压整个文件。**

很多客户端用**两遍自定义压缩**：

1. 第一遍用 **Bentley-McIlroy** 方案，在**大窗口内压缩长的公共字符串**；
2. 第二遍用快速算法，在 **16 KB 小窗口**里找重复。

**速度数字**（论文给的）：两遍都很快，**编码 100–200 MB/s，解码 400–1000 MB/s**。

论文承认选算法时**重速度而轻压缩率**，但结果超出预期：Webtable 里存网页内容的实验达到 **10:1 的空间压缩**，**远好于 HTML 页面上 Gzip 典型的 3:1 到 4:1**。原因在于**行的布局**——同一主机的所有页面存在相邻位置，Bentley-McIlroy 因此能识别出同主机页面之间大量的共享样板。

论文由此给出一条可迁移的经验：**很多应用都把 row name 设计成让相似数据聚在一起，因此能拿到很好的压缩率**；而存同一值的多个版本时压缩率还会更好。

### 两级缓存

| 缓存 | 缓存什么 | 对哪类负载有用 |
| --- | --- | --- |
| **Scan Cache**（上层） | SSTable 接口返回给 tablet server 代码的**键值对** | **反复读同一数据** |
| **Block Cache**（下层） | 从 GFS 读来的 **SSTable block** | **读与刚读过的数据相邻**（顺序读，或热行内同一 locality group 的不同列的随机读） |

### Bloom filter

一次读要读**组成该 tablet 状态的全部 SSTable**，它们不在内存时会造成大量磁盘访问。客户端可以指定对某个 locality group 的 SSTable 建 **Bloom filter**，用来问"该 SSTable **是否可能**含有指定 row/column 对的数据"。

效果是两条：**用少量 tablet server 内存大幅减少读的磁盘寻道**；以及**对不存在行/列的查询大多完全不必碰盘**。

### commit log 的合流与排序恢复

这一条最能体现"工程取舍是多米诺式的"。

**先看问题**：如果每个 tablet 一个独立日志文件，GFS 里会**并发写极大量文件**，可能造成大量磁盘寻道；而且**日志独立会削弱 group commit**（组会更小）。

**修法**：**每个 tablet server 只追加到一个 commit log**，把不同 tablet 的 mutation **混在同一个物理日志文件里**。

**于是恢复变复杂**：tablet server 挂掉后，它服务的 tablet 会被迁到很多其他 server 上，每个新 server 要从原 server 的日志里**重放属于自己那个 tablet 的 mutation**，而这些 mutation 是混在一起的。

**朴素解法的代价可以算**：如果 100 台机器各分到一个 tablet，那个日志文件会被**读 100 遍**。

**解法**：先把 commit log 条目按

$$\langle \text{table},\ \text{row name},\ \text{log sequence number} \rangle$$

**排序**。排序输出里，**某个 tablet 的所有 mutation 变成连续的**，于是**一次寻道加一次顺序读**就能读完。为了并行化，**日志文件被切成 64 MB 段，在不同 tablet server 上并行排序**；整个过程由 **master 协调**，在某个 tablet server 表示需要从 commit log 恢复时发起。

## 判据速查

| 问题 | 答案 |
| --- | --- |
| 数据模型一句话 | 稀疏、分布式、持久的**多维有序 map**，$(row, column, timestamp) \to bytes$ |
| 支持跨行事务吗 | **不支持**，只有单行事务 |
| SSTable 是什么 | 持久的、**有序的、不可变的** key→value map，内部是一串 block（典型 64 KB） |
| SSTable 一次查寻几次寻道 | **一次**（内存里的 block index 二分 + 读一个 block） |
| Bigtable 对 Chubby 的依赖有几处 | **五处**：唯一 master、bootstrap 位置、tablet server 发现与死亡判定、schema、ACL |
| Chubby 不可用的影响有多大 | 14 个集群实测：数据不可用时间占比平均 **0.0047%**，最差单集群 **0.0326%** |
| client 会打扰 master 吗 | **一般不会** —— client 不依赖 master 拿 tablet 位置，多数 client 从不联系 master |
| memtable 是什么 | 存**最近提交**更新的内存有序缓冲 |
| 恢复怎么重建 memtable | 按 METADATA 里的 **redo points**，重放自那以来提交的所有更新 |
| minor / merging / major compaction 的目标 | 降内存、限 SSTable 数量、**彻底清掉已删数据** |
| 为什么 non-major compaction 会留删除条目 | 用它抑制仍存活的旧 SSTable 里的已删数据；只有 major compaction 才真正清干净 |
| locality group 解决什么 | 把不一起访问的 column family 隔到不同 SSTable；还可标为 **in-memory** |
| 压缩为什么要按 block 做 | 牺牲一点压缩率，换"只解压一小块"的能力 |
| 实测压缩率 | Webtable 页面内容 **10:1**（同主机页面相邻，样板被识别出来） |
| 两级缓存各管什么 | Scan Cache 管"重复读同一数据"，Block Cache 管"读相邻数据" |
| commit log 为什么合流 | 避免 GFS 里海量并发小文件写入，并让 group commit 的组更大 |
| 合流后恢复怎么不变慢 | 按 $\langle table, row, log\ seq \rangle$ 排序 → 每个 tablet 的 mutation 连续；切成 64 MB 段并行排序 |

## 相关

- [[01-GFS|GFS]] —— Bigtable 的底层存储：SSTable 与 commit log 都写在 GFS 上
- [[04-Zab 与 ZooKeeper|Zab 与 ZooKeeper]] —— Chubby 的同类系统；Bigtable 对 Chubby 的五处依赖说明"协调服务"在这一代系统里的位置
- [[03-一致性模型]] —— Bigtable 只给单行事务，跨行一致性的缺失是上层（如 MegaStore）要补的

## 参考

- Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes, Robert E. Gruber. *Bigtable: A Distributed Storage System for Structured Data*. OSDI 2006.（数据模型、SSTable、Chubby 的五处依赖与可用性实测、三档 compaction、locality group / 压缩 / 两级缓存 / Bloom filter / commit log 排序恢复）
