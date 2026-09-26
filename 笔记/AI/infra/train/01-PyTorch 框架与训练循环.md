---
tags:
  - AI/infra/框架
---

# PyTorch 框架与训练循环

PyTorch 是 AI Infra 领域最主流的框架，也是后续分布式训练与推理优化的载体。这一篇走通「一个训练 step 到底发生了什么」，以及怎么用工具找到它慢在哪。

三个层次要先分清：**Tensor 是数据单元，autograd 是求导引擎，`nn.Module` 是组织方式的约定**。训练循环把三者串起来。

## Tensor

Tensor 是多维数组 —— NumPy `ndarray` 的 GPU 加速版，外加自动微分支持。

### 形状操作

「把 Tensor 想成一块可以任意捏形的橡皮泥：数据不变，只是换个排列方式。」

```python
import torch

x = torch.arange(12)
a = x.view(3, 4)          # 要求内存连续
b = x.reshape(3, 4)       # 不连续时也能用
c = x.view(-1, 4)         # -1 自动推断为 3

# permute：交换维度。Transformer 里最常用的一步
t = torch.randn(2, 8, 12, 64)          # (batch, seq_len, heads, dim)
t_permuted = t.permute(0, 2, 1, 3)     # (batch, heads, seq_len, dim)

# squeeze / unsqueeze：去掉或插入大小为 1 的维度
torch.randn(1, 3, 1, 4).squeeze().shape    # torch.Size([3, 4])
torch.randn(3, 4).unsqueeze(0).shape       # torch.Size([1, 3, 4])
```

`view` 与 `reshape` 的差别是**是否需要连续内存** —— `view` 只是改 stride（零拷贝），内存不连续时会报错；`reshape` 在必要时会拷贝一次。**改 stride 不搬数据，拷贝要搬数据** —— 在显存敏感的代码里这个区别很实在。

### 设备管理

```python
x_gpu = x.to('cuda')                  # 推荐写法
x_gpu = x.cuda()                       # 等价简写
x_cpu = x_gpu.cpu()
x_gpu = torch.randn(3, 4, device='cuda')   # 直接在 GPU 上创建，省一次搬运
```

> **CPU–GPU 搬运走 PCIe，带宽远低于 HBM。** 频繁的 `.cpu()` / `.cuda()` 是隐蔽的性能瓶颈 —— 训练循环里出现这类调用基本都要重新想一下数据流。

### dtype

| dtype | 每元素字节 | 数值范围 | 场景 |
| --- | --- | --- | --- |
| `float32` | 4 | 大 | 默认精度、优化器状态 |
| `float16` | 2 | 小，易溢出 | 混合精度（需 Loss Scaling） |
| `bfloat16` | 2 | **与 fp32 相同** | 混合精度（**推荐**，Ampere+） |

```python
print(x_fp32.nelement() * x_fp32.element_size() / 1024)   # 1000×1000 fp32 → 3906 KB
print(x_bf16.nelement() * x_bf16.element_size() / 1024)   # 同形状 bf16 → 1953 KB
```

fp16 与 bf16 同为 2 字节，差别全在指数位 —— 完整机制见 [[01-数值计算与精度]]。

## autograd

计算图是 PyTorch 记下的「食谱」：前向时把每一步操作记下来，反向时沿图从结果推回每个输入的梯度。

PyTorch 用的是**动态计算图（Define-by-Run）** —— 图在前向时构建，因此支持 `if`/`else`、`for` 这类 Python 控制流。这是它与静态图框架的核心区别。

```python
x = torch.tensor([2.0, 3.0], requires_grad=True)
z = (x * 3).sum()        # z = 15
z.backward()
print(x.grad)            # tensor([3., 3.])，dz/dx = 3
```

四条规则：

- `backward()` **只能对标量调用** —— 非标量要先 `.sum()` 或 `.mean()`
- 只有**叶节点**（`requires_grad=True` 且由用户创建）会保存 `.grad`
- `model.eval()` 与 `torch.no_grad()` 都会停止建图，但**语义不同**：前者切 Dropout/BatchNorm 行为，后者关梯度记录
- 计算图用完即释放，所以想二次反向要传 `retain_graph=True`

### 梯度是累加的

**PyTorch 默认累加梯度，不自动清零。** 这是有意设计 —— 梯度累积（用多个 micro-batch 模拟大 batch）正依赖它。

```python
x = torch.tensor([1.0], requires_grad=True)
(x * 2).sum().backward()
print(x.grad)     # tensor([2.])
(x * 3).sum().backward()
print(x.grad)     # tensor([5.]) ← 2+3，被累加了
x.grad.zero_()
(x * 4).sum().backward()
print(x.grad)     # tensor([4.])
```

**标准训练循环里每个 step 开始前必须清零**，否则上一轮梯度会混进来。实践中用 `optimizer.zero_grad()` 一次清掉所有参数。

> [!warning] `zero_grad(set_to_none=True)` 是更优的默认
> 默认的 `zero_grad()` 是把梯度张量**填 0**；`set_to_none=True` 是把它**置为 None**。后者省掉一次全量写显存，且在第一次 `backward()` 时会重新分配而不是累加到 0 上。新版 PyTorch 已在部分场景默认用后者。

## `nn.Module`

`nn.Module` 是「积木说明书」 —— 定义积木怎么拼（`forward`），并清点所有零件（参数管理）。

```python
import torch.nn as nn

class SimpleModel(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.linear1 = nn.Linear(input_dim, hidden_dim)
        self.relu = nn.ReLU()
        self.linear2 = nn.Linear(hidden_dim, output_dim)

    def forward(self, x):
        return self.linear2(self.relu(self.linear1(x)))

model = SimpleModel(784, 256, 10)
output = model(torch.randn(32, 784))    # 用 model(x)，不要直接调 forward
```

**必须用 `model(x)` 而不是 `model.forward(x)`** —— 前者会触发 `__call__` 上的 hook 机制（分布式训练、混合精度、梯度检查点都挂在这上面），后者绕过全部 hook。

### 参数管理

```python
model = nn.Linear(10, 5)
for name, param in model.named_parameters():
    print(f"{name}: {param.shape}")     # weight: [5, 10] / bias: [5]

total = sum(p.numel() for p in model.parameters())    # 参数量统计
print(model.state_dict().keys())        # odict_keys(['weight', 'bias'])
```

`state_dict()` 返回的是名字到张量的映射，**是 checkpoint 的标准载体**。

### 常用层与大模型的关系

| 层 | 在大模型里的位置 |
| --- | --- |
| `nn.Linear` | Q/K/V 投影、FFN —— 用最多 |
| `nn.Embedding` | 第一层，token id → 稠密向量 |
| `nn.LayerNorm` | 每个子层之后，稳定训练 |
| `nn.Dropout` | `model.eval()` 后自动关闭 |

## 训练循环

### DataLoader 的两个参数值得单独说

```python
loader = DataLoader(dataset, batch_size=64, shuffle=True,
                    num_workers=4, pin_memory=True, drop_last=True)
```

- **`num_workers` 过小 → GPU 等数据**（data loading bottleneck）。这是「GPU 利用率上不去」最常见的非模型原因
- **`pin_memory=True` 让 Host→Device 传输走异步 DMA**，避免一次同步拷贝

### 一个 step 的五步

```
forward → loss → backward → optimizer.step() → optimizer.zero_grad()
```

顺序上有一处容易搞错：`zero_grad()` 放在 `step()` 之后是经典写法，但**放在 `step()` 之前也等价**；真正不能做的是「`backward()` 之前忘了清」。另外 `optimizer.step()` 必须在 `backward()` 之后。

一个训练 step 的五步，以及 `zero_grad` 该放在哪：

```
  ┌─────────┐  ┌──────┐  ┌──────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │ forward │─▶│ loss │─▶│ backward │─▶│ optimizer.step() │─▶│ optimizer.zero_grad()│
  └─────────┘  └──────┘  └──────────┘  └──────────────────┘  └──────────────────────┘
                                             │                          │
                                  必须在 backward 之后        放在 step 之后是经典写法，
                                                             放在 step 之前也等价

  真正不能做的只有一件：`backward()` 之前忘了清零
      ⇒ 上一轮的梯度会混进来（PyTorch 默认累加，不自动清零）
      ⇒ 而这个「默认累加」正是梯度累积所依赖的行为
```

### 学习率调度与 checkpoint

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=100)
# 每个 epoch 结束后调用 scheduler.step()

torch.save({
    'epoch': epoch,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
}, 'checkpoint.pt')
```

> **checkpoint 必须同时存优化器状态。** 只存模型权重的话，断点恢复后动量/二阶动量从零开始，训练曲线会出现明显扰动。**训练显存的大头就是优化器状态**（见 [[02-分布式训练总论与显存账本]]），把它存下来不是可选项。

## GPU 训练

### 混合精度

思路是「草稿用铅笔、定稿用钢笔」：前向反向用低精度提速，参数更新用高精度保精度。

```python
# BF16（推荐，Ampere+，不需要 GradScaler）
with torch.autocast(device_type='cuda', dtype=torch.bfloat16):
    loss = nn.functional.cross_entropy(model(images), labels)
loss.backward()
optimizer.step()

# FP16（需要 GradScaler 防梯度下溢）
scaler = torch.cuda.amp.GradScaler()
with torch.autocast(device_type='cuda', dtype=torch.float16):
    loss = nn.functional.cross_entropy(model(images), labels)
scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

### 显存查看

```python
print(torch.cuda.memory_allocated() / 1024**3)        # 当前已分配
print(torch.cuda.max_memory_allocated() / 1024**3)    # 峰值
print(torch.cuda.memory_summary(abbreviated=True))

torch.cuda.reset_peak_memory_stats()   # 只重置峰值计数器
```

> [!warning] 测混合精度收益时的两个坑
> 1. **`reset_peak_memory_stats()` 不释放已占显存**，只重置峰值计数器。测 BF16 时如果上一轮的 FP32 模型和优化器状态还活着，会被算进 BF16 的峰值里，结果可能算出「节省为负」。必须 `del` 掉对象再 `torch.cuda.empty_cache()`。
> 2. **混合精度省的是激活值，不是权重与优化器状态** —— 后两者仍是 FP32。模型小、batch 小时这部分固定开销占绝对主导，测出来会是「节省 0%」。要把 batch/序列长度放大到激活值占主导，才看得到预期的约 30%–50% 节省。

## 性能分析

`torch.profiler` 回答「每个操作花了多少时间、GPU 利用率如何、哪里在空等」。

```python
from torch.profiler import profile, record_function, ProfilerActivity

with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
             record_shapes=True, profile_memory=True) as prof:
    with record_function("forward"):
        loss = criterion(model(d), l)
    with record_function("backward"):
        loss.backward()
    with record_function("optimizer_step"):
        optimizer.step(); optimizer.zero_grad()

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=15))
prof.export_chrome_trace("trace.json")   # 可在 chrome://tracing 可视化
```

读输出的三个模式：

| 现象 | 含义 |
| --- | --- |
| **CPU total 远大于 CUDA total** | GPU 在等 CPU —— 数据预处理或 Python 开销 |
| **大量小 CUDA kernel** | launch overhead 累积，考虑算子融合 |
| **频繁 CPU-GPU 同步** | `.item()` / `print(tensor)` 会触发同步，阻塞 GPU 流水线 |

第三类最隐蔽，也最容易改：

```python
# 不好：每步都同步
for batch in dataloader:
    loss = train_step(batch)
    print(f"loss: {loss.item()}")        # 每步触发一次同步

# 好：每 100 步记录一次
for i, batch in enumerate(dataloader):
    loss = train_step(batch)
    if i % 100 == 0:
        print(f"step {i}, loss: {loss.item()}")
```

> `.item()` 之所以昂贵，是因为它要把一个 GPU 标量取回 CPU —— 这**要求所有已入队的 GPU 工作先做完**，等于把异步流水线截断。`.cpu()`、`print(tensor)`、`if tensor > 0` 都属于同一类隐式同步。

`.item()` / `.cpu()` / `print(tensor)` 为什么昂贵：

```
  正常情况下，CPU 只负责「排队」，GPU 异步执行：

    CPU  ├─launch─┬─launch─┬─launch─┬─launch─▶   （不停下来，继续往下走）
    GPU           ├──kernel──┼──kernel──┼──kernel──▶

  一旦出现 .item()：

    CPU  ├─launch─┬─launch─┬──[等全部做完]──┬─launch─▶
    GPU           ├──kernel──┼──kernel──────┤    ← 流水线被截断，GPU 空等

  ⇒ `.item()` 要把一个 GPU 标量取回 CPU，这要求**所有已入队的 GPU 工作先做完**
  ⇒ 同一类隐式同步还有：`.cpu()`、`print(tensor)`、`if tensor > 0`
  ⇒ 改法：每 N 步记录一次，而不是每步都记
```

## 相关

- [[05-反向传播与梯度优化]] —— autograd 背后的链式法则与线性层反向公式
- [[01-数值计算与精度]] —— fp16/bf16 的指数位差异与 Loss Scaling 机制
- [[02-分布式训练总论与显存账本]] —— checkpoint 为什么必须存优化器状态
- [[01-GPU 硬件架构与存储层次]] —— CPU–GPU 搬运的带宽差距有多大

## 参考

- https://pytorch.org/docs/stable/
- https://pytorch.org/docs/stable/notes/autograd.html
- https://pytorch.org/docs/stable/profiler.html
- https://pytorch.org/docs/stable/amp.html
- https://pytorch.org/tutorials/recipes/recipes/tuning_guide.html
- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/pyroch/pytorch%E6%A1%86%E6%9E%B6%E5%85%A5%E9%97%A8
