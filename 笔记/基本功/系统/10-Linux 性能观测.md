---
tags:
  - 基本功/系统
  - 基本功/性能观测
---

# Linux 性能观测

`01-硬件结构与 CPU` 到 `08-I/O 多路复用与 Reactor` 讲内核怎么工作，本篇讲怎么在一台机器上把它看出来。观测的价值不在工具清单，而在把「哪个资源吃满、排了多长的队、错了多少次」变成一条能照着走的判定链：每个字段对应内核里一个计数器，每个计数器指向一个可以改的参数。

网络观测量在内核收发包路径上的定义已经写在 [[14-Linux 内核收发网络包]] §4：`softnet_stat` 的三列、`ethtool -S` 的硬件计数、协议栈的丢包计数器。那篇从「包走到哪一层、在哪里被丢」的机制角度讲；本篇讲另一面——用命令行工具做系统级观测的实操。两篇计数器口径一致，遇到网络丢包时按下文顺序走，落到具体计数器含义时回那篇。

## 1. 观测方法论

排查最容易犯的错是拿到一个工具就用，而不是先问「我怀疑哪个资源、要看它哪一面」。方法论把这个问题固定下来。

### 1.1 USE 与 RED：两套问题清单

USE 方法对**每一类资源**问三个问题（来源：Brendan Gregg, *The USE Method*）：

- **Utilization（利用率）**：资源有多少时间在忙。CPU 看 `%usr+%sys`，磁盘看 `%util`，网卡看 `sar -n DEV` 的 `%ifutil`。
- **Saturation（饱和度）**：有没有请求在排队等这个资源。这是利用率看不出来的那一面——CPU 的 run-queue 长度、磁盘的平均队列深度 `aqu-sz`、网卡的 `txqueuelen` 积压。
- **Errors（错误）**：资源报错了没有。磁盘的 I/O error、网卡的 `rx_crc_errors`、内存的 ECC 计数。

判据：磁盘 `%util` 100% 不一定是瓶颈，若 `aqu-sz` 始终是 1，它只是没有空闲周期。

RED 方法面向**请求驱动的服务**（来源：Tom Wilkie, *The RED Method*）：**Rate** 是每秒请求数，**Errors** 是每秒失败的请求数（含超时与重试），**Duration** 是请求耗时的分布（P50/P95/P99）——用分布而不是平均值，平均值会把尾延迟掩盖掉。

两者的粒度不同：USE 的对象是**资源**，RED 的对象是**服务**。一台机器上跑几十个服务，先用 RED 定位「是哪个端点、哪类请求变慢」，再用 USE 定位「它把哪个资源吃满了」。反过来也成立：RED 三个指标都正常而延迟仍高，问题多半在 USE 的 Saturation 一列——资源没被用满，请求却在排队。

### 1.2 瓶颈定位的顺序

顺序是**先全局后单进程，先资源后应用**。理由在因果链的方向上：应用层的超时、重试、队列堆积，多数是下层资源排队的投影；从异常的应用指标往回头看，容易被中间层症状带走——重试放大流量，放大后的流量又加重下层排队，形成正反馈。三步走：

1. **全局快照**：`top` 或 `vmstat 1` 看 CPU、内存、swap；`iostat -x 1` 看磁盘；`sar -n DEV 1` 看网络。目标先给异常归类。
2. **落到单进程**：`pidstat -u -p <pid> 1` 看该进程吃 CPU 的多少，`-r` / `-d` 同理看内存与磁盘；`top -Hp <pid>` 把线程唤出来。
3. **落到系统调用与内核**：`strace -c -p <pid>` 看系统调用分布，`perf top` 看用户态热点函数，`bpftrace` 挂内核探测点。

一个例子：`top` 报 CPU 高，但 `pidstat` 逐进程看没有一个进程超过 10%。此时瓶颈不在用户进程，而在内核侧——`top` 的 `%si`（软中断）会揭穿它，因为收包软中断跑在中断上下文，不计入任何进程的 CPU 时间（机制见 [[14-Linux 内核收发网络包]] §1.3）。从应用侧起步会一路查到「应用代码没问题」的死胡同。

### 1.3 四个可观测来源

Linux 上能拿到系统状态的地方有四类，侵入性和信息粒度差别很大：

- **`/proc`（procfs）**：内核把每个进程、子系统的状态导成文本文件。读一次就是一次内核侧的快照调用，不改变被观测对象；字段随内核版本变化（来源：man 5 proc）。
- **`/sys`（sysfs）**：设备、驱动、cgroup 的视图，一个文件对应一个内核对象的属性。读是取属性，写入即调用该属性的 `store` 回调——它不只是读，也是配置入口（来源：Linux kernel documentation, *sysfs - The filesystem for exporting kernel objects*）。
- **系统调用（`strace`）**：基于 `ptrace`，让目标进程在每次进入和退出系统调用时停住。代价是进程速度下降数倍，信息也止于 syscall 边界（来源：man 1 strace）。
- **tracepoint 与 BPF（`perf`、`bpftrace`）**：内核在关键路径上预埋的静态探测点，以及可挂在这些点或任意函数上的 kprobe/uprobe。采样模式开销可忽略，适合生产（来源：Linux kernel documentation, *Tracepoints*）。

```text
   四个可观测来源：从左到右侵入性递增，信息粒度递减
   ┌──────────────┬───────────────┬──────────────┬────────────────────┐
   │ 来源          │ 读什么         │ 代价          │ 信息粒度            │
   ├──────────────┼───────────────┼──────────────┼────────────────────┤
   │ /proc /sys   │ 内核导出的计数  │ 一次读文件     │ 粗（只有累计值）     │
   │ perf 采样     │ tracepoint/PMU │ 低，可常开     │ 中（函数级、采样）   │
   │ bpftrace     │ 探测点 + 聚合   │ 低到中         │ 细（参数级、可聚合）  │
   │ strace       │ 系统调用边界    │ 高，降速数倍   │ 细但只到边界         │
   └──────────────┴───────────────┴──────────────┴────────────────────┘
       ▲ 排查顺序：先左后右 —— 便宜的来源答不上来，再动贵的
```

判据：能在 `/proc` 里读到的（丢包计数、队列长度），不要用 `strace` 去抓；`strace` 留给「系统调用卡在哪一个」这类问题，且尽量用 `-c`（只统计不打印）。

### 1.4 四类资源 × USE 三问总表

把 USE 铺成一张速查图，每个资源对着三问找入口：

```text
   总表速查：先定资源，再定 USE 那一列
   资源 ─┬─ CPU  ─┬─ 用满？  mpstat %usr+%sys
         │        ├─ 排队？  vmstat r 列（> 核数即排队）
         │        └─ 错？    dmesg | grep -i mce
         ├─ 内存 ─┬─ 用满？  free 的 available
         │        ├─ 排队？  vmstat si/so（换页=内存不够）
         │        └─ 错？    dmesg | grep -i "out of memory"
         ├─ 磁盘 ─┬─ 用满？  iostat -x %util
         │        ├─ 排队？  iostat -x aqu-sz / await
         │        └─ 错？    dmesg | grep -iE "I/O error"
         └─ 网络 ─┬─ 用满？  sar -n DEV %ifutil
                  ├─ 排队？  softnet_stat dropped / ss Send-Q
                  └─ 错？    ethtool -S 的 rx_crc_errors / nstat
```

表里每类资源的「用满」一列都是最便宜的全局命令，「排队」和「错」才需要更细的工具，这解释了 §1.2 的定位顺序。

## 2. 网络性能指标

网络的性能由一组互相约束的指标定义。带宽、吞吐、PPS 描述容量，延迟、抖动、丢包描述质量，连接数与重传描述连接稳定性。

**带宽（bandwidth）与吞吐（throughput）。** 带宽是链路的理论最大速率，单位 `b/s`（比特每秒），由硬件协商决定：`ethtool eth0 | grep Speed` 报的就是它，千兆网卡是 `1000Mb/s`，万兆是 `10000Mb/s`（来源：man 8 ethtool）。吞吐是实际传输速率，单位可以是 `b/s` 或 `B/s`，一定不超过带宽。读命令时要留意单位——网卡速率用比特，`sar` 的吞吐用 `kB/s`（字节），1 `Gb/s` 约等于 125 `MB/s`，把 `125000kB/s` 当成「远未到千兆」是常见误判。

**PPS（Packet Per Second）。** 以包为单位的转发速率，衡量系统对包的转发能力。它与带宽是两种不同的天花板：同样 1 Gb/s 的网卡，跑 1500 字节的大包，约 8.1 万 pps 就打满带宽；跑 64 字节的小包，每包还要带以太网首部、帧间隙与前导码（约 20 字节开销），线速要求约 148.8 万 pps——此时 PPS 先到顶，带宽利用率上不去（推导见 §5.2）。

**延迟（latency）与抖动（jitter）。** 延迟是一次请求从发出到收到响应的时间，网络场景里通常指 RTT（round-trip time，往返时延）。它分几层看：`ping` 报的 `time` 是 ICMP 往返时延（口径见 `09-ICMP 与 ping`）；TCP 的 RTT 由内核按 ACK 往返估计，`ss -ti` 的 `rtt` 就是这个值。抖动是延迟的波动幅度，`ping` 输出的 `mdev` 就是往返时延的标准差；均值 10 ms、抖动 50 ms 的链路对音视频的破坏大于稳定 30 ms 的链路。

**丢包率（packet loss）与重传率（retransmission）。** 丢包率是丢弃包占发送包的比例，从 `ping` 的 `packet loss` 看端到端，从 `ethtool -S` 与 `/proc/net/softnet_stat` 看本机在哪一层丢（见 §3.2）。重传率是重传段占已发段的比例，`ss -ti` 的 `retrans` 给出累计重传段数，`nstat` 的 `TcpRetransSegs` 给出系统累计。判据：稳态下重传率低于 0.1% 属正常局域网水平，持续高于 1% 说明链路或对端有问题，此时先查物理层而不是调协议栈参数。

**连接数与并发（connections）。** 当前 TCP 连接数、各状态分布、监听队列长度。`ss -s` 给汇总，`/proc/net/sockstat` 给 `inuse`/`tw` 分解（见 §4）。判据：`TIME_WAIT` 数量大是短连接频繁的表现，不等于故障；`orphan`（孤儿 socket）大量增长才指向应用没有正确 `close()`。

这些指标并不独立：丢包触发重传、重传抬高延迟、延迟又拖慢吞吐。看到吞吐上不去，先确认丢包和重传，再看带宽——顺序反了会把「因为丢包导致的慢」误判成「带宽不够」。各计数器口径见 [[14-Linux 内核收发网络包]] §4。

## 3. 网络配置与网卡计数

看网络从最底下两层开始：先用 `ip` 看接口的配置和链路层统计，再用 `ethtool` 看网卡驱动与硬件的计数器。

### 3.1 ip addr / ip route / ip -s link

`ip` 属于 `iproute2` 软件包，仍在维护；`ifconfig`、`netstat`、`route` 属于 `net-tools`，已经不再更新，新系统里默认不安装（来源：man 8 ip）。三条最常用的子命令：

- **`ip addr`（简写 `ip a`）**：看接口的 IP、子网掩码、MAC、MTU 与状态标志。状态里出现 `LOWER_UP` 表示物理链路已连通（`ifconfig` 里对应的标志是 `RUNNING`）；只有 `UP` 没有 `LOWER_UP`，说明接口被 `up` 了但没有载波，通常是没插网线或对端端口关闭。
- **`ip route`**：看路由表。默认路由形如 `default via <网关> dev <接口>`；`ip route get <目标 IP>` 能直接问内核「发往这个地址会走哪条路由、从哪个源地址出」。
- **`ip -s link show eth0`**：看链路层收发统计。`RX` 段给出 `bytes packets errors dropped overrun mcast`，`TX` 段给出 `bytes packets errors dropped carrier collsns`。

MTU 默认 `1500` 字节，超过就由 IP 层分片；容器网络、隧道网络常要调小（`ip link set eth0 mtu 1400`）。统计字段里 `errors`、`dropped`、`overrun`、`carrier`、`collisions` 任一持续增长都说明这一层出了问题，展开在 §3.2。

### 3.2 ethtool -S 与 dropped / errors 的分层

`ip -s link` 给的是链路层汇总，`ethtool -S eth0` 给的是驱动和网卡硬件的原始计数，名字随芯片不同但语义可对应（来源：man 8 ethtool）：

- **`rx_fifo_errors`**（链路层汇总里的 `overrun`）：网卡 FIFO 溢出，收包速度超过网卡内部缓冲的处理速度，属于硬件来不及。
- **`rx_missed_errors`**：DMA 已把帧写进 RX 环，但主机没在环写满前取走，网卡只能丢弃，属于驱动或中断来不及。
- **`rx_no_buffer_count`**：RX 环里没有可用的描述符缓冲区交给网卡。
- **`rx_length_errors` / `rx_crc_errors` / `rx_frame_errors`**：物理层或链路问题（线缆、光模块、双工不匹配），这三类增长通常与协议栈参数无关，先查硬件。

```text
   收包路径上的丢包点与对应计数器（自左向右）
   网线 ─▶ 网卡 FIFO ─▶ RX ring ─▶ backlog 队列 ─▶ socket 缓冲 ─▶ 应用
              │             │            │                │
        rx_fifo_errors  rx_missed_   softnet_stat     RcvQDrop /
        (overrun)       errors /     第 2 列 dropped  RcvbufErrors
                        rx_no_buffer
              ▼             ▼            ▼                ▼
        硬件来不及      环太小/中断   内核取包跟不上   应用读得太慢
                        太慢           → netdev_max_  → 改应用
                        → ethtool -G   backlog
```

判读顺序是「网卡 → 内核 → 协议栈 → 应用」：先看 `ethtool -S` 有没有硬件层错误，再看 `/proc/net/softnet_stat` 有没有内核队列溢出，再看 `nstat` 的协议栈丢包，最后看 `ss` 的 socket 队列。每一层独立计数，上层的症状会掩盖下层根因——只在最上层调参，效果往往是把丢包推迟到下一处。各计数器的完整含义和对应参数见 [[14-Linux 内核收发网络包]] §4。

## 4. socket 观测

接口统计只到链路层，协议栈和 socket 的状态要用 `ss`，汇总计数在 `/proc` 里有原始形式。

### 4.1 ss 的三种用法与 /proc/net/sockstat

`ss` 与 `netstat` 显示的字段大体相同（状态、接收队列 `Recv-Q`、发送队列 `Send-Q`、本地地址、远端地址、进程），但在繁忙机器上应优先用 `ss`（来源：man 8 ss）。差别在取数方式：`netstat` 逐个读取并解析 `/proc/net/tcp`、`/proc/net/udp` 等文本文件，每个文件是一个顺序遍历内核结构的接口，连接数上万时既要遍历又要解析文本；`ss` 通过 `netlink` 的 `NETLINK_SOCK_DIAG` 接口向内核请求 socket 列表，内核按 socket 表批量填充二进制结构直接返回，少了文本序列化和解析两步（来源：man 8 ss；man 5 proc）。

三个最常用的形态：

- **`ss -s`**：汇总。输出 `Total` 总 socket 数，`TCP` 一行给出 `estab`（已建立）、`closed`、`orphaned`（孤儿）、`timewait` 等计数。要展开的协议栈级统计（`active/passive openings`、`segments send/received`），用 `nstat` 更合适。
- **`ss -lntp`**：列出监听中的 TCP socket（`-l` 监听、`-n` 不解析名字、`-t` TCP、`-p` 显示进程）。关键是 `Recv-Q` / `Send-Q` 两列在**监听态**与**连接态**含义不同。监听态下，`Recv-Q` 是当前已完成三次握手、还没有被 `accept()` 取走的连接数，`Send-Q` 是这个队列的上限；连接态下，`Recv-Q` 是内核缓冲区里没被应用读走的字节数，`Send-Q` 是已发出但还没被对端确认的字节数。内核里对监听 socket 取的是 `sk_ack_backlog` 与 `sk_max_ack_backlog`（来源：Linux kernel source, `net/ipv4/tcp_diag.c` 的 `tcp_diag_get_info()`），也就是 `06-TCP 的边界与失败模式` 里的全连接队列。
- **`ss -ti`**：`-i` 展开 socket 的内部信息，`-t` 限定 TCP。每行尾部追加一段 TCP 内部状态，字段取自内核的 `struct tcp_info`（来源：Linux kernel source, `include/uapi/linux/tcp.h`）。

```text
   ss -ti 一行里关键字段的来源与读法
   ┌────────────────┬──────────────────────────────┬────────────────────┐
   │ 字段            │ 含义                          │ 内核域              │
   ├────────────────┼──────────────────────────────┼────────────────────┤
   │ rtt:0.24/0.05  │ 平滑 RTT / RTT 方差（毫秒）    │ tcpi_rtt(rttvar)   │
   │ cwnd:10        │ 拥塞窗口（段，非字节）         │ tcpi_snd_cwnd      │
   │ retrans:0/12   │ 当前重传中 / 累计重传段数      │ tcpi_retrans /     │
   │                │                              │ tcpi_total_retrans │
   │ unacked:3      │ 已发未确认的段数               │ tcpi_unacked       │
   │ lost:0         │ 判定为丢失的段数               │ tcpi_lost          │
   │ ssthresh:7     │ 慢启动阈值                    │ tcpi_snd_ssthresh  │
   └────────────────┴──────────────────────────────┴────────────────────┘
```

判据：`rtt` 单位是微秒，`ss` 显示时换算成毫秒；`retrans` 第二个数（累计重传）在稳态连接上应几乎不涨，持续增长说明丢包；`cwnd` 卡在很小的值且不增长，多半被丢包或零窗口压住，回到 §2 对照重传率与延迟。监听态下 `Recv-Q` 接近 `Send-Q` 是危险信号——accept 队列快满了，应用 `accept()` 跟不上，连接随时会被丢。

`ss -s` 的汇总在 `/proc/net/sockstat` 里有原始形式，TCP 行是 `TCP: inuse <n> orphan <n> tw <n> alloc <n> mem <n>`，各字段的构造在内核里是（来源：Linux kernel source, `net/ipv4/proc.c` 的 `sockstat_seq_show()`）：`inuse` 是 `sock_prot_inuse_get(net, &tcp_prot)`，当前处于「使用中」状态的 TCP socket 数；`orphan` 是 `tcp_orphan_count_sum()`，已没有文件描述符引用、但仍在内核里存在的 socket 数（应用 `close()` 后若还有未完成 I/O，socket 会先进 orphan）；`tw` 是 `TIME_WAIT` 状态的 socket 数；`alloc` 是已分配的 TCP socket 总数；`mem` 是 TCP 协议占用的内存页数。这五个数一起看能定性：`inuse` 接近 `alloc` 说明 socket 大多在使用中；`orphan` 持续增长说明应用没有及时回收连接；`tw` 高是短连接频繁的结果；`mem` 增长指向 socket 缓冲区占用（见 §8 的 `rmem_max` / `wmem_max`）。

## 5. 吞吐、PPS 与压测

§2 把指标立住了，这一节讲怎么把它们量出来。

### 5.1 sar -n DEV 各列怎么读

`sar -n <类型> [间隔] [次数]` 把网络统计按间隔刷新，三类常用（来源：man 1 sar）：**`sar -n DEV 1`** 按接口给吞吐与 PPS；**`sar -n EDEV 1`** 按接口给错误统计（`rxerr/s`、`txerr/s`、`rxdrop/s`、`coll/s` 等）；**`sar -n TCP 1`** 给协议栈层统计（`active/s`、`passive/s`、`iseg/s`、`oseg/s`）。

`-n DEV` 的列：`IFACE` 接口名，`rxpck/s`、`txpck/s` 是收发 PPS（包/秒），`rxkB/s`、`txkB/s` 是收发吞吐（千字节/秒），`rxcmp/s`、`txcmp/s` 是压缩包数，`rxmcst/s` 是组播包数，`%ifutil` 是接口利用率——内核估算的「实际流量占链路速率」的比例。判据：`%ifutil` 接近 100% 时吞吐才真正被带宽限制；若 `%ifutil` 很低而应用却报慢，瓶颈不在带宽，去查延迟、丢包或对端。

### 5.2 带宽与 PPS 的双重上限

同一张网卡有两个天花板：**带宽上限**和 **PPS 上限**，哪个先到顶取决于包的大小。以千兆网卡（1 Gb/s）为例算一遍。以太网上一个「最小帧」传到线路上需要 64 字节帧加上 7 字节前导码和 1 字节帧起始定界符，再加 12 字节帧间隙，共 84 字节，换算成比特是 672 bit。1 Gb/s 的线速下，每秒能传的包数上限是：

```text
   1_000_000_000 bit/s ÷ 672 bit/包 ≈ 1_488_095 包/s
```

约 148.8 万 pps，这是千兆网卡在最小包下的 PPS 天花板。再看大包：1500 字节帧加 20 字节固定开销共 1520 字节，换算 12160 bit，则 pps 上限约 8.2 万，此时 8.2 万 × 1500 字节 ≈ 1 Gb/s，恰好把带宽打满。两种极端的对照：

| 包大小（字节） | 每包线路开销（字节） | 线速 PPS 上限（千兆） | 打满带宽时 | 谁是瓶颈 |
| --- | --- | --- | --- | --- |
| 64（最小） | 20 | ≈ 148.8 万 | 8.6% 带宽 | PPS |
| 512 | 20 | ≈ 23.5 万 | 96% 带宽 | 接近带宽 |
| 1500（最大） | 20 | ≈ 8.2 万 | 100% 带宽 | 带宽 |

```text
   带宽上限与 PPS 上限：小包时 PPS 先到顶
   吞吐 (Gb/s)
      1.0 ├───────────────────────────● 大包：带宽先满
          │                        ╱
          │                     ╱
          │                  ╱   ← 小包：PPS 先到顶，
          │              ╱         带宽利用率上不去
      0.086 ●─────────╱
          └────┬───────┬──────────┬──▶ 包长 (字节)
              64      512       1500
```

这也解释了一个常见现象：机器收小包（心跳、DNS、ACK 密集）到每秒上百万包时，`%ifutil` 才 8%，但 CPU 已经跑满在软中断上——瓶颈是 PPS 和每包的处理开销，不是带宽。此时该做的是减少中断（中断合并、多队列），不是加带宽。相关机制在 [[14-Linux 内核收发网络包]] §1。

### 5.3 实时工具与压测工具

要知道「这些流量是谁的」，用三个实时工具：**`iftop`** 按连接对显示带宽，适合定位「哪对 IP 在占带宽」；**`nload`** 按设备显示进出总吞吐和曲线；**`nethogs`** 按**进程**显示带宽。

要回答「这条链路或这台服务能到多少」，用压测工具。**`iperf3`** 测端到端网络路径的吞吐，一端 `-s` 起服务端，一端 `-c <host>` 发起，`-u` 切 UDP 看丢包与抖动，`-P` 并发多流，`-t` 定时长（来源：iperf 官方文档）。它测的是「内核网络栈 + 链路」的净吞吐，不含业务逻辑。**`wrk` 与 `ab`** 测 HTTP 服务，包含应用处理与协议栈；`ab` 是单线程、每连接一次请求的模型，会把客户端自身变成瓶颈，`wrk` 是多线程加事件驱动。判据：先用 `iperf3` 确认网络不是瓶颈，再用 `wrk`/`ab` 测服务；反过来会把应用的慢误算到网络上。

## 6. 连通性与延时

连通性和延时是网络体验最直接的两个指标。这一节按「先确认通不通、再拆解延迟构成、最后抓包看细节」的顺序组织。

### 6.1 ping 与 mtr / traceroute

`ping <host>` 发 ICMP Echo 并汇总。输出里 `icmp_seq` 是序列号（丢包时会跳号），`ttl` 是生存时间，`time` 是往返时延，末尾的 `rtt min/avg/max/mdev` 是时延的最小/平均/最大/标准差（来源：man 8 ping）。判据：`mdev` 相对 `avg` 偏大说明抖动大；`packet loss` 非零说明有丢包。但 `ping` 不通不代表服务不通——防火墙常直接丢弃 ICMP，`ping` 失败时下一步该用 `curl` 或 `nc` 测目标端口。

`mtr <host>` 把 `ping` 和 `traceroute` 合起来：逐跳显示每一跳的延迟和丢包，并持续刷新（来源：man 8 mtr）。它的价值在区分「丢包发生在哪一跳」——若丢包从某一跳开始并且后面每一跳都丢，是本跳之后的问题；若只有中间某一跳丢而后面恢复正常，那一跳多半只是不响应探测（ICMP 限速），不等于真丢包。

### 6.2 curl -w 分段计时

`ping` 只测到网络层，HTTP 请求慢在哪一段要用 `curl -w` 拆开。`curl` 通过 `-w` 输出一组计时变量，各段首尾相接（来源：curl 官方手册 `-w, --write-out`）：

- **`time_namelookup`**：DNS 解析耗时，从开始到域名解析完成。
- **`time_connect`**：从开始到 TCP 三次握手完成。减去 `time_namelookup` 就是纯 TCP 握手时间。
- **`time_appconnect`**：从开始到 TLS 握手完成。
- **`time_starttransfer`**：从开始到**第一个字节**到达（TTFB）。减去 `time_pretransfer` 得到服务端处理时间。
- **`time_total`**：从开始到全部数据读完。

```text
   curl 一次 HTTPS 请求的分段计时（各段首尾相接）
   │
   ├─ time_namelookup ─┤                              DNS 解析
   │                    ├── time_connect ──┤          TCP 握手
   │                                        ├── time_appconnect ──┤  TLS 握手
   │                                                              ├ time_pretransfer ┤
   │                                                                                ├ time_starttransfer ┤  首字节
   │                                                                                                    ├─ time_total ─┤
   t=0 ─────────────────────────────────────────────────────────────────────────────────────────────▶ 时间
```

判据：`time_namelookup` 大是 DNS 问题（换解析器或加缓存）；`time_connect - time_namelookup` 大是 TCP 握手慢（RTT 高或 SYN 丢包）；`time_appconnect - time_connect` 大是 TLS 握手慢；`time_starttransfer - time_pretransfer` 大是服务端处理慢；`time_total - time_starttransfer` 大是响应体大或下游带宽受限。一次请求的耗时被拆成四段，每段指向不同的负责人。

### 6.3 tcpdump

`tcpdump` 抓包看的是「线上到底发了什么」。用法核心是**过滤表达式**（来源：man 7 pcap-filter）：按主机 / 网段 `host 10.0.0.1`、`net 10.0.0.0/24`；按端口 `port 443`、`src port 80`；按协议 `tcp`、`udp`、`icmp`、`arp`；按标志位 `tcp[tcpflags] & (tcp-syn) != 0` 抓 SYN。组合用 `and`、`or`、`not`，例如 `tcpdump -ni eth0 'host 10.0.0.1 and port 443'`。

落盘用 `-w file.pcap`，`-r file.pcap` 回读，`-c <n>` 抓够 n 个包就停。代价要算清楚：抓包要把每个包从内核复制到用户态并写文件，包速率高时 `tcpdump` 自身会成为瓶颈并丢包（结束时报告 `packets dropped by kernel`）。大流量排查更稳的做法是按过滤表达式缩小范围，或用 `-s <snaplen>` 只抓包头的若干字节。判据：看到 `packets dropped by kernel` 非零，说明抓包丢包了，结论要以过滤后的那部分为准。

## 7. 日志分析

日志分析的素材是几万到几亿行的大文件。多数人的第一反应是 `cat` 或 `grep`，问题恰恰出在这里——几条看似无害的命令在大文件上会把机器拖垮。这一节从「别把机器打爆」出发，把 PV、UV、TOP 分析做成几条可控的流水线。

### 7.1 cat 的性能陷阱与 wc / head

`cat big.log | grep x` 与 `grep x big.log` 结果一样，代价不一样。管道版本起一个额外的 `cat` 进程，并让数据从 `cat` 经管道缓冲区转到 `grep`。管道缓冲区默认约 64 KB，内核每搬 64 KB 就要写端一次 `write`、读端一次 `read`，加上两进程之间的调度切换；数据还被多拷贝一次（`read` 到 `cat` 的用户缓冲，再 `write` 进管道，再被 `grep` 读出）。直接 `grep x big.log` 只有一次读取。文件越大、匹配越稀疏，多出来的开销越明显。

`cat` 大文件还会把内容全推到终端，终端渲染慢于磁盘，会卡住终端并占满 CPU。看大文件用 `less`（按需加载一屏，翻页时才继续读，来源：man 1 less）；看末尾用 `tail`，实时跟踪用 `tail -f`（轮转后用 `tail -F`）。

统计总行数有个细节：`wc -l` 有专门的换行计数路径，一次扫描只数换行字节；`grep -c '^'` 要用正则引擎逐行判断，大文件上更慢。判据：只关心行数用 `wc -l`。但 `wc -l` 统计的是换行符个数，文件最后一行若没有结尾换行，它会少算 1（来源：man 1 wc）。`head -n N` 读够 N 行就退出并关闭管道，上游进程收到 `SIGPIPE` 会停下，所以它在大文件上优于 `sed -n '1,Np'`（后者读完 N 行后仍会继续读到文件尾，除非加 `q`）。

### 7.2 awk 做 PV 分析与 sort 的内存行为

nginx 的 `access.log` 每行是一次访问，默认格式是 `$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"`。以空格分隔后，`$1` 是客户端 IP，`$4` 是带方括号的本地时间，`$7` 是请求路径，`$9` 是状态码，`$12` 是 User-Agent。PV（Page View）等于日志行数，`wc -l access.log` 一次给出总量。按天分组看 `$4` 里的日期：`$4` 形如 `[18/Sep/2026:13:20:01`，用 `substr($4, 2, 11)` 从第 2 个字符起取 11 个字符，得到 `18/Sep/2026`。经典流水线是 `awk` 取字段、`sort` 排序、`uniq -c` 计数：

```text
   PV 按天分组：每一段把上一段的输出当输入
   access.log
     │ awk '{print substr($4,2,11)}'       取出日期列
     ▼
   日期流（每天一行，共 N 行）
     │ sort                                使相同日期相邻
     ▼
   有序日期流
     │ uniq -c                             相邻重复行合并并计数
     ▼
   每天 PV 计数
```

`uniq` 的去重原理是比较**相邻**行、只保留连续重复行的一份，所以它前面必须先 `sort`，否则相同日期的行不相邻，去重就是错的（来源：man 1 uniq）。

`sort` 在数据量超过内存缓冲时会退化：默认拿出一块内存做内存排序，超出后改用**外部归并排序**，把已排序的块写到临时文件再归并。临时文件默认落在 `$TMPDIR` 或 `/tmp`，如果这些目录在内存文件系统上（如 `tmpfs`），大排序会直接吃内存（来源：man 1 sort）。三个控制手段：**`-S <size>`** 指定主内存缓冲上限（如 `sort -S 2G`），**`-T <dir>`** 把临时文件指到磁盘而不是 `tmpfs`，**`LC_ALL=C`** 关闭区域设置的多字节排序规则、按字节比较，能快几倍——默认区域下 `sort` 要按本地语言的字符序比较，开销远大于逐字节比较（来源：man 1 sort）。日志分析这类只需要稳定排序的场景都该加上它。

### 7.3 UV 去重与超大基数

UV（Unique Visitor）是去重后的访问人数。`access.log` 没有用户身份，通常用客户端 IP 近似：`awk '{print $1}' access.log | sort -u | wc -l`。按天分组的 UV 要把「日期 + IP」作为去重键，再按日期计数：

```text
   awk '{print substr($4,2,11), $1}' access.log \
     | sort -u \
     | awk '{uv[$1]++} END {for (d in uv) print d, uv[d]}'
```

第一段给每行拼出「日期 IP」，`sort -u` 让相同的「日期 IP」只剩一条，最后一段用 awk 的关联数组按日期累加。`END` 是 awk 在所有输入处理完后才执行的触发器，`uv[$1]++` 靠的是 awk 的关联数组——它的内存占用随**不同键**的数量增长（来源：POSIX awk 规范对关联数组与 `END` 的定义）。

大数据量下的取舍由此而来。`sort -u` 与 `awk '!seen[$0]++'`（读到新行时输出、见过就不输出）都需要把「已见过的键」放进内存：`sort` 是排序后比较相邻行，`awk` 是维护一张哈希表。基数百万级两种方式都能扛；基数到亿级时，哈希表或排序缓冲区就会撞上内存上限。超大基数用两种近似结构：

- **布隆过滤器（Bloom filter）**：只回答「某个元素在不在集合里」，用固定大小的位数组换空间，适合「过滤已经处理过的键」。
- **HyperLogLog**：只估计「集合有多少个不同元素」，不判断成员。用 16384 个 6 位寄存器（约 12 KB）就能把标准误差控制在约 0.81%（`1.04/√16384`）（来源：Flajolet 等, *HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm*, 2007）。Redis 的 `PFADD`/`PFCOUNT` 就是它的实现，详见 `02-数据类型与底层数据结构`。

判据：要**精确**的 UV 且基数在内存可承受范围内，用哈希或排序去重；只要**数量级**、基数可能到亿级，用 HyperLogLog；要**成员判定**（是否访问过），用布隆过滤器。

### 7.4 终端分析、TOP3、分片并行与 journalctl

默认空格分隔会把 UA 里的空格拆碎，分析 User-Agent 要按双引号分隔：`awk -F'"' '{print $6}'` 取第 6 个引号段，正是 UA。统计终端种类：

```text
   awk -F'"' '{print $6}' access.log | sort | uniq -c | sort -rn | head
```

`sort -rn` 的 `-r` 是逆序、`-n` 是按数值排序，缺了 `-n` 会按字符串比较（`9` 会排在 `10` 前面）。**TOP3 请求路径**是同一条流水线换个字段：

```text
   awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -n 3
```

`grep` 在流水线里有两个提速点：`-F` 把模式当固定字符串而不是正则，省掉正则引擎；`-m <n>` 找到 n 条匹配就退出，省掉后续读取（来源：man 1 grep）。**`grep --line-buffered`** 解决管道里的缓冲问题：`grep` 输出到管道时默认按块缓冲，导致 `tail -f access.log | grep x` 看不到实时结果；加 `--line-buffered` 让它每行都刷（来源：man 1 grep）。

**大文件处理的原则**收成四条：流式处理（一次遍历、不把整个文件读进内存）、`LC_ALL=C`、避免 `sort` 爆内存（`-S`/`-T`）、必要时用 `split` 分片并行：

```text
   split -l 1000000 access.log part_ && \
     ls part_* | xargs -P 4 -I{} sh -c "awk '{print \$7}' {} | sort | uniq -c" \
     | awk '{a[$2]+=$1} END {for (k in a) print a[k], k}' | sort -rn | head -3
```

每个分片独立做 TOP 统计，最后把各分片的计数按路径合并。`split` 与 `xargs -P` 的组合把一次遍历变成多核并行（来源：man 1 split；man 1 xargs 的 `-P` 并行选项）。

systemd 系统上服务日志进的是 `journald`，用 `journalctl` 读（来源：man 1 journalctl）：**`-u <unit>`** 只看某个服务；**`--since "1 hour ago"` / `--until`** 按时间过滤；**`-p err`** 按优先级过滤，`-p` 取 `0`（emerg）到 `7`（debug）的级别名或数字，`-p err` 即 `3` 及以上；**`-f`** 实时跟踪，**`-n <n>`** 只看最后 n 行。判据：`journalctl -u nginx --since today -p err` 能直接把「今天的错误」筛出来；不限制 `-u` 和 `--since` 时它会扫全部日志，在大机器上同样昂贵。

## 8. 参数与观测

调参的前提是先能观测到对应的计数器。这一节把观测里最常需要动的四个内核参数按「默认值 → 语义 → 什么时候改 → 改大改小的后果 → 联动 → 失败模式」六项写齐。默认值取自内核文档与源码，发行版常在自己的 sysctl 配置里覆盖，动手前以本机 `/proc/sys` 的实际值为准。

**`net.core.netdev_max_backlog`：收包队列的长度。**

- **默认值**：1000（来源：Linux kernel documentation, *Documentation/admin-guide/sysctl/net.rst*）。
- **语义**：单个 CPU 的收包队列（backlog）能容纳的包数上限。接口收包快于内核处理时，多余包进这个队列；队列满则丢，计入 `/proc/net/softnet_stat` 第 2 列。
- **什么时候改**：`softnet_stat` 第 2 列 `dropped` 随流量上涨，而 `ethtool -S` 的 `rx_missed_errors` 不涨——说明环没满，是内核处理队列溢出。
- **改大 / 改小的后果**：改大能吸收更大突发、减少丢包，代价是该 CPU 上积压更多包，端到端延迟上升、内存占用增加；改小则突发更容易丢，但排队延迟低。
- **联动**：与 `net.core.netdev_budget`（单轮处理的包数上限）联动。队列够大而预算不够时，包在队列里多停留，问题从「丢」变成「延迟」，`softnet_stat` 第 3 列 `time_squeeze` 上涨。也受多队列 / RPS 影响。
- **失败模式**：只调大本参数不动预算，表现为丢包变成延迟；把值设得极大，高突发下可能耗尽内存。
- **改法与验证**：`sysctl -w net.core.netdev_max_backlog=16384`，落盘到 `/etc/sysctl.d/` 持久化；验证看 `softnet_stat` 第 2 列的**增速**是否下降，累计值不会回落。

**`net.core.rmem_max` / `net.core.wmem_max`：socket 缓冲区上限。**

- **默认值**：两者均为 4194304 字节（4 MiB）（来源：Linux kernel documentation, *Documentation/admin-guide/sysctl/net.rst*）。
- **语义**：`rmem_max` 是单个 socket 接收缓冲区的上限，`wmem_max` 是发送缓冲区的上限。应用若用 `setsockopt(SO_RCVBUF)` 显式设置，值会被本参数截断；TCP 的自动调优（`net.ipv4.tcp_rmem` / `tcp_wmem` 的第三个值）也受本参数约束（来源：Linux kernel documentation, *Documentation/networking/ip-sysctl.rst*）。
- **什么时候改**：高带宽高延迟链路（长肥管道，BDP 大）上单连接吞吐上不去，`ss -ti` 的 `cwnd` 已顶到上限、`Recv-Q`/`Send-Q` 常年非零，说明窗口被缓冲区限制。
- **改大 / 改小的后果**：改大让单连接容纳更多未确认数据、吞吐上限提高，代价是每个连接占更多内存，连接数很多时内存增长明显；改小省内存，但单连接吞吐被压住。
- **联动**：应用层 `SO_RCVBUF`/`SO_SNDBUF` 与 `net.ipv4.tcp_rmem`/`tcp_wmem` 三者一起决定实际窗口，只改一个通常没用——应用显式 `setsockopt` 会覆盖自动调优。
- **失败模式**：只调大 `rmem_max` 而应用仍用默认 `SO_RCVBUF`，或 `tcp_rmem` 的 max 仍是旧值，窗口不会变大；连接数上百万时大缓冲区可能触发 TCP 内存压力（`nstat` 的 `TcpExtTCPMemoryPressures` 增长）。
- **改法与验证**：`sysctl -w net.core.rmem_max=16777216 net.core.wmem_max=16777216`；验证看单连接 iperf3 吞吐是否上升。

**`vm.dirty_ratio` 与 `fs.file-max`：回写与文件句柄。**

- **默认值**：`vm.dirty_ratio` 为 20，`vm.dirty_background_ratio` 为 10，`vm.dirty_expire_centisecs` 为 3000（30 秒），`vm.dirty_writeback_centisecs` 为 500（5 秒）（来源：Linux kernel source, `mm/page-writeback.c`）。`fs.file-max` 不是常数，由 `files_maxfiles_init()` 按内存算：`max_files = max((可用内存页数 × 4) / 10, 8192)`，约每 10 KB 内存对应一个文件句柄，下限 8192（来源：Linux kernel source, `fs/file_table.c`）。
- **语义**：`dirty_ratio` 是「产生脏页的进程自己被强制同步回写」的阈值（占可用内存的百分比），`dirty_background_ratio` 是后台回写线程开始工作的阈值。`file-max` 是全系统能同时打开的文件句柄总数。
- **什么时候改**：写入大的机器上 `%iowait` 周期性飙升、`vmstat` 的 `bo` 呈尖峰、应用出现秒级写停顿，多半是脏页集中回写造成的，可下调 `dirty_ratio` 或改用字节阈值 `dirty_bytes`；`fs.file-max` 在 `dmesg` 报 `VFS: file-max limit reached` 时才需要调。回写机制与 [[08-文件系统与 IO 栈]] 的 PageCache 一节是同一件事，那里讲机制，这里只给观测入口。
- **改大 / 改小的后果**：`dirty_ratio` 调大让内核攒更多脏页再回写，顺序写吞吐可能更高，但回写尖峰更大、掉电丢更多数据；调小则回写更频繁、尖峰更平滑。`file-max` 调大上限更高，代价是内核可为更多打开文件分配对象。
- **联动**：`dirty_ratio` 与 `dirty_bytes` 互斥（设了字节数就忽略比例）；`file-max` 与每个进程的 `ulimit -n`（`fs.nr_open`）是不同层级，进程上限先撞上时调 `file-max` 无效。
- **失败模式**：只看 `%util` 高就调 `dirty_ratio`，可能掩盖真实的磁盘带宽不足；`file-max` 调大而应用没有及时 `close()`，句柄泄漏会把内存耗尽。

## 9. 一套完整的排查剧本

把前面各节收成四条能照着走的路径。每条都按「先全局、再单进程、后系统调用或内核」的顺序，每步给出**判据**——满足什么条件才进下一步。

**CPU 高。**

1. `top` 看首行与各列，先归类：`%us` 高是用户态热点，`%sy` 高是内核态（系统调用或锁），`%si` 高是软中断（网络收包），`%wa` 高是等 I/O（本质是磁盘问题），`%st` 高是被宿主机偷走（虚拟化）。
2. 用户态热点：`pidstat -u -p <pid> 1` 确认是哪个进程，再 `perf top -p <pid>` 或 `perf record -g -p <pid>` 找热函数。
3. 内核态：`strace -c -p <pid>` 看系统调用分布，某类调用占比高就是它。
4. 软中断：`cat /proc/softirqs` 看 `NET_RX` 是否集中在单核，`/proc/net/softnet_stat` 看预算是否用尽（口径见 [[14-Linux 内核收发网络包]] §4.1）。
5. 判据：若 `load average` 高而 `%idle` 也高，查 `D` 状态进程（`ps -eo state,pid,cmd | awk '$1=="D"'`），这类进程在等磁盘，CPU 其实是闲置的。

**内存涨。**

1. `free -m` 看 `available` 而不是 `free` 列——`free` 低不一定是问题，`available` 才是应用真正能拿到的量。
2. `vmstat 1` 看 `si`/`so`：非零说明正在换页（swap in/out），内存已不够。
3. 定位进程：`pidstat -r -p <pid> 1` 看单进程缺页（`minflt/s`、`majflt/s`），`smem` 或 `ps_mem` 按 PSS 排序看谁占得最多（RSS 会重复计算共享页，PSS 更准）。
4. 内核侧：`slabtop` 看 slab 缓存增长，`/proc/meminfo` 的 `Dirty`/`Writeback`/`SReclaimable` 分别对应脏页、回写中、可回收。
5. 判据：`available` 低于阈值且 `si`/`so` 非零，才有内存压力；`dmesg` 出现 `Out of memory: Killed process` 说明已经触发 OOM killer。详见 `03-虚拟内存与页面置换`。

**磁盘 I/O 打满。**

1. `iostat -x 1`：`%util` 接近 100% 表示设备没有空闲周期，但 NVMe、RAID 上 `%util` 可能虚高，要结合 `aqu-sz`（平均队列深度）与 `await`（平均等待，毫秒）一起看。
2. 判据：`await` 明显超过设备标称延迟且 `aqu-sz` 大于 1，说明请求在排队；此时瓶颈才是磁盘。
3. 定位进程：`pidstat -d 1` 或 `iotop -o` 看哪个进程在读写在排队。
4. 区分读写：`iostat` 的 `r/s`、`w/s` 与 `r_await`、`w_await` 分清是读慢还是写慢——写慢且与回写相关时，回到 §8 的 `dirty_ratio` 与 [[08-文件系统与 IO 栈]] 的回写机制。

**网络丢包。**

1. `ip -s link show eth0` 看 `RX`/`TX` 的 `errors`/`dropped`/`overrun` 是否增长，先确认这一层有没有丢。
2. `ethtool -S eth0` 细看硬件计数：`rx_missed_errors`/`rx_fifo_errors` 指向环太小或中断太慢，`rx_crc_errors` 指向物理层。
3. `/proc/net/softnet_stat` 第 2 列 `dropped` 增长指向 `netdev_max_backlog`，第 3 列 `time_squeeze` 增长指向 `netdev_budget` / 多队列。
4. `nstat -az | grep -iE 'drop|overflow'` 看协议栈层的丢包（accept 队列溢出、接收缓冲区满），`ss -ti` 看单连接重传。
5. 判据：网卡层先排除（`rx_crc_errors` 这类与协议栈参数无关），再往上看内核队列，最后才动协议栈参数。完整对照见 [[14-Linux 内核收发网络包]] §4.3。

```text
   四类症状的定位顺序：先全局 → 再单进程 → 后 syscall / 内核
   症状       ① 全局                ② 单进程              ③ 系统调用 / 内核
   ─────────────────────────────────────────────────────────────────────
   CPU 高     top（%us/%sy/%si）   pidstat -u -p        perf top / strace -c
              └ %si 高 → 直接进 ③ 的 softirqs / softnet_stat
   内存涨     free -m（available）  pidstat -r -p        slabtop / meminfo
              └ si/so 非零 → 有压力；dmesg 有 OOM → 已触发
   磁盘满     iostat -x（%util）    pidstat -d 1         iotop -o
              └ aqu-sz>1 且 await 大 → 真排队；否则看回写参数
   网络丢包   ip -s link（errors）  ss -ti（retrans）    ethtool -S / softnet_stat
              └ 顺序：网卡 → 内核队列 → 协议栈 → 应用，不可跳跃
```

这四条路径共用同一条原则：**先用最便宜、最全局的命令给问题归类，再决定要不要动用更贵、更细的工具**。所有计数器只比增量不比绝对值，`nstat`、`/proc` 里的数都是自开机的累计值，采两次做差才有意义。

## 相关

- [[00-系统专栏导览]] —— 本专栏的入口、边界与阅读顺序
- [[04-进程、线程与调度]] —— CPU 与上下文切换那几类指标的来源
- [[09-IO 多路复用与 Reactor]] —— 网络与连接类指标的来源
- [[14-Linux 内核收发网络包]] —— 内核侧的观测量与排查入口

## 参考

- 小林coding. *图解系统 v1.0*. https://xiaolincoding.com/os/
- Brendan Gregg. *The USE Method*. https://www.brendangregg.com/usemethod.html
- Tom Wilkie. *The RED Method*. https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/
- Linux man-pages. *ss(8)*. https://man7.org/linux/man-pages/man8/ss.8.html
- Linux man-pages. *ip(8)*. https://man7.org/linux/man-pages/man8/ip.8.html
- Linux man-pages. *ethtool(8)*. https://man7.org/linux/man-pages/man8/ethtool.8.html
- Linux man-pages. *sar(1)*. https://man7.org/linux/man-pages/man1/sar.1.html
- Linux man-pages. *strace(1)*. https://man7.org/linux/man-pages/man1/strace.1.html
- Linux man-pages. *proc(5)*. https://man7.org/linux/man-pages/man5/proc.5.html
- Linux man-pages. *ping(8)*. https://man7.org/linux/man-pages/man8/ping.8.html
- Linux man-pages. *tcpdump(8)*. https://man7.org/linux/man-pages/man8/tcpdump.8.html
- Linux kernel documentation. *Documentation/admin-guide/sysctl/net.rst*. https://www.kernel.org/doc/html/latest/admin-guide/sysctl/net.html
- Linux kernel documentation. *Tracepoints*. https://www.kernel.org/doc/html/latest/trace/tracepoints.html
- Linux kernel source. *net/ipv4/tcp_diag.c*, *net/ipv4/proc.c*, *mm/page-writeback.c*, *fs/file_table.c*. https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/
- curl. *curl -w, --write-out*. https://curl.se/docs/manpage.html
- Flajolet, Fusy, Gandouet, Meunier. *HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm*. 2007. https://doi.org/10.46298/dmtcs.3545
- iperf. *iperf3 Documentation*. https://software.es.net/iperf/
