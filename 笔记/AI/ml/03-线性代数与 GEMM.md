---
tags:
  - AI/ml/数学
---

# 线性代数与 GEMM

AI Infra 对数学的要求是**看到公式能立刻回答三个工程问题**：

1. **形状是否匹配？** 内维对不对，输出是什么 shape
2. **代价是多少？** 多少 FLOPs、多少访存
3. **实现风险是什么？** 大数据类型、内存布局、分块、数值稳定性

以语言模型的输出投影为例：

$$\mathbf{Y} = \mathbf{X}\mathbf{W}, \qquad \mathbf{X} \in \mathbb{R}^{(BS) \times H},\quad \mathbf{W} \in \mathbb{R}^{H \times V}$$

- 形状：内维 $H$ 相同，输出 $(BS) \times V$
- 代价：每个输出元素做 $H$ 次乘加，共约 $2BSHV$ FLOPs
- 风险：权重很大、输出 logits 很大 —— 要考虑 dtype、内存布局、分块、并行策略，以及 Softmax 的数值稳定性

这三问贯穿整个技术栈，后面每一层优化都是在回答其中某一个。

## 一、符号与约定

| 符号 | 含义 |
| --- | --- |
| $x$ | 标量 |
| $\mathbf{x}$ | 向量 |
| $\mathbf{X}$ | 矩阵或高阶张量 |
| $X_{ij}$ | 矩阵第 $i$ 行第 $j$ 列 |
| $\mathbf{X}^\top$ | 转置 |
| $\lVert \mathbf{x} \rVert_2$ | L2 范数 |
| $\odot$ | 逐元素乘法 |

深度学习代码用 **batch-first 记法**：

| 符号 | 含义 |
| --- | --- |
| $B$ | batch size |
| $S$ | sequence length |
| $H$ | hidden size |
| $V$ | vocabulary size |
| $N_h$ | attention head 数 |
| $D_h = H / N_h$ | 每个 head 的维度 |

## 二、张量不只是 shape

工程里的「张量」是这几项信息的组合：

```
数据地址 + shape + stride + dtype + device + layout
```

**两个张量 shape 相同，不代表内存布局相同；数值相同，也不代表 dtype 与误差特性相同。**

### stride 与偏移

一个 shape 为 $(2,3)$ 的行主序矩阵：

$$\mathbf{X} = \begin{bmatrix}1 & 2 & 3\\4 & 5 & 6\end{bmatrix}$$

底层连续存储为 `[1,2,3,4,5,6]`，以元素为单位的 stride 是 $(3,1)$。元素 $X_{ij}$ 的线性偏移：

$$\text{offset}(i,j) = 3i + j$$

转置视图 $\mathbf{X}^\top$ 的 shape 变成 $(3,2)$，但 **stride 变成 $(1,3)$，数据一个字节都不用搬**。代价是：按转置后的最后一维扫描时访问不再连续。

> **看到 `view` / `reshape` / `transpose` / `contiguous`，要同时想逻辑维度和物理布局。** `view` 与 `reshape` 的差别就在这 —— `view` 只改 stride（要求能表示），`reshape` 必要时会拷贝一次。`contiguous()` 则是显式把数据重排成连续。

### 逐元素、广播、点积、矩阵乘

| 运算 | 形状规则 |
| --- | --- |
| 逐元素 | 要求对应元素能配对 |
| **广播** | 从尾部维度向前对齐，每对维度「相等 / 有一个是 1 / 有一方不存在」 |
| 点积 | $n \times n \to$ 标量 |
| 矩阵乘 | $(M,K)@(K,N) \to (M,N)$，内维 $K$ 归约 |
| **batch matmul** | 前若干维是 batch 维，**只对最后两维做矩阵乘** |

$$\mathbf{Q}\mathbf{K}^\top \in \mathbb{R}^{B \times N_h \times S_q \times S_k}$$

前两维 $(B, N_h)$ 是 batch 维，每个 `(batch, head)` 独立做一次矩阵乘。**「高阶张量乘法」不神秘，就是最后一维做矩阵乘、前面维度负责批处理。**

> [!warning] 广播的危险在于「不报错但算错」
> 广播是逻辑视图，不会先复制出完整张量。但**错误 shape 也可能恰好能广播** —— 代码不报错，语义已经错了。这类 bug 在自定义算子里很常见，形状检查要写严格。

### 范数与容差

$$\lVert \mathbf{x} \rVert_1 = \sum_i |x_i|, \qquad \lVert \mathbf{x} \rVert_2 = \sqrt{\sum_i x_i^2}, \qquad \lVert \mathbf{x} \rVert_\infty = \max_i |x_i|$$

范数在工程里用于：衡量参数/激活/梯度规模、梯度裁剪、比较参考实现与优化实现的误差、正则化、向量归一化。

比较浮点结果时，测试库通常**组合绝对与相对容差**：

$$|\hat{x}-x| \le \text{atol} + \text{rtol} \cdot |x|$$

为什么不能只用相对容差：参考值 $x$ 接近 0 时相对误差会被放大。这也是 `torch.allclose` / `numpy.testing` 的默认行为。

### 线性层其实是仿射变换

严格的线性变换满足 $f(a\mathbf{x}+b\mathbf{y}) = af(\mathbf{x})+bf(\mathbf{y})$。神经网络里的「Linear 层」还带 bias：

$$\mathbf{y} = \mathbf{W}\mathbf{x} + \mathbf{b}$$

这是**仿射变换**。一条推论值得记住：**多个没有激活函数的线性/仿射层可以合并成一个** —— 所以非线性激活是深层网络能表达复杂函数的关键，不是可选项。

### 秩、低秩分解与 LoRA

矩阵的秩可以理解为「独立方向的数量」：

$$\operatorname{rank}(\mathbf{A}) \le \min(M,N)$$

秩低意味着信息冗余，可用两个小矩阵近似：

$$\mathbf{A} \approx \mathbf{U}\mathbf{V}, \qquad \mathbf{U} \in \mathbb{R}^{M \times r},\ \mathbf{V} \in \mathbb{R}^{r \times N},\ r \ll \min(M,N)$$

**参数量从 $MN$ 降到 $r(M+N)$** —— 这是低秩适配（LoRA）与模型压缩的数学基础。一个 $4096 \times 4096$ 的权重，rank 8 的 LoRA 只增加 $8 \times 8192 = 65{,}536$ 个参数，不到原矩阵的 0.4%。

## 三、GEMM 与分块

### 分块矩阵乘法是恒等式

把 $\mathbf{A}$、$\mathbf{B}$ 按块划分，则 $\mathbf{A}\mathbf{B}$ 的每个块可以由小块的乘加得到：

$$\begin{bmatrix}\mathbf{A}_{11} & \mathbf{A}_{12}\\ \mathbf{A}_{21} & \mathbf{A}_{22}\end{bmatrix}\begin{bmatrix}\mathbf{B}_{11} & \mathbf{B}_{12}\\ \mathbf{B}_{21} & \mathbf{B}_{22}\end{bmatrix}=\begin{bmatrix}\mathbf{A}_{11}\mathbf{B}_{11}+\mathbf{A}_{12}\mathbf{B}_{21} & \cdots\\ \cdots & \cdots\end{bmatrix}$$

**这是 tiling 的数学依据 —— 它改变计算顺序与数据搬运，不改变目标公式。** 理解这一点就不会把 tiling 当成「工程近似」。

### 朴素实现浪费在哪

对 $(M,K)@(K,N)$：输出元素 $MN$ 个，每个做 $K$ 次乘加，FLOPs 约 $2MKN$。

朴素实现若每次乘加都从全局内存加载 $A_{ik}$ 与 $B_{kj}$，**同一个元素会被反复读取**：

| 数据复用 | 说明 |
| --- | --- |
| $A$ 的一个元素 | 被同一输出行的多个列复用 |
| $B$ 的一个元素 | 被同一输出列的多个行复用 |
| 累加器 | 在寄存器中被复用 $K$ 次 |

GPU kernel 的 tiling 就是把这件事变成：取一小块 $\mathbf{A}$ 与一小块 $\mathbf{B}$ → 搬到片上共享内存/寄存器 → 复用这些数据算出多个输出元素 → 沿 $K$ 方向迭代累加 → 写回 HBM。

### 算术强度可以估

理想情况下每个输入矩阵只从 HBM 读一次、输出写一次，粗略字节数 $2(MK + KN + MN)$：

$$AI \approx \frac{2MKN}{2(MK+KN+MN)}$$

这不是精确性能模型，但足以判断一个 GEMM 更可能受算力还是带宽限制 —— 判据是把它和硬件的 ops:byte 比（H100 约 295 FLOP/Byte）比较，见 [[01-GPU 硬件架构与存储层次]]。

### tile 不是越大越好

更大的 tile 提高复用，但会消耗更多：

- 共享内存
- 寄存器
- 每 block 的线程数与同步开销
- 边界处理成本

资源占用过高会**降低一个 SM 上同时驻留的 block/warp 数**，影响延迟隐藏。tile 的选择是在数据复用、占用率、指令效率、形状适配之间的折中。

### 尾块

$M, N, K$ 不是 tile size 整数倍时，最后一块越界。常见处理：加边界判断/掩码；padding 到对齐尺寸；为常见整齐 shape 提供快路径。三种方式的开销结构不同 —— padding 增加计算和存储，分支增加控制开销，**哪个更快取决于具体 shape 与硬件**。

### 浮点分块结果可能不同

实数加法满足结合律，浮点加法不严格满足：

$$(a+b)+c \ne a+(b+c)$$

不同 tile 划分、不同线程归约树、是否走 Tensor Core 路径，都会改变累加顺序，所以**优化前后结果可能不逐位一致**。正确性验证应使用适合 dtype 与问题规模的误差容限，并关注误差是否随归约长度系统性放大。机制见 [[01-数值计算与精度]]。

## 四、从数学式到 Kernel 的检查表

看到一个算子，按这个顺序拆：

1. 输入、输出、中间量的 **shape** 是什么？
2. 哪些维度**保留**，哪些维度**归约**？
3. 能否**分块**？分块状态如何**合并**？
4. **FLOPs** 与理论最小**访存量**是多少？
5. 是否存在**广播、转置或非连续 stride**？
6. 哪些操作对**精度敏感**，需要高精度累加？
7. 是否会出现 **exp 溢出、除零、消减或长归约误差**？
8. 中间张量能否**融合消除**？
9. **边界 shape 与尾块**如何处理？
10. 用什么**参考实现、dtype 容差和极端输入**验证？

这份清单把抽象数学变成可执行的工程动作。第 3、6、7 条分别指向后面三块内容：分块状态的合并方式决定能否写成分块 kernel（Online Softmax 就是一例），精度敏感的归约决定 accumulate dtype，数值风险决定要不要用稳定形式。

## 相关

- [[01-机器学习介绍]] —— 特征与参数在计算机里是什么（本专栏的术语口径）
- [[02-线性回归]] —— 线性层 ^\top x$ 的最小二乘解，GEMM 的最小规模实例
- [[01-GPU 硬件架构与存储层次]] —— 算术强度与 Roofline 的硬件侧
- [[04-概率、Softmax 与信息论]] —— 从 logits 到分布的数学
- [[05-反向传播与梯度优化]] —— 线性层的反向为什么还是 GEMM
- [[01-数值计算与精度]] —— 浮点非结合性与归约顺序
- [[02-Transformer 架构]] —— 这些运算在模型里的具体位置

## 参考

- **AIInfraGuide 第2章 数学基础 §1–3**（三问框架、stride 与布局、四种乘法、秩与低秩、GEMM tiling 的数学依据、算术强度估算）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80
- **Deep Learning Book — Linear Algebra**：https://www.deeplearningbook.org/contents/linear_algebra.html
- **PyTorch Broadcasting Semantics**（广播规则的确切定义）：https://docs.pytorch.org/docs/stable/notes/broadcasting.html
- **PyTorch Numerical Accuracy**（`atol` / `rtol` 的口径）：https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html
- **LoRA: Low-Rank Adaptation of Large Language Models**（低秩分解的工程应用）：https://arxiv.org/abs/2106.09685
