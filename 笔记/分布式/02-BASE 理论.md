---
tags:
  - 分布式
  - 分布式/一致性
---

# BASE 理论

`BASE` 是 Dan Pritchett（eBay）在 ACM Queue 2008 年 5/6 月号 *BASE: An Acid Alternative* 里给出的缩写：**Basically Available, Soft state, Eventually consistent**。它给的是「分区数据库里怎么用可用性换一致性」这个取舍的命名与拆解，本身不含具体算法 —— 论文的实际内容是把取舍一步步落到具体的 SQL 改写与表结构上。

## 先把缩放的两条路分清楚

论文开篇把水平扩展拆成两个可以同时使用的维度：

| 维度 | 做法 |
| --- | --- |
| **功能分区（functional partitioning）** | 按功能把数据分组，把功能组分散到不同库 |
| **分片（sharding）** | 在功能组内部把数据切到多个库 |

垂直扩展的局限论文列了三条：撞到最大机器的容量上限、加容量的成本是阶梯式的（必须买下一档）、容易产生供应商锁定。

**功能分区会遇到一个具体的阻力**：靠数据库自身的约束（外键等）保证跨功能组一致性，会把 **schema 与部署策略耦合起来** —— 约束要生效，相关表必须在同一台服务器上，水平扩展就到此为止。想在事务量上继续扩，就得把约束从数据库里搬进应用层。搬出来之后的一系列挑战，就是下面几节逐条解决的问题。

## CAP 在这里的落点，以及一个可以算的代价

论文引用的 CAP 表述是：

- **一致性**：客户端感知到一组操作是**同时**发生的；
- **可用性**：每个操作都必须以预期响应终止；
- **分区容错**：即使个别组件不可用，操作也能完成。

**ACID 路线选一致性，代价可以直接算出来。** 论文给的算式是：可用性等于**所需组件可用性的乘积**。2PC 事务涉及两个数据库，于是事务可用性是两个库可用性之积 —— 设单库可用性 99.9%：

$$0.999 \times 0.999 = 0.998$$

即 99.8%，**比单库多出 43 分钟/月的停机时间**。这就是「选一致性」在可用性账本上的具体读数。

有一句判据值得单独拎出来：**「可能被使用但不是必需」的组件不降低系统可用性。** 这是 BASE 全部设计的立足点 —— 只要能让某个组件从"必需"退成"可选"，系统可用性就不受它影响。

## BASE 与 ACID 的对立

| | ACID | BASE |
| --- | --- | --- |
| 姿态 | 悲观 | 乐观 |
| 对一致性的要求 | 每个操作结束即强制一致 | 接受一致性处于**流动状态**（soft state） |
| 一致性达成时点 | 操作边界 | 最终（eventually） |

BASE 的可用性靠**支持部分失败而不引起整体失败**实现。论文的例子：把用户分到 5 台数据库服务器上，按 BASE 设计的操作应当做到「某一台用户库故障只影响该主机上的 20% 用户」。论文强调这里没有魔法 —— 数据本来就分开了，只是设计时主动利用了这一点，于是用户**感知到的**可用性提高了。

## 一致性模式的完整推演

论文用同一个 schema 走完了从 ACID 到 BASE 的四步改写，每一步都在解上一个问题、同时引入下一个。

### 基线：一条 ACID 事务

```sql
-- 表：user(id, name, amt_sold, amt_bought)
--     transaction(xid, seller_id, buyer_id, amount)
Begin transaction
  Insert into transaction(xid, seller_id, buyer_id, amount);
  Update user set amt_sold    = amt_sold    + $amount where id = $seller_id;
  Update user set amt_bought  = amt_bought  + $amount where id = $buyer_id;
End transaction
```

判据在这里就出现了：**`user` 表里的汇总列可以看作 `transaction` 表的缓存**，它存在的理由是查询效率。既然只是缓存，一致性约束就可以放松 —— 买卖双方的"当前余额"不要求立刻反映这笔交易。论文指出这种延迟在生活中很常见：ATM 取现与手机通话的余额显示都有延迟。

### 第一步：直接拆开（figure 4）

```sql
Begin transaction; Insert into transaction(...);      End transaction
Begin transaction; Update user set amt_sold=...; ...;  End transaction
```

两个事务不再耦合，一致性也无法保证。论文点明后果：**若在两次事务之间发生故障，`user` 表会永久不一致**。这在"余额只是估计值"的语义下可以接受，但语义必须先这么定。

### 第二步：持久消息队列（figure 5）

如果估计值不可接受，就在**同一个事务内**把更新汇总值所需的信息落成持久消息：

```sql
Begin transaction
  Insert into transaction(id, seller_id, buyer_id, amount);
  Queue message "update user('seller', seller_id, amount)";
  Queue message "update user('buyer',  buyer_id,  amount)";
End transaction
```

处理端独立消费：

```sql
For each message in queue
  Begin transaction
    Dequeue message
    If message.balance == 'seller'
      Update user set amt_sold   = amt_sold   + message.amount where id = message.id;
    Else
      Update user set amt_bought = amt_bought + message.amount where id = message.id;
    End if
  End transaction
End for
```

**这一步有一条硬约束**：队列的后端持久化**必须与数据库在同一份资源上**。只有在同一资源上，入队才能跟着落库事务一起提交，从而**不引入 2PC**。事务因此完全落在一个数据库实例内，不影响系统可用性。

但新问题立刻出来了：**如果出队发生在一个涉及 user host 的事务里，2PC 又回来了。** 论文给了两个出路：

- **什么都不做**：把更新拆到独立的后端组件，保住了面向客户的组件可用性；消息处理器本身可用性低一点，业务上可能可以接受。
- **彻底消除 2PC**：需要幂等。

### 第三步：用幂等表消除 2PC（figure 7）

**幂等**的定义是「执行一次与执行多次结果相同」。它的价值在于**允许部分失败** —— 反复执行不改变最终状态，于是可以不依赖分布式事务，只靠"重试到成功"。

难点在于 **Update 天然不幂等**：例子里的 `amt_sold = amt_sold + $amount` 执行两次余额就错了。论文进一步指出更隐蔽的一点：**即使是把值直接 set 进去的 update，在无法保证执行顺序时也不幂等** —— 最终状态会取决于执行顺序。

解法是一张"已应用记录"表：

```
updates_applied(trans_id, balance, user_id)
```

伪码改成 **peek → 事务内查重并更新 → 成功后移除**：

```sql
For each message in queue
  Peek message
  Begin transaction
    Select count(*) as processed
      where trans_id = message.trans_id
        and balance  = message.balance
        and user_id  = message.user_id
    If processed == 0
      If message.balance == 'seller'
        Update user set amt_sold   = amt_sold   + message.amount where id = message.id;
      Else
        Update user set amt_bought = amt_bought + message.amount where id = message.id;
      End if
      Insert into updates_applied(message.trans_id, message.balance, message.user_id);
    End if
  End transaction
  If transaction successful
    Remove message from queue
  End if
End for
```

两个细节值得注意：

- 用 **peek** 而不是 dequeue，是为了让"移除"发生在业务事务**成功之后**；队列操作与业务库操作可以是两个独立事务。
- 队列操作**只有在数据库操作成功提交后才算提交**；失败就重试，靠 `updates_applied` 的查重把重复执行挡掉。

### 第四步：让更新与顺序无关（figure 9）

换一个场景：除了汇总值，还要记 `last_sale` / `last_purchase`（最近一次卖出/买入的时间）。若两笔购买在很短的窗口内发生、而消息系统不保证顺序，`last_purchase` 就会取错。

解法是把 **"时间不能倒退"这个业务语义直接写进 SQL 的 where 条件**：

```sql
For each message in queue:
  Peek message
  Begin transaction
    Update user set last_purchase = message.trans_date
      where id = message.buyer_id
        and last_purchase < message.trans_date;
  End transaction
  If transaction successful
    Remove message from queue
  End if
End for
```

这样更新就**与执行顺序无关**了。论文指出同一手法可以保护任何怕乱序的更新，把时间换成**单调递增的 transaction id** 也可以。

## 消息顺序：保序昂贵，而且给的是虚假安全感

论文专门用一节劝退"依赖消息系统保序"：

- 保序**贵**，而且**经常没必要**；上面两次改写的开销都远小于在消息系统里强制保序。
- **Web 应用语义上就是事件驱动的**：客户端请求以任意顺序到达，每个请求的处理时间不同，各组件内的请求调度不确定 —— 结果就是消息入队顺序不确定。**要求保序会给人虚假的安全感。**
- 论文的结论句很直接：**不确定的输入必然导致不确定的输出。**

这条判据解释了为什么上面第四步要用 `where` 条件把顺序问题消掉，而不是去要求消息系统保序。

## soft state / 最终一致对应用设计的影响

论文最后把这层影响讲清楚：软件工程师习惯把系统看成一个**闭环**（可预测输入 → 可预测输出），这是写出正确系统的前提。用 BASE **不会改变闭环意义上的可预测性**，但要求**把行为当整体来看**。

它给的例子是资产转账：把"从一个用户扣掉"和"给另一个用户加上"用消息队列解耦后，**存在一段时间资产两边都不在**。这个窗口的大小由消息系统设计决定。从用户视角看，这个延迟可能不可见 —— 如果收发双方本就在直接沟通，几秒的延迟可以容忍，系统行为在用户看来就是一致的。

**需要知道"何时状态已一致"时，用事件驱动架构（EDA）**：在把资产提交给接收方的那个事务内产生一个事件，后续处理由事件触发。论文说 EDA 能显著改善扩展性与架构解耦，但展开超出该文范围。

## 判据与边界

| 判据 | 说明 |
| --- | --- |
| 可用性是所需组件可用性的**乘积** | 每引入一个「必需」组件就乘一个小于 1 的系数；2PC 的两个库 = 两次相乘，99.9%×99.9% → 每月多停 43 分钟 |
| 「可选」组件不影响可用性 | BASE 的核心手法：把组件从「必需」改成「可选」 |
| 跨功能组的一致性更容易放松 | 功能组**内部**的一致性通常与业务语义绑定，更难动 |
| 放松一致性是产品决策 | **暂时不一致无法对终端用户隐藏**，必须工程与产品共同定 |
| 汇总列是缓存 | 它存在的理由是查询效率，这个判断是「可以放松」的依据 |
| 幂等是消除 2PC 的前提 | Update 天然不幂等；靠「已应用记录表 + peek/retry」实现幂等 |
| 顺序问题用 `where` 条件消掉 | 把「不能倒退」的语义写进更新条件，比要求消息系统保序便宜得多 |
| 代价 | 约束从数据库搬到应用层；查询会读到中间状态；需要额外的记录表与消息组件 |

论文与 [[01-CAP 定理|CAP 定理]] 的关系：CAP 说清「分区时 C 与 A 只能选一个」，BASE 给出选 A 之后工程上逐步怎么做。两者取舍口径一致，区别在层次 —— 一个在定理层，一个在模式层。

## 相关

- [[01-CAP 定理]] —— 本层的理论依据；本篇是它「选 A」那一侧的工程落地
- [[03-一致性模型]] —— 最终一致性在这个谱系里的位置，以及它与 CAP 中 A 的对应
- [[01-分布式事务]] —— 补偿型事务（TCC / Saga）与本篇的「软状态 + 补偿」是同一种思路

## 参考

- Dan Pritchett（eBay）. *BASE: An Acid Alternative*. ACM Queue, Vol. 6, No. 3, May/June 2008, pp. 48–55.
