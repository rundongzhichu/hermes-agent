# Hermes 自主进化与学习系统

## 概述

Hermes Agent 在其核心运行时内置了**完整闭环学习回路**——这是内置能力，非外挂。学习回路跨越三个持久化层级，从最短暂到最持久：

```
压缩（Compression）──→ 记忆（Memory）──→ 技能（Skills）
  ↓ 会话中保留上下文      ↓ 跨会话回忆        ↓ 沉淀为持久"如何做"知识
```

---

## 一、学习回路全景

```
┌─────────────────────────────────────────────────────────────┐
│                     对话执行循环                              │
│  run_conversation() → LLM API → 工具调用 → 结果              │
└──────────────────┬──────────────────────────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
  ┌──────┐   ┌──────────┐   ┌──────────────┐
  │ 压缩  │   │ 后回调   │   │ 后台自改进审查 │
  │(会话中)│   │(turn结束)│   │ (fork agent)  │
  └──────┘   └──────────┘   └──────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              ┌─────────┐  ┌────────────┐  ┌──────────┐
              │ 更新技能 │  │ 沉淀记忆   │  │ 压缩轨迹  │
              │SKILL.md  │  │MEMORY.md   │  │离线训练   │
              └─────────┘  └────────────┘  └──────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │      Curator（技能库维护）      │
                    │  自动老化 → 归档 → LLM 合并  │
                    └─────────────────────────────┘
```

---

## 二、后台自改进审查 — 学习的发动机

### 代码路径

```
agent/background_review.py       # 审查 fork 核心实现
agent/agent_init.py              # 设定审查触发间隔
agent/turn_context.py            # 回合开始时触发 Memory 审查检查
agent/turn_finalizer.py          # 回合结束时触发 Skill 审查检查
```

### 触发机制

两个独立计数器驱动：

| 审查类型 | 计数器 | 配置键 | 默认间隔 |
|----------|--------|--------|----------|
| Memory Review | `_memory_nudge_interval` | `memory.nudge_interval` | 每 10 个用户回合 |
| Skill Review | `_skill_nudge_interval` | `skills.creation_nudge_interval` | 每 10 次工具调用迭代 |

**触发位置**（两处）：

1. **Memory 审查** — `agent/turn_context.py:211-217`：回合开始时，基于用户回合数检查
2. **Skill 审查** — `agent/turn_finalizer.py:377-381`：回合结束时，基于工具调用迭代数检查

**实际触发代码**（`turn_finalizer.py:393-401`）：
```python
if final_response and not interrupted and (_should_review_memory or _should_review_skills):
    agent._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

### 审查 Fork 详解（`background_review.py`）

`_run_review_in_thread()`（行 331-570）创建一个独立的 AIAgent fork，在与主线程**并行的后台线程**中运行，不会阻塞用户的下一轮交互：

**Fork 的属性：**
- 继承父级的运行时（provider、model、base_url、credentials）
- 继承缓存的系统提示（`_cached_system_prompt`）— 关键：可与父级共享 prompt cache
- 工具白名单仅限于 memory 和 skill 管理工具（`get_tool_definitions(enabled_toolsets=["memory", "skills"])`）
- `skip_memory=True` — 避免审查过程的交互泄露到用户记忆中
- `suppress_status_output=True` — 用户只看到最终动作摘要
- `compression_enabled=False` — 审查永不压缩

**三种审查提示变体**：

| 提示 | 位置 | 关注点 |
|------|------|--------|
| `_MEMORY_REVIEW_PROMPT` | 行 34 | 用户是否透露了身份/偏好/目标？ |
| `_SKILL_REVIEW_PROMPT` | 行 45 | **主动审查** — 大部分会话应产生至少一个技能更新 |
| `_COMBINED_REVIEW_PROMPT` | 行 150 | 一次性完成两项审查 |

### Skill Review 提示的核心原则

`_SKILL_REVIEW_PROMPT` 的关键指令：

> **Be ACTIVE** — most sessions produce at least one skill update.

优先级顺序：
1. **更新已加载的技能** — 如果当前会话使用了某个技能并发现不足
2. **更新已有的 umbrella skill** — 在已有大技能上追加而非创建新文件
3. **添加 support file** — 仅添加参考/模板文件到已有技能目录
4. **创建新的 umbrella skill** — 仅在全新领域时创建

> "When the user complains about how you handled a task, update the skill that governs that task — memory alone isn't enough."

---

## 三、技能系统 — 最持久的"如何做"知识

### 代码路径

```
tools/skill_manager_tool.py     # 技能 CRUD（创建、编辑、补丁、删除、写文件）
tools/skill_usage.py            # 使用遥测、生命周期状态、归档/恢复
tools/skill_provenance.py       # 区分后台审查 vs 用户直接写入
tools/skills_tool.py            # skill_view、skills_list 工具
agent/skill_utils.py            # frontmatter 解析、平台匹配、配置提取
agent/curator.py                # 后台技能库维护调度器（1836 行）
agent/curator_backup.py         # Curator 快照/回滚安全机制
```

### 技能目录布局

```
~/.hermes/skills/
  my-skill/
    SKILL.md                    # 主技能文件（必须）
    references/                 # 会话特定细节、知识库
    templates/                  # 可复制的模板文件
    scripts/                    # 静态可重复执行的脚本
    assets/                     # 辅助资源
  category/another-skill/SKILL.md
  .archive/                     # Curator 归档的技能（可恢复）
  .usage.json                   # 每技能遥测数据（原子写入）
  .curator_state                # Curator 调度状态
  .bundled_manifest             # 跟踪从仓库植入的内置技能
  .hub/lock.json                # Hub 安装技能锁文件
  .curator_suppressed           # 重植入器必须保持归档的内置技能
```

### 技能生命周期状态（`skill_usage.py`）

```
active ──(30天未使用)──→ stale ──(90天未使用)──→ archived → .archive/
  ↑                         │                        │
  └────(再次使用 reactivate)┘                        │
                                                     │
  pinned（正交布尔标记，阻止所有自动状态变更）←────────┘
```

### 技能来源区分（`skill_provenance.py`）

使用 `ContextVar` 区分写入来源：
- 默认值：`"foreground"` — 用户或正常工具调用创建
- 后台审查 fork 设置为 `"background_review"`
- 审查 fork 写入时调用 `mark_agent_created()`，在 `.usage.json` 中标记 `created_by: "agent"`

**意义**：`created_by: "agent"` 决定了哪些技能符合 Curator 管理的条件。

### 受保护内置技能（`skill_usage.py`）

`PROTECTED_BUILTIN_SKILLS` 当前只有 `"plan"` — 它支持 `/plan` 斜杠命令，绝不可被触碰。

---

## 四、Curator 系统 — 技能库后台维护

### 代码路径

```
agent/curator.py            # Curator 调度器 + LLM 合并审查
agent/curator_backup.py     # 快照/回滚安全网
```

### 运行时机

Curator 不是 cron 守护进程——它在 agent **空闲** 时运行，满足以下全部条件时触发：

| 条件 | 配置键 | 默认值 |
|------|--------|--------|
| 启用 | `curator.enabled` | `true` |
| 未暂停 | `hermes curator pause` | — |
| 距上次运行超过 | `curator.interval_hours` | 168 小时（7 天） |
| Agent 空闲超过 | `curator.min_idle_hours` | 2 小时 |

触发检查在 `should_run_now()`（行 198）。

### 两阶段执行（`run_curator_review()`，行 1407）

#### 阶段 1：自动状态变更（`apply_automatic_transitions()`，行 255）

纯时间驱动，无需 LLM：
- 遍历所有 Curator 管理的技能
- 未活跃超过 `stale_after_days`（30 天）→ 标记 `stale`
- 未活跃超过 `archive_after_days`（90 天）→ 移至 `.archive/`
- 之前标记 stale 但又被使用了 → `reactivated`
- 首次遇到的技能 → 初始化其不活跃时钟

#### 阶段 2：LLM 合并审查（`_run_llm_review()`，行 1676）

创建一个带综合合并提示的 fork agent：
- **目标形态**：类级 umbrella 技能，而非狭窄的单次会话技能
- **识别 "前缀簇"**：共享首词或领域关键词的技能
- **三种合并策略**：
  1. 合并到已有 umbrella 技能
  2. 创建新 umbrella 技能
  3. 降级为 support files
- **硬规则**：不删除（仅归档）、不碰 hub 安装的技能、不碰 pinned 技能
- **预期产出**：每次运行至少 10 条归档，否则 "你停得太早了"
- 输出结构化 YAML 以支持下游分类

### 安全机制（`curator_backup.py`）

每次 Curator 运行前自动创建快照：
- 快照内容：所有 SKILL.md + 目录、`.usage.json`、`.archive/`、`.curator_state`、`.bundled_manifest`、cron 作业
- 存储位置：`.curator_backups/<utc-iso>/`，tar.gz 格式
- 保留最近 5 个快照（可通过 `curator.backup.keep` 配置）
- 回滚：`rollback()` 先再拍一次安全快照，再提取选定快照

### Cron 作业引用重写（curator.py:1111-1136）

当 Curator 将技能 X 合并到 umbrella 技能 Y 时，任何引用 X 的 cron 作业会被就地重写为引用 Y。

---

## 五、记忆系统 — 跨会话知识固存

### 代码路径

```
agent/memory_manager.py       # MemoryManager 编排器
agent/memory_provider.py      # MemoryProvider 抽象基类
run_agent.py                  # 在 build_system_prompt/prefetch/sync 中接线
```

### 架构

```
MemoryManager
├── 内置 Provider（MEMORY.md / USER.md）
│   └── 存储位置：~/.hermes/memories/
└── 至多一个外部 Provider（Honcho / Hindsight / Mem0 / ...）
    └── 插件位置：plugins/memory/<name>/
    └── 激活方式：memory.provider 配置键
```

### MemoryProvider 生命周期（ABC 接口）

| 方法 | 调用时机 | 用途 |
|------|----------|------|
| `initialize()` | Agent 启动时 | 连接、创建资源 |
| `system_prompt_block()` | 构建系统提示时 | 注入记忆上下文 |
| `prefetch(query)` | 每回合开始前 | 召回相关记忆 |
| `queue_prefetch(query)` | 异步预取 | 为下一回合准备 |
| `sync_turn(user, assistant)` | 每回合结束后 | 持久化该回合 |
| `on_session_end(messages)` | 会话结束时 | 批量提取 |
| `on_pre_compress(messages)` | 压缩前 | 从即将丢弃的消息中提取 |
| `on_memory_write(...)` | 写入操作时 | 同步到外部 provider |
| `on_delegation(task, result)` | 子代理完成时 | 父代理观察子任务结果 |

### 在主管道中的接线位置（`run_agent.py`）

- `build_system_prompt()` — 从所有 provider 收集记忆块
- `prefetch_all(query)` — 每回合前的记忆召回
- `sync_all(user, assistant)` — 每回合后的持久化（**后台守护线程**执行，不阻塞 UI）
- `queue_prefetch_all(query)` — 为下一回合预填充

## 六、上下文压缩 — 会话中信息保留

### 代码路径

```
agent/context_compressor.py          # ContextCompressor 内置实现
agent/conversation_compression.py    # 压缩编排 + 会话轮换
agent/context_engine.py              # ContextEngine 抽象基类
agent/manual_compression_feedback.py # 压缩摘要面向用户展示
```

### 压缩流程（`ContextCompressor`）

1. **触发条件**：`should_compress()` — 回合后 `prompt_tokens >= threshold_tokens`，带防抖动保护
2. **预处理**：
   - `_prune_old_tool_results()` — 将旧工具输出替换为单行摘要
   - 去重相同工具结果
   - 截断过长工具参数
   - 剥离历史图片
3. **识别压缩区间**：保护头部（首个系统提示 + 前 3 回合）和尾部（按 token 预算保留最后 ~6 回合），仅压缩中间回合
4. **LLM 摘要**（`_generate_summary()`）：使用辅助摘要模型生成结构化摘要，包含：
   - 历史任务快照、目标、已完成操作、活跃状态、进行中、已阻塞
   - 关键决策、已解决/未决问题、相关文件、剩余工作、关键上下文
5. **迭代更新**：已有摘要时生成增量更新而非从头摘要
6. **降级兜底**：LLM 摘要失败时生成确定性降级摘要

### 会话轮换（`conversation_compression.py`）

- 获取压缩锁（原子操作，基于 `state.db`）
- 通知外部 memory provider（`on_pre_compress`）
- 轮换 `session_id`
- 创建新 SQLite 会话行，父级指向旧会话

### 防抖动

- 连续 2 次无效压缩（< 10% 节省）后跳过
- 摘要失败后冷却 600 秒
- 预检推迟：信任上次真实 provider 的 `prompt_tokens` 而非粗略估算

---

## 七、知识沉淀路径 — 从暂态到恒态

```
┌──────────────────────────────────────────────────────┐
│                   对话（暂态）                         │
│  每轮 LLM 交互 + 工具调用 + 结果                       │
└────────────┬─────────────────────────────────────────┘
             │
    ┌────────▼────────┐
    │   压缩 (Compression)  │  ← 会话中间回合被摘要为结构化参考
    │   保留上下文，防止窗口溢出  │    摘要仅作参考，非活动指令
    └────────┬────────┘
             │  on_pre_compress()
    ┌────────▼────────┐
    │   记忆 (Memory)       │  ← 外部 provider 从即将丢弃的消息中提取
    │   跨会话回忆           │  ← sync_turn 每回合持久化
    │   MEMORY.md / USER.md │  ← 系统提示块跨会话召回
    └────────┬────────┘
             │  后台审查 fork
    ┌────────▼────────┐
    │   技能 (Skills)       │  ← 审查 fork 主动创建/更新 SKILL.md
    │   持久 "如何做" 知识    │  ← 失败反馈 → 更新对应技能
    │   SKILL.md            │  ← provenance: "agent" vs "foreground"
    └────────┬────────┘
             │  Curator 定期维护
    ┌────────▼────────┐
    │   归档 (Archive)      │
    │   30天→stale           │  ← 自动老化
    │   90天→.archive/       │  ← 窄技能合并为 umbrella
    │   合并到类级技能        │  ← cron 引用自动重写
    └──────────────────────┘
```

### 离线训练数据管道（附加闭环）

```
trajectory_compressor.py  ← 离线后处理工具
         │
         ▼
  生产轨迹 (JSONL) → 压缩 → 训练数据集 → 模型微调 → 强化 Agent 能力
```

---

## 八、关键文件速查表

| 文件 | 职责 |
|------|------|
| `agent/background_review.py` | 回合后 Memory/Skill 自改进审查 fork |
| `agent/curator.py` | 后台技能库维护（自动老化 + LLM 合并） |
| `agent/curator_backup.py` | Curator 运行前快照/回滚安全网 |
| `agent/memory_manager.py` | 多 Provider 记忆编排 |
| `agent/memory_provider.py` | MemoryProvider 抽象基类 |
| `agent/context_compressor.py` | LLM 摘要生成 + 预处理优化 |
| `agent/conversation_compression.py` | 压缩编排 + 会话轮换 |
| `agent/context_engine.py` | ContextEngine 抽象基类（可插拔） |
| `agent/turn_context.py` | 回合开始前的记忆审查触发 |
| `agent/turn_finalizer.py` | 回合结束后的技能审查触发 + fork 启动 |
| `agent/agent_init.py` | 初始化 `_memory_nudge_interval` / `_skill_nudge_interval` |
| `agent/skill_utils.py` | 技能 frontmatter 解析、配置提取 |
| `tools/skill_manager_tool.py` | 技能 CRUD 工具 |
| `tools/skill_usage.py` | 技能遥测、生命周期、归档/恢复 |
| `tools/skill_provenance.py` | ContextVar 区分后台 vs 前台写入来源 |
| `tools/skills_tool.py` | skill_view / skills_list 工具 |
| `agent/insights.py` | 被动分析引擎（SQLite → 使用报告） |
| `trajectory_compressor.py` | 离线轨迹压缩（训练数据管道） |
| `hermes_state.py` | SQLite 会话存储（FTS5, 压缩锁） |
| `cron/` | 调度器（cron 作业引用技能，Curator 自动重写） |

---

## 九、关键配置

```yaml
# 审查触发
memory:
  nudge_interval: 10            # 每 N 个用户回合触发 Memory 审查
skills:
  creation_nudge_interval: 10   # 每 N 次工具迭代触发 Skill 审查

# Curator
curator:
  enabled: true
  interval_hours: 168           # 7 天
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    keep: 5                     # 保留最近 N 个快照

# 压缩
compression:
  threshold_tokens: 64000       # 触发压缩的 token 阈值
  anti_thrashing_threshold: 0.1 # 节省 < 10% 视为无效
  summary_failure_cooldown: 600 # 失败后 600 秒冷却

# 训练数据
trajectory_compressor:
  target_tokens: 15250
  protect_last_turns: 4
```
