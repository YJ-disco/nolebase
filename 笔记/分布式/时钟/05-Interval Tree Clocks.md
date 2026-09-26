---
tags:
  - 分布式/时钟
---

# Interval Tree Clocks

[[03-逻辑时钟：Lamport 与向量|Vector 向量时钟]] 的表示是"一个全局 id 到一个整数计数器"的映射：每个实体记下它知道的、每个其他实体的事件数。这要求 **id 集合是固定的**，或者至少要求 id 的唯一性与退休能被全局协调。在动态系统里这两条都不成立。

`Interval Tree Clocks`（ITC）是 Almeida、Baquero、Fonte 在 *Interval Tree Clocks: A Logical Clock for Dynamic Systems* 里给出的机制。它的定位是**泛化版本向量与向量时钟**，同时去掉全局 id 的前提：任意实体可以自主 fork 出一个新实体，任意两个实体可以 join 成一个，不需要全局协调。

## 动态系统里向量时钟坏在哪

先前方案的失效点列得很具体：

| 方案的问题 | 具体表现 |
| --- | --- |
| id 需要全局唯一 | 通常靠 MAC 地址、外部 id 服务这类外部来源 |
| 退休 id 需要全局协调 | 必须所有活跃实体都确认"我见过这个 id 的终止"才能安全移除；**一个不可达实体就能让垃圾回收永远挂住** |
| 元数据无法释放 | 回收不了 id，随 id 增长的元数据也释放不了；当不可达概率高时，"不回收"反而更划算 |
| 局部退休只支持受限形态 | 有方案只允许"由直接祖先 join 掉"这一种终止模式 |

一个对照是 **Dynamo** 的做法：Dynamo 为控制版本向量增长，会把不活跃的旧条目激进剪枝。该工程的做法是"参数调过、生产环境出错概率很低"，但一般意义上这会导致旧更新**复活**（resurgence of old updates）。ITC 想避免的正是"必须靠剪枝才能控制向量长度"这个处境。

## fork-event-join 模型

所有因果跟踪机制可以抽成三个核心操作，作用在 stamp 上。stamp 是一个二元组 $(i, e)$：$i$ 是 id，$e$ 是**事件分量**（编码"因果上已知的事件"）。事件分量之间的偏序记作 $(E, \sqsubseteq)$ —— 版本向量里它是逐分量比较 $e \sqsubseteq e' \iff \forall k: e[k] \le e'[k]$；因果历史（causal histories）里它是集合包含。

**fork**：克隆一个 stamp 的因果过去，产出两个事件分量相同、id 不同的 stamp。

$$\mathrm{fork}(i, e) = ((i_1, e), (i_2, e)), \quad i_1 \ne i_2$$

**peek**：fork 的特例 —— 只需要一份匿名 stamp $(0, e)$，可以用来传递因果信息但不能登记事件。

$$\mathrm{peek}((i, e)) = ((0, e), (i, e))$$

**event**：往事件分量里加一个新事件，使它**严格推进**偏序：对系统中任何其他事件分量 $x$，$e' \not\sqsubseteq x$；且当 $x \sqsubset e'$ 时必有 $x \sqsubseteq e$ ——"推进刚好够多"。版本向量里的实现是给自己的计数器加一。

**join**：合并两个 stamp，要求结果是两者的**序论上确界**：

$$e_3 = e_1 \sqcup e_2, \quad 且\ e_1 \sqsubseteq e_3,\ e_2 \sqsubseteq e_3$$

这要求偏序构成一个**交半格（join semilattice）**——所有对都有上确界。版本向量里它是逐分量取最大；因果历史里是集合并。

三个操作能拼出经典操作：

| 经典操作 | 分解 | 对应到版本向量的动作 |
| --- | --- | --- |
| send | event + peek | 推进本地计数器，再生成一份匿名副本随消息发出 |
| receive | join + event | 先逐分量取最大，再给自己加一 |
| sync | join + fork | 版本向量系统的同步 |

### 框架再看一层：id 固定在哪里

这件事可以抽象成一个**函数空间**框架：因果跟踪机制可以用"定义在某个域上的函数"来刻画，机制之间的差别落在两处 ——

| | 经典机制 | ITC |
| --- | --- | --- |
| id 的性质 | 每个参与者用**固定、预定义**的函数 | id 分量本身**被操纵**，用来适应动态的参与者数量 |
| 定义域 | 离散、通常有限 | **连续无限域** $\mathbb{R}$，重心在区间 $[0, 1)$，可任意细分 |

这条对照解释了后面所有设计差异：**ITC 之所以能自治 fork/join，是因为它把 id 从"编号"变成了"可切分的资源"。**

## ITC 的结构：两个函数、两棵树

**单位脉冲函数**先定义出来：

$$\mathbf{1} := \lambda x.\ \begin{cases} 1 & 0 \le x < 1 \\ 0 & x < 0 \ \vee\ x \ge 1 \end{cases}$$

### id 树

$$i ::= 0 \mid 1 \mid (i_1, i_2)$$

语义把 id 树解释成 $[0,1)$ 上的函数：

$$\llbracket 0 \rrbracket = 0, \qquad \llbracket 1 \rrbracket = \mathbf{1}, \qquad \llbracket (i_1, i_2) \rrbracket = \lambda x.\ \llbracket i_1 \rrbracket(2x) + \llbracket i_2 \rrbracket(2x - 1)$$

$(i_1, i_2)$ 里的两个子树被变换到**两个不相交的子区间**：$i_1$ 落在 $[0, 1/2)$，$i_2$ 落在 $[1/2, 1)$。函数取 1 的那段区间就是这个实体"拥有"的身份区间。

举例：$(1, (0, 1))$ 表示函数 $\lambda x.\ \mathbf{1}(2x) + (\lambda x.\ \mathbf{1}(2x-1))(2x-1)$。

id 树在 $[0,1)$ 上的含义（树 ↔ 区间）：

```
   1              [────────── 整段 [0,1) ──────────]
   (1, 0)         [──── 左半 ────][      空       ]
   (0, 1)         [      空      ][──── 右半 ────]
   (i₁, i₂)       [── i₁ 的区间 ──][── i₂ 的区间 ──]      两个子树各占不相交的一半

   换成树来看，区间是从根往下切出来的：

       1                     (i₁, i₂)
       │                     ╱       ╲
     [0,1)                 i₁          i₂
                        [0, ½)      [½, 1)

   split 的四条分支：
     split(1)         = ((1,0), (0,1))      把一个节点劈成左右两份
     split((0, i))    = ((0,i₁), (0,i₂))    往非空的那一侧递归
     split((i, 0))    = ((i₁,0), (i₂,0))
     split((i₁, i₂))  = ((i₁,0), (0,i₂))    ← 不新增区间：已有的两个子树各拿一份

   ⇒ 新 id 完全是本地产物 —— 不需要全局唯一编号，也不需要外部 id 服务。
```

### 事件树

$$e ::= n \mid (n, e_1, e_2)$$

$$\llbracket n \rrbracket = n \cdot \mathbf{1}, \qquad \llbracket (n, e_1, e_2) \rrbracket = n \cdot \mathbf{1} + \lambda x.\ \llbracket e_1 \rrbracket(2x) + \llbracket e_2 \rrbracket(2x - 1)$$

某个子区间上的取值 = **整个区间共有的基值** $n$，加上对应子树的相对值。"共有基值"这一层让 event 可以在不展开整棵树的前提下推进。

一个 **stamp** 是 $(i, e)$ 这一对；初始态叫 **seed stamp** $(1, 0)$，从它可以 fork 出任意初始配置。

## 归一化形式：同一函数有多种树

一个函数可以对应多棵等价的树。例如单位脉冲：

$$\mathbf{1} \equiv (1, 1) \equiv (1, (1, 1)) \equiv ((1, 1), 1) \equiv \cdots$$

ITC 的设计要求 stamp **始终保持在归一化形式（normal form）** —— 这既是为表示紧凑，也是为了让四个操作的定义可以写成简单的递归（否则每次操作都要先判断是否等价）。

### id 的归一化

$$\mathrm{norm}((0,0)) = 0, \qquad \mathrm{norm}((1,1)) = 1, \qquad \mathrm{norm}(i) = i$$

含义直接：两个子树都是 0（谁都不占）就等于 0；两个子树都是 1（合起来覆盖整个区间）就等于 1。

### event 的归一化：lift 与 sink

先定义两个"抬升/下沉"算子（$m, n$ 是整数，$e_1, e_2$ 是归一化事件树）：

$$n \uparrow m = n + m, \qquad (n, e_1, e_2) \uparrow m = (n + m, e_1, e_2)$$

$$n \downarrow m = n - m, \qquad (n, e_1, e_2) \downarrow m = (n - m, e_1, e_2)$$

即只动节点上的基值，不动子树结构。归一化函数是：

$$\mathrm{norm}(n) = n$$

$$\mathrm{norm}((n, m, m)) = n + m$$

$$\mathrm{norm}((n, e_1, e_2)) = (n + m,\ e_1 \downarrow m,\ e_2 \downarrow m), \quad m = \min(\min(e_1), \min(e_2))$$

其中 $\min(e) = \min_{x \in [0,1)} \llbracket e \rrbracket(x)$，递归实现是 $\min(n) = n$、$\min((n, e_1, e_2)) = n + \min(\min(e_1), \min(e_2))$。**对归一化事件树有一个简化**：其中一个子树的 min 必定是 0，所以 $\min((n, e_1, e_2)) = n$，不用往下走。

两个例子：

$$(2, 1, 1) \equiv 3, \qquad (2, (2, 1, 0), 3) \equiv (4, (0, 1, 0), 1)$$

同理还有 $\max$：$\max(n) = n$、$\max((n, e_1, e_2)) = n + \max(\max(e_1), \max(e_2))$。

## 四个操作的精确定义

下面每个函数都以归一化 stamp 为输入和输出。

### 比较

比较直接由对应函数的逐点比较导出：

$$(i_1, e_1) \sqsubseteq (i_2, e_2) \ :=\ \llbracket e_1 \rrbracket \sqsubseteq \llbracket e_2 \rrbracket$$

注意 **id 不参与比较** —— 比较只看事件函数。它可以在归一化事件树上递归地算，记作 `leq(e1, e2)`（$l$、$r$ 代表左、右子树）：

```
leq(n1, n2)                   = n1 ⊑ n2
leq(n1, (n2, l2, r2))         = n1 ⊑ n2
leq((n1,l1,r1), n2)           = n1 ⊑ n2 ∧ leq(l1 ↑ n1, n2) ∧ leq(r1 ↑ n1, n2)
leq((n1,l1,r1), (n2,l2,r2))   = n1 ⊑ n2 ∧ leq(l1 ↑ n1, l2 ↑ n2) ∧ leq(r1 ↑ n1, r2 ↑ n2)
```

递归里出现的 $\uparrow$ 是因为要减掉当前节点的基值，把子树放回同一基准上比较。

### fork：切分 id

fork 保持事件分量不变，把 id 切成两块。切分函数要满足

$$(i_1, i_2) = \mathrm{split}(i) \ \Rightarrow\ \llbracket i_1 \rrbracket \cdot \llbracket i_2 \rrbracket = 0 \ \wedge\ \llbracket i_1 \rrbracket + \llbracket i_2 \rrbracket = \llbracket i \rrbracket$$

即两块**不重叠**、且加起来还原原来那块。id 树的两个子树天然满足这两条，所以递归很直接：

```
split(0)         = (0, 0)
split(1)         = ((1, 0), (0, 1))
split((0, i))    = ((0, i1), (0, i2))    where (i1, i2) = split(i)
split((i, 0))    = ((i1, 0), (i2, 0))    where (i1, i2) = split(i)
split((i1, i2))  = ((i1, 0), (0, i2))
```

最后一条是**不新增区间的分支**：已经在树里的两个子树各自作为一份，直接分给两个实体。

### join：id 相加、事件取上确界

$$\mathrm{join}((i_1, e_1),\ (i_2, e_2)) := (\mathrm{sum}(i_1, i_2),\ \mathrm{join}(e_1, e_2))$$

要求 $\llbracket \mathrm{sum}(i_1, i_2) \rrbracket = \llbracket i_1 \rrbracket + \llbracket i_2 \rrbracket$、$\llbracket \mathrm{join}(e_1, e_2) \rrbracket = \llbracket e_1 \rrbracket \sqcup \llbracket e_2 \rrbracket$。

`sum` 递归（每层调 `norm` 保持归一化）：

```
sum(0, i) = i
sum(i, 0) = i
sum((l1,r1), (l2,r2)) = norm((sum(l1,l2), sum(r1,r2)))
```

事件树的 `join` 更绕，因为要处理"一棵树有子树、另一棵只是整数"：

```
join(n1, n2)                 = max(n1, n2)
join(n1, (n2,l2,r2))         = join((n1,0,0), (n2,l2,r2))
join((n1,l1,r1), n2)         = join((n1,l1,r1), (n2,0,0))
join((n1,l1,r1), (n2,l2,r2)) = join((n2,l2,r2), (n1,l1,r1))                     if n1 > n2
join((n1,l1,r1), (n2,l2,r2)) = norm((n1, join(l1, l2 ↓ (n2−n1)), join(r1, r2 ↓ (n2−n1))))
```

最后一条把基值差用 $\downarrow$ 抹平，再递归合并左右子树 —— 这保证了结果在归一化形式上。

### event：在可用区间上抬升，并尽量化简

event 的定义给的是一组**约束**，没有指定唯一实现：

$$\mathrm{event}((i, e)) = (i, e'), \quad \text{subject to}\ \llbracket e' \rrbracket = \llbracket e \rrbracket + f \cdot \llbracket i \rrbracket \ \ \text{对任意使}\ f \cdot \llbracket i \rrbracket > 0 \ \text{的}\ f$$

读法是：**只能在 id 覆盖的区间上抬升事件值**，抬升多少有自由度。前置条件是 $i \ne 0$ —— **匿名 stamp 不能登记事件**（因为匿名 id 对应的函数处处为 0，没有可抬升的区间）。

ITC 在实现里把这个自由度用来**化简事件树**。做法是先用 `fill` 尝试所有"给定 id 树就能做的化简"：

```
fill(i, e)   # 对给定 id 树做一次或多次化简，返回化简后的树
```

若 `fill` 没能改动树，就退回 `grow`，"长"某个子树 —— **优先只把一个整数加一**。于是整体是：

$$\mathrm{event}(i, e) = \begin{cases} (i,\ \mathrm{fill}(i, e)) & \mathrm{fill}(i, e) \ne e \\ (i, e') & \text{否则}\ (e', c) = \mathrm{grow}(i, e) \end{cases}$$

`fill` 有一条重要性质：**它不会去递增一个"递加了也不能化简树"的整数**。

## 一次完整运行：stamp 怎么长大又怎么缩回

用最大的那个例子说明 ITC 自适应的过程：

1. 单个参与者，持有 seed stamp；
2. fork 成两个；其中一个发生一次 event 后再 fork，另一个发生两次 event —— 此时三个参与者；
3. **前两次 fork 必须切分 id 树里的节点，第三次 fork 直接用上了已有的两个子树**（走 `split((i1, i2)) = ((i1,0),(0,i2))` 这条分支，不新增区间）；
4. 一个参与者发生 event，另外两个先 join 再 fork 完成同步；
5. **最后的 join 让 id 合并两个子树、发生化简**；紧接着的那次 event 恰好把事件树缺的那块填满，于是**事件函数退化成一个单独的整数**。

这个例子的注解是两条：**每次 event 都只在自己 id 覆盖的区间上抬升事件树**；以及 **join 之后能发生化简**（`sum` 里的 `norm` 就是干这个的）。这就是"stamp 会随参与者数量收缩"的具体来源。

一次完整运行里的两件事，都能在 id 区间上直接看出来：

```
   ① 每次 event 只在自己 id 覆盖的区间上抬升事件树

        A 占左半：event 只在左半抬升
        [── A 抬升 ──][── 别人管 ──]

   ② join 会把相邻区间并回去：sum 里的 norm 负责这一步

        [── A ──][── B ──]  ──join──▶  [──── A∪B ────]

        紧接着那次 event 用自己那份把事件树补满
        ⇒ 事件函数退化成一个单独的整数，表示随之变短

   ⇒ 表示随参与者数量可增可减；向量时钟的长度则单调不减，只能靠剪枝控制。
```

## 空间性质

仿真评估空间需求的结论是：**空间需求随实体数量良性地扩展，并且随时间只温和增长**。与向量时钟的关键差别是"会缩"——向量时钟的长度单调不减，只能靠剪枝控制。

## 判据与边界

| 判据 | 说明 |
| --- | --- |
| 不需要全局 id | 新 id 由本地 `split` 产生，无需外部来源 |
| 退休不需要全局协调 | 一个实体被 join 掉就自动回收，不依赖其他节点是否可达 |
| id 可复用 | `split` 归还的区间可以再分给后继 |
| **匿名 stamp 不能记事件** | event 的前置条件是 $i \ne 0$；`peek` 产出的匿名副本只能用于传递因果信息 |
| 比较不看 id | $(i_1,e_1) \sqsubseteq (i_2,e_2)$ 只比较事件函数 |
| 必须保持归一化 | 否则 `sum` / `join` 的递归会得到非最小表示，且等价判断要先做归一化 |
| 适用范围 | 高 churn 的 P2P、副本数动态变化的复制存储 |
| 代价 | 结构是树，实现与理解成本高于定长向量；区间无限细分时深度增长 |

## 相关

- [[03-逻辑时钟：Lamport 与向量]] —— ITC 要泛化的对象（向量时钟与 Dynamo 版本向量的剪枝问题）；fork / event / join 三操作的历史起点也在那篇
- [[04-全局快照与虚拟时间]] —— 同为"保留偏序"的时钟构造，但那一条线走的是格与一致割

## 参考

- Paulo Sérgio Almeida, Carlos Baquero, Victor Fonte. *Interval Tree Clocks: A Logical Clock for Dynamic Systems*. 12th International Conference on Principles of Distributed Systems (OPODIS 2008), LNCS 5401, pp. 259–274.
