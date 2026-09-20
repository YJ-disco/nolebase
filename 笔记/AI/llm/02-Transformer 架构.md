---
tags:
  - AI/llm
---

# Transformer 架构

Transformer 由谷歌团队 2017 年提出（`Vaswani et al., Attention is all you need, NeurIPS 2017`）。它**完全抛弃循环结构**，只靠**注意力（Attention）**捕捉序列内依赖，从而做到真正的并行计算——RNN 第 $t$ 步必须等第 $t-1$ 步，无法并行，这是它被替换掉的直接原因。

## Encoder-Decoder 整体结构

最初是为端到端机器翻译设计的，宏观上分两半：

| 部分 | 职责 |
| --- | --- |
| 编码器（Encoder） | 「理解」输入的整个句子，为每个词元生成富含上下文信息的向量表示 |
| 解码器（Decoder） | 「生成」目标句子，参考自己已生成的前文，并「咨询」编码器的理解结果来生成下一个词 |

编码器层 `EncoderLayer` 的结构：多头自注意力 → Add & Norm → 前馈网络 → Add & Norm。
解码器层 `DecoderLayer` 多一个子层：掩码多头自注意力 → Add & Norm → 交叉注意力（Q 来自解码器自己，K/V 来自编码器输出）→ Add & Norm → 前馈网络 → Add & Norm。

```python
class EncoderLayer(nn.Module):
    def forward(self, x, mask):
        attn_output = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        return x
```

## 自注意力：Q、K、V

以句子 `The agent learns because **it** is intelligent.` 为例：读到 `it` 时，要理解它的指代，就得把注意力放到前面的 `agent` 上。自注意力就是对这个过程的数学建模——处理每个词时兼顾所有其他词，并分配不同的注意力权重。

每个输入词元向量被投影成三个可学习的角色：

| 角色 | 含义 |
| --- | --- |
| 查询（Query, Q） | 代表当前词元，它正在主动「查询」其他词元以获取信息 |
| 键（Key, K） | 代表句子中可被查询的词元的「标签」或「索引」 |
| 值（Value, V） | 代表词元本身携带的「内容」或「信息」 |

三者由原始词嵌入乘以三个可学习权重矩阵 $W^Q,W^K,W^V$ 得到。计算过程：

1. 为每个词生成 $Q,K,V$
2. **相关性得分**：用当前词的 $Q$ 与所有词（含自己）的 $K$ 做点积
3. **稳定化与归一化**：除以缩放因子 $\sqrt{d_k}$（$d_k$ 是 $K$ 的维度）防止梯度过小，再 Softmax 成总和为 1 的权重
4. **加权求和**：权重分别乘以各词的 $V$ 再相加，得到融合全局上下文的新表示

公式：

$$\text{Attention}(Q,K,V)=\text{softmax}\left(\frac{QK^{T}}{\sqrt{d_{k}}}\right)V$$

### 从单头到多头

只做一次注意力，模型可能只学会关注一种关系（处理 `it` 时只学会关注主语）。语言里的关系是多种并存的——指代、时态、从属。多头注意力的做法是：把 Q、K、V 在维度上切成 $h$ 份，每份独立做一次单头注意力，最后拼接再过一个线性变换整合。相当于让 $h$ 个「专家」从不同表示子空间审视同一个句子。

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        assert d_model % num_heads == 0
        self.d_model, self.num_heads = d_model, num_heads
        self.d_k = d_model // num_heads
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

    def scaled_dot_product_attention(self, Q, K, V, mask=None):
        attn_scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        if mask is not None:
            attn_scores = attn_scores.masked_fill(mask == 0, -1e9)
        attn_probs = torch.softmax(attn_scores, dim=-1)
        return torch.matmul(attn_probs, V)

    def split_heads(self, x):
        batch_size, seq_length, d_model = x.size()
        return x.view(batch_size, seq_length, self.num_heads, self.d_k).transpose(1, 2)

    def combine_heads(self, x):
        batch_size, num_heads, seq_length, d_k = x.size()
        return x.transpose(1, 2).contiguous().view(batch_size, seq_length, self.d_model)

    def forward(self, Q, K, V, mask=None):
        Q = self.split_heads(self.W_q(Q))
        K = self.split_heads(self.W_k(K))
        V = self.split_heads(self.W_v(V))
        attn_output = self.scaled_dot_product_attention(Q, K, V, mask)
        return self.W_o(self.combine_heads(attn_output))
```

两个实现细节值得记：

- `d_model` 必须能被 `num_heads` 整除，否则 `split_heads` 的 reshape 不成立
- 掩码用 `masked_fill(mask == 0, -1e9)`：把要屏蔽的位置填成极小负数，过 Softmax 后概率趋近 0

## 为什么是它胜出：三个维度上的对照

「并行」只是表面说法。原论文用三个量把自注意力和循环/卷积放在一起比，才是它胜出的完整理由。设 $n$ 为序列长度、$d$ 为表示维度、$k$ 为卷积核宽度：

| 层类型 | **每层计算复杂度** | **顺序操作数** | **最大路径长度** |
| --- | --- | --- | --- |
| **自注意力** | $O(n^2 \cdot d)$ | **$O(1)$** | **$O(1)$** |
| 循环层（RNN/LSTM） | $O(n \cdot d^2)$ | $O(n)$ | $O(n)$ |
| 卷积层 | $O(k \cdot n \cdot d^2)$ | $O(1)$ | $O(\log_k n)$ |

三列各回答一个不同的问题，要分开读：

**一、计算复杂度：注意力不是无条件更便宜。**

$$O(n^2 \cdot d) \quad\text{vs}\quad O(n \cdot d^2)$$

**只有当 $n < d$ 时，自注意力才比循环层更便宜**（可以先把 $n^2 d$ 与 $n d^2$ 约掉一个 $nd$，剩下 $n$ 与 $d$ 的对比）。

> **这条判据很重要**：它说明注意力的二次项**在序列长度超过表示维度时才成为瓶颈**。2017 年的机器翻译任务里 $n$ 通常在几十到几百、$d$ 是 512 —— **正好落在注意力更便宜的那一侧**。

这也预告了后来的事：**当上下文从几十上百扩到 32K、128K 时，$n$ 远远超过 $d$，二次项就成了主要矛盾**——这才是 FlashAttention、稀疏注意力、滑窗注意力这一系列工作的动机来源。

**二、顺序操作数：这是「能不能并行」的精确定义。**

自注意力是 **$O(1)$**——所有位置的计算互不依赖，一次矩阵乘全部算完。循环层是 **$O(n)$**——第 $t$ 步必须等第 $t-1$ 步。**这一列的差距是训练能不能吃满算力的问题**（见 [[03-Decoder-Only 与自回归]] 里「训练并行、生成串行」那条不对称是怎么来的）。

**三、最大路径长度：这是「长程依赖好不好学」的精确定义。**

自注意力里任意两个位置**直接相连，路径长度恒为 1**；循环层要走 $O(n)$ 步；卷积层要 $O(\log_k n)$ 层堆叠。

> **路径越长，梯度回传经过的连乘越多，越容易衰减** —— 这正是 [[01-语言模型演进：N-gram 到 RNN|RNN 的梯度消失]]的根源。**路径长度从 $O(n)$ 压到 $O(1)$，是 Transformer 能解决长程依赖的根本原因**，而不是「注意力比循环更聪明」。

**三列合起来**：算力上打平（在 $n<d$ 时还占优）、并行上碾压、长程依赖上从根上解决。这才是一个架构替换另一个架构的完整账。

## 逐位置前馈网络 FFN

每个 Encoder / Decoder 层里，多头注意力之后都跟一个**逐位置前馈网络（Position-wise Feed-Forward Network）**。分工是：注意力层从整个序列「动态聚合」信息，前馈网络从聚合后的信息里提取更高阶特征。

「逐位置」指它独立作用于每一个词元向量——长度为 `seq_len` 的序列实际会调用 `seq_len` 次，但**所有位置共享同一组权重**，既保留独立加工能力，又大幅减少参数量。

$$\text{FFN}(x)=\max(0, xW_1+b_1)W_2+b_2$$

通常 `d_ff = 4 * d_model`：先把维度放大，过 ReLU，再映射回 `d_model`。这种「先扩大再缩小」被认为有助于学到更丰富的特征表示。

## 残差连接与层归一化

每个子模块都被 `Add & Norm` 包裹，作用有两个：

| 操作 | 解决的问题 | 机制 |
| --- | --- | --- |
| 残差连接（Add） | 梯度消失 | $\text{Output}=x+\text{Sublayer}(x)$，反向传播时梯度可绕过子模块直接前传 |
| 层归一化（Norm） | 内部协变量偏移（Internal Covariate Shift） | 对单个样本的所有特征归一化到均值 0、方差 1，使每层输入分布稳定 |

## 位置编码

自注意力本身**不含任何位置信息**——对它来说 `agent learns` 和 `learns agent` 完全等价。位置编码（Positional Encoding）解决这个：给每个词元的嵌入向量额外加一个代表绝对/相对位置的「位置向量」。

它的关键特点是**不通过学习得到，而是用固定数学公式直接算**：

$$PE_{(pos,2i)}=\sin\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right),\qquad PE_{(pos,2i+1)}=\cos\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)$$

其中 $pos$ 是词元在序列中的位置，$i$ 是位置向量的维度索引（$0$ 到 $d_{\text{model}}/2$），$d_{\text{model}}$ 是词嵌入维度。偶数维用 sin、奇数维用 cos。

这样即使两个词元同叫 `agent`、嵌入完全相同，由于位置不同，加上不同的位置编码后，输入到模型的向量就变得独一无二。

**注意这是 2017 年原版的做法**（绝对位置编码，加到嵌入上）。这一层后来演进了很长一段：绝对 → 相对 → **RoPE** → 长上下文外推（位置插值 / NTK-aware / YaRN）→ ALiBi。**今天主流模型用的 RoPE 与这里的做法差别很大**——它不对嵌入做加法，而是旋转注意力里的 Q、K。完整演进见 [[09-位置编码]]。

```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model: int, dropout: float = 0.1, max_len: int = 5000):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        position = torch.arange(max_len).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2) * (-math.log(10000.0) / d_model))
        pe = torch.zeros(max_len, d_model)
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        self.register_buffer('pe', pe.unsqueeze(0))   # buffer 不是参数，但会随模型 to(device)

    def forward(self, x):
        x = x + self.pe[:, :x.size(1)]
        return self.dropout(x)
```

## 相关

- [[01-语言模型演进：N-gram 到 RNN]] —— Transformer 替换掉的那套循环结构
- [[03-Decoder-Only 与自回归]] —— 从完整架构砍到只剩解码器

## 参考

- 来源：《Hello-Agents》第三章 §3.1.2
- Vaswani, A., et al. Attention is all you need. NeurIPS, 2017.
