---
tags:
  - AI/infra/CUDA
---

# CUDA 开发环境与第一个 Kernel

这一篇走通两件事：**环境怎么搭**（驱动 / Toolkit / nvcc 三者关系与版本约束），以及**一个 kernel 从 CPU 到 GPU 要经过哪五步**。

## 驱动与 Toolkit 不是一回事

```
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

判据是：**驱动让硬件能工作，Toolkit 让你能写程序。**

### 版本兼容是单向的

NVIDIA 采用**前向兼容**：**新版驱动能跑旧版 Toolkit 编译的程序，反过来不行。** 三条规则：

- 每个 Toolkit 版本有**最低驱动版本要求**
- 驱动版本 ≥ 要求即可，**不必精确匹配**
- `nvidia-smi` 显示的 `CUDA Version` 是驱动**最高支持**的 Toolkit 版本，**不是当前安装的 Toolkit 版本**

| Toolkit 版本 | 最低 Linux 驱动 | 最低 Windows 驱动 |
| --- | --- | --- |
| CUDA 12.6 | ≥ 560.28 | ≥ 560.70 |
| CUDA 12.4 | ≥ 550.54 | ≥ 551.61 |
| CUDA 12.2 | ≥ 535.54 | ≥ 536.25 |
| CUDA 11.8 | ≥ 520.61 | ≥ 520.06 |

> **优先升级驱动，而不是降级 Toolkit。** 新驱动向后兼容旧 Toolkit，也为后续升级留出空间。这条与 [[04-Python、C++ 与 GPU 软件栈工程基本功]] 里「`nvidia-smi` 的 CUDA Version 不代表装了同版本 nvcc」是同一件事的两面。

### Toolkit 里有什么

| 组件 | 作用 |
| --- | --- |
| `nvcc` | CUDA C/C++ 编译器 |
| `cuBLAS` | 线性代数库 |
| `cuDNN` | 深度学习原语库（**需单独下载**） |
| `cuFFT` / `cuRAND` | FFT / 随机数 |
| `Nsight Systems` / `Nsight Compute` | 系统级 / Kernel 级性能分析 |
| `cudart` | 运行时 API 库 |
| `cuda-gdb` | GPU 调试器 |

> [!warning] 安装 Toolkit 时不要勾选 Driver
> 用 runfile 安装时若不取消勾选 Driver，会覆盖已装好的驱动。更稳的方式是 `apt install cuda-toolkit-12-6` —— **不会捆绑驱动**。
>
> `.run` 方式安装驱动前必须先禁用 `nouveau` 并切到文本模式（tty），否则会失败。

安装后配置环境变量：

```bash
export CUDA_HOME=/usr/local/cuda-12.6
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
nvcc --version      # 验证
```

从 CUDA 11.6 起 **Samples 不再随 Toolkit 附带**，要单独 clone。跑通 `deviceQuery` 显示 `Result = PASS` 才算整条链（驱动 + Toolkit + GPU）正常。

## nvcc 是编译协调器，不是编译器

它把 `.cu` 里的**设备代码与主机代码分离**，分别交给 NVIDIA PTX 编译器和系统 C++ 编译器，最后链接成可执行文件：

```
.cu 源文件
  ├─ 设备代码 → ptxas → cubin（GPU 机器码）
  └─ 主机代码 → g++/MSVC → .o
                              ↓
                          链接器 → 可执行文件
```

这条链路解释了后面两个常见现象：**核函数报错会在链接期而不是编译期暴露**；以及 **fatbinary** —— 一个可执行文件里可以打包多个架构的机器码。

### 常用编译选项

| 选项 | 作用 |
| --- | --- |
| `-arch=sm_XX` | 目标 GPU 架构 |
| `-gencode arch=compute_XX,code=sm_XX` | 生成多架构代码（fatbinary），可重复指定 |
| `-G` | 生成设备代码调试信息 |
| `-lineinfo` | **在优化代码中保留行号** —— 性能分析必开 |
| `-Xcompiler` | 把后续选项传给主机编译器 |
| `-maxrregcount=N` | 限制每线程寄存器数（影响 Occupancy，见 `04-Occupancy 与资源分配`） |
| `--ptxas-options=-v` | **打印每个 kernel 的寄存器与共享内存用量** —— 调优第一步 |

```bash
nvcc -arch=sm_89 hello.cu -o hello
nvcc -gencode arch=compute_80,code=sm_80 -gencode arch=compute_89,code=sm_89 hello.cu -o hello
nvcc -g -G -lineinfo hello.cu -o hello_debug
```

### 架构代号速查

| 代号 | 代表 GPU | 计算能力 |
| --- | --- | --- |
| `sm_70` | V100 | 7.0 |
| `sm_75` | T4、RTX 2080 | 7.5 |
| `sm_80` | A100 | 8.0 |
| `sm_86` | RTX 3090 | 8.6 |
| `sm_89` | RTX 4090、L40 | 8.9 |
| `sm_90` | H100 | 9.0 |
| `sm_100` | B200 | 10.0 |

**为错误的 arch 编译是「明明代码对、性能就是差」的常见原因** —— 新架构的指令（Hopper 的 `wgmma`、TMA）不会被生成。

### CMake 集成

CMake 从 3.8 起把 CUDA 当一等语言：

```cmake
cmake_minimum_required(VERSION 3.18)
project(MyCUDAProject LANGUAGES CXX CUDA)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CUDA_ARCHITECTURES 80 89 90)     # 多架构

add_executable(main src/main.cu)
target_link_libraries(main PRIVATE CUDA::cudart)
```

混合 C++ 与 CUDA 源文件时，把 `.cu` 编成静态库、再由 C++ 主程序链接即可：

```cmake
add_library(kernels STATIC src/kernels/vector_add.cu src/kernels/matrix_mul.cu)
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE kernels CUDA::cudart)
```

## 第一个 Kernel 的五步

从 CPU 搬到 GPU，就像把工作从一个员工交给一支千人团队 —— **要准备材料、分配任务、等待完成、收回成果**：

| 步 | API | 作用 |
| --- | --- | --- |
| 1 · 分配设备内存 | `cudaMalloc()` | 类似 `malloc()` |
| 2 · Host → Device | `cudaMemcpy(..., HostToDevice)` | 传输入 |
| 3 · 启动 Kernel | `kernel<<<grid, block>>>(args)` | GPU 并行执行 |
| 4 · Device → Host | `cudaMemcpy(..., DeviceToHost)` | 取结果 |
| 5 · 释放 | `cudaFree()` | 类似 `free()` |

### Kernel 的写法

```cpp
__global__ void vectorAdd(const float* A, const float* B, float* C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;   // 全局线程索引
    if (idx < N) {                                     // 边界检查
        C[idx] = A[idx] + B[idx];
    }
}
```

关键点：

- **`idx = blockIdx.x * blockDim.x + threadIdx.x`** —— 全局索引公式，后面所有 kernel 都用它
- **`if (idx < N)` 不是可选的。** 这是最容易被初学者省略、也最容易出问题的一行

### 为什么必须做边界检查

$N = 1000$、Block 大小 $= 256$：

```
需要 Block 数 = ceil(1000/256) = 4
总线程数     = 4 × 256        = 1024
多出的线程索引 = 1000 ~ 1023   ← 越界
```

**没有检查，这 24 个线程会访问非法地址，导致未定义行为或崩溃。** 向量加法这类规整问题尚且如此，尾块处理在 GEMM、Reduce 里更复杂（见 `06-Reduce 算子优化` 与 `07-GEMM 性能优化`）。

Host 端启动方式：

```cpp
int blockSize = 256;
int gridSize  = (N + blockSize - 1) / blockSize;    // 向上取整的标准写法
vectorAdd<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
```

### 启动是异步的

`kernel<<<...>>>` **立即返回**，CPU 不等 GPU 跑完。这带来两个后果：

1. **计时必须同步。** 直接用 CPU 时钟包住 kernel 启动，测到的是「发射耗时」而不是「执行耗时」。要么在计时前后加 `cudaDeviceSynchronize()`，要么用 CUDA Event：

```cpp
cudaEvent_t start, stop;
cudaEventCreate(&start); cudaEventCreate(&stop);
cudaEventRecord(start);
vectorAdd<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
cudaEventRecord(stop);
cudaEventSynchronize(stop);
float ms = 0; cudaEventElapsedTime(&ms, start, stop);
```

2. **错误可能推迟暴露。** 很多 CUDA API 调用只是把任务入队，报错要到后面某个同步点才出现。所以**每次 API 调用后都应检查返回码**，而 kernel 启动本身的错误要用 `cudaGetLastError()` 抓：

```cpp
#define CUDA_CHECK(call)                                                   \
  do {                                                                     \
    cudaError_t err = call;                                                \
    if (err != cudaSuccess) {                                              \
      printf("CUDA error at %s:%d: %s\n", __FILE__, __LINE__,              \
             cudaGetErrorString(err));                                     \
      exit(EXIT_FAILURE);                                                  \
    }                                                                      \
  } while (0)

CUDA_CHECK(cudaMalloc(&d_A, size));
vectorAdd<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
CUDA_CHECK(cudaGetLastError());          // 抓 kernel 启动错误
CUDA_CHECK(cudaDeviceSynchronize());     // 抓执行错误
```

> **没有错误检查的 CUDA 代码，出错时只会告诉你「结果不对」，不会告诉你错在哪。** 这是把「能跑」变成「能调试」的第一道门槛。

## 相关

- [[04-Python、C++ 与 GPU 软件栈工程基本功]] —— 驱动 / Toolkit / Runtime 的分层与诊断顺序
- [[02-CUDA 编程模型与执行模型]] —— `idx` 公式背后的三级线程层次
- [[03-CUDA 内存模型与访存优化]] —— `cudaMalloc` 分配的是什么内存
- [[01-GPU 硬件架构与存储层次]] —— SM、Warp 与存储层次的硬件侧

## 参考

- https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%BA%8C-cuda%E7%BC%96%E7%A8%8B%E4%B8%8E%E7%AE%97%E5%AD%90%E4%BC%98%E5%8C%96/11-cuda%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA
- https://docs.nvidia.com/cuda/cuda-installation-guide-linux/
- https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/
- https://github.com/NVIDIA/cuda-samples
- https://cmake.org/cmake/help/latest/prop_tgt/CUDA_ARCHITECTURES.html
