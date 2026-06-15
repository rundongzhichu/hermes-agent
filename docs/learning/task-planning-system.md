# Hermes 任务规划系统

## 概述

Hermes Agent 拥有**两套并行的任务规划系统**，分别服务于不同的场景：

| 系统 | 持久性 | 跨度 | 适用场景 |
|------|--------|------|----------|
| **Kanban Board** | 持久化（SQLite） | 跨会话、多 Profile | 长期协作、跨会话任务接力 |
| **Subagent Delegation** | 内存级 | 单次会话内 | 短推理子任务、并行研究 |

---

## 一、Agent 主循环 — 一切执行的基础

### 代码路径

```
run_agent.py                          # AIAgent 类（~12k LOC），对外入口
agent/conversation_loop.py            # run_conversation() 核心循环
agent/chat_completion_helpers.py      # API 调用辅助
agent/tool_executor.py                # 工具执行调度
```

### 核心流程：`run_conversation()`（`agent/conversation_loop.py:436`）

```
while (api_call_count < max_iterations and budget.remaining > 0) or budget_grace_call:
    1. 检查中断信号
    2. 构建 API 消息（系统提示 + 对话历史 + 工具定义）
    3. 发起可中断的 LLM API 调用
    4. 解析响应：
       ├─ 如果是 tool_calls:
       │   ├─ 校验/修复工具名称和 JSON 参数
       │   ├─ 限制 delegate_task 并发数
       │   ├─ 去重
       │   ├─ 执行工具（串行或 ThreadPoolExecutor 并发）
       │   ├─ 检查是否需要压缩上下文
       │   └─ 回到步骤 1
       └─ 如果是文本回复:
           ├─ 持久化会话
           ├─ 刷新内存
           └─ 返回最终结果
```

- **API 模式**：支持 `chat_completions`（OpenAI 兼容）、`codex_responses`（OpenAI Codex）、`anthropic_messages`（Anthropic 原生），运行时自动解析
- **预算控制**：每个 agent 有独立的 `IterationBudget`（默认 90 次迭代），子代理受 `delegation.max_iterations` 限制（默认 50）
- **回调系统**：`tool_progress_callback`、`step_callback`、`thinking_callback` 等支持实时进度展示

---

## 二、Kanban Board — 持久化多 Agent 工作队列

### 代码路径

```
hermes_cli/kanban.py               # Kanban 调度器（2830 行），常驻后台循环
hermes_cli/kanban_decompose.py     # 任务分解引擎，Triage → 子任务图
hermes_cli/kanban_swarm.py         # Swarm 拓扑层，树形多 Agent 协作
gateway/kanban_watchers.py         # Gateway 模式下的 Kanban 看守
tools/kanban_tools.py              # 工人工具（kanban_show、kanban_complete 等）
agent/prompt_builder.py            # KANBAN_GUIDANCE 提示注入（行 181）
```

### 调度器生命周期（`hermes_cli/kanban.py`）

调度器是一个常驻后台循环，按固定节奏运行：

1. **回收过期声明** — 超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）无心跳的任务被回收
2. **提升就绪任务** — 前置任务全部完成的任务进入 `ready` 状态
3. **原子声明任务** — 以原子操作声明任务，避免并发冲突
4. **启动分配 Profile** — 为每个声明到的任务启动 AIAgent 实例

关键约束：
- 连续失败超过 `kanban.failure_limit`（默认 2 次）自动 block
- 通过 `HERMES_KANBAN_BOARD` 环境变量实现 Board 级隔离
- 数据存储在 `~/.hermes/kanban.db`（SQLite）

### 任务分解引擎（`hermes_cli/kanban_decompose.py`）

入口：`decompose_task(task_id)`（行 271）

工作流程：
1. 从 Triage 列取出任务
2. 调用辅助 LLM（"kanban decomposer"），系统提示要求将工作拆分为 **2-6 个子任务**
3. 每个子任务有 `parents` 数组（0-based 索引），表达数据依赖
4. 无 parent 的子任务可以 **并行** 执行
5. 如果 LLM 判断任务不可拆分，返回 `fanout: false`，任务保持单个

输出结构 `DecomposeOutcome`：
```
{
  ok: bool,
  tasks: [{title, body, assignee, parents[]}],
  fanout: bool
}
```

### Swarm 拓扑（`hermes_cli/kanban_swarm.py`）

入口：`create_swarm(conn, goal, workers, verifier_assignee, synthesizer_assignee)`（行 77）

一次 API 调用创建树形任务图：

```
planning root（立即完成）
    ├── 并行 specialist workers（就绪）
    └── verifier（TODO，等待所有 workers 完成）
         └── synthesizer（TODO，等待 verifier 完成）
```

- 使用 `task_comments` 中的结构化 JSON 评论作为共享**黑板**（`post_blackboard_update` / `latest_blackboard`）
- 幂等性：如果检测到已存在相同拓扑的非归档 root，则恢复图而非重复创建
- 依赖关系：worker 的 parent 指向 root，verifier 的 parents 是所有 worker ID，synthesizer 的唯一 parent 是 verifier

### 工人工具（`tools/kanban_tools.py`）

仅在调度器模式下（`HERMES_KANBAN_TASK` 环境变量）或 profile 启用 `kanban` toolset 时可见：

- `kanban_show()` — 展示任务详情
- `kanban_complete(summary, metadata)` — 完成并传递结构化 handoff
- `kanban_block(reason)` — 遇到模糊性时阻塞
- `kanban_heartbeat(note)` — 长时间操作期间保持任务活跃
- `kanban_comment(body)` — 添加评论
- `kanban_create(title, assignee, parents)` — 编排器创建子任务
- `kanban_link(source, target, kind)` — 链接任务
- `kanban_list(status, limit)` — 列出任务（需显式 toolset 启用）

### KANBAN_GUIDANCE 提示（`agent/prompt_builder.py:181`）

系统提示中注入的完整协议，包含生命周期指令：定向、心跳、阻塞、完成（带 handoff）、创建后续工作。其中 "Orchestrator mode"（行 231）指示 planner profile 使用 `kanban_create` 派生子任务而非自己执行。

---

## 三、Subagent Delegation — 内存级分层 Agent 树

### 代码路径

```
tools/delegate_tool.py              # delegate_task() 核心实现（2956 行）
agent/prompt_builder.py             # TOOL_USE_ENFORCEMENT_GUIDANCE（行 257）
```

### 入口：`delegate_task(goal, context, toolsets, tasks, role, parent_agent)`（行 2012）

**两种模式：**

| 模式 | 输入 | 行为 |
|------|------|------|
| **Single** | 一个 goal 字符串 | 生成一个子 agent |
| **Batch (Parallel)** | `tasks: [{goal, context, toolsets, role}]` | 每个在独立线程并发执行 |

最大并发数受 `delegation.max_concurrent_children` 限制（默认 3）。

### 角色系统

| 角色 | 能力 | 限制 |
|------|------|------|
| **leaf**（默认） | 专注执行 | 禁止 `delegate_task`、`clarify`、`memory`、`send_message`、`execute_code` |
| **orchestrator** | 可递归创建子代理 | 受 `delegation.orchestrator_enabled` 和 `max_spawn_depth` 限制 |

### 树深度控制

`delegation.max_spawn_depth`（默认 2）：
```
parent (depth 0) → orchestrator child (depth 1) → leaf grandchild (depth 2)
```

### 子代理构建（`_build_child_agent`，行 931）

在主线程构建完整的 `AIAgent`：
- **隔离会话**：不继承父级对话历史
- **独立 task_id**：独立的终端会话和文件缓存
- **受限工具集**：可配置，始终移除 blocked tools
- **专注系统提示**：由 goal + context 构建
- **凭据覆盖**：可将子代理路由到不同的 provider:model

### 子代理执行（`_run_single_child`，行 1412）

在 `ThreadPoolExecutor` 工作线程中运行：
- **心跳监控**：防止 Gateway 端不活跃超时
- **停滞检测**：检测迭代/工具是否不推进
- **可选硬超时**：`delegation.child_timeout_seconds`
- **文件状态协调**：父子之间的文件状态同步
- **可观测性**：`_active_subagents` 字典供 TUI 展示

### 全局暂停开关

`set_spawn_paused()` 允许 TUI 冻结新的 fan-out 而不中断活跃的子代理。

### 编排器系统提示（`_build_child_system_prompt`，行 624）

告知子代理其深度、嵌套能力，role=orchestrator 时包含 "orchestrator mode" 指令。

---

## 四、两套系统的组合关系

提示构建器明确区分两者（`prompt_builder.py:252`）：

> `delegate_task` is for short reasoning subtasks inside your own run; board tasks are for cross-agent handoffs that outlive one API loop.

| 维度 | Kanban Board | Subagent Delegation |
|------|-------------|---------------------|
| 持久化 | SQLite，重启后存活 | 内存，会话结束后消失 |
| 并发模型 | 调度器按节奏分配 | ThreadPoolExecutor 立即执行 |
| 上下文 | 每个 worker 独立系统提示 | 子代理隔离会话 |
| 适用场景 | 跨 Profile 协作、接力 | 单次运行内并行研究 |
| 工具集 | kanban_* 系列 | 受限/继承 |

---

## 五、关键配置项

```yaml
# Kanban
kanban:
  dispatch_stale_timeout_seconds: 14400  # 4 小时
  failure_limit: 2
  orchestrator_profile: "planner"
  dispatch_in_gateway: true

# Delegation
delegation:
  max_concurrent_children: 3
  max_spawn_depth: 2
  max_iterations: 50
  orchestrator_enabled: true
  child_timeout_seconds: 300
```
