---
tags:
  - 基本功/网络
  - 基本功/网络栈
---

# Linux 内核收发网络包

[[01-网络分层模型]] 把 TCP/IP 四层讲成一个骨架；本篇看这套骨架在 Linux 里怎么落地：一个帧从网卡进入内存、被内核逐层剥开、最后被某个进程读走，以及应用 `send()` 的字节怎么反向走到网线上。

这条路径的每一段运行在不同的上下文里——网卡硬件、硬件中断（上半部）、软中断（下半部）、内核线程 `ksoftirqd`、进程系统调用。分清哪一段在哪个上下文，决定了哪些操作是安全的、为什么某些计数器会涨、以及丢包时该动哪个参数。接收与发送还有一条关键的不对称：**发送可以排队等，接收必须及时取走**，后者是 Linux 上大部分「丢包」问题的来源。

## 1. 网卡到协议栈之间发生了什么

网卡收到帧之后并不直接调用协议栈函数。硬件事先被内核安排了一段内存（环形缓冲区），收到帧就由 `DMA` 写进去，再通知 `CPU`；通知和处理被刻意拆成两截，为的是不让协议栈的长逻辑卡在中断里。这一段要弄清四件事的关系：`DMA`、中断上半部、软中断、`NAPI` 轮询。

拆成两截的原因可以量出来。网卡每收一个帧都要通知一次 `CPU`，而处理一个 `TCP` 包要走到哈希表查找、丢进 socket 队列、唤醒进程，这些动作耗时远大于「确认中断」本身。若全放在硬中断里做，`CPU` 会把大量时间花在进出中断、保存寄存器现场上，用户任务被频繁打断。于是内核把任务切成两半：**上半部在硬中断里只做「记账 + 排队」，下半部在软中断里做真正的协议栈处理**。

一次收包经过的上下文顺序是固定的：网卡硬件（`DMA`）→ 硬中断（上半部）→ 软中断（下半部）→ 进程系统调用（`recv` 拷贝数据）。后面每一节都锚在这条链路的某一段上。

### 1.1 DMA 与环形缓冲区

网卡不把帧交给 `CPU` 再让 `CPU` 拷贝，而是用 `DMA`（Direct Memory Access，直接内存访问）直接写进内核预分配的缓冲区。收发两侧各有一组**描述符环**（`descriptor ring`）：每个描述符记录一块缓冲区的地址和状态，网卡硬件认这个环，内核驱动也认这个环。

环是「环形」的，因为生产者（网卡）和消费者（驱动）各持一个指针，绕圈复用同一批缓冲区，避免每次收包都分配内存。接收环的大小由驱动决定，可用 `ethtool -g eth0` 查看、`ethtool -G eth0 rx 4096` 调整（`rx` / `tx` 分别是收 / 发环的条目数）。

这一层是丢包发生的第一处，也最容易被忽略：**驱动还没把描述符消费掉，环就写满了，网卡只能直接丢帧**，并在网卡自己的计数器里记一笔（`rx_missed_errors`、`rx_no_buffer_count`，见 §4.2）。这个丢包发生在协议栈之前，所以协议栈侧的任何计数都不会变化。

描述符环由两类东西组成：描述符数组本身（网卡 `DMA` 读它来判断「下一个缓冲区在哪」）和描述符指向的数据缓冲区。驱动初始化时一次性分配好这两样，运行期只更新描述符的状态位，所以正常收包路径上不为每个帧申请内存。较新的驱动用 `page_pool` 管理缓冲区，把页在环与协议栈之间循环复用，进一步压低每包的内存分配开销。

对照早期做法更能看清 `DMA` 的价值：没有 `DMA` 时（`PIO`，Programmed I/O）`CPU` 要亲自把帧从网卡寄存器搬进内存，每字节一次读指令，收到 1 Gbps 满速流量时 `CPU` 会被搬数据占满。`DMA` 让网卡直接写内存，`CPU` 只在结束时被通知一次。

这也是为什么接收侧调优的第一个动作往往是看 `ring` 大小：`ethtool -g eth0` 报出当前 `RX` / `TX` 环的 `Pre-set maximums` 与 `Current hardware settings`，虚拟机和部分驱动的默认值只有 256 或 512（待验证，随驱动而异）。环越大，能扛的突发越长，代价是每个条目都占着一块缓冲区内存。

### 1.2 中断上半部：只做最少的事

网卡写满一个描述符后发起硬件中断（`IRQ`）。`CPU` 暂停当前任务，按中断描述符表跳到驱动注册的处理函数（不同驱动函数名不同，如 `e1000_intr`、`ixgbe_intr`）。这段代码运行在**硬中断上下文**，是整条路径上最不能停留的地方。

上半部做的事只有三件：读中断状态寄存器、确认并清理中断、调用 `napi_schedule()` 把该设备的 `NAPI` 实例挂到当前 `CPU` 的轮询链表上，然后返回。它不解析帧、不动协议栈。

这个「少做」是刻意的。中断上下文里不能睡眠、不能拿会阻塞的锁，而且它执行期间同级中断被屏蔽；上半部停留得越久，其它中断的响应被推迟得越久，系统抖动越大。对高包速场景，还可以用**中断合并**（interrupt coalescing）主动降低中断频率——`ethtool -C eth0 rx-usecs` 把中断延迟若干微秒，让网卡攒几个帧再打断一次，用延迟换 `CPU` 占用。

中断的另一个维度是「分给谁」。支持多队列的网卡用 `MSI-X` 为每个收发队列分配独立中断向量，配合 `RSS`（Receive Side Scaling）让不同流落到不同 `CPU`，把收包处理摊到多核上。`cat /proc/interrupts | grep eth0` 能看到每个队列的中断号与各 `CPU` 上的计数，计数严重偏向某一核就说明分载没生效。

「上半部不能睡眠」这条限制会向上传染：凡可能从上半部调用的函数，都不能用会睡眠的内存分配接口（如带 `GFP_KERNEL` 的那些）。把重活推到下半部，正是为了让那部分代码从这条限制里解出来。

### 1.3 下半部：软中断与 ksoftirqd

`napi_schedule()` 最终落到 `____napi_schedule()`，它把当前 `CPU` 的 `NET_RX_SOFTIRQ` 标记为待处理（`raise_softirq_irqoff`）。这就是「下半部」：真正的收包处理被推迟到这里。

软中断不等于内核线程。它在两种时机执行：一是中断返回路径上（`irq_exit()` 发现有待处理软中断就调用 `invoke_softirq()`），这时它仍在中断上下文；二是由每 `CPU` 的 `ksoftirqd/N` 内核线程被唤醒后执行。接收软中断 `NET_RX_SOFTIRQ` 的处理函数是 `net_rx_action()`；对应的发送软中断 `NET_TX_SOFTIRQ` 由 `net_tx_action()` 处理，负责释放发送完成的 `skb`。

把协议栈处理推迟到软中断，换来的是「一段时间里批量处理一批帧」，而不是每帧一次中断。代价是软中断若长时间跑不完会饿死用户进程，于是 `net_rx_action()` 带预算：单轮处理的包数上限是 `net.core.netdev_budget`，耗时上限是 `net.core.netdev_budget_usecs`。预算用尽而仍有活儿时，内核记一次 `time_squeeze`，把剩下的交给 `ksoftirqd` 线程继续跑，不长期占住中断返回路径（来源：Linux kernel documentation, *Documentation for /proc/sys/net/*）。

软中断本身还有一套更细的调度规则：`__do_softirq()` 在中断返回路径上循环处理待办软中断，但循环次数有上限（`MAX_SOFTIRQ_RESTART`，值为 10）（待验证），超过就不再在中断上下文里耗下去，转交 `ksoftirqd`。所以「软中断跑在哪个上下文」有两个可能的位置，同一段处理逻辑按负载落在不同处：空闲时在中断返回路径上跑完，繁忙时被推给 `ksoftirqd` 线程。

观察软中断总量的地方是 `/proc/softirqs`：按 `CPU` 分列，`NET_RX` 一列即各核累计执行的收包软中断次数。把它与 `/proc/interrupts` 对照，能判断收包处理是否均匀落在各核上。

### 1.4 NAPI：中断加轮询的混合

纯中断模式的问题可以算清楚：若每个包触发一次中断，100 万 pps 就是每秒 100 万次中断，`CPU` 的时间几乎全花在进中断、保存现场、出中断上，留给协议栈的反而更少。

`NAPI`（New API，Linux 2.6 引入）用「中断 + 轮询」的混合方式替代纯中断。第一帧到达时用中断把 `NAPI` 实例唤醒；随后驱动**关掉该队列的中断**，改为主动轮询，一次 `poll` 调用来连续取走多帧，直到把环取空或达到配额，再重新打开中断等待下一次通知。

驱动的轮询方法是 `int (*poll)(struct napi_struct *napi, int budget)`（来源：Linux kernel documentation, *NAPI*）。参数 `budget` 是本次允许处理的收包数上限，来自 `net.core.dev_weight`（默认 64）。返回值是本次实际处理的帧数：**若返回值恰好等于 `budget`，说明环里还有剩余，`NAPI` 实例会被再次调度，无须再等中断**；小于 `budget` 才认为取空。

中断与轮询的关系：

```text
   网卡收到帧
      │  DMA 把帧写进 RX ring（描述符 + 缓冲区）
      ▼
   ┌─ 上半部（硬中断上下文）──────────────────────────┐
   │  ① 读状态寄存器、确认中断                        │
   │  ② napi_schedule()：napi 实例挂到本 CPU poll 链表│
   │  ③ raise_softirq(NET_RX_SOFTIRQ)                │
   └─────────────────────────────────────────────────┘
      │
      ▼
   ┌─ 下半部（软中断上下文）──────────────────────────┐
   │  net_rx_action()                                │
   │    └─ 遍历 poll 链表 → 驱动 poll(napi, budget)   │
   │         循环取描述符，直到环空或 budget 用尽      │
   │  预算：netdev_budget（包数）/ netdev_budget_usecs │
   └─────────────────────────────────────────────────┘
      │  预算用尽且仍有工作
      ▼  记一次 time_squeeze；余下交给 ksoftirqd/N 继续
   协议栈逐层处理 → socket 接收队列 → 唤醒等待的进程
```

这也解释了 §4.1 里两个计数器为什么会分别增长：轮询跟不上到达速率时，包会堆在收包队列里溢出，`dropped` 上涨；而单轮预算不够时是 `time_squeeze` 上涨。两者指向的参数并不相同。

还有一点与并行度有关：`NAPI` 倾向「一对一」映射——现代网卡的每个队列通常对应一个中断、一个 `NAPI` 实例（来源：Linux kernel documentation, *NAPI*），`ethtool` 里的「channel」概念（`rx` / `tx` / `combined`）就是「一个服务某类队列的中断 / `NAPI`」的近似说法。多队列网卡的收包并行度正来自这里；单队列网卡只能靠 `RPS` 在软件层补。

配额与唤醒阈值一起决定一次收包的「批大小」。设 `dev_weight` 为 64，则驱动一次 `poll` 最多交 64 个帧给协议栈；若环里有 500 个帧而预算只有 64，剩下的会在 `net_rx_action` 的下一轮继续处理，中间可能让出 `CPU`。批越大，单位帧分摊的开销（进软中断、加锁、遍历链表）越低；批越小，软中断越短，对延迟敏感的任务越不容易被拖住。这是吞吐与延迟之间的取舍，没有一组值对所有场景都最优——所以 §5.2 给了这个参数单独的调优落点。

「先关中断、后开中断」这个模式还有一个边界要注意：两次开中断之间到达的帧必须靠轮询取走，如果驱动在轮询里提前返回（比如错误地判断环已空），这些帧要一直等到下次中断或超时才被处理，表现为固定间隔的延迟。驱动的 `poll` 返回值语义正为此约定——返回小于 `budget` 代表「确实取空了」，返回值等于 `budget` 代表「还有活儿，请再调我一次」。读驱动源码或自己适配网卡时，这个返回值是判断「会不会漏帧」的关键。同一机制也解释了为什么中断合并参数（`ethtool -C`）会间接影响丢包：合并窗口越长，环的消费越依赖轮询是否及时，环就更容易被写满。

## 2. 接收路径

从网卡收到帧到应用 `read()` 返回，数据经过网络接口层、网络层、传输层三层剥壳，最后跨过内核与用户态边界。下面按「先定上下文、再走层次」组织：每一层做什么、在哪一层上下文执行。

路径上有两个边界值得盯住：一个是 `IP` 首部里的**协议号**，决定交给 `TCP` 还是 `UDP`，这是层与层之间的边界；另一个是 socket 的**四元组**，决定交给哪个进程，这是内核与进程之间的边界。两次「查表分发」决定了包最终去哪，也是排查「包收到了但进程读不到」时最先要看的地方。

### 2.1 网络接口层：校验、剥帧、定上层协议

驱动 `poll` 从 `RX ring` 取到一个描述符，据此构造一个 `struct sk_buff`（内核里所有网络包都用同一个结构体描述）。`sk_buff` 用 `head` / `data` / `tail` / `end` 四个指针描述这块缓冲区里数据的有效区间；各层加首部就是把 `data` 往低地址移（`skb_push`），剥首部就是往高地址移（`skb_pull`）。整条路径上协议栈不靠反复拷贝来加头去头，靠的就是移动这几个指针。

帧的 `FCS`（尾部 4 字节 `CRC-32`）通常由网卡硬件校验，通过则给 `skb` 打上 `CHECKSUM_UNNECESSARY`，内核不再重算。接着 `eth_type_trans()` 读以太网首部的类型字段，决定交给谁：`0x0800` 给 `IPv4`、`0x86DD` 给 `IPv6`、`0x0806` 给 `ARP`，结果写进 `skb->protocol`。

在此之前还有一步 `GRO`（Generic Receive Offload）：`napi_gro_receive()` 把属于同一 TCP 流的、连续到达的段合并成一个更大的 `skb`，让上层只需处理一次。之后 `__netif_receive_skb()` → `__netif_receive_skb_core()` 按注册的 `ptype` 链表分发，若有 `AF_PACKET` 抓包会先复制一份，最后调用网络层的接收函数。

`sk_buff` 的四个指针各管一段：`head` 指向缓冲区起点，`end` 指向终点，`data` 到 `tail` 之间是当前有效数据，`tail` 到 `end` 之间是预留的尾部空间。加首部让 `data` 前移，剥首部让它后移，同一块内存就这样在各层之间流转而不被重拷。缓冲区末尾还挂着一个 `skb_shared_info`，记录分片信息与 `frags` 数组，供 `GRO` / `GSO` / 分片使用。

`GRO` 常与 `LRO`（Large Receive Offload）一起被提到，差异在合并发生在哪：`LRO` 由网卡硬件合并，对软件不可见，转发场景下可能破坏端到端语义；`GRO` 在内核软件里合并，并保留 `gso_size`，需要时能再拆回原段，所以更通用。合并后的 `skb` 带着 `gso_size` 往上走，到传输层仍能被正确切分。

### 2.2 网络层：本地交付还是转发

`ip_rcv()` 先做 `ip_rcv_core()` 里的合法性检查：版本、首部长度、`IP` 首部校验和，任一不合法就丢弃并计入 `IpExtInHdrErrors`（`/proc/net/snmp` 的 `Ip:` 段）。

接着 `ip_rcv_finish()` 调路由判定（`ip_route_input_noref`），决定这个包的去向：目的 `IP` 是本机地址，就走 `ip_local_deliver()` → `ip_local_deliver_finish()`；目的地址不是本机且本机开启了转发，就走 `ip_forward()`（这是主机变路由器的路径）。

本地交付时，按 `IP` 首部里的**协议号**查 `inet_protos` 表分派：`6` → `tcp_v4_rcv()`、`17` → `udp_rcv()`、`1` → `icmp_rcv()`。若有分片，还要先经 `ip_defrag()` 重组；重组队列的内存上限受 `net.ipv4.ipfrag_high_thresh` 约束，超过会丢弃并计入 `IpExtReasmFails`。协议号这个「解释开关」与 [[01-网络分层模型]] §3.3 里那张解封装图的位置一致。

### 2.3 传输层：四元组分用，找到 socket

`tcp_v4_rcv()` 拿到剥掉首部后的包，用四元组（源 `IP`、源端口、目的 `IP`、目的端口）调 `__inet_lookup_skb()` 查 socket：先查已建立连接的哈希表（`ehash`），未命中再查监听哈希表（`lhash`）。任一命中就把包交给对应 socket 的处理函数（`tcp_v4_do_rcv()`）；都未命中则回一个 `RST`。

命中已建立连接后走 `tcp_rcv_established()`，数据段经 `tcp_data_queue()` → `tcp_queue_rcv()` 放进该 socket 的接收队列 `sk_receive_queue`（一个 `struct sk_buff_head`，把 `skb` 串成链表）。乱序到达的段先进 `out_of_order_queue` 等缺口补齐。

socket 接收缓冲区有上限 `sk_rcvbuf`。它由 `net.ipv4.tcp_rmem`（三元组 min/default/max）在连接生命周期内自动调节，硬上限是 `net.core.rmem_max`。**缓冲区被填满时，`TCP` 会在通告窗口里回一个零窗口，让对端停止发送**——这是流量控制在接收侧的落点。

如果这个包是三次握手的最后一个 `ACK`，连接会从半连接队列转入 accept 队列（`inet_csk_reqsk_queue_add()`）。accept 队列满了，新连接可能被丢弃，计入 `TcpExtListenOverflows`；`SYN` 到 `LISTEN` socket 被丢则计入 `TcpExtListenDrops`（见 §4.2）。

`__inet_lookup_skb()` 的查找是哈希，冲突用拉链法解决。`SO_REUSEPORT` 让多个 socket 绑同一端口，内核按四元组哈希把连接分摊给其中一个，这是多进程服务器（如 `nginx` 多 worker）分摊新连接的常用手段。监听哈希表也查不到时，包被判为「不属于任何连接」，回一个 `RST`。

接收队列里的 `skb` 会在内存压力下被合并以减少碎片（`tcp_collapse()`），计入 `TcpExtTCPRcvCollapsed`——这个计数增长说明内核在收缩内存，不一定是故障。

把 §2.1 到 §2.3 的分层处理串起来：

```text
   驱动 poll 从 RX ring 取到描述符
        │  构造 struct sk_buff，data 指向帧首
        ▼
   网络接口层   eth_type_trans() 读类型 ── 0x0800 / 0x86DD / 0x0806
        │  硬件已校验 FCS；GRO 合并同流 TCP 段；skb_pull 剥掉帧头
        ▼
   网络层      ip_rcv() 校验 → 查路由
        │  目的 IP 是本机？ ── 否 → ip_forward() 转发走掉
        │                   是 → ip_local_deliver()
        │  按协议号分派：6 → tcp_v4_rcv / 17 → udp_rcv / 1 → icmp_rcv
        ▼
   传输层      tcp_v4_rcv() 用四元组查 socket
        │  命中 → tcp_data_queue() 入 sk_receive_queue
        │  未命中 → 回 RST
        ▼
   （接 §2.4：sk_data_ready 唤醒进程，进程在 recv 里拷进用户缓冲区）
```

### 2.4 进接收缓冲区、唤醒进程

数据入队后，`tcp_data_queue()` 末尾调用 `sk->sk_data_ready(sk)`，通常是 `sock_def_readable()`。它唤醒在 socket 等待队列上睡眠的进程（`wake_up_interruptible_sync_poll()`），同时唤醒挂在同一等待队列上的 `epoll` 实例。**这一步在软中断上下文里执行**——数据刚到达时，是中断路径主动叫醒进程，而不是进程自己不停轮询。

被唤醒的进程从 `recv()` / `read()` 系统调用返回前，还要在 `tcp_recvmsg()` 里把数据从内核的 `skb` 拷贝到用户缓冲区（`skb_copy_datagram_msg()` 最终走 `copy_to_user()`）。**这一步在进程上下文。** 所以一次收包实际是两段搬家：软中断把数据搬进内核的 socket 缓冲区，进程再把数据搬进用户缓冲区。

### 2.5 中断上下文与进程上下文的分界

把整条接收路径按上下文标出来：

```text
   上下文自下而上：硬件 → 硬中断 → 软中断 → 进程
   │
   ├─ 进程上下文    read() / recv() → tcp_recvmsg() → copy_to_user
   │                （把数据从内核缓冲区搬进用户缓冲区，允许睡眠）
   │  ┄┄┄┄┄┄┄┄┄ 系统调用边界 ┄┄┄┄┄┄┄┄┄
   ├─ 软中断上下文   ip_rcv → tcp_v4_rcv → tcp_data_queue → sk_data_ready
   │                （协议栈剥壳、查表、入队、唤醒，不能睡眠）
   │  ┄┄┄┄┄┄┄┄┄ 软中断边界 ┄┄┄┄┄┄┄┄┄
   ├─ 中断上下文     net_rx_action → 驱动 poll（处理环里的帧）
   │  ┄┄┄┄┄┄┄┄┄ 中断边界 ┄┄┄┄┄┄┄┄┄
   ├─ 硬中断上下文   网卡中断处理函数（确认中断、napi_schedule）
   │
   └─ 硬件          DMA 把帧写进 RX ring
```

三层的约束不同，这决定了实现上的取舍：

- **硬中断上下文**最受限。不能睡眠，不能拿会睡眠的锁，停留时间要尽量短。
- **软中断上下文**同样不能睡眠，`in_interrupt()` 为真。协议栈接收逻辑全在这里，所以任何在接收路径上阻塞的操作都会拖住整个 `CPU` 的软中断。
- **进程上下文**才允许睡眠、允许 `copy_to_user`。`recv()` 侧拿不到数据时会睡眠在等待队列上，由 §2.4 的软中断唤醒。

`ksoftirqd/N` 是个例外：它是内核线程，有自己的 `task_struct`，可以被调度器抢占；但它执行软中断处理逻辑时仍然不能主动睡眠。理解这条分界能解释两个常见现象：为什么 recv 侧锁争用会表现为软中断里的 `time_squeeze` 上涨（消费者慢，软中断队列下不去），以及为什么 `SO_BUSY_POLL`、`XDP` 这类机制要把处理搬到更早的上下文里。

这条分界也解释了为什么收包路径上要尽量避免「每包一次内存分配」和「每包一次锁竞争」：`NAPI` 的批量轮询、`page_pool`、`GRO`、`RPS` 都是为了让软中断一次多做点事、少进退几次上下文。反过来，若协议栈处理里出现 O(连接数) 的线性扫描，成本会直接落在软中断时长上，表现为 `time_squeeze` 与整体延迟一起上涨。

## 3. 发送路径

发送方向是接收的反向封装，但中间多了两级缓冲：传输层的发送队列和网络接口层的排队规则（`qdisc`）。这两级缓冲让发送可以延迟，也决定了发送侧的丢包点在哪。

还有一处与接收不同：接收路径只有一次数据拷贝（内核缓冲区 → 用户缓冲区，发生在 `recv` 侧），发送路径的拷贝次数取决于是否触发分片，下面逐段标出哪一步在拷数据、哪一步只是在移指针。

发送侧还要多回答一个问题：数据什么时候算「发出去了」。答案分层——驱动认为写进网卡就算完成，`TCP` 认为收到 `ACK` 才算完成，§3.5 会区分这两个 `skb` 的释放时机。

### 3.1 系统调用与传输层

应用调 `send()` / `write()`，系统调用进内核后经 `sock_sendmsg()` 落到 `tcp_sendmsg()` → `tcp_sendmsg_locked()`。内核在该函数里申请一个 `sk_buff`（`sk_stream_alloc_skb()`），用 `copy_from_iter()` 把用户态数据拷贝进 `skb` 的数据区——这是发送路径上的**第一次内存拷贝**。装好数据的 `skb` 挂到 socket 的发送队列 `sk_write_queue`。

真正发出去由 `tcp_push()` → `tcp_write_xmit()` 决定，能不能发受两个窗口共同约束：对端通告的接收窗口和本端的拥塞窗口，取两者较小值。这一步也解释了 §2.3 收到的零窗口会怎样反过来卡住发送。

`__tcp_transmit_skb()` 在真正交出去之前会做一次 `skb_clone()`。这里有个常见误解需要澄清：

> [!warning] `skb_clone` 复制的是描述符，不是数据
> 内核为「发送完成后 `skb` 会被释放、但 `TCP` 在收到 `ACK` 前不能删数据」做的处理，是克隆一个 `sk_buff` 描述符：新的 `skb` 与原 `skb` **共享同一块数据缓冲区**（靠引用计数管理），只复制描述符本身的字段。所以在传输层这个位置发生的是一次描述符复制，数据区并没有被拷第二遍。原始 `skb` 留在 `sk_write_queue` 里等 `ACK`，克隆件被交下去发送，发完释放的是描述符。

克隆件随后填上 `TCP` 首部：源 / 目的端口、序号、确认号、通告窗口、标志位（`tcp_transmit_skb()` 内完成）。

把发送路径上的内存拷贝列一张账：第一次在 `tcp_sendmsg()`，用户数据拷进 `skb`；第二次在 §3.2 的分片，只有超过 `MTU` 时才发生；§3.1 的 `skb_clone()` 共享数据区，不产生数据拷贝。所以**不分片时，一个字节从 `send()` 到上网线只被拷过一次**，其余步骤都在移动指针。

队列里的未确认数据计入 `sk_wmem_queued`，它受 `SO_SNDBUF` 与 `net.ipv4.tcp_wmem` 约束；写满时阻塞 socket 上的 `send()` 会睡眠，非阻塞 socket 返回 `EAGAIN`。这是应用侧「写不进去」的直接原因，与内核队列参数是两回事。

### 3.2 网络层：查路由、加 IP 首部、netfilter

`ip_queue_xmit()` → `ip_local_out()`，先过 netfilter 的 `LOCAL_OUT` 钩子，再到 `ip_output()` → `ip_finish_output()`。路由通常已缓存在 `sk->sk_dst_cache` 上，未命中才重新查（`ip_route_output_flow()`）。

网络层填 `IP` 首部：源 / 目的 `IP`、`TTL`、协议号 `6`，以及分片用的标识（`ip_select_ident()`）。若数据超过出口的 `MTU`，会在此分片（`ip_fragment()`）——每个分片都要申请一个新的 `skb` 并把数据拷过去，这是发送路径上的**第二次数据拷贝**，它只在发生分片时出现（`MTU` 足够时不发生）。

处理完交给邻居子系统（`ip_finish_output2()`）。

这一段的钩子点是 netfilter 五个 `hook` 之一 `NF_INET_LOCAL_OUT`；`iptables` / `nftables` 的 `OUTPUT` 链、`conntrack` 的 `NAT` 都在这里介入。若链上有 `DROP`，包在此消失，计数记在链自身而不是网卡——所以「网卡计数正常但进程收不到回应」时，netfilter 规则是一个要先排除的方向。

超过 `MTU` 时有两种处理：`IPv4` 允许中途分片（`ip_fragment()`），`IPv6` 只允许源端分片；若首部带 `DF`（Don't Fragment）标志，则回一条 `ICMP` 需要分片（Type 3 Code 4），`TCP` 据此调整 `MSS`，这就是路径 `MTU` 发现（`PMTUD`）的落点。分片会给每片都加一份首部，也让丢包影响放大（丢一片整包重传），所以工程上更倾向把 `MSS` 调小而不是依赖分片。

### 3.3 邻居子系统：ARP 拿到下一跳 MAC

网络层只给出了下一跳的 `IP`，以太网只认 `MAC`，中间靠邻居子系统补全。它先查邻居表（`arp_tbl`）：命中且状态为 `NUD_VALID`（可达）就直接取 `MAC`（`neigh_output()`）。

未命中时走 `neigh_resolve_output()`：把待发的包先压进邻居的未决队列，同时触发 `ARP`——`arp_solicit()` 以广播 `who-has` 询问下一跳 `IP` 对应的 `MAC`；对端单播回 `arp_reply` 后，`neigh_update()` 更新表项并冲刷队列，积压的包随即发出。可达状态有超时，过期后条目降级、下次再解析。

拿到 `MAC` 后，用 `dev_hard_header()`（以太网用 `eth_header()`）填上帧头：目的 `MAC` = 下一跳，源 `MAC` = 出口网卡，类型 `0x0800`；`FCS` 由网卡硬件在发送时补齐。之后就交给排队规则。

邻居表项的回收由邻居子系统的定时器驱动：可达（`NUD_REACHABLE`）状态被确认后先保持一段时间，再降级、超时、重新解析，`arp -n` 能看到表项与状态。表项数量与哈希桶由 `net.ipv4.neigh.default.gc_thresh1/2/3` 控制，超过第三档开始主动回收，条目过多时新解析可能失败——大规模容器网络里这个坑很常见。

### 3.4 排队：qdisc 与发送队列

`dev_queue_xmit()` → `__dev_queue_xmit()` → `__dev_xmit_skb()`。每个发送队列（`struct netdev_queue`）关联一个排队规则 `qdisc`。内核默认的 `qdisc` 是 `pfifo_fast`（来源：Linux kernel documentation, *Documentation for /proc/sys/net/* 的 `default_qdisc`），可以通过 `net.core.default_qdisc` 改成 `fq_codel` 之类；多队列网卡的根 `qdisc` 固定为 `mq`，叶子再用默认值。

包经 `qdisc_enqueue()` 入队，再由 `__qdisc_run()` → `sch_direct_xmit()` 取出来交给驱动。每个队列有长度上限（传统上由设备的 `txqueuelen` 决定，默认 1000，用 `ip link set eth0 txqueuelen 2000` 调整）。**队列满时新包被丢弃**，`tc -s qdisc show dev eth0` 的 `dropped` 就是这一处的计数。拥塞控制与 `BQL`（byte queue limits）也在这层调节注入速率。

### 3.5 驱动取走与网卡 DMA

驱动从队列里取到 `skb` 后调用 `ndo_start_xmit`（如 `e1000_xmit_frame`）：把 `skb` 的数据地址写进 `TX` 描述符、做 `DMA` 映射，最后写网卡的门铃寄存器，网卡随即通过 `DMA` 读走数据并发出。

发送完成后网卡再发一次中断，`NET_TX_SOFTIRQ` 的 `net_tx_action()` 处理完成队列，释放这部分描述符与 `skb`。但要注意区分两个 `skb`：此时释放的是 §3.1 里那个克隆件。**发送完成不等于数据可以删**——原始 `skb` 还在 `sk_write_queue` 里，直到收到对端 `ACK`，`tcp_clean_rtx_queue()` 才把它清出队列。这就是未确认数据在发送侧的内存占用来源。

排队与驱动之间还有一层 `BQL`（Byte Queue Limits）：内核按链路速率动态限制「已交给网卡但尚未发完」的字节数，避免应用一次塞进太多数据导致排队延迟（bufferbloat）暴涨。调大 `txqueuelen` 并不能绕开 `BQL`，它限的是驱动层而不是 `qdisc` 层。

发送异常时驱动记 `tx_timeout`，触发 `NETDEV_WATCHDOG`，最终可能 `reset` 网卡并重传；`tx_restart_queue` 表示因环满而暂缓、稍后重试的次数。这两个计数持续增长通常指向网卡 / 驱动 / 链路问题，与协议栈参数无关。

把 §3.1 到 §3.5 的发送链路串起来：

```text
   应用 send() / write()
      │  sys_sendto → tcp_sendmsg：申请 skb，copy_from_iter 拷用户数据
      ▼  挂进 sk_write_queue
   传输层   __tcp_transmit_skb
      │  skb_clone（共享数据区，原始 skb 留队列等 ACK）；填 TCP 首部
      ▼
   网络层   ip_queue_xmit → ip_output
      │  netfilter LOCAL_OUT；查路由；填 IP 首部；超 MTU 则分片
      ▼
   邻居子系统  查 ARP 缓存 → 未命中则发 who-has，未决包先入队
      │  dev_hard_header 填以太网首部（目的 MAC = 下一跳）
      ▼
   qdisc   __dev_xmit_skb → qdisc_enqueue → __qdisc_run 取包
      │  队列满则丢（tc -s qdisc 的 dropped 计数）
      ▼
   驱动 ndo_start_xmit → DMA 映射 + 门铃 → 网卡发出
      │  发送完成：NET_TX_SOFTIRQ 释放描述符副本
      ▼  收到 ACK：tcp_clean_rtx_queue 释放原始 skb
```

### 3.6 发送可以延迟，接收必须及时

收发两侧的缓冲与丢包行为并不对称：

| | 发送侧 | 接收侧 |
| --- | --- | --- |
| 缓冲位置 | `sk_write_queue` + `qdisc` 队列 | 网卡 `RX ring` + 内核收包队列 |
| 满了怎么办 | 排队；排不下则丢包后由 `TCP` 重传 | 直接丢帧，只能靠对端重传 |
| 丢包代价 | 一次重传延迟（通常仍是本端可控） | 一次 `RTT`，且本端无从知晓 |
| 反压机制 | 拥塞窗口 / `qdisc` / `BQL` 限流 | 网卡没有反压，只能丢 |
| 主要调优项 | `txqueuelen`、`qdisc`、拥塞控制 | `netdev_max_backlog`、`dev_weight`、`RX ring`、多队列 / `RPS` |

发送侧可以「先存起来慢慢发」，因为 `TCP` 本来就会重传，丢一个包的影响被重传机制吸收；接收侧收到的是别人发来的段，环或队列满了就是真的丢了，本端唯一能做的是通知对端少发（`TCP` 零窗口）或赶紧取走。这解释了 `netdev_max_backlog` 为什么只出现在接收方向的调优清单里：它是收包队列溢出时唯一的缓冲区。

## 4. 观测量与排查

每个丢包位置都有对应的计数器。排查的顺序是先确定丢在哪一层，再改对应的参数——盲目调大某个队列，往往只是把丢包推迟到下一处。

这些计数器分三个来源：网卡 / 驱动的硬件计数（`ethtool -S`）、内核软中断与队列的计数（`/proc/net/softnet_stat`）、协议栈统计（`nstat` / `netstat -s`）。三者彼此独立，所以判读要按「网卡 → 内核 → 协议栈 → 应用」的顺序，先排除下层，否则会被上层症状带偏。

### 4.1 软中断与收包队列：/proc/net/softnet_stat

`/proc/net/softnet_stat` 每个 `CPU` 一行，数值以十六进制输出。现代内核的行含义如下（来源：prometheus/procfs `net_softnet.go`，对应内核 `net/core/net-procfs.c`）：

| 列 | 字段 | 含义 |
| --- | --- | --- |
| 1 | `processed` | 已处理的帧数 |
| 2 | `dropped` | 因收包队列（`backlog`）满而丢弃的帧数 |
| 3 | `time_squeeze` | `net_rx_action` 因预算用尽或超时而退出、但仍有工作可做的次数 |
| 9 | `cpu_collision` | 发送时争抢设备锁的冲突次数 |
| 10 | `received_rps` | 本 `CPU` 被 `RPS` 的处理器间中断唤醒处理包的次数 |
| 11 | `flow_limit_count` | 达到流量限制的次数（`RFS` 的可选限制） |
| 12 | `backlog_len` | 当前 `backlog` 队列长度（Linux 5.14 起） |
| 13 | `cpu` | 该行对应的 `CPU` 编号 |

判读要点：

- **第 2 列 `dropped` 持续增长** → 收包队列溢出，瓶颈在内核取包速度跟不上到达速度，改 `net.core.netdev_max_backlog`（§5.1）。
- **第 3 列 `time_squeeze` 持续增长** → 单轮 `poll` 预算不够，改 `net.core.netdev_budget` / `netdev_budget_usecs`，或把负载分散到多队列 / 多 `CPU`。
- 两列同时涨时，先看是哪一列先起——队列溢出在前说明取包速度才是根因，单纯加预算治不了。

`/proc/softirqs` 的 `NET_RX` / `NET_TX` 两列给出每 `CPU` 的软中断次数，用来判断收包是否集中在单核（多队列 / `RSS` / `RPS` 是否生效）。

读数时注意两点：文件里每行是十六进制，用 `awk '{printf "%d\n", "0x"$2}' /proc/net/softnet_stat` 把第 2 列转成十进制更直观；每行列数会随内核版本变化，脚本解析这个文件要按列数分支，不能写死 13 列。

第 10 列 `received_rps` 与第 11 列 `flow_limit_count` 只在启用了 `RPS` / `RFS` 后才有意义。`RPS` 用软件把包分发给其它 `CPU` 的接收软中断（通过 `/sys/class/net/eth0/queues/rx-0/rps_cpus` 配置），是单队列网卡把收包摊到多核的主要手段；`received_rps` 增长说明分载在工作。

### 4.2 协议栈与网卡层的计数

按层次从下往上，各有各的计数器：

- **网卡层**：`ethtool -S eth0` 报驱动与硬件计数。`rx_missed_errors`（描述符耗尽、环满）、`rx_fifo_errors`（`FIFO` 溢出）、`rx_no_buffer_count`（驱动无可用缓冲）、`rx_crc_errors` / `rx_frame_errors` / `rx_length_errors`（物理层或链路问题）、`tx_restart_queue` / `tx_timeout`（发送异常）。环参数用 `ethtool -g eth0` 看、`ethtool -G` 调。
- **驱动 / 链路层汇总**：`ip -s link show eth0` 报 `RX errors/dropped/overrun/mcast` 与 `TX errors/dropped/carrier/collsns`。其中收侧的 `overrun` 对应 `FIFO` 溢出，通常意味着环太小或中断太慢。
- **协议栈层**：`nstat -az` 或 `netstat -s`。接收相关的关键项有 `TcpExtListenDrops`（发往 `LISTEN` socket 的 `SYN` 被丢）、`TcpExtListenOverflows`（accept 队列溢出）、`TcpExtTCPReqQFullDrop`（半连接队列满）、`TcpExtTCPRcvQDrop`（接收队列因内存不足丢）、`TcpExtTCPRcvCollapsed`（接收队列 `skb` 被合并）、`UdpRcvbufErrors`（`UDP` 接收缓冲区满）、`IpExtInHdrErrors` / `IpExtInDiscards`。

`ss -m` 可以看到每个 socket 的 `Recv-Q` / `Send-Q` 与内存占用；`Recv-Q` 长期非零说明应用读得不够快，属于应用侧瓶颈。

几个容易混的计数器要点名区分：`TcpExtListenDrops` 统计「发往 `LISTEN` socket 的 `SYN` 被丢弃」的总次数（半连接队列满与 accept 队列满都会计入），`TcpExtListenOverflows` 专指 accept 队列溢出的次数；`TcpExtTCPReqQFullDrop` 是半连接（`SYN`）队列满造成的丢弃；`TcpExtTCPRcvQDrop` 是已建立连接的接收队列因内存压力丢包。前两者常一起涨，排查时先确认哪个队列先满，再决定改 `somaxconn` 还是 `tcp_max_syn_backlog`。

`netstat -s` 把这些名字翻成人话（`listen queue of a socket overflowed`、`SYNs to LISTEN sockets dropped`），`nstat -az` 给的是原始计数器名。要做趋势分析，用 `sar -n EDEV` 或自己定时采集，单次快照的累计值说明不了变化。

网卡层的计数名字随驱动与芯片不同，语义大体对应：`rx_missed_errors` 是 `DMA` 已写但主机来不及取（环满或中断被长时间屏蔽），`rx_fifo_errors` 是网卡 `FIFO` 溢出，`rx_no_buffer_count` 是驱动没有可用缓冲区交给网卡，`rx_length_errors` / `rx_crc_errors` / `rx_frame_errors` 指向物理层或链路（线缆、光模块、双工不匹配）——后三者增长通常与协议栈参数无关，要先查硬件与链路。

`ip -s link` 的 `overrun` 是 `FIFO` 溢出在链路层的汇总口径，与 `ethtool -S` 的 `rx_fifo_errors` 常同时增长；同一条输出里的 `dropped` 则是驱动 / 内核主动丢的（例如环满后回收描述符时丢弃）。两者成因不同：`overrun` 偏硬件来不及，`dropped` 偏软件来不及时。

协议栈侧还有一组与内存相关的计数容易被忽略：`TcpExtTCPMemoryPressures`（进入 `TCP` 内存压力状态）、`TcpExtTCPMemoryPressuresChrono`（累计处于压力状态的时间）、`TcpExtTCPRcvQDrop`（因内存丢接收队列）。它们增长时，问题往往在 socket 缓冲区上限或应用读得太慢，而不是队列深度。

判读这些统计有一条通用原则：**只比增量，不比绝对值**。`nstat` 的输出是自开机的累计值，一个百万级的 `TcpExtTCPRcvCollapsed` 可能来自很久以前的抖动；做法是间隔若干秒采两次做差，或直接用 `sar -n EDEV` / `nstat` 的滚动模式看增速。

### 4.3 症状到计数器的对照

排查顺序：

```text
   现象：接收丢包 / 吞吐上不去
        │
        ▼  ① 网卡层还够吗？     ethtool -S eth0 | grep -iE 'miss|fifo|buffer'
     rx_missed_errors / rx_no_buffer_count 增长
        └─ 环太小或中断太慢 → ethtool -G 调大 ring / 中断合并 / 多队列
        │
        ▼  ② 内核取包跟得上吗？ cat /proc/net/softnet_stat
     第 2 列 dropped 增长       └─ backlog 溢出 → net.core.netdev_max_backlog
     第 3 列 time_squeeze 增长  └─ poll 预算用尽 → netdev_budget / 多队列
        │
        ▼  ③ 协议栈丢在哪？     nstat -az | grep -iE 'drop|overflow'
     ListenDrops / ListenOverflows └─ accept 队列满 → net.core.somaxconn
     RcvQDrop / RcvbufErrors      └─ socket 缓冲区或内存不足
        │
        ▼  ④ 应用读得够快吗？   ss -m 看 Recv-Q 是否长期非零
     Recv-Q 持续非零 └─ 应用侧瓶颈，先改应用再动内核参数
```

| 症状 | 优先看什么 | 判据 | 方向 |
| --- | --- | --- | --- |
| 收包无规律丢，`rx_missed_errors` 涨 | `ethtool -g` / `ethtool -S` | 环满或中断过于集中 | 调大 `RX ring`、开多队列 / `RPS` |
| `softnet_stat` 第 2 列涨 | `/proc/net/softnet_stat` | 收包队列溢出 | 调大 `netdev_max_backlog` |
| `softnet_stat` 第 3 列涨 | `/proc/net/softnet_stat` | 单轮预算不足 | 调大 `netdev_budget` / 分散队列 |
| 高并发下 `ListenDrops` 涨 | `nstat -az` | accept 队列满 | `somaxconn` 与应用 `backlog` 同步调大 |
| `UDP` 收包丢，`RcvbufErrors` 涨 | `nstat -az` / `ss -m` | 接收缓冲区满 | 应用加大 `SO_RCVBUF` / 及时读 |
| 吞吐上不去但几乎不丢 | `ss -m` / `sar -n DEV` | 窗口或应用处理受限 | 查 `TCP` 窗口、拥塞控制与应用逻辑 |

最后一列的「方向」只给最直接的动作，落地前还要回到 §5.4 那张队列图确认位置：比如 `rx_missed_errors` 增长既可能是环太小，也可能是软中断处理太慢导致环长期填满，两种情况该动的参数不同。判据是同时看 `/proc/net/softnet_stat` 的 `time_squeeze`——若它也在涨，瓶颈在 `CPU` 侧的轮询预算，先把预算或多队列调上去，再考虑加环大小；若它不涨，环才是瓶颈。

## 5. 内核参数与取舍

下列参数的默认值来自内核文档或源码锚点；标 `（待验证）` 的表示本次未在官方文档页面上直接核到默认值，使用前应以实际机器的 `/proc/sys/...` 为准。

每个参数按「默认值 → 语义 → 什么时候改 → 改大 / 改小的后果 → 联动 → 失败模式」六项写齐。动参数前先想清楚要改的是**哪个队列**：不同队列满了报不同计数器，参数之间不能互相替代。

这些参数几乎都是**队列上限或预算**，调大它们只是在「多等一会儿」与「多占一点内存」之间换，真正的吞吐上限由网卡、`CPU` 与应用处理速度决定。所以调参前先确认瓶颈不在应用（判据见 §5.4 末尾）。

### 5.1 net.core.netdev_max_backlog

- **默认值**：1000（待验证）。
- **语义**：单个 `CPU` 的收包队列（`backlog`）能容纳的包数上限。当接口收包快于内核处理速度时，多余包进入该队列；队列满则丢弃，计入 `softnet_stat` 第 2 列。
- **什么时候该改**：`softnet_stat` 第 2 列 `dropped` 随流量上涨，且 `rx_missed_errors` 不涨（说明环没满，是内核处理队列溢出）。
- **改大的后果**：能吸收更大的突发，减少丢包；代价是该 `CPU` 上积压的包更多，端到端延迟上升，内存占用增加。
- **改小的后果**：突发更容易丢，但排队延迟低。
- **联动**：与 `net.core.netdev_budget` / `netdev_budget_usecs` 联动——队列够大但预算不够时，包会在队列里多停留，`time_squeeze` 上涨，问题从「丢」变成「延迟」。也受多队列 / `RPS` 影响：分散到多 `CPU` 后每队列压力下降。
- **失败模式**：只调大本参数而不动预算，表现为丢包变成延迟与 `time_squeeze`；把值设得极大则在高突发下可能耗尽内存。
- **改法与验证**：`sysctl -w net.core.netdev_max_backlog=16384`，落盘到 `/etc/sysctl.d/` 持久化；验证看 `softnet_stat` 第 2 列的**增速**是否下降（累计值不会回落，只能看增量）。

### 5.2 net.core.dev_weight

- **默认值**：64（来源：Linux kernel documentation, *Documentation for /proc/sys/net/*）。
- **语义**：单次 `NAPI` 轮询允许处理的收包数上限，即驱动 `poll` 方法的 `budget` 参数。它是每 `CPU` 变量。
- **什么时候该改**：`softnet_stat` 第 3 列 `time_squeeze` 频繁增长、且单核软中断占用很高时，可适度调大以提高单次轮询吞吐。
- **改大的后果**：一次 `poll` 取走更多帧，中断次数与调度次数下降；但单次软中断持续时间变长，用户进程被抢占的间隔变长，调度抖动增大，尾延迟可能变差。
- **改小的后果**：软中断更短、更易被抢占，延迟更平滑；但轮询次数增加，`CPU` 花在进出软中断上的开销上升。
- **联动**：与 `dev_weight_rx_bias` / `dev_weight_tx_bias` 相乘生效（两者的默认值均为 1），并与 `netdev_budget` 一起限制单轮总量。
- **失败模式**：调得过大时，接收软中断会长时间占用 `CPU`，表现为应用线程的 `sy` 占比高、调度延迟变大。
- **改法与验证**：`sysctl -w net.core.dev_weight=128`，与 `netdev_budget` 一起观察 `time_squeeze` 与 `sy` 占比；多数场景调到 128 以内足够，继续加大收益递减。

### 5.3 net.core.somaxconn

- **默认值**：4096（待验证；较早内核为 128）。
- **语义**：`listen()` 的 `backlog` 参数上限，即 `TCP` **accept 队列**的最大长度。应用传入的 `backlog` 会被本值截断，取二者较小值。
- **什么时候该改**：服务端出现 `TcpExtListenOverflows` / `TcpExtListenDrops` 增长，且已确认应用 `accept()` 跟得上时，说明纯队列容量不足。
- **改大的后果**：能顶住短时连接突发，减少「连接被丢 / 被 `RST`」；代价是积压的连接占用内存与文件描述符。
- **改小的后果**：连接突发时更早开始丢新连接，客户端表现为连接被拒或超时。
- **联动**：必须与应用层 `listen(sockfd, backlog)` 的值一起调，否则应用传的小值会继续截断；握手阶段还受半连接队列 `net.ipv4.tcp_max_syn_backlog` 约束，后者先满时表现为 `TCPReqQFullDrop`，改 `somaxconn` 无效。连接建立与队列的细节见 [[05-TCP 协议]]。
- **失败模式**：只调大 `somaxconn` 而应用仍用默认小 `backlog`，或半连接队列太浅，都会出现「参数看起来够大但仍在丢连接」。
- **改法与验证**：`sysctl -w net.core.somaxconn=8192`，同时在应用里把 `listen()` 的 `backlog` 传成同等量级；验证看 `TcpExtListenOverflows` 是否停止增长。

### 5.4 参数之间的联动与失败模式

这几个参数分别管不同的队列，改动时必须成对看：

```text
   到达速率 ──▶ RX ring ──▶ backlog 队列 ──▶ 协议栈 ──▶ socket 接收缓冲区 ──▶ 应用
                │            │                          │
             ring 大小    netdev_max_backlog          tcp_rmem / rmem_max
             丢包:        丢包:                      丢包/零窗口:
             rx_missed    softnet_stat 第 2 列        RcvQDrop / RcvbufErrors

   预算侧:    netdev_budget(包数) / netdev_budget_usecs(时间)  → time_squeeze
   单轮侧:    dev_weight  → 单次 poll 处理多少帧
   连接侧:    somaxconn / tcp_max_syn_backlog  → ListenOverflows / TCPReqQFullDrop
```

常见的两类失败模式：

- **只加队列不加预算**：`netdev_max_backlog` 调大后包不再丢，但积压在队列里，`time_squeeze` 与接收延迟上升。根治要么提高预算，要么把负载分散到多队列 / 多 `CPU`。
- **只调内核不动应用**：`somaxconn`、`rmem_max` 都调大了，但应用 `accept()` / `read()` 慢，瓶颈只是从队列溢出变成了 `Recv-Q` 长期非零。内核参数只负责缓冲，消化速度由应用决定。

## 相关

- [[01-网络分层模型]] —— 四层骨架与逐层封装的字节账，本篇看它在 Linux 里的落地
- [[05-TCP 协议]] —— 连接建立、窗口与重传，收发的队列与缓冲区在协议侧的定义
- [[08-IP 协议与路由]] —— 网络层交付与路由判定，`ip_rcv` 与 `ip_queue_xmit` 背后的规则
- [[10-以太网与 ARP]] —— 帧格式、`FCS` 与 `ARP` 解析，邻居子系统补全下一跳 `MAC` 的依据
- [[00-网络专栏导览]] —— 本专栏的边界、目录与阅读顺序
- [[03-多卡互联与集群网络]] —— 多队列、`RSS` 与集群网络下的收发包路径

## 参考

- 小林coding. *图解网络 v4.0*. https://xiaolincoding.com/network/
- Linux kernel documentation. *NAPI*. https://www.kernel.org/doc/html/latest/networking/napi.html
- Linux kernel documentation. *Documentation for /proc/sys/net/*. https://www.kernel.org/doc/html/latest/admin-guide/sysctl/net.html
- Linux kernel source. *net/core/dev.c*. https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dev.c
- Linux kernel source. *net/ipv4/tcp_input.c*. https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/tcp_input.c
- prometheus/procfs. *net_softnet.go*. https://github.com/prometheus/procfs/blob/master/net_softnet.go
- *ethtool(8) manual page*. https://man7.org/linux/man-pages/man8/ethtool.8.html
