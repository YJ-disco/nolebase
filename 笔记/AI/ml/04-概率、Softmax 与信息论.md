---
tags:
  - AI/ml/数学
---

# 概率、Softmax 与信息论

上一层（[[03-线性代数与 GEMM]]）管的是张量怎么变换。这一层管的是**模型输出的含义** —— logits 经 Softmax 变成分布，交叉熵对应负对数似然，KL 衡量分布差异。

## 一、随机变量与分布

离散随机变量用概率质量函数，连续随机变量用概率密度函数：

$$p_X(x) = P(X=x),\ \sum_x p_X(x) = 1 \qquad\qquad \int_{-\infty}^{\infty} p_X(x)\,dx = 1$$

连续变量在单点的概率为 0，区间概率由密度积分给出。

| 分布 | 取值 | 典型场景 |
| --- | --- | --- |
| Bernoulli | $0/1$ | 二分类、Dropout mask |
| **Categorical** | $1,\ldots,V$ | **下一个 token 的分布** |
| Uniform | 区间 / 有限集合 | 初始化、随机采样 |
| Gaussian | 实数 | 初始化、噪声、近似分析 |

### 联合、边缘、条件

$$P(X=x) = \sum_y P(X=x, Y=y) \qquad\qquad P(X=x \mid Y=y) = \frac{P(X=x, Y=y)}{P(Y=y)}$$

自回归语言模型用链式法则分解序列概率：

$$P(x_1, x_2, \ldots, x_T) = \prod_{t=1}^{T} P(x_t \mid x_1, \ldots, x_{t-1})$$

> **这条式子解释了为什么训练时的 next-token prediction 与推理时逐 token 生成是同一个概率模型的两个过程。** 训练在每个位置同时算这一项，推理按顺序一次算一项 —— 数学上是同一个分解式。

### 独立与条件独立

若 $P(X,Y) = P(X)P(Y)$ 则独立。**独立比「不相关」更强**：协方差为 0 不一定独立。给定 $Z$ 后条件独立写作 $P(X,Y\mid Z) = P(X\mid Z)P(Y\mid Z)$。

> **分布式训练隐含了 i.i.d. 假设。** 数据并行切分、梯度取平均、loss 汇总都建立在「各卡样本独立同分布」上。真实数据里的重复样本、分片偏差、序列相关性会破坏这个假设 —— 这也是「按样本加权」而不是「简单平均」的原因（见 [[02-分布式训练总论与显存账本]] 的加权均值算例）。

### Bayes

$$P(X\mid Y) = \frac{P(Y\mid X)P(X)}{P(Y)}$$

先验 × 似然 → 后验。常规 LLM 训练不直接算它，但它是概率推断、参数估计与不确定性建模的基础。

### 期望、方差、协方差

$$\mathbb{E}[X] = \sum_x x P(X=x) \qquad\qquad \mathbb{E}[aX+bY] = a\mathbb{E}[X] + b\mathbb{E}[Y]$$

**期望的线性性不要求变量独立** —— 这条在推导里常用。

$$\operatorname{Var}(X) = \mathbb{E}[(X-\mu)^2] = \mathbb{E}[X^2] - \mu^2$$

$\mathbb{E}[X^2] - \mu^2$ 这个形式**数值上不稳定**（两个大数相减），正确做法见 [[01-数值计算与精度]] 的灾难性消减一节。

协方差与相关系数：

$$\operatorname{Cov}(X,Y) = \mathbb{E}[(X-\mu_X)(Y-\mu_Y)], \qquad \rho_{X,Y} = \frac{\operatorname{Cov}(X,Y)}{\sigma_X\sigma_Y}$$

**协方差矩阵** $\boldsymbol{\Sigma} = \mathbb{E}[(\mathbf{x}-\boldsymbol{\mu})(\mathbf{x}-\boldsymbol{\mu})^\top]$ 是对称半正定矩阵。PCA 对它做特征分解，找出方差最大的正交方向。

### 除以 n 还是 n−1

$$s^2 = \frac{1}{n-1}\sum_{i=1}^{n}(x_i - \bar{x})^2$$

深度学习中「方差」是否除以 $n$ 还是 $n-1$ **取决于具体定义**：

- **LayerNorm / BatchNorm 用总体方差形式（除以元素数 $n$）**
- 统计库的默认值可能是无偏形式（除以 $n-1$）

> 复现算子时要查 API 语义，不要只看名字。这两者在小 $n$ 下差异可观，是「照论文实现却对不上数值」的一个常见来源。

### 最大似然与负对数似然

$$\theta^* = \arg\max_\theta \prod_i p_\theta(y_i\mid x_i) = \arg\max_\theta \sum_i \log p_\theta(y_i\mid x_i)$$

取对数把乘积变成求和 —— 既避免数值下溢，也便于求导。等价地最小化负对数似然：

$$\mathcal{L}_{NLL} = -\sum_i \log p_\theta(y_i\mid x_i)$$

**这就是语言模型训练损失的定义本身。**

## 二、Softmax

### logits 不是概率

模型最后一层输出 $V$ 个任意实数 $\mathbf{z} = (z_1,\ldots,z_V)$ —— 可为负，不需要和为 1。Softmax 把它们转成分布：

$$p_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$$

**Softmax 对统一平移不敏感**：

$$\operatorname{softmax}(\mathbf{z}+c) = \operatorname{softmax}(\mathbf{z})$$

因为分子分母同乘 $e^c$。**这个性质正是数值稳定实现的依据** —— 既然平移不改变结果，那就平移到一个不会溢出的位置。

### 数值稳定的实现

直接算 $e^{1000}$ 会溢出。令 $m = \max_j z_j$：

$$p_i = \frac{e^{z_i - m}}{\sum_j e^{z_j - m}}$$

最大的指数输入变成 0，因此最大指数值为 1，其他都不大于 1。

```python
def stable_softmax(x):
    m = max(x)
    exps = [exp(v - m) for v in x]
    return [v / sum(exps) for v in exps]
```

减最大值防的是**正向溢出**。很小的项仍可能下溢到 0 —— 这通常说明它相对最大项确实可忽略，但**如果后续还要取对数，就不能走这条路径**：应该直接用稳定的 `log_softmax`，而不是先变成 0 再 `log(0)`。

### LogSumExp

$$\operatorname{LSE}(\mathbf{z}) = \log\sum_j e^{z_j} = m + \log\sum_j e^{z_j - m}$$

于是：

$$\log \operatorname{softmax}(\mathbf{z})_i = z_i - \operatorname{LSE}(\mathbf{z})$$

> 交叉熵实现通常**融合 `log_softmax + NLLLoss`** —— 既减少中间张量与访存，也避开了「先 Softmax 再取 log」这条不稳定路径。这是「数值稳定与性能优化方向一致」的典型例子。

### 温度

$$p_i(T) = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}$$

| $T$ | 效果 |
| --- | --- |
| $< 1$ | 放大 logit 差异，分布更尖锐 |
| $> 1$ | 缩小差异，分布更平坦 |
| $\to 0^+$ | 趋近只选最大 logit（贪心） |
| $\to \infty$ | 趋近均匀分布 |

**温度改变的是采样分布，不改变模型权重。** 实现上应先缩放 logits 再应用稳定 Softmax；非常小的 $T$ 要谨慎处理（$z/T$ 可能溢出）。

## 三、交叉熵、熵与 KL

### 交叉熵

$$H(\mathbf{q}, \mathbf{p}) = -\sum_i q_i \log p_i$$

真实标签是 one-hot（正确类别为 $y$）时：

$$\mathcal{L} = -\log p_y$$

模型给正确类别的概率越高，损失越小；$p_y \to 1$ 时损失为 0。

### 熵

$$H(\mathbf{p}) = -\sum_i p_i \log p_i$$

全部概率集中在一类时熵最小；$V$ 类均匀分布时熵最大，为 $\log V$。对数底决定单位（自然对数 → nat，底 2 → bit），机器学习损失通常用自然对数。

### KL 散度

$$D_{KL}(\mathbf{q} \lVert \mathbf{p}) = \sum_i q_i \log\frac{q_i}{p_i} \ge 0$$

两分布相同时为 0，但**通常不对称**：

$$D_{KL}(\mathbf{q}\lVert\mathbf{p}) \ne D_{KL}(\mathbf{p}\lVert\mathbf{q})$$

**所以 KL 不是严格意义的距离。**

交叉熵可以分解：

$$H(\mathbf{q},\mathbf{p}) = H(\mathbf{q}) + D_{KL}(\mathbf{q}\lVert\mathbf{p})$$

> 训练时真实分布 $\mathbf{q}$ 固定，$H(\mathbf{q})$ 与模型参数无关 —— **所以最小化交叉熵等价于最小化 KL 散度**。这是「用交叉熵做损失」的理论依据。

KL 出现在知识蒸馏、分布匹配、RLHF/PPO 的约束项、投机解码的分析里。实现时要明确三件事：**API 接收的是概率、log 概率还是 logits**，**KL 的方向**（不对称！），以及**沿哪个维度归约**。

### Perplexity

$$\operatorname{PPL} = e^{\bar{L}}$$

（$\bar{L}$ 为平均 token 负对数似然。）直觉上可理解为「模型每步面对的有效候选数」。

> [!warning] PPL 不能脱离评测设置横比
> tokenizer、数据预处理、上下文长度、是否忽略特殊 token，都会显著改变 PPL 数值。**只有词表与测试集相同才有比较意义** —— 这一点在 [[01-语言模型演进：N-gram 到 RNN]] 有更完整的讨论。

### Top-k 与 Top-p

| 策略 | 做法 |
| --- | --- |
| Greedy | 选最大概率 token |
| Top-k | 只在概率最高的 $k$ 个中采样 |
| Top-p（nucleus） | 取累计概率至少达 $p$ 的**最小**候选集，再归一化采样 |

Top-p 的集合大小随分布尖锐程度自动变化 —— 这是它成为多数聊天 API 默认的原因。两者的失效场景与参数交互见 [[04-采样参数]]。

**实现要点**：通常先过滤 logits（把被排除项设为负无穷），再执行稳定 Softmax 与采样。

> [!warning] 分布式推理下的 top-k / top-p
> 若 vocabulary 被张量并行切分，**global top-k / top-p 需要跨设备聚合局部候选**，或使用等价的分布式算法。这是一个容易漏掉的正确性问题：各卡各自取局部 top-k 会得到与单卡不同的候选集。

## 相关

- [[01-机器学习介绍]] —— 交叉熵就是 P 的一种具体形式；指标选错等于优化目标错
- [[02-线性回归]] —— 平方损失对应高斯噪声假设，是「换分布假设就换损失」的第一个例子
- [[03-线性代数与 GEMM]] —— 上一层的张量与矩阵运算
- [[05-反向传播与梯度优化]] —— Softmax + 交叉熵的梯度为什么是 $p - y$
- [[04-采样参数]] —— Top-k / Top-p / 温度在推理侧的完整语义与失效场景
- [[01-数值计算与精度]] —— 方差的不稳定算法、下溢与消减
- [[01-语言模型演进：N-gram 到 RNN]] —— 困惑度的完整定义与量级参照
- [[10-KV Cache 与推理优化]] —— 采样分布与推理阶段的关系

## 参考

- **AIInfraGuide 第2章 §4–5**（概率分布、链式法则、期望方差、稳定 Softmax、LogSumExp、交叉熵与 KL 的关系、PPL、top-k/top-p）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80
- **Deep Learning Book — Probability and Information Theory**：https://www.deeplearningbook.org/contents/prob.html
- **PyTorch Numerical Accuracy**（`log_softmax` 与融合实现的口径）：https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html
- **Attention Is All You Need**（除以 $\sqrt{D_h}$ 的方差推导在 [[11-Self-Attention 机制]] 展开）：https://arxiv.org/abs/1706.03762
