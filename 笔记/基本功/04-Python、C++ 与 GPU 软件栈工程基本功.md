---
tags:
  - 基本功/工程能力
---

# Python、C++ 与 GPU 软件栈工程基本功

AI Infra 对语言能力的要求落在**沿调用链跨层追踪问题**上，背语法没有用。一行 `torch.matmul(x, w)` 可能经历：

```
Python API
  └─ 算子分发器（C++）
      └─ cuBLAS / 自定义 CUDA Kernel
          └─ GPU 指令与显存访问
```

**卡在哪一层，决定了该用什么工具。** 这一篇按「层」组织：三层语言各负责什么、Python 的性能陷阱、C++ 的边界与代价、跨语言互操作的契约、以及 GPU 软件栈的分层诊断。

## 一、三层语言各自负责什么

| 层次 | 常用技术 | 优势 | 典型工作 |
| --- | --- | --- | --- |
| **编排层** | Python、Shell | 开发快、生态丰富 | 模型定义、训练脚本、任务编排、数据处理 |
| **系统层** | C/C++ | 可控的内存与性能、易连接底层 API | 框架核心、算子调度、通信库、Python 扩展 |
| **设备层** | CUDA C++、Triton | 显式表达 GPU 并行 | Kernel、算子融合、访存优化 |

三层不是互相替代的关系。**同一个功能在不同层实现，性能与开发成本能差几个数量级** —— 判断「这件事该在哪一层做」是基本功的核心。

### 最小工具箱

```bash
python --version / which python / python -m pip --version
g++ --version / cmake --version
uname -a / ps aux / free -h / df -h
nvidia-smi / nvcc --version
```

这些命令不要求全掌握，但要**知道它们各自回答什么问题**。

> **诊断的第一原则：环境问题出现时，先记录命令、输出、工作目录和版本，不要立刻反复重装。** 可观察的信息越完整，定位越快。一次只改一个变量，保留原始报错。

## 二、Python 的性能陷阱

### 名字绑定，不是变量

Python 的赋值建立**引用**，不复制对象：

```python
a = [1, 2]; b = a
b.append(3)
print(a)          # [1, 2, 3]
print(a is b)     # True：同一个对象
```

`is` 比身份，`==` 比内容。函数参数同样是名字绑定 —— **传的是对象引用** —— 「按值传递」和「按引用传递」这两个传统标签都不准确。

这一套模型解释了三个高频 Bug：

| Bug | 成因 |
| --- | --- |
| **默认可变参数被共享** | 默认参数在**函数定义时**求值一次，`def f(x, acc=[])` 的 `acc` 跨调用复用 |
| **浅拷贝改到了原对象** | `.copy()` 只复制最外层，内层对象仍是同一引用 |
| **多线程共享状态** | 见下 |

正确的默认值写法：用 `None` 表示「未提供」，函数体内再建列表。

> [!warning] 深拷贝不是默认答案
> 对大型张量或模型状态做无意复制会产生巨大内存与时间开销。**更好的做法是先明确：哪些数据需要独占，哪些可以只读共享。**

### GIL 到底限制了什么

CPython 的全局解释器锁保护解释器内部状态：**同一进程中通常只有一个线程能同时执行 Python 字节码**。所以两个纯 Python CPU 循环的线程拿不到两倍速度。

**但这不等于「Python 线程没用」**：

- 阻塞 I/O 时解释器**会释放 GIL**
- NumPy / PyTorch 等原生扩展**在耗时计算时释放 GIL**
- 后台日志、预取、轻量控制任务适合线程

> **GIL 是 CPython 的实现细节**，不应泛化成「所有 Python 实现」或「永远如此」的规则。判断性能要测当前解释器与依赖，而不是背结论。

### 四类任务的并发选型

| 任务类型 | 典型例子 | 常见选择 |
| --- | --- | --- |
| I/O 密集 | 请求服务、读大量小文件 | 线程或 `asyncio` |
| **Python CPU 密集** | 纯 Python 解析、复杂循环 | **多进程或原生扩展** |
| 原生算子密集 | NumPy / PyTorch / CUDA 运算 | 由底层线程池或 GPU 并行，Python 只负责调度 |
| 大数据跨进程 | 数据加载、共享缓存 | 多进程 + 共享内存，谨慎序列化 |

**多进程的成本容易被低估**：进程创建、参数与返回值的序列化/反序列化、内存复制或映射、进程间通信。**每个任务只做几十微秒工作时，调度成本可能高于计算本身** —— 应增大任务粒度，并避免来回传大数组或张量。

> [!warning] CUDA 与多进程的交互
> CUDA 上下文与多进程启动方式组合不当会引发初始化错误或额外显存占用。**不要在父进程初始化 CUDA 之后随意 `fork`** —— 使用框架的数据加载与分布式能力时，遵循框架推荐的启动器和 start method。

`asyncio` 的注意点：协程只在 `await` 处让出控制权。**在事件循环里直接执行阻塞 I/O 或长 CPU 计算会卡住所有协程** —— 应改用异步库，或把阻塞工作放到线程/进程执行器。

## 三、C++ 侧的三个硬概念

| 概念 | 一句话 |
| --- | --- |
| **RAII** | 把资源绑定到对象生命周期 —— 构造获取、析构释放，异常安全 |
| **智能指针** | `unique_ptr` 独占、`shared_ptr` 共享（引用计数）、`weak_ptr` 打破循环 |
| **ABI** | **源码兼容不代表二进制兼容** —— 换编译器、换标准库版本、换 `-D_GLIBCXX_USE_CXX11_ABI` 都可能让已编译的扩展加载失败 |

编译错误、链接错误、运行时装载错误是三件不同的事：

| 阶段 | 典型症状 |
| --- | --- |
| 编译 | 语法、类型、模板实例化失败 |
| 链接 | `undefined reference` —— 符号没找到 |
| 运行时装载 | 缺 `.so`、符号版本不匹配、ABI 不兼容 |

## 四、跨语言互操作的契约

`ctypes` 调稳定的 C ABI；`pybind11` 把 C++ 函数、类和异常映射成 Python 对象。**绑定大型数组时要逐项核对**：

- dtype 是否匹配
- shape 与 stride 是否符合算法假设
- 数据是否**连续**
- 内存位于 **CPU 还是 GPU**
- **谁拥有底层存储**，返回后是否仍有效
- 计算期间是否应该**释放 GIL**
- 原生异常如何转换为 Python 异常

> **所谓「零拷贝」只是共享同一块底层内存**，它要求双方对这份契约达成一致：
>
> ```
> 地址 + 元素类型 + 形状 + 步长 + 设备 + 生命周期 + 可变性
> ```
>
> 任何一项不匹配都可能导致隐式复制、错误结果或越界访问。**转置后的 NumPy 数组可能不是 C 连续布局，只拿首地址按连续数组读会得到错误数据** —— 这是「Python 侧看着对、C++ 侧算错」的经典成因，机制见 [[03-线性代数与 GEMM]] 的 stride 一节。

> **核心原则：把数据布局和所有权当作接口的一部分，而不是实现细节。**

## 五、GPU 软件栈：四个容易混淆的部分

```text
应用 / PyTorch / 自定义扩展
        │
CUDA Runtime 与数学库（cudart、cuBLAS、cuDNN 等）
        │
用户态驱动库（libcuda）
        │
内核态 NVIDIA Driver
        │
GPU
```

| 层 | 是什么 |
| --- | --- |
| **NVIDIA Driver** | 操作系统与 GPU 通信的基础。`nvidia-smi` 反映的**主要是驱动状态** |
| **CUDA Toolkit** | 开发工具集合 —— `nvcc`、头文件、调试分析工具与库 |
| **CUDA Runtime / 数学库** | 应用运行时加载的用户态组件 |
| **cuDNN** | 面向深度学习算子的独立加速库，**不等于整个 CUDA** |

> [!warning] `nvidia-smi` 里的 CUDA Version 不代表装了同版本 nvcc
> 它通常表示**驱动可支持的最高 CUDA 兼容级别**。判断编译工具链要看 `nvcc --version`；判断框架实际用的运行时要看框架自身信息。这三者经常不一致，是「版本都对但就是编不过」的根源。
>
> 另一条相关事实：**运行预编译的框架 wheel 未必需要 `nvcc`** —— 只有要编译自定义扩展时才需要。

### 环境诊断的固定顺序

```bash
nvidia-smi                     # 1. 驱动是否看到设备
which nvcc; nvcc --version     # 2. 编译工具链来自哪里
which python; python -c 'import sys; print(sys.executable)'   # 3. Python 来自哪里
python - <<'PY'                # 4. 框架看到什么
import torch
print("torch:", torch.__version__)
print("built with CUDA:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
print("device count:", torch.cuda.device_count())
PY
```

| 现象 | 优先检查 |
| --- | --- |
| `nvidia-smi` 失败 | 驱动、设备挂载、宿主机状态 |
| `nvidia-smi` 成功但找不到 `nvcc` | Toolkit 未装或 PATH 错；**运行预编译框架未必需要 nvcc** |
| 框架导入成功但 CUDA 不可用 | 装了 CPU 构建、驱动兼容、容器未传 GPU、设备被屏蔽 |
| 自定义扩展编译失败 | 编译器、Toolkit、头文件、**ABI**、目标 GPU 架构 |
| 运行时报缺 `.so` | 动态库搜索路径、wheel/conda 依赖、架构不匹配 |

### 环境隔离的三层

| 工具 | 隔离范围 | 适合 |
| --- | --- | --- |
| `venv` | Python 包与解释器环境 | 纯 Python 项目 |
| `conda` | Python + 一部分原生库与工具链 | 科学计算、复杂二进制依赖 |
| 容器 | 用户态文件系统、依赖、进程环境 | 部署、CI、跨机器复现 |

**容器与宿主机共享内核，只隔离用户态。** GPU 容器通常仍使用**宿主机的 NVIDIA 驱动**，靠 NVIDIA Container Toolkit 把设备与驱动库暴露进容器。

> **不要把一个 `requirements.txt` 当成完整的环境描述。** 锁定直接依赖版本有助于复现，但完整环境还包括操作系统、驱动、编译器、硬件架构和启动参数。

## 相关

- [[03-线性代数与 GEMM]] —— stride 与布局：跨语言传数组为什么会算错
- [[01-PyTorch 框架与训练循环]] —— profiler 怎么定位「GPU 在等 CPU」
- [[01-GPU 硬件架构与存储层次]] —— 硬件层的四个核心指标

## 参考

- **AIInfraGuide 第1章 编程语言基础**（三层语言分工、Python 引用模型与并发选型、RAII 与 ABI、pybind11 与零拷贝的契约、GPU 软件栈四层与诊断顺序）：https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80
- **pybind11 Documentation**：https://pybind11.readthedocs.io/
- **NVIDIA CUDA Installation Guide**（驱动、Toolkit、Runtime 的关系）：https://docs.nvidia.com/cuda/cuda-installation-guide-linux/
- **NVIDIA Container Toolkit**：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/
- **CPython — Global Interpreter Lock**：https://docs.python.org/3/glossary.html#term-global-interpreter-lock
