# Hermes Agent 核心架构原理详解

本文档深入解析 Hermes Agent 的核心系统架构，包括 Skills、Plugins、Cron、Subagent、Hooks、Tools 和自进化机制的工作原理。

---

## 目录

1. [Skills 技能系统](#1-skills-技能系统)
2. [Plugins 插件系统](#2-plugins-插件系统)
3. [Cron 定时任务系统](#3-cron-定时任务系统)
4. [Subagent 子代理委托系统](#4-subagent-子代理委托系统)
5. [Hooks 钩子系统](#5-hooks-钩子系统)
6. [Tools 工具注册与发现](#6-tools-工具注册与发现)
7. [自进化机制](#7-自进化机制)

---

## 1. Skills 技能系统

### 1.1 核心理念

Skills 是 **按需加载的知识文档**，采用 **渐进式披露（Progressive Disclosure）** 模式，最小化 Token 消耗。遵循 [agentskills.io](https://agentskills.io/specification) 开放标准。

**关键特性：**
- **Agent 可管理**：Agent 可以创建、修改、删除自己的 Skills
- **渐进式加载**：先看到列表 → 按需加载完整内容 → 按需加载引用文件
- **条件激活**：根据平台、可用工具集动态显示/隐藏
- **单一数据源**：所有 Skills 存储在 `~/.hermes/skills/`

### 1.2 存储结构

```
~/.hermes/skills/
├── software-development/          # 分类目录
│   ├── subagent-driven-development/
│   │   ├── SKILL.md               # 主文件（必需）
│   │   ├── references/            # 引用文档
│   │   ├── templates/             # 模板文件
│   │   └── assets/                # 资源文件
│   └── code-review/
│       └── SKILL.md
└── mlops/
    └── axolotl/
        └── SKILL.md
```

### 1.3 SKILL.md 格式

```markdown
---
name: my-skill
description: Brief description of what this skill does
version: 1.0.0
platforms: [macos, linux]     # 可选：限制特定操作系统
metadata:
  hermes:
    tags: [python, automation]
    category: devops
    fallback_for_toolsets: [web]    # 当 web 工具集可用时隐藏此 Skill
    requires_toolsets: [terminal]   # 仅当 terminal 工具集可用时显示
    config:                          # 可选：声明需要的配置项
      - key: my.setting
        description: "What this controls"
        default: "value"
        prompt: "Prompt for setup"
---

# Skill Title

## When to Use
触发条件说明

## Procedure
1. Step one
2. Step two

## Pitfalls
- Known failure modes and fixes

## Verification
如何确认执行成功
```

### 1.4 渐进式加载流程

```
Level 0: skills_list()           → [{name, description, category}, ...]   (~3k tokens)
Level 1: skill_view(name)        → Full content + metadata       (varies)
Level 2: skill_view(name, path)  → Specific reference file       (varies)
```

**工作原理：**
1. **启动时**：只扫描 Skills 索引（名称、描述、分类），生成轻量级清单
2. **Agent 调用时**：通过 `skills_list()` 获取清单，判断是否需要加载
3. **匹配时**：调用 `skill_view(name)` 加载完整 SKILL.md 内容
4. **需要参考文件时**：调用 `skill_view(name, path)` 加载特定文件

### 1.5 条件激活机制

Skills 可以根据以下条件动态显示/隐藏：

#### 平台过滤
```yaml
platforms: [macos]            # 仅 macOS（如 iMessage、Apple Reminders）
platforms: [macos, linux]     # macOS 和 Linux
```

#### 工具集依赖
```yaml
metadata:
  hermes:
    fallback_for_toolsets: [web]    # 当 web 工具集可用时隐藏（作为备选方案）
    requires_toolsets: [terminal]   # 仅当 terminal 工具集可用时显示
```

**逻辑：**
- `fallback_for_toolsets`: 如果指定的工具集**可用**，则隐藏此 Skill（因为已有更好的原生工具）
- `requires_toolsets`: 如果指定的工具集**不可用**，则隐藏此 Skill（因为无法执行）

### 1.6 Skill 管理工具（skill_manage）

Agent 可以通过 `skill_manage` 工具自主创建和管理 Skills：

```python
skill_manage(
    action="create",
    name="my-new-skill",
    content="""---
name: my-new-skill
description: A new skill created by the agent
---

# My New Skill
...
""",
    category="devops"
)
```

**支持的操作：**

| 操作 | 用途 | 关键参数 |
|------|------|----------|
| `create` | 从头创建新 Skill | `name`, `content`（完整 SKILL.md）, `category`（可选） |
| `patch` | 针对性修复（推荐） | `name`, `old_string`, `new_string` |
| `edit` | 大规模结构重写 | `name`, `content`（完整替换） |
| `delete` | 完全删除 Skill | `name` |
| `write_file` | 添加/更新辅助文件 | `name`, `file_path`, `file_content` |
| `remove_file` | 删除辅助文件 | `name`, `file_path` |

**Agent 何时创建 Skills：**
- 成功完成复杂任务（5+ 次工具调用）后
- 遇到错误但最终找到解决方案时
- 用户纠正了 Agent 的方法时
- 发现了非平凡的工作流程时

### 1.7 Skills Hub（技能市场）

从在线注册表、skills.sh、官方可选 Skills 等来源浏览、搜索、安装和管理 Skills。

**支持的来源：**
- **ClawHub**: OpenClaw 社区的 Skills 注册表
- **Official Optional Skills**: 官方提供的可选 Skills
- **Direct URLs**: 直接从 GitHub 仓库或 well-known endpoint 安装
- **Local Registry**: 本地配置的 Skills 目录

**安装流程：**
1. 搜索：`skills_hub(search="web scraping")`
2. 查看：`skills_hub(fetch="skill-identifier")`
3. 安装：自动下载到 `~/.hermes/skills/`
4. 安全扫描：检测危险代码、shell 注入、网络请求等

### 1.8 外部 Skills 目录

可以在 `config.yaml` 中配置额外的 Skills 扫描目录：

```yaml
skills:
  external_dirs:
    - ~/.agents/skills/
    - /shared/team-skills/
```

**特性：**
- **只读**：外部目录仅用于发现，Agent 创建的 Skills 始终写入 `~/.hermes/skills/`
- **本地优先**：同名 Skill 在本地和外部都存在时，本地版本优先
- **完全集成**：外部 Skills 出现在系统提示、`skills_list`、`skill_view` 和 `/skill-name` 命令中
- **容错**：不存在的目录被静默跳过

---

## 2. Plugins 插件系统

### 2.1 核心理念

Plugins 是 **可扩展的模块化组件**，允许开发者在不修改核心代码的情况下扩展 Hermes 的功能。插件可以注册工具、钩子、CLI 命令等。

**关键原则：**
- **零侵入**：插件不得修改核心文件（`run_agent.py`、`cli.py`、`gateway/run.py` 等）
- **显式启用**：所有插件默认禁用，必须在 `config.yaml` 中明确启用
- **多来源**：支持捆绑插件、用户插件、项目插件、pip 安装的插件

### 2.2 插件来源优先级

插件从四个来源发现，**后面的覆盖前面的**（同名冲突时）：

1. **捆绑插件**：`<repo>/plugins/<name>/`（随 Hermes 发布）
2. **用户插件**：`~/.hermes/plugins/<name>/`
3. **项目插件**：`./.hermes/plugins/<name>/`（需设置 `HERMES_ENABLE_PROJECT_PLUGINS=1`）
4. **Pip 插件**：暴露 `hermes_agent.plugins` entry-point 的包

**注意**：`memory/` 和 `context_engine/` 子目录有独立的发现路径，不在通用扫描范围内。

### 2.3 插件结构

每个目录插件必须包含：

```
plugins/my-plugin/
├── plugin.yaml              # 元数据清单
└── __init__.py              # 入口点，包含 register(ctx) 函数
```

#### plugin.yaml 示例

```yaml
name: my-plugin
version: 1.0.0
description: A sample plugin that adds custom tools
author: Your Name
kind: standalone  # standalone | backend | exclusive
provides_tools:
  - my_custom_tool
provides_hooks:
  - pre_tool_call
  - post_llm_call
requires_env:
  - MY_API_KEY
```

**插件类型（kind）：**

| 类型 | 说明 | 启用方式 |
|------|------|----------|
| `standalone` | 独立的工具/钩子提供者 | `plugins.enabled` 配置 |
| `backend` | 现有核心工具的后备实现（如 image_gen） | 内置后端自动加载；用户安装的后端需 `plugins.enabled` |
| `exclusive` | 独占类别，只有一个活跃提供者（如 memory） | `<category>.provider` 配置键 |

#### __init__.py 示例

```python
"""My Plugin - provides custom tools and hooks."""

def register(ctx):
    """Register tools and hooks with the plugin context."""
    
    # 注册自定义工具
    ctx.register_tool(
        name="my_custom_tool",
        toolset="my_plugin",
        schema={
            "name": "my_custom_tool",
            "description": "Does something useful",
            "parameters": {
                "type": "object",
                "properties": {
                    "param1": {"type": "string"}
                }
            }
        },
        handler=lambda args, **kw: do_something(args.get("param1")),
        check_fn=lambda: bool(os.getenv("MY_API_KEY")),
        requires_env=["MY_API_KEY"],
    )
    
    # 注册生命周期钩子
    def on_pre_tool_call(tool_name, args, task_id, **kwargs):
        print(f"Tool {tool_name} is about to run")
        # 可以返回 {"action": "block", "message": "..."} 来阻止工具执行
    
    ctx.register_hook("pre_tool_call", on_pre_tool_call)
    
    # 注册 CLI 子命令
    def handle_my_command(args):
        print("Custom command executed")
    
    ctx.register_cli_command(
        name="mycommand",
        description="A custom CLI command",
        handler=handle_my_command
    )
```

### 2.4 插件发现与加载流程

```python
# 1. 扫描目录
for source in [bundled, user, project, pip]:
    for plugin_dir in source:
        if plugin_dir/plugin.yaml exists:
            parse manifest
            
# 2. 检查启用状态
if plugin.key not in plugins.enabled:
    skip  # 默认禁用
    
if plugin.key in plugins.disabled:
    skip  # 显式禁用列表优先
    
# 3. 加载插件模块
importlib.import_module(f"hermes_plugins.{plugin.name}")

# 4. 调用 register(ctx)
plugin_module.register(PluginContext(manifest, manager))

# 5. 记录注册的工具和钩子
manager._plugin_tool_names.add(tool_name)
manager._hooks[hook_name].append(callback)
```

### 2.5 插件上下文（PluginContext）

`PluginContext` 是传递给每个插件 `register()` 函数的接口对象，提供以下方法：

#### 工具注册
```python
ctx.register_tool(
    name="tool_name",
    toolset="toolset_name",
    schema={...},
    handler=lambda args, **kw: ...,
    check_fn=lambda: True,
    requires_env=["API_KEY"],
    is_async=False,
    description="",
    emoji=""
)
```

#### 钩子注册
```python
ctx.register_hook("pre_tool_call", callback_function)
```

支持的钩子见 [Hooks 系统](#5-hooks-钩子系统)。

#### CLI 命令注册
```python
ctx.register_cli_command(
    name="mycommand",
    description="Command description",
    handler=lambda args: ...,
    category="Tools & Skills",
    aliases=("mc",),
    args_hint="[arg]"
)
```

#### 消息注入
```python
# 向当前对话注入消息（可用于远程监控、消息桥接等）
success = ctx.inject_message("External message content", role="user")
```

#### 内存提供者注册
```python
ctx.register_memory_provider(MyMemoryProvider())
```

#### 上下文引擎注册
```python
ctx.register_context_engine(MyContextEngine())
```

### 2.6 插件配置

在 `~/.hermes/config.yaml` 中管理插件：

```yaml
plugins:
  enabled:
    - disk-cleanup
    - my-custom-plugin
  disabled:       # 可选的拒绝列表 — 优先级高于 enabled
    - noisy-plugin
```

**管理命令：**
```bash
hermes plugins                    # 交互式切换（空格键勾选/取消）
hermes plugins enable <name>      # 添加到允许列表
hermes plugins disable <name>     # 从允许列表移除 + 添加到禁用列表
hermes plugins list               # 列出所有发现的插件
```

### 2.7 插件命名空间

目录插件在 `hermes_plugins.<name>` 命名空间下可导入：

```python
import hermes_plugins.my_plugin
```

这使得插件之间可以相互引用，也便于调试。

### 2.8 插件生命周期

```
启动阶段：
  1. discover_and_load() - 扫描并加载启用的插件
  2. 调用每个插件的 register(ctx)
  3. 注册工具到全局 registry
  4. 注册钩子到 HookRegistry
  5. 注册 CLI 命令到 argparse

运行阶段：
  - 钩子在特定点被 invoke_hook() 触发
  - 工具通过 registry.dispatch() 调用
  
关闭阶段：
  - 无特殊清理（Python GC 处理）
```

---

## 3. Cron 定时任务系统

### 3.1 核心理念

Cron 系统允许用户调度 **周期性或一次性的 AI 任务**，由后台调度器自动执行。任务可以是自然语言提示、脚本执行或附加 Skills 的组合。

**关键特性：**
- **灵活的调度**：支持 cron 表达式、间隔时间、一次性时间戳
- **多平台交付**：任务结果可以发送到 Telegram、Discord、Slack 等平台
- **隔离执行**：每个任务在独立的 AIAgent 实例中运行
- **输出持久化**：任务输出保存到 `~/.hermes/cron/output/{job_id}/{timestamp}.md`

### 3.2 任务存储

```
~/.hermes/cron/
├── jobs.json                  # 任务定义
├── output/                    # 任务输出
│   ├── job-id-1/
│   │   ├── 2026-04-27T10-00-00.md
│   │   └── 2026-04-27T11-00-00.md
│   └── job-id-2/
│       └── 2026-04-27T09-30-00.md
└── .tick.lock                 # 防止并发执行的锁文件
```

### 3.3 任务数据结构

```python
job = {
    "id": "uuid-4-random-chars",
    "name": "Daily News Summary",
    "prompt": "Summarize today's tech news",
    "skills": ["web-research", "summarization"],  # 可选：附加 Skills
    "skill": "web-research",  # 向后兼容的单技能字段
    "model": "gpt-4",         # 可选：覆盖默认模型
    "provider": "openrouter", # 可选：覆盖默认提供者
    "base_url": "...",        # 可选：自定义 API 端点
    "script": None,           # 可选：执行脚本而非 LLM
    "context_from": ["job-id-2"],  # 可选：从其他任务获取上下文
    "schedule": {
        "kind": "cron",       # once | interval | cron
        "expr": "0 9 * * *",  # cron 表达式
        "display": "0 9 * * *"
    },
    "repeat": {
        "times": None,        # None = 永远重复；整数 = 执行次数
        "completed": 0
    },
    "enabled": True,
    "state": "scheduled",     # scheduled | running | paused | completed
    "paused_at": None,
    "paused_reason": None,
    "created_at": "2026-04-27T08:00:00",
    "next_run_at": "2026-04-28T09:00:00",
    "last_run_at": "2026-04-27T09:00:00",
    "last_status": "ok",      # ok | error
    "last_error": None,
    "last_delivery_error": None,
    # 交付配置
    "deliver": {
        "platform": "telegram",  # telegram | discord | slack | ... | origin
        "chat_id": "@mychannel"
    },
    "origin": {               # 追踪任务创建来源
        "platform": "telegram",
        "chat_id": "123456"
    },
    "enabled_toolsets": ["terminal", "file", "web"],  # 任务可用的工具集
    "workdir": "/path/to/workspace"  # 可选：工作目录
}
```

### 3.4 调度解析

支持三种调度类型：

#### 1. 一次性（once）
```python
# 30 分钟后执行
parse_schedule("30m")
→ {"kind": "once", "run_at": "2026-04-27T10:30:00", "display": "once in 30m"}

# 指定时间戳执行
parse_schedule("2026-04-27T14:00")
→ {"kind": "once", "run_at": "2026-04-27T14:00:00", "display": "once at 2026-04-27 14:00"}
```

#### 2. 间隔（interval）
```python
# 每 30 分钟执行
parse_schedule("every 30m")
→ {"kind": "interval", "minutes": 30, "display": "every 30m"}

# 每 2 小时执行
parse_schedule("every 2h")
→ {"kind": "interval", "minutes": 120, "display": "every 120m"}
```

#### 3. Cron 表达式
```python
# 每天早上 9 点执行
parse_schedule("0 9 * * *")
→ {"kind": "cron", "expr": "0 9 * * *", "display": "0 9 * * *"}

# 每小时执行
parse_schedule("0 * * * *")
→ {"kind": "cron", "expr": "0 * * * *", "display": "0 * * * *"}
```

**注意**：Cron 表达式需要安装 `croniter` 包：`pip install croniter`

### 3.5 调度器工作原理

```python
# gateway/run.py 中的后台线程
def cron_tick_loop():
    while True:
        time.sleep(60)  # 每分钟检查一次
        try:
            from cron.scheduler import tick
            tick()  # 检查并执行到期任务
        except Exception as e:
            logger.error(f"Cron tick failed: {e}")
```

#### tick() 流程

```python
def tick(verbose=False):
    # 1. 获取文件锁（防止多进程并发执行）
    acquire_lock("~/.hermes/cron/.tick.lock")
    
    # 2. 加载所有任务
    jobs = load_jobs()
    
    # 3. 筛选到期任务
    now = datetime.now()
    due_jobs = [
        j for j in jobs
        if j["enabled"] 
        and j["state"] != "paused"
        and j["next_run_at"] <= now.isoformat()
    ]
    
    # 4. 并行执行到期任务
    with ThreadPoolExecutor(max_workers=max_parallel) as executor:
        futures = [executor.submit(process_job, job) for job in due_jobs]
        for future in as_completed(futures):
            future.result()
    
    # 5. 释放锁
    release_lock()
```

#### process_job() 流程

```python
def process_job(job):
    # 1. 执行任务
    success, output, final_response, error = run_job(job)
    
    # 2. 保存输出
    output_file = save_job_output(job["id"], output)
    
    # 3. 交付结果（除非 Agent 返回 [SILENT]）
    if final_response and "[SILENT]" not in final_response:
        delivery_error = deliver_result(job, final_response)
    
    # 4. 标记执行状态
    mark_job_run(job["id"], success, error, delivery_error)
    
    # 5. 计算下次执行时间
    advance_next_run(job)
```

### 3.6 任务执行（run_job）

```python
def run_job(job):
    # 1. 确定执行模式
    if job.get("script"):
        # 脚本模式：直接执行 shell 命令
        result = subprocess.run(job["script"], capture_output=True, text=True)
        return result.returncode == 0, result.stdout, result.stdout, None
    else:
        # LLM 模式：创建 AIAgent 实例
        agent = AIAgent(
            model=job.get("model") or config["model"],
            provider=job.get("provider"),
            base_url=job.get("base_url"),
            enabled_toolsets=resolve_cron_enabled_toolsets(job, config),
            max_iterations=config.get("max_iterations", 90),
            quiet_mode=True,
            platform="cron",  # 标记为 cron 平台
            session_id=f"cron-{job['id']}",
        )
        
        # 2. 构建提示词
        system_message = build_cron_system_message(job)
        user_message = job["prompt"]
        
        # 3. 附加 Skills（如果有）
        if job.get("skills"):
            user_message += f"\n\nUse these skills: {', '.join(job['skills'])}"
        
        # 4. 注入上下文（如果有）
        if job.get("context_from"):
            context = gather_context_from_jobs(job["context_from"])
            user_message += f"\n\nContext from previous jobs:\n{context}"
        
        # 5. 执行对话
        result = agent.run_conversation(user_message, system_message)
        
        return True, "", result["final_response"], None
```

### 3.7 交付系统

任务结果可以交付到多个平台：

```python
def deliver_result(job, content, adapters=None, loop=None):
    deliver_config = job.get("deliver", {})
    platform = deliver_config.get("platform", "origin")
    
    if platform == "origin":
        # 发送到任务创建时的来源聊天
        origin = job.get("origin", {})
        platform = origin.get("platform")
        chat_id = origin.get("chat_id")
    
    # 获取对应平台的适配器
    adapter = get_adapter(platform)
    
    # 发送消息
    adapter.send_message(chat_id, content)
```

**支持的平台：**
- Telegram、Discord、Slack、WhatsApp、Signal
- Matrix、Mattermost、Home Assistant
- DingTalk（钉钉）、Feishu（飞书）、WeCom（企业微信）
- Email、SMS、Webhook

### 3.8 任务管理 API

```python
# 创建任务
create_job(
    schedule="every 30m",
    prompt="Check system health",
    name="Health Check",
    deliver={"platform": "telegram", "chat_id": "@alerts"}
)

# 列出任务
list_jobs(include_disabled=False)

# 获取任务详情
get_job("job-id")

# 更新任务
update_job("job-id", {"enabled": False})

# 暂停任务
pause_job("job-id", reason="Maintenance")

# 恢复任务
resume_job("job-id")

# 手动执行任务
run_job_now("job-id")

# 删除任务
delete_job("job-id")
```

### 3.9 配置选项

在 `~/.hermes/config.yaml` 中：

```yaml
cron:
  max_parallel_jobs: 3  # 最大并行执行的任务数（默认无限制）
  
# 任务级别的工具集配置（通过 hermes tools 命令设置）
tools:
  cron:
    enabled:
      - terminal
      - file
      - web
```

---

## 4. Subagent 子代理委托系统

### 4.1 核心理念

`delegate_task` 工具允许父 Agent ** spawn 隔离的子 Agent**，在独立的上下文中并行执行任务。子 Agent 有自己的对话历史、终端会话和工具集，只有最终摘要返回给父 Agent。

**关键特性：**
- **上下文隔离**：子 Agent 对父 Agent 的对话历史一无所知
- **并行执行**：默认最多 3 个并发子 Agent（可配置）
- **工具集限制**：子 Agent 不能访问某些工具（如 delegation、clarify、memory）
- **中断传播**：中断父 Agent 会同时中断所有活跃的子 Agent

### 4.2 使用场景

#### 单任务委托
```python
delegate_task(
    goal="Debug why tests fail",
    context="Error: assertion in test_foo.py line 42",
    toolsets=["terminal", "file"]
)
```

#### 批量并行委托
```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Research topic B", "toolsets": ["web"]},
    {"goal": "Fix the build", "toolsets": ["terminal", "file"]}
])
```

### 4.3 子 Agent 上下文工作原理

:::warning 关键：子 Agent 一无所知
子 Agent 以**完全空白的对话**开始。它们对父 Agent 的对话历史、之前的工具调用或之前讨论的内容**零知识**。子 Agent 的唯一上下文来自父 Agent 在调用 `delegate_task` 时填充的 `goal` 和 `context` 字段。
:::

**这意味着父 Agent 必须在调用中传递子 Agent 需要的一切：**

```python
# ❌ 错误 - 子 Agent 不知道"错误"是什么
delegate_task(goal="Fix the error")

# ✅ 正确 - 子 Agent 拥有它需要的所有上下文
delegate_task(
    goal="Fix the TypeError in api/handlers.py",
    context="""The file api/handlers.py has a TypeError on line 47:
    'NoneType' object has no attribute 'get'.
    The function process_request() receives a dict from parse_body(),
    but parse_body() returns None when Content-Type is missing.
    The project is at /home/user/myproject and uses Python 3.11."""
)
```

### 4.4 子 Agent 构建流程

```python
def _build_child_agent(parent_agent, goal, context, toolsets, ...):
    # 1. 确定子 Agent 深度
    child_depth = parent_agent.depth + 1
    
    # 2. 检查深度限制
    max_spawn_depth = config.get("delegation.max_spawn_depth", 1)
    if child_depth > max_spawn_depth:
        raise ValueError(f"Max spawn depth exceeded ({max_spawn_depth})")
    
    # 3. 确定角色（leaf vs orchestrator）
    role = args.get("role", "leaf")
    if role == "orchestrator" and child_depth >= max_spawn_depth:
        role = "leaf"  # 强制降级为 leaf
    
    # 4. 构建系统提示
    system_prompt = _build_child_system_prompt(
        goal, context, role, child_depth, max_spawn_depth
    )
    
    # 5. 过滤工具集
    allowed_toolsets = _strip_blocked_tools(toolsets or ["terminal", "file", "web"])
    if role == "leaf":
        allowed_toolsets = [t for t in allowed_toolsets if t != "delegation"]
    
    # 6. 继承凭证池
    credential_pool = _resolve_child_credential_pool(
        args.get("provider"), parent_agent
    )
    
    # 7. 创建 AIAgent 实例
    child = AIAgent(
        model=args.get("model") or config.get("delegation.model"),
        provider=args.get("provider") or config.get("delegation.provider"),
        base_url=parent_agent.base_url,
        api_key=parent_agent.api_key,
        max_iterations=args.get("max_iterations", 50),
        enabled_toolsets=allowed_toolsets,
        credential_pool=credential_pool,
        platform=parent_agent.platform,
        session_id=f"{parent_agent.session_id}-child-{uuid4().hex[:8]}",
        quiet_mode=True,
    )
    
    # 8. 注册到活跃子 Agent 列表（用于中断传播）
    _register_subagent({
        "subagent_id": child.session_id,
        "agent": child,
        "goal": goal,
        "depth": child_depth,
    })
    
    return child
```

### 4.5  blocked 工具集

子 Agent **永远无法访问**以下工具：

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 防止递归委托（leaf 角色）
    "clarify",         # 不允许与用户交互
    "memory",          # 不允许写入共享 MEMORY.md
    "send_message",    # 不允许跨平台副作用
    "execute_code",    # 子 Agent 应逐步推理，而不是编写脚本
])
```

**额外限制：**
- `delegation` 工具集对 `leaf` 角色子 Agent 隐藏
- `role="orchestrator"` 子 Agent 保留 `delegate_task`，但仍不能使用其他四个工具

### 4.6 并行执行架构

```python
def delegate_task(goal=None, tasks=None, ...):
    if tasks:
        # 批量模式：并行执行
        max_concurrent = _get_max_concurrent_children()  # 默认 3
        
        if len(tasks) > max_concurrent:
            return error(f"Too many tasks ({len(tasks)}). Max: {max_concurrent}")
        
        # 创建线程池
        approval_cb = _get_subagent_approval_callback()
        with ThreadPoolExecutor(
            max_workers=max_concurrent,
            initializer=_set_subagent_approval_cb,
            initargs=(approval_cb,)
        ) as executor:
            futures = []
            for i, task in enumerate(tasks):
                child = _build_child_agent(...)
                future = executor.submit(_run_single_child, i, task["goal"], child, ...)
                futures.append((i, future))
            
            # 等待所有完成
            results = []
            for i, future in sorted(futures):
                try:
                    result = future.result(timeout=child_timeout)
                    results.append(result)
                except TimeoutError:
                    results.append({"status": "timeout", "error": "Child timed out"})
        
        return format_batch_results(results)
    else:
        # 单任务模式：直接执行
        child = _build_child_agent(...)
        result = _run_single_child(0, goal, child, ...)
        return format_single_result(result)
```

### 4.7 中断传播机制

```python
# 父 Agent 的中断方法
class AIAgent:
    def interrupt(self, message: str):
        self._interrupt_requested = True
        self._interrupt_message = message
        
        # 传播到所有活跃的子 Agent
        with self._active_children_lock:
            for child in self._active_children:
                child.interrupt(message)
```

**工作流程：**
1. 用户在 CLI 中输入新消息
2. CLI 调用 `parent.interrupt("User typed a new message")`
3. `parent._interrupt_requested` 设为 `True`
4. 遍历 `parent._active_children`，调用每个子 Agent 的 `interrupt()`
5. 子 Agent 在下一次迭代边界检查 `_interrupt_requested` 并退出循环

### 4.8 深度限制与嵌套编排

默认情况下，委托是**扁平的**：父 Agent（深度 0）spawn 子 Agent（深度 1），这些子 Agent 不能再委托。

**启用嵌套委托：**

```yaml
# ~/.hermes/config.yaml
delegation:
  max_spawn_depth: 2  # 允许 orchestrator 子 Agent spawn leaf 孙 Agent
  orchestrator_enabled: true  # 全局开关（默认 true）
```

```python
# 父 Agent 调用
delegate_task(
    goal="Survey three code review approaches and recommend one",
    role="orchestrator",  # 允许此子 Agent spawn 自己的 workers
    context="...",
)
```

**深度层级：**
- `max_spawn_depth: 1`（默认）：扁平委托，`role="orchestrator"` 无效
- `max_spawn_depth: 2`：父 → orchestrator → leaf（最多 3×3=9 个并发 leaf）
- `max_spawn_depth: 3`（上限）：父 → orchestrator → orchestrator → leaf（最多 3×3×3=27 个并发 leaf）

**成本警告**：每增加一层，并发 Agent 数量呈指数增长。谨慎提高 `max_spawn_depth`。

### 4.9 子 Agent 进度回调

CLI 模式下，子 Agent 的工具调用实时显示在树形视图中：

```
🤖 Parent Agent: Delegating task...
  ├─ 🌐 web_search  "latest Python features"
  ├─ 📄 read_file   "src/main.py"
  └─ ✅ Task complete
```

**实现：**
```python
def _build_child_progress_callback(task_index, goal, parent_agent, ...):
    def _callback(event_type, tool_name, preview, args, **kwargs):
        if event_type == "subagent.start":
            spinner.print_above(f"  ├─ 🚀 Starting: {goal[:55]}")
        elif event_type == "subagent.tool":
            emoji = get_tool_emoji(tool_name)
            spinner.print_above(f"  ├─ {emoji} {tool_name}  \"{preview[:35]}\"")
        elif event_type == "subagent.complete":
            spinner.print_above(f"  └─ ✅ Task {task_index + 1} complete")
    
    return _callback
```

### 4.10 配置选项

```yaml
# ~/.hermes/config.yaml
delegation:
  model: "google/gemini-flash-2.0"  # 子 Agent 使用的模型（默认同父 Agent）
  provider: "openrouter"             # 可选：路由到不同的提供者
  max_concurrent_children: 3         # 最大并发子 Agent 数
  child_timeout_seconds: 600         # 单个子 Agent 超时（秒）
  max_spawn_depth: 1                 # 最大委托深度
  orchestrator_enabled: true         # 全局启用 orchestrator 角色
  subagent_auto_approve: false       # 自动批准危险命令（谨慎启用）
```

---

## 5. Hooks 钩子系统

### 5.1 核心理念

Hooks 是 **事件驱动的回调机制**，允许在 Hermes 生命周期的关键点插入自定义逻辑。有两种类型的钩子：

1. **Plugin Hooks**：通过插件系统注册的 Python 回调
2. **Gateway Hooks**：从 `~/.hermes/hooks/` 目录动态加载的独立脚本

### 5.2 Plugin Hooks

#### 支持的钩子事件

| 钩子 | 触发时机 | 返回值 |
|------|---------|--------|
| `pre_tool_call` | 任何工具执行前 | `{"action": "block", "message": str}` 阻止调用 |
| `post_tool_call` | 任何工具返回后 | 忽略 |
| `transform_terminal_output` | 终端输出返回后 | 转换后的输出字符串 |
| `transform_tool_result` | 工具结果返回后 | 转换后的结果 JSON |
| `pre_llm_call` | 每轮工具调用循环前 | `{"context": str}` 注入到用户消息 |
| `post_llm_call` | 每轮工具调用循环后 | 忽略 |
| `pre_api_request` | API 请求发送前 | 忽略 |
| `post_api_request` | API 响应接收后 | 忽略 |
| `on_session_start` | 新会话创建（仅第一轮） | 忽略 |
| `on_session_end` | 会话结束 | 忽略 |
| `on_session_finalize` | CLI/Gateway 清理会话时 | 忽略 |
| `on_session_reset` | Gateway 交换新会话密钥时 | 忽略 |
| `subagent_stop` | `delegate_task` 子 Agent 退出时 | 忽略 |
| `pre_gateway_dispatch` | Gateway 收到用户消息后，分发前 | `{"action": "skip"/"rewrite"/"allow", ...}` |

#### 注册钩子

```python
def register(ctx):
    # 注册 pre_tool_call 钩子
    def block_dangerous_commands(tool_name, args, task_id, **kwargs):
        if tool_name == "terminal":
            cmd = args.get("command", "")
            if any(dangerous in cmd for dangerous in ["rm -rf", "dd if=", "> /dev/sda"]):
                return {"action": "block", "message": "Dangerous command blocked"}
    
    ctx.register_hook("pre_tool_call", block_dangerous_commands)
    
    # 注册 pre_llm_call 钩子（唯一返回值被使用的钩子）
    def inject_memory(session_id, user_message, is_first_turn, **kwargs):
        memories = recall_memories(session_id, user_message)
        if memories:
            return {"context": f"Recalled memories:\n" + "\n".join(memories)}
        return None
    
    ctx.register_hook("pre_llm_call", inject_memory)
```

#### 钩子执行流程

```python
# model_tools.py 中的 handle_function_call()
def handle_function_call(function_name, function_args, ...):
    # 1. 触发 pre_tool_call 钩子
    block_message = get_pre_tool_call_block_message(
        function_name, function_args, task_id, ...
    )
    if block_message:
        return json.dumps({"error": block_message})
    
    # 2. 执行工具
    start_time = time.monotonic()
    result = registry.dispatch(function_name, function_args, ...)
    duration_ms = int((time.monotonic() - start_time) * 1000)
    
    # 3. 触发 post_tool_call 钩子
    invoke_hook(
        "post_tool_call",
        tool_name=function_name,
        args=function_args,
        result=result,
        duration_ms=duration_ms,
        ...
    )
    
    return result
```

#### pre_llm_call 钩子的特殊行为

`pre_llm_call` 是唯一返回值被使用的钩子。它可以注入上下文到当前轮次的用户消息中：

```python
def my_hook(session_id, user_message, conversation_history, **kwargs):
    # 返回字典格式
    return {"context": "Additional context to prepend"}
    
    # 或纯字符串（等价）
    return "Additional context to prepend"
    
    # 或不注入
    return None
```

**注入位置**：始终是**用户消息**，而不是系统提示。这保留了提示缓存——系统提示在各轮次间保持相同，因此缓存的 Token 被重用。

**多插件合并**：当多个插件返回上下文时，它们的输出按插件发现顺序（按目录名字母顺序）用双换行符连接。

### 5.3 Gateway Hooks

Gateway Hooks 是从 `~/.hermes/hooks/` 目录动态加载的独立脚本。

#### 目录结构

```
~/.hermes/hooks/
├── boot-md/
│   ├── HOOK.yaml
│   └── handler.py
└── my-custom-hook/
    ├── HOOK.yaml
    └── handler.py
```

#### HOOK.yaml

```yaml
name: boot-md
description: Run ~/.hermes/BOOT.md on gateway startup
events:
  - gateway:startup
```

#### handler.py

```python
"""Boot MD Hook - runs BOOT.md on startup."""

async def handle(event_type, context):
    """Handle gateway:startup event."""
    from pathlib import Path
    from hermes_constants import get_hermes_home
    
    boot_md = get_hermes_home() / "BOOT.md"
    if boot_md.exists():
        content = boot_md.read_text()
        # Do something with the content
        print(f"Loaded BOOT.md: {len(content)} chars")
```

#### 支持的事件

| 事件 | 触发时机 |
|------|---------|
| `gateway:startup` | Gateway 进程启动 |
| `session:start` | 新会话创建（会话的第一条消息） |
| `session:end` | 会话结束（用户运行 /new 或 /reset） |
| `session:reset` | 会话重置完成（创建新会话条目） |
| `agent:start` | Agent 开始处理消息 |
| `agent:step` | 工具调用循环中的每一轮 |
| `agent:end` | Agent 完成处理 |
| `command:*` | 任何斜杠命令执行（通配符匹配） |

#### 通配符匹配

Handlers 可以使用通配符注册：

```yaml
events:
  - "command:*"  # 匹配所有 command:xxx 事件
```

```python
# handler.py
async def handle(event_type, context):
    # event_type 可能是 "command:reset", "command:new", 等
    command_name = event_type.split(":")[1]
    print(f"Command executed: {command_name}")
```

#### 内置 Hooks

Hermes 自带一些内置 Hooks：

- **boot-md**：在 Gateway 启动时运行 `~/.hermes/BOOT.md`

### 5.4 Shell Hooks（外部脚本钩子）

Shell Hooks 允许在执行特定命令时触发外部脚本。

#### 配置

在 `~/.hermes/config.yaml` 中：

```yaml
hooks:
  pre_tool_call:
    - command: ~/.hermes/scripts/log-tool.sh
      matcher: "terminal"  # 可选：只匹配特定工具
      timeout: 5           # 超时（秒）
```

#### 同意机制

首次运行时，Hermes 会提示用户同意执行外部脚本：

```
⚠️  Shell hook wants to run: ~/.hermes/scripts/log-tool.sh
    Event: pre_tool_call
    Matcher: terminal

Do you allow this? [y/N] y
✓ Allowed. This consent is recorded in ~/.hermes/shell-hooks-allowlist.json
```

同意记录保存在 `~/.hermes/shell-hooks-allowlist.json` 中，包括脚本的 mtime 哈希，以便检测脚本修改。

#### 管理命令

```bash
hermes hooks list          # 列出所有配置的钩子
hermes hooks test <event>  # 测试钩子
hermes hooks revoke <cmd>  # 撤销同意
hermes hooks doctor        # 诊断钩子健康状态
```

### 5.5 钩子用例

#### 1. 内存召回

```python
def recall(session_id, user_message, **kwargs):
    try:
        resp = httpx.post(f"{MEMORY_API}/recall", json={
            "session_id": session_id,
            "query": user_message,
        }, timeout=3)
        memories = resp.json().get("results", [])
        if not memories:
            return None
        text = "Recalled context:\n" + "\n".join(f"- {m['text']}" for m in memories)
        return {"context": text}
    except Exception:
        return None

def register(ctx):
    ctx.register_hook("pre_llm_call", recall)
```

#### 2. 安全防护

```python
POLICY = "Never execute commands that delete files without explicit user confirmation."

def guardrails(**kwargs):
    return {"context": POLICY}

def register(ctx):
    ctx.register_hook("pre_llm_call", guardrails)
```

#### 3. 工具使用指标跟踪

```python
from collections import Counter, defaultdict
import json

_tool_counts = Counter()
_error_counts = Counter()
_latency_ms = defaultdict(list)

def track_metrics(tool_name, result, duration_ms=0, **kwargs):
    _tool_counts[tool_name] += 1
    _latency_ms[tool_name].append(duration_ms)
    try:
        parsed = json.loads(result)
        if "error" in parsed:
            _error_counts[tool_name] += 1
    except (json.JSONDecodeError, TypeError):
        pass

def register(ctx):
    ctx.register_hook("post_tool_call", track_metrics)
```

#### 4. 阻止危险命令

```python
DANGEROUS_PATTERNS = ["rm -rf /", "dd if=", "> /dev/sda"]

def block_dangerous(tool_name, args, **kwargs):
    if tool_name == "terminal":
        cmd = args.get("command", "")
        if any(pattern in cmd for pattern in DANGEROUS_PATTERNS):
            return {"action": "block", "message": "Dangerous command blocked by security policy"}

def register(ctx):
    ctx.register_hook("pre_tool_call", block_dangerous)
```

---

## 6. Tools 工具注册与发现

### 6.1 核心理念

Tools 是 Hermes 的**执行能力**，每个工具是一个 Python 函数，通过中央注册表统一管理。工具采用**自注册模式**，无需维护手动导入列表。

**关键特性：**
- **自动发现**：`tools/*.py` 文件中任何调用 `registry.register()` 的模块都会被自动导入
- **集中注册**：所有工具的 schema、handler、元数据存储在 `ToolRegistry` 中
- **工具集分组**：工具按功能分组为 toolsets（如 `terminal`、`file`、`web`）
- **可用性检查**：每个工具可以声明环境变量依赖和检查函数

### 6.2 工具文件结构

每个工具文件遵循统一的模式：

```python
#!/usr/bin/env python3
"""Tool Description"""

import json
import os
from tools.registry import registry

def check_requirements() -> bool:
    """Check if the tool's dependencies are available."""
    return bool(os.getenv("REQUIRED_API_KEY"))

def my_tool(param1: str, param2: int = 10, task_id: str = None) -> str:
    """
    Tool implementation.
    
    Args:
        param1: Description
        param2: Description
        task_id: Session/task identifier (injected by the framework)
    
    Returns:
        JSON string with the result
    """
    try:
        # Do something
        result = {"success": True, "data": "..."}
        return json.dumps(result, ensure_ascii=False)
    except Exception as e:
        return json.dumps({"error": str(e)}, ensure_ascii=False)

# Schema definition
MY_TOOL_SCHEMA = {
    "name": "my_tool",
    "description": "What this tool does",
    "parameters": {
        "type": "object",
        "properties": {
            "param1": {
                "type": "string",
                "description": "Description of param1"
            },
            "param2": {
                "type": "integer",
                "description": "Description of param2",
                "default": 10
            }
        },
        "required": ["param1"]
    }
}

# Register the tool
registry.register(
    name="my_tool",
    toolset="my_category",
    schema=MY_TOOL_SCHEMA,
    handler=lambda args, **kw: my_tool(
        param1=args.get("param1"),
        param2=args.get("param2", 10),
        task_id=kw.get("task_id")
    ),
    check_fn=check_requirements,
    requires_env=["REQUIRED_API_KEY"],
    emoji="🔧",
)
```

### 6.3 注册表架构

```python
# tools/registry.py

class ToolEntry:
    """单个注册工具的元数据"""
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars",
    )

class ToolRegistry:
    """单例注册表，收集工具的 schema + handlers"""
    
    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}
        self._toolset_checks: Dict[str, Callable] = {}
        self._lock = threading.RLock()
    
    def register(self, name, toolset, schema, handler, check_fn=None, ...):
        """注册工具。由每个工具文件在模块导入时调用。"""
        with self._lock:
            self._tools[name] = ToolEntry(...)
    
    def dispatch(self, name, args, **kwargs):
        """分派工具调用。由 model_tools.py 调用。"""
        entry = self._tools.get(name)
        if not entry:
            return json.dumps({"error": f"Unknown tool: {name}"})
        
        # 异步桥接
        if entry.is_async:
            return asyncio.run(async_dispatch(entry.handler, args, **kwargs))
        
        # 同步调用
        return entry.handler(args, **kwargs)
    
    def get_definitions(self, enabled_toolsets, disabled_toolsets):
        """获取 OpenAI 格式的工具定义列表。"""
        entries = self._snapshot_entries()
        definitions = []
        for entry in entries:
            if entry.toolset in enabled_toolsets:
                definitions.append(entry.schema)
        return definitions
```

### 6.4 自动发现机制

```python
# tools/registry.py

def _module_registers_tools(module_path: Path) -> bool:
    """检查模块是否包含顶层 registry.register() 调用。"""
    try:
        source = module_path.read_text(encoding="utf-8")
        tree = ast.parse(source)
    except (OSError, SyntaxError):
        return False
    
    # 只检查模块体语句，避免误判辅助模块
    return any(_is_registry_register_call(stmt) for stmt in tree.body)

def discover_builtin_tools(tools_dir=None) -> List[str]:
    """导入内置的自注册工具模块。"""
    tools_path = Path(tools_dir) if tools_dir else Path(__file__).parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)
    ]
    
    imported = []
    for mod_name in module_names:
        try:
            importlib.import_module(mod_name)
            imported.append(mod_name)
        except Exception as e:
            logger.warning(f"Could not import tool module {mod_name}: {e}")
    return imported
```

**工作流程：**
1. `model_tools.py` 在模块级别调用 `discover_builtin_tools()`
2. 扫描 `tools/*.py`，使用 AST 解析检查是否有 `registry.register()` 调用
3. 导入符合条件的模块
4. 模块导入时执行 `registry.register()`，将工具注册到全局注册表

### 6.5 工具集（Toolsets）

工具按功能分组为 toolsets：

```python
# toolsets.py

TOOLSETS = {
    "terminal": {
        "label": "💻 Terminal",
        "description": "Execute shell commands",
        "tools": ["terminal"],
    },
    "file": {
        "label": "📁 File Operations",
        "description": "Read, write, search files",
        "tools": ["read_file", "write_file", "patch", "search_files"],
    },
    "web": {
        "label": "🌐 Web Search",
        "description": "Search and extract web content",
        "tools": ["web_search", "web_extract"],
    },
    "browser": {
        "label": "🌍 Browser Automation",
        "description": "Control a headless browser",
        "tools": [
            "browser_navigate", "browser_snapshot", "browser_click",
            "browser_type", "browser_scroll", "browser_back",
        ],
    },
    # ... 更多 toolsets
}
```

**工具集解析：**
```python
def resolve_toolset(toolset_name: str) -> List[str]:
    """解析工具集名称为工具列表。"""
    if toolset_name in TOOLSETS:
        return TOOLSETS[toolset_name]["tools"]
    
    # 检查别名
    alias_target = registry.get_toolset_alias_target(toolset_name)
    if alias_target and alias_target in TOOLSETS:
        return TOOLSETS[alias_target]["tools"]
    
    return []
```

### 6.6 工具调用流程

```python
# model_tools.py

def handle_function_call(function_name, function_args, task_id, ...):
    """处理工具调用。由 run_agent.py 调用。"""
    
    # 1. 触发 pre_tool_call 钩子
    block_message = get_pre_tool_call_block_message(...)
    if block_message:
        return json.dumps({"error": block_message})
    
    # 2. 测量工具执行延迟
    start_time = time.monotonic()
    
    # 3. 分派到注册表
    result = registry.dispatch(function_name, function_args, task_id=task_id, ...)
    
    duration_ms = int((time.monotonic() - start_time) * 1000)
    
    # 4. 触发 post_tool_call 钩子
    invoke_hook(
        "post_tool_call",
        tool_name=function_name,
        args=function_args,
        result=result,
        duration_ms=duration_ms,
        ...
    )
    
    return result
```

### 6.7 MCP 工具集成

MCP（Model Context Protocol）工具是动态发现的：

```python
# tools/mcp_tool.py

def discover_mcp_tools():
    """从配置中发现 MCP 服务器并注册其工具。"""
    config = _load_mcp_config()  # 从 config.yaml 读取 mcp_servers
    
    for server_name, server_config in config.items():
        asyncio.run(_discover_and_register_server(server_name, server_config))

async def _discover_and_register_server(name, config):
    """连接 MCP 服务器并注册其工具。"""
    server = await _connect_server(name, config)
    tools = await server.list_tools()
    
    for tool in tools:
        tool_name = f"mcp_{name}_{tool.name}"
        
        registry.register(
            name=tool_name,
            toolset=f"mcp-{name}",
            schema=tool.schema,
            handler=lambda args, t=tool: t.execute(args),
        )
```

**配置示例：**
```yaml
# ~/.hermes/config.yaml
mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
  github:
    url: https://api.githubcopilot.com/mcp
    headers:
      Authorization: "Bearer ${GITHUB_TOKEN}"
```

### 6.8 插件工具

插件可以通过 `PluginContext.register_tool()` 注册工具：

```python
def register(ctx):
    ctx.register_tool(
        name="my_plugin_tool",
        toolset="my_plugin",
        schema={...},
        handler=lambda args, **kw: do_something(),
    )
```

插件工具与内置工具无缝集成，出现在相同的注册表中。

### 6.9 工具可用性检查

每个工具可以声明环境变量依赖：

```python
registry.register(
    name="web_search",
    toolset="web",
    schema=WEB_SEARCH_SCHEMA,
    handler=...,
    check_fn=lambda: bool(os.getenv("FIRECRAWL_API_KEY")),
    requires_env=["FIRECRAWL_API_KEY"],
)
```

**检查流程：**
```python
def check_tool_availability(quiet=False):
    """检查所有工具的可用性。"""
    available = []
    unavailable = []
    
    for entry in registry._snapshot_entries():
        if entry.check_fn:
            try:
                if entry.check_fn():
                    available.append(entry.name)
                else:
                    unavailable.append(entry.name)
            except Exception:
                unavailable.append(entry.name)
        else:
            available.append(entry.name)
    
    return available, unavailable
```

---

## 7. 自进化机制

### 7.1 核心理念

Hermes 的自进化能力体现在 **Agent 可以自主创建和管理 Skills**，将成功的经验转化为可重用的程序性知识。这是 Hermes 的"长期记忆"形式。

**关键概念：**
- **程序性记忆 vs 陈述性记忆**：
  - 陈述性记忆（MEMORY.md、USER.md）：广泛、声明性的事实（"用户喜欢 Python"）
  - 程序性记忆（Skills）：狭窄、可操作的"如何做"知识（"如何使用 Axolotl 微调 Llama 3"）
- **Agent 驱动**：Agent 在完成任务后自主决定何时创建 Skill
- **持续改进**：Skills 可以被 Agent 自己修改和优化

### 7.2 自进化工作流

```
1. Agent 执行复杂任务
   ↓
2. 任务成功完成（或经过多次尝试后成功）
   ↓
3. Agent 反思："这个方法值得记住"
   ↓
4. Agent 调用 skill_manage(action="create", ...)
   ↓
5. 新 Skill 保存到 ~/.hermes/skills/
   ↓
6. 下次遇到类似任务时，Skill 被自动加载
   ↓
7. Agent 可以 patch/edit Skill 以改进它
```

### 7.3 Agent 何时创建 Skills

根据 `skills/SKILL_CREATION_GUIDELINES.md`，Agent 应在以下情况创建 Skills：

#### ✅ 应该创建
- 成功完成复杂任务（5+ 次工具调用）
- 遇到错误但最终找到解决方案
- 用户纠正了 Agent 的方法
- 发现了非平凡的工作流程
- 学习了新的 API 或工具用法
- 优化了某个流程的性能

#### ❌ 不应该创建
- 简单的单步操作
- 临时性的、不太可能重用的知识
- 已经存在于现有 Skills 中的内容
- 过于特定于单次任务的细节

### 7.4 Skill 创建示例

假设 Agent 刚刚学会了如何使用 Firecrawl 进行网页抓取：

```python
# Agent 调用
skill_manage(
    action="create",
    name="web-scraping-firecrawl",
    content="""---
name: web-scraping-firecrawl
description: Scrape websites using Firecrawl API with proper rate limiting and error handling
version: 1.0.0
metadata:
  hermes:
    tags: [web, scraping, firecrawl]
    category: research
    requires_toolsets: [web]
---

# Web Scraping with Firecrawl

## When to Use
- Need to extract structured data from websites
- JavaScript-rendered pages that need browser execution
- Large-scale scraping with rate limiting

## Prerequisites
- FIRECRAWL_API_KEY set in ~/.hermes/.env
- web toolset enabled

## Procedure

### 1. Basic Extraction
\`\`\`python
web_extract(urls=["https://example.com"])
\`\`\`

### 2. Structured Data Extraction
\`\`\`python
web_extract(
    urls=["https://example.com/products"],
    formats=["markdown"],
    only_main_content=True
)
\`\`\`

### 3. Handling Rate Limits
- Firecrawl allows 100 requests/minute on free tier
- Batch URLs in groups of 10
- Add 1-second delays between batches

## Common Pitfalls
- Don't scrape robots.txt-disallowed paths
- Handle pagination manually (Firecrawl doesn't auto-follow "next" links)
- Some sites block Firecrawl's user agent

## Verification
- Check returned markdown for expected content
- Verify all requested URLs were processed
- Ensure no sensitive data was extracted
""",
    category="research"
)
```

### 7.5 Skill 改进示例

Agent 发现 Firecrawl 的新功能后，可以 patch 现有 Skill：

```python
skill_manage(
    action="patch",
    name="web-scraping-firecrawl",
    old_string="### 2. Structured Data Extraction\n```python\nweb_extract(\n    urls=[\"https://example.com/products\"],\n    formats=[\"markdown\"],\n    only_main_content=True\n)\n```",
    new_string="""### 2. Structured Data Extraction
```python
web_extract(
    urls=["https://example.com/products"],
    formats=["markdown"],
    only_main_content=True
)
```

### 2b. Extract to JSON Schema (New!)
```python
web_extract(
    urls=["https://example.com/products"],
    formats=["json"],
    json_schema={
        "type": "object",
        "properties": {
            "products": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "price": {"type": "number"}
                    }
                }
            }
        }
    }
)
```"""
)
```

### 7.6 安全扫描

为了防止 Agent 创建恶意的 Skills，有一个可选的安全扫描机制：

```python
# tools/skill_manager_tool.py

def _security_scan_skill(skill_dir: Path) -> Optional[str]:
    """扫描 Skill 目录。如果被阻止则返回错误字符串，否则返回 None。"""
    if not _GUARD_AVAILABLE:
        return None
    if not _guard_agent_created_enabled():  # 默认关闭
        return None
    
    try:
        result = scan_skill(skill_dir, source="agent-created")
        allowed, reason = should_allow_install(result)
        
        if allowed is False:
            report = format_scan_report(result)
            return f"Security scan blocked this skill ({reason}):\n{report}"
        
        if allowed is None:  # "ask" verdict
            report = format_scan_report(result)
            logger.warning("Agent-created skill blocked (dangerous findings): %s", reason)
            return f"Security scan blocked this skill ({reason}):\n{report}"
    except Exception as e:
        logger.warning(f"Security scan failed for {skill_dir}: {e}")
    
    return None
```

**扫描内容：**
- Shell 注入模式
- 危险的网络请求
- 文件系统操作风险
- 已知的恶意代码模式

**注意**：默认情况下，Agent 创建的 Skills **不经过安全扫描**，因为 Agent 已经可以通过 `terminal()` 执行相同的代码路径。用户可以通过 `hermes config set skills.guard_agent_created true` 启用扫描。

### 7.7 自进化的优势

1. **个性化**：每个用户的 Hermes 实例根据其使用模式演化出独特的 Skills 集合
2. **持续改进**：Skills 随着时间推移变得越来越好
3. **减少 Token 消耗**：复用的 Skills 避免了每次都重新学习
4. **知识积累**：Agent 的经验被永久保存，不会因会话结束而丢失
5. **社区共享**：用户可以通过 Skills Hub 分享他们的 Skills

### 7.8 未来方向

潜在的自进化增强：

1. **自动 Skill 提炼**：Agent 主动识别可提炼为 Skill 的模式
2. **Skill 质量评分**：基于使用频率和成功率评估 Skills
3. **Skill 合并**：自动检测和合并相似的 Skills
4. **跨用户学习**：匿名聚合成功的 Skills 模式
5. **自适应提示优化**：根据历史性能调整系统提示

---

## 总结

Hermes Agent 的核心架构围绕七个关键系统构建：

1. **Skills**：按需加载的知识文档，采用渐进式披露模式
2. **Plugins**：可扩展的模块化组件，支持零侵入扩展
3. **Cron**：灵活的定时任务调度系统
4. **Subagent**：隔离的子代理委托，支持并行执行
5. **Hooks**：事件驱动的回调机制，贯穿整个生命周期
6. **Tools**：自动发现的执行能力，通过中央注册表管理
7. **自进化**：Agent 自主创建和管理 Skills，实现持续学习

这些系统共同构成了一个**高度可扩展、可定制、自学习**的 AI Agent 框架，能够适应用户的特定需求和工作流程。

---

## 附录：关键文件索引

| 组件 | 核心文件 | 说明 |
|------|---------|------|
| Skills | `tools/skills_tool.py` | Skills 查询工具 |
| | `tools/skill_manager_tool.py` | Skill 管理工具 |
| | `tools/skills_hub.py` | Skills Hub 集成 |
| | `agent/prompt_builder.py` | Skills 提示构建 |
| Plugins | `hermes_cli/plugins.py` | 插件管理器 |
| | `plugins/*/` | 插件目录 |
| Cron | `cron/jobs.py` | 任务存储和管理 |
| | `cron/scheduler.py` | 调度器 |
| Subagent | `tools/delegate_tool.py` | 委托工具 |
| | `run_agent.py` | AIAgent 类（中断传播） |
| Hooks | `hermes_cli/plugins.py` | Plugin Hooks |
| | `gateway/hooks.py` | Gateway Hooks |
| | `agent/shell_hooks.py` | Shell Hooks |
| Tools | `tools/registry.py` | 工具注册表 |
| | `model_tools.py` | 工具编排层 |
| | `toolsets.py` | 工具集定义 |
| 自进化 | `tools/skill_manager_tool.py` | Skill 创建/编辑 |
| | `tools/skills_guard.py` | 安全扫描 |

---

**文档版本**：v1.0  
**最后更新**：2026-04-27  
**作者**：Hermes Agent Team
