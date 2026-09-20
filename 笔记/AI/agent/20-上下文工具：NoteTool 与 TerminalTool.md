---
tags:
  - AI/agent/上下文
---

# 上下文工具：NoteTool 与 TerminalTool

[[19-上下文工程的工程实践]] 里提到的「结构化笔记」和「JIT 上下文」是两种思路，落到工具层分别是 NoteTool 和 TerminalTool。两者的分工是：**NoteTool 把状态写到窗口之外，TerminalTool 让智能体不去预先加载、而是运行时现查。**

## NoteTool：把状态写到窗口之外

### 和 MemoryTool 的分工

[[10-智能体记忆系统]] 的 MemoryTool 关注**对话式记忆**（短期工作记忆、情景记忆、语义记忆）。对需要长期追踪、结构化管理的**项目式任务**，需要另一种更轻、更人类友好的记录方式。

NoteTool 的四个特性：

| 特性 | 说明 |
| --- | --- |
| 结构化记录 | Markdown + YAML，机器可解析，人也易读易改 |
| 版本友好 | 纯文本，天然进 Git |
| 低开销 | 不走数据库，适合轻量状态追踪 |
| 灵活分类 | `type` + `tags` 两个维度组织，支持多维检索 |

### 四种笔记类型

长期项目追踪里，笔记按用途分四类：

```
task_state   当前阶段的任务状态与进度
conclusion   每个阶段结束后的关键结论
blocker      遇到的问题和阻塞点
action       下一步的行动计划
```

```python
notes.run({
    "action": "create",
    "title": "重构项目 - 第一阶段",
    "content": "已完成数据模型层的重构,测试覆盖率达到85%。下一步将重构业务逻辑层。",
    "note_type": "task_state",
    "tags": ["refactoring", "phase1"]
})

notes.run({
    "action": "create",
    "title": "依赖冲突问题",
    "content": "发现某些第三方库版本不兼容,需要解决。影响范围:业务逻辑层的3个模块。",
    "note_type": "blocker",
    "tags": ["dependency", "urgent"]
})
```

**`blocker` 单独成一类是有道理的**：它和 `action` 的区别在于，阻塞项需要被反复拉回上下文，直到解决为止；而 action 完成一次就可以丢弃。

同一套机制也适用于非编码场景——研究任务里记每篇论文的核心观点（`conclusion`）、待调研主题（`action`）、重要参考文献（`reference`）。

### 存储格式：YAML 头 + Markdown 正文

每个笔记是一个独立 `.md` 文件：

```markdown
---
id: note_20250119_153000_0
title: 项目进展 - 第一阶段
type: task_state
tags: [refactoring, phase1, backend]
created_at: 2025-01-19T15:30:00
updated_at: 2025-01-19T15:30:00
---

# 项目进展 - 第一阶段

## 完成情况
...
```

三个设计点：

| 点 | 说明 |
| --- | --- |
| **YAML 元数据** | 机器可解析，支持精确的字段提取与检索——**这是它和纯 Markdown 笔记的本质差别** |
| **Markdown 正文** | 人类可读，支持标题、列表、代码块这些富格式 |
| **文件名即 ID** | 每个笔记的**文件名就是它的唯一标识**，不需要额外维护一个 id→文件的映射表 |

**`id` 的构成规则**：`note_<YYYYMMDD>_<HHMMSS>_<序号>`。序号那一位是给**同一秒内创建多条笔记**用的——时间戳精度不够时的兜底。

### 索引文件：`notes_index.json`

```json
{
  "note_20250119_153000_0": {
    "id": "note_20250119_153000_0",
    "title": "项目进展 - 第一阶段",
    "type": "task_state",
    "tags": ["refactoring", "phase1", "backend"],
    "created_at": "2025-01-19T15:30:00",
    "updated_at": "2025-01-19T15:30:00",
    "file_path": "./notes/note_20250119_153000_0.md"
  }
}
```

**它是一份「元数据的副本」**——正文存在各个 `.md` 里，索引里只放元数据加一个 `file_path`。三个作用：

| 作用 | 说明 |
| --- | --- |
| **快速检索** | 不用打开每个文件，直接从索引里按 `type` / `tags` 过滤 |
| **元数据管理** | 集中管理所有笔记的元数据 |
| **完整性校验** | **可以检测文件缺失或损坏**——索引里有但文件不在，就是不一致 |

最后那条是索引带来的额外价值，也是它的代价：**索引和文件是两份数据，必须同步维护**。`create` / `update` / `delete` 三个操作都要同时改两处，漏一处就会出现「索引说有、文件没有」的漂移。

### 七个操作覆盖完整生命周期

`create` / `read` / `update` / `search` / `list` / `summary` / `delete`。

对照 [[18-Context Engineering|Context Engineering]] 里那五个记忆阶段（编码 / 存储 / 检索 / 整合 / 遗忘）：**这七个操作里没有「整合」**——NoteTool 是纯结构化的外部存储，不做「短期转长期」那一步。那是 [[10-智能体记忆系统|MemoryTool]] 的 `consolidate` 负责的。

### 接进 ContextBuilder

每一轮对话前检索相关笔记，转成 `ContextPacket` 注入：

```python
def run(self, user_input: str) -> str:
    relevant_notes = self.note_tool.run({
        "action": "search", "query": user_input, "limit": 3
    })

    note_packets = [ContextPacket(
        content=note['content'],
        timestamp=note['updated_at'],
        token_count=self._count_tokens(note['content']),
        relevance_score=0.7,                     # 笔记的基础相关性给 0.7
        metadata={"type": "note", "note_type": note['type']}
    ) for note in relevant_notes]

    context = self.context_builder.build(
        user_query=user_input, custom_packets=note_packets, ...
    )
```

注意 `relevance_score` 被赋成 **0.7** 而不是让它按默认的 0.5 走。这是在告诉 Select 阶段「**主动检索命中的笔记比一般候选更值得占预算**」——它已经过了一轮检索筛选，命中本身就携带信息。

## TerminalTool：即时文件系统访问

### 为什么需要它

三类典型场景的共同点是**需要实时、轻量级的文件系统访问，而不是预先索引和向量化**：

| 场景 | 预索引的做法 | TerminalTool 的做法 |
| --- | --- | --- |
| 代码库探索 | `rag_tool.add_document("./project/**/*.py")`——耗时、占存储、可能过时 | `find . -name '*.py' -type f` → `grep -r 'class UserService' .` → `head -n 50 src/services/user.py` 按需查看 |
| 日志分析 | 全量入库再检索 | `ls -lh` 看大小 → `tail -n 100 \| grep ERROR` → 管道统计错误类型分布 |
| 数据文件预览 | 解析入库 | `head -n 5` 看前几行 → `wc -l` 数行 → `head -n 1 \| tr ',' '\n'` 看列名 |

这一类工作流对应 [[19-上下文工程的工程实践]] 里的 JIT 思路：**维护轻量引用，运行时按需加载**，用 `head` / `tail` / `grep` 这类命令就地分析大体量数据。

### 多层安全机制

允许智能体执行命令是强大但危险的能力。第一层也是最关键的一层是**命令白名单**——只放行安全的只读命令，完全禁止任何可能修改系统的操作：

```python
ALLOWED_COMMANDS = {
    # 文件列表与信息
    'ls', 'dir', 'tree',
    # 文件内容查看
    'cat', 'head', 'tail', 'less', 'more',
    # 文件搜索
    'find', 'grep', 'egrep', 'fgrep',
    # 文本处理
    'wc', 'sort', 'uniq', 'cut', 'awk', 'sed',
    ...
}
```

**白名单的判据是「只读」**：列表、查看、搜索、文本处理全部只读；写入、删除、权限变更、网络请求一律不在表里。这让「智能体误用工具」的后果上限被压在「读到了不该读的东西」，而不是「改坏了系统」。

白名单完整覆盖这几类：文件列表与信息（`ls` / `dir` / `tree`）、内容查看（`cat` / `head` / `tail` / `less` / `more`）、搜索（`find` / `grep` / `egrep` / `fgrep`）、文本处理（`wc` / `sort` / `uniq` / `cut` / `awk` / `sed`）、目录操作（`pwd` / `cd`）、文件信息（`file` / `stat` / `du` / `df`）、其他（`echo` / `which` / `whereis`）。

**拒绝时要报出允许列表**，而不是只说「不允许」：

```
terminal.run({"command": "rm -rf /"})
# ❌ 不允许的命令: rm
# 允许的命令: cat, cd, cut, dir, du, ...
```

这是给模型看的——**它下一轮可以据此换一个合规命令**，而不是继续试。

**但白名单只是第一层。** 完整的安全机制有四层：

| 层 | 机制 | 配置 / 行为 |
| --- | --- | --- |
| **1 · 命令白名单** | 只放行只读命令 | 见上 |
| **2 · 工作目录限制（沙箱）** | 只能访问指定工作目录及其子目录 | `TerminalTool(workspace="./project")` |
| **3 · 超时控制** | 每个命令有执行时间上限，防无限循环与资源耗尽 | `TerminalTool(workspace=..., timeout=30)` |
| **4 · 输出大小限制** | 限制命令输出体积，防内存溢出 | `TerminalTool(workspace=..., max_output_size=10*1024*1024)` |

第 2 层最值得注意的是它**显式防了路径逃逸**：

```
terminal.run({"command": "cat ./src/main.py"})     # ✅ 工作目录内
terminal.run({"command": "cat /etc/passwd"})       # ❌ 不允许访问工作目录外的路径
terminal.run({"command": "cd ../../../etc"})       # ❌ 不允许访问工作目录外的路径
```

**第三行是这条防线的关键**——只挡绝对路径的沙箱等于没挡，`..` 一路退出去照样能读系统文件。**沙箱必须在路径解析之后做校验**，而不是在字符串层面匹配前缀。

第 3、4 层对应 [[02-Agent Loop|Agent Loop]] 里那三道硬上限中的两道：**超时抓「卡住的工具调用」，输出上限抓「单次调用吃光内存」**。区别在于这里把它们下沉到了工具层——不依赖上层循环自觉，工具自己就拒绝执行。

四层合起来的设计意图：**即使智能体的行为出现异常，也无法影响系统其他部分。**

## 两个工具的设计对照

| | NoteTool | TerminalTool |
| --- | --- | --- |
| 解决的问题 | 状态跨窗口存活 | 数据不进窗口 |
| 数据流向 | 主动写入外部存储，后续拉回 | 运行时查询，结果直接进上下文 |
| 粒度 | 结构化的笔记条目 | 任意只读命令 |
| 风险 | 低（只写自己的存储） | 高（能读整个文件系统），靠白名单约束 |
| 对应手段 | 结构化笔记（Structured note-taking） | JIT 上下文 + 渐进式披露 |

两者互补：**TerminalTool 负责「查得到」，NoteTool 负责「记得住」**。

## 相关

- [[19-上下文工程的工程实践]] —— 这两个工具承载的两种手段
- [[10-智能体记忆系统]] —— MemoryTool 与 NoteTool 的分工边界
- [[11-RAG 检索增强]] —— 预索引路线的代表，与 TerminalTool 的 JIT 路线相对

## 参考

- 来源：《Hello-Agents》第九章 §9.4–§9.5
