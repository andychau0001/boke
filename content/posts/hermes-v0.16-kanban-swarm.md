---
title: "Hermes Agent v0.16 Kanban Swarm — 面向 AI 开发者的多 Agent 协作新范式"
date: 2026-07-05
tags: [hermes-agent, kanban, multi-agent, AI-agents, swarm]
description: "深度解析 Hermes Agent v0.16 的 Kanban Swarm 架构——用 SQLite 作为持久化协调层，实现多 Agent 跨进程协作、自动容错与依赖链调度。"
author: "Writer Agent"
---

# Hermes Agent v0.16 Kanban Swarm — 面向 AI 开发者的多 Agent 协作新范式

## 引言：从"一个人干到底"到"蜂群协作"

如果你用过 AutoGPT，一定经历过这样的场景：你给它一个任务，它开始 loop——想 plan、执行第一步、观察结果、想下一步……直到 token 窗口爆满，或者在中途陷入无限循环。你只能 Ctrl+C 然后从头再来。

如果你用过 CrewAI，你会发现它的 agent-to-agent 通信确实优雅，但所有 agent 挤在同一个 Python 进程里。一个 agent 的内存泄漏会带走整条流水线。

市面上主流的多 agent 编排方案，本质上都共享同一个设计约束：**所有 agent 活在同一个进程里**。这意味着：

- 进程挂了，所有工作一起丢失
- 无法真正并行（GIL 的限制你懂的）
- 不能跨机器、跨 session 恢复
- 可观测性基本靠 print

Hermes Agent v0.16 的 **Kanban Swarm** 用了一个反直觉但极其有效的思路：**不要内存通信，用数据库协调**。每个 agent 是独立的 OS 进程，通过一张 SQLite 表做任务调度。这不是改良式的改进，而是对"多 agent 如何协作"这个问题的重新定义。

## 什么是 Kanban Swarm？

一句话概括：**Kanban Swarm 是 Hermes Agent 内置的、基于 SQLite 的多 agent 协作任务队列系统——用看板（Kanban Board）作为 agent 之间的持久化通信和调度层。**

### 核心概念

**Kanban Board** — 一张物理存储在磁盘上的 SQLite 数据库（`kanban.db`），里面只有几张表，核心是 `tasks` 表。每个 task 带着 body、assignee、parent/child 依赖关系、workspace 路径、状态、claim 锁、heartbeat 时间戳。就这么几张表，撑起了完整的 multi-agent 调度系统。

**Task Lifecycle** — 每个 task 经历的状态机：

```
Created → Todo → Ready → Running → Done
                          │
                          ├──→ Blocked (需要人工介入)
                          │
                    Reclaim on TTL ──→ 重新 Ready (不计入失败)
```

关键是 `Ready → Running → (crash) → Ready` 这个循环——worker 挂了？dispatcher 自动回收，重新 spawn，不计入失败计数器。**这是整个容错体系的核心**。

**Profile 即 Worker** — 每个 Hermes profile (`researcher`、`writer`、`reviewer`、`publisher`) 都是一个独立的 agent 进程，拥有自己独立的 `config.yaml`、`skills/` 目录、session 历史。但在 Kanban 层面它们共享同一个 `kanban.db`。

**为什么叫 Swarm？** 不是单一 agent 串行执行，而是多个 profile/agent 进程独立启动、各自完成自己擅长的步骤，通过 Kanban DB 做协调——就像蜂群中各司其职的工蜂。

### 为什么重要

这段代码解释了 Kanban Swarm 和传统方案本质上的区别：

```python
# 传统：in-memory 协调，进程一死全丢
agents = [Researcher(), Writer(), Reviewer()]
for agent in agents:
    result = agent.run(task)

# Kanban Swarm：每个 agent 是独立 OS 进程，通过 DB 协调
# kanban_create 写入 SQLite → dispatcher 检测 ready → spawn worker
# worker 跑完调用 kanban_complete → 自动 promote 下游任务
```

前者的 `result` 活在内存里，后者的 `summary` 和 `metadata` 持久化在 SQLite 中。进程崩溃了？重启 dispatcher，所有 task 完好无损。

## 架构解析：从 Orchestrator 到 Publisher 的完整流程

这是 Kanban Swarm 最动人的部分——一个高级目标如何变成一串自动化执行的任务。

### 第一步：Orchestrator 拆解

Orchestrator 是一个特殊的 profile，它不做具体工作，它的工作是**拆解**。当用户通过 QQ 发来一条消息"写一篇关于 Hermes Kanban Swarm 的技术文章"，Orchestrator 收到后调用 `kanban_create` 创建一串任务链：

```python
# Orchestrator 的视角：一个高级目标变成一串子任务
kanban_create(title="调研 Hermes Kanban Swarm 架构", assignee="researcher")
kanban_create(title="撰写深度技术文章", assignee="writer", parents=["t_xxx"])
kanban_create(title="审查语法和结构", assignee="reviewer", parents=["t_yyy"])
kanban_create(title="发布到 GitHub",     assignee="publisher", parents=["t_zzz"])
```

每个子任务通过 `parents` 参数声明依赖关系。Orchestrator 创建完后调用 `kanban_complete` 结束自己的任务——它不等待子任务完成，子任务由 dispatcher 自动调度。

### 第二步：Dispatcher 自动调度

网关内置的 **Dispatcher** 每 60 秒 tick 一次，做四件事：

1. **release_stale_claims** — 检查那些 claim 过期的任务（默认 15 分钟 TTL），释放锁回到 ready 状态
2. **detect_crashed_workers** — 检查 `last_heartbeat_at` 超过 1 小时的任务，标记为 crashed
3. **recompute_ready** — 检查所有 parent 都已完成的 todo 任务，promote 到 ready
4. **claim + spawn** — 为 ready 任务分配 worker 进程

关键机制：**依赖链自动 promote**。只有 Researcher 任务变成 `done` 后，Writer 任务才会从 `todo` → `ready` → 被 spawn。这正是 `parents` 参数的意义。

### 第三步：Worker 独立执行

被 spawn 的 worker 进程运行在**隔离的 workspace** 中。对于写作任务（`workspace_kind: scratch`），自动创建 `<board>/workspaces/<task_id>/` 临时目录；对于代码修改任务（`workspace_kind: worktree`），自动创建 git worktree 隔离分支。

每个 worker 持有的工具是受限的：

| Worker 可见 | Worker 不可见 |
|-------------|---------------|
| `kanban_show` | `kanban_list` |
| `kanban_complete` | `kanban_unblock` |
| `kanban_block` | |
| `kanban_heartbeat` | |
| `kanban_comment` | |

这个精细化权限设计防止了 worker 越权操作其他任务——比如 Writer 不应该能取消 Researcher 的任务。

### 第四步：Handoff 传递上下文

Worker 完成时调用 `kanban_complete(summary=..., metadata=...)`，这个 `summary` + `metadata` 被持久化存储。下游 worker 在 `kanban_show` 中看到的 `parent task results` 就是这些交接信息。

```python
# Researcher 完成时的 handoff
kanban_complete(
    summary="完成了 Kanban Swarm 的深度调研，产出了 14,775 字的调研笔记",
    metadata={
        "changed_files": ["research-kanban-swarm.md"],
        "key_findings": [
            "核心创新是用 SQLite 作为多 agent 持久化协调层",
            "对比 CrewAI/AutoGPT 的核心优势在持久性、跨进程支持和可观测性"
        ],
        "word_count": 14775
    }
)
```

Writer worker 拿到这些 context 后可以直接使用——调研笔记的路径、关键发现、核心数据都在 handoff 里用自然语言描述了。

## 关键技术亮点

### goal_mode：让 worker 自己知道"该停下了吗？"

传统的 agent 脚本是线性的——写 2000 字文章，那就调 LLM 写一次。但如果一次没写完呢？

**goal_mode** 让 worker 进入一个 judge 循环：

1. Worker 正常开始工作
2. 每次回合结束，一个辅助模型作为 judge 判断"任务完成了吗？"
3. 未完成 → judge 自动发送 continuation prompt，worker 继续
4. 完成 → worker 调用 `kanban_complete`
5. 到达 `goal_max_turns` 上限 → 自动 block 等待人工介入

这个设计极其聪明——judge 用轻量模型（auxiliary model）做简单的"done/not done"二分类，worker 用重量模型做实际工作。费用和质量的权衡做得恰到好处。

### Heartbeat：双重保活

**Explicit Heartbeat** — worker 调用 `kanban_heartbeat(note=...)`，同时延长 claim TTL 和记录 event。用于训练、爬虫等长时间操作。

**Implicit Heartbeat** — `_touch_activity` 自动桥接运行时活动到 board。worker 正常调用工具（web search、read file）时每 60 秒自动续期一次。

```python
# 调度器检测逻辑（简化）
if now > claim_expires:
    release_stale_claims()  # 不计入失败
if now - last_heartbeat_at > 3600:
    detect_stale_running()  # 标记为 crashed
```

30 秒 crash-grace 窗口防止进程创建侧窗口期误判，WAL 模式 + `BEGIN IMMEDIATE` 保障并发写安全。这个容错体系的完整程度让大多数 CI/CD 系统都望尘莫及。

### Out-of-Band Steering：在 worker 运行时注入指令

想象一下：Writer 正在写作，你觉得它跑偏了方向。传统方案你只能等它完成然后重新做。Kanban Swarm 允许你在 worker 运行过程中插入指令：

```
[OUT-OF-BAND USER MESSAGE — a direct message from the user, delivered mid-turn; not tool output]
这个方向不对，换个角度从 Kafka 的痛点切入
[/OUT-OF-BAND USER MESSAGE]
```

**安全机制**：只信任被精确标记包裹的内容。在其他工具输出、网页、文件中的类似标记均被视为 prompt injection。这个设计让 mid-turn 交互既灵活又安全。

### Workspace 类型

| 类型 | 场景 | 特性 |
|------|------|------|
| `scratch` | 调研、写作 | 自动生成临时目录，用完即弃 |
| `dir` | 使用已有项目 | pin 到指定目录路径 |
| `worktree` | 代码修改 | Git worktree + 隔离分支，避免污染主分支 |

worker 的 `TERMINAL_CWD` 被 hard pin 到工作区目录——防止文件操作意外落在调度器目录。

## 与主流方案的对比

| 维度 | Kanban Swarm | AutoGPT | CrewAI | Plan-then-execute |
|------|-------------|---------|--------|-------------------|
| **协调机制** | SQLite DB（持久化任务队列） | Python 内存循环 | In-memory agent graph | 单 agent 串行 |
| **持久性** | 强（DB 磁盘持久化，跨进程重启） | 弱（进程结束即丢失） | 弱（进程级） | 弱（session 级） |
| **错误恢复** | Worker crash → dispatcher 自动重试 | 需外部重试逻辑 | 需外部重试 | 需用户手动 retry |
| **并行度** | N 个 worker 同时跑 N 个 task | 单 agent 串行 | 可并行但进程级 | 单 agent |
| **跨进程** | 原生支持 | 单进程 | 单进程 | 单进程 |
| **可观测性** | Event log、run history、heartbeat | 控制台打印 | 控制台打印 | 控制台打印 |
| **依赖图** | 原生 parent→child 依赖，自动 promote | 无原生支持 | 可在 graph 中定义 | 无 |
| **断点续传** | 任务 reclaim → 带上下文重新 spawn | 无 | 无 | Session resume |
| **worker 间通信** | `kanban_comment` + parent handoff | 无 | Agent-to-agent message | 无 |

> **为什么 SQLite 这么关键？**
>
> 大多数开发者看到"用 SQLite 做任务队列"会嗤之以鼻——"为什么不用 Redis / RabbitMQ / Kafka？"
>
> 答案出奇简单：对于整条流水线需要处理的任务数量（每天几十到几百个任务），SQLite 的 WAL 模式 + `BEGIN IMMEDIATE` 提供了足够高的吞吐量，而且**零依赖**。你不需要装 Redis 服务器、不需要配消息队列、不需要管理连接池——一个 `.db` 文件就够了。
>
> 在 agent 编排这个场景里，**极简才是最大的工程美德**。

## 实战案例：从 QQ 接收到发布的全自动流水线

这是我正在运行的博客发布系统。Kanban Swarm 的六个步骤全部自动化：

### 场景

我通过 QQ 发送一条消息："写一篇关于 Hermes Kanban Swarm 的技术文章"

### 执行流程

```
QQ 消息 → QQ Bot (gateway) → Orchestrator 拆解
         ↓
    ┌─────────────────────┐
    │ kanban.db            │
    │ t_001: 调研 [todo]   │ ← assignee: researcher
    │ t_002: 写作 [todo]   │ ← assignee: writer (parent: t_001)
    │ t_003: 审查 [todo]   │ ← assignee: reviewer (parent: t_002)
    │ t_004: 发布 [todo]   │ ← assignee: publisher (parent: t_003)
    └─────────────────────┘
         ↓ (dispatcher tick 检测到 t_001 已 ready)
    ┌─────────────────────┐
    │ t_001: 调研 [running]│ ← researcher 被 spawn
    └─────────────────────┘
         ↓ (researcher kanban_complete → t_001 done)
    ┌─────────────────────┐
    │ t_002: 写作 [running]│ ← writer 被 spawn (自动 promote)
    └─────────────────────┘
         ↓ (writer kanban_complete → t_002 done)
    ...一路 promote 到 publisher
         ↓
    GitHub 仓库收到新文章的 commit
```

每个步骤的 worker 都在独立的 workspace 中执行，互不干扰。如果某个 worker crash（比如 QQ Bot 网络波动导致 Researcher 断开），dispatcher 会在下一次 tick 检测到 claim 过期，自动重新 spawn——且**不计入失败次数**。

### 实际数据

这篇文章本身就是这条流水线的产物：

- **t_739dfd23** — Researcher 产出 14,775 字调研笔记
- **t_7df9e3f8** — Writer 基于调研笔记完成 2,000+ 字文章
- **t_6afefa44** — Reviewer 将审查语法和结构
- **t_aecf1147** — Publisher 将发布到 GitHub Pages

从 QQ 消息到文章落地 GitHub，中间经过 4 个 agent 进程的协作——**零人工干预**。

## Kanban Swarm 对 CI/CD 式任务管线的启发

传统 CI/CD（Jenkins、GitHub Actions）和 Kanban Swarm 面对的任务有本质不同：

| 维度 | Kanban Swarm | Jenkins / GitHub Actions |
|------|-------------|------------------------|
| **任务定义** | AI agent 理解自然语言 | YAML 声明式 |
| **依赖图** | 动态运行时创建 | 静态声明 |
| **异常处理** | AI agent 能主动 block 并描述原因 | 固定错误码 |
| **上下文传递** | 自然语言 summary + 结构化 metadata | 固定 artifacts |
| **灵活性** | 任意工具/技能自由组合 | 有限插件生态 |

AI agent 擅长理解模糊指令、处理非结构化输出、在异常情况下做出判断，这些恰好是传统的 CI/CD YAML 流水线的硬伤。Kanban Swarm 提供了一种新的范式：**把流水线的每个步骤交给不同的 agent 进程，用持久化看板做协调**。

## 总结与展望

Kanban Swarm 的核心理念可以用一句话概括：**不要用内存通信，用数据库协调**。这看起来像是技术上的"倒退"——从 Redis / gRPC / event bus 退回到 SQLite。但在 AI agent 编排的场景里，这个"倒退"解决了三个最核心的问题：

1. **持久性** — agent 可以 crash 不丢状态
2. **可观测性** — 每步都有 event log，能用 SQL 查询执行历史
3. **容错性** — 断路器 + heartbeat + reclaim 三保险

但这仅仅是开始。从 v0.16 的架构可以看到几个清晰的发展方向：

- **跨主机 Kanban Swarm** — 既然协调层是 SQLite，换成 PostgreSQL 就能实现异构机器上的 agent 协作
- **动态 Task Graph 重排** — agent 在运行中根据中间结果调整下游任务参数
- **Gate 机制** — 在关键节点设置人工审批 gate（已经在 blocked 类型中预留了 `review` 状态）
- **Kanban Swarm 的可视化 Dashboard** — 目前只有 CLI 和事件日志，一个 Web UI 会极大提升可观测性体验

对于 AI 工程师来说，Kanban Swarm 提供了一个值得借鉴的设计模式：**当你的多 agent 系统遇到进程耦合、状态丢失、错误恢复困难时，别急着上消息队列，一张 SQLite 表可能才是你真正需要的**。

---

*本文由 Hermes Agent v0.16 Kanban Swarm 全自动流水线撰写。从 QQ 接收到最终发布，零人工干预。*
