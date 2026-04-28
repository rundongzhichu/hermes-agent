# Hermes Agent 项目结构与核心文件说明

## 项目概述

Hermes Agent 是一个功能强大的 AI 代理系统，支持多种交互方式（CLI、TUI、网关平台）、丰富的工具集、技能系统和插件架构。该项目采用模块化设计，具有良好的扩展性和可维护性。

---

## 一、顶层目录结构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 — 核心对话循环 (~12k LOC)
├── model_tools.py        # 工具编排层，discover_builtin_tools(), handle_function_call()
├── toolsets.py           # 工具集定义，_HERMES_CORE_TOOLS 列表
├── cli.py                # HermesCLI 类 — 交互式 CLI 编排器 (~11k LOC)
├── hermes_state.py       # SessionDB — SQLite 会话存储 (FTS5 搜索)
├── hermes_constants.py   # get_hermes_home(), display_hermes_home() — profile-aware 路径
├── hermes_logging.py     # setup_logging() — agent.log / errors.log / gateway.log
├── batch_runner.py       # 并行批处理
├── agent/                # Agent 内部组件（提供商适配器、记忆、缓存、压缩等）
├── hermes_cli/           # CLI 子命令、设置向导、插件加载器、皮肤引擎
├── tools/                # 工具实现 — 通过 tools/registry.py 自动发现
│   └── environments/     # 终端后端（本地、docker、ssh、modal、daytona、singularity）
├── gateway/              # 消息网关 — run.py + session.py + platforms/
│   ├── platforms/        # 各平台适配器（telegram, discord, slack, whatsapp 等）
│   └── builtin_hooks/    # 始终注册的网关节点钩子
├── plugins/              # 插件系统
│   ├── memory/           # 记忆提供者插件（honcho, mem0, supermemory 等）
│   ├── context_engine/   # 上下文引擎插件
│   └── <others>/         # 仪表板、图像生成、磁盘清理、示例等
├── optional-skills/      # 较重/小众技能（默认不激活）
├── skills/               # 内置技能包
├── ui-tui/               # Ink (React) 终端 UI — `hermes --tui`
│   └── src/              # entry.tsx, app.tsx, gatewayClient.ts + 组件/钩子/工具
├── tui_gateway/          # TUI 的 Python JSON-RPC 后端
├── acp_adapter/          # ACP 服务器（VS Code / Zed / JetBrains 集成）
├── cron/                 # 调度器 — jobs.py, scheduler.py
├── environments/         # RL 训练环境（Atropos）
├── scripts/              # run_tests.sh, release.py, 辅助脚本
├── website/              # Docusaurus 文档站点
└── tests/                # Pytest 套件（~15k 测试，~700 文件）
```

### 关键配置文件

| 文件 | 用途 |
|------|------|
| `pyproject.toml` | Python 项目配置、依赖管理 |
| `.env.example` | 环境变量模板（API 密钥等） |
| `cli-config.yaml.example` | CLI 配置模板 |
| `docker-compose.yml` | Docker 编排配置 |
| `Dockerfile` | Docker 镜像构建 |
| `flake.nix` | Nix 包管理器配置 |

### 用户配置位置

- **配置文件**: `~/.hermes/config.yaml`（设置）
- **环境变量**: `~/.hermes/.env`（仅 API 密钥）
- **日志目录**: `~/.hermes/logs/` — `agent.log` (INFO+), `errors.log` (WARNING+), `gateway.log`

---

## 二、核心模块详解

### 2.1 Agent 核心 (`run_agent.py`)

**文件大小**: ~640KB (12,828 行)

**主要职责**:
- 实现 `AIAgent` 类，负责完整的对话循环
- 处理工具调用、预算跟踪、中断检查
- 管理消息历史、上下文压缩、轨迹保存
- 支持多模型提供商适配（OpenAI、Anthropic、Gemini、Bedrock 等）

**核心方法**:
```python
class AIAgent:
    def __init__(self, base_url, api_key, provider, model, ...):
        """初始化 Agent，接受约 60 个参数"""
        
    def chat(self, message: str) -> str:
        """简单接口 — 返回最终响应字符串"""
        
    def run_conversation(self, user_message, system_message, ...) -> dict:
        """完整接口 — 返回包含 final_response 和 messages 的字典"""
```

**对话循环逻辑**:
```python
while (api_call_count < max_iterations and iteration_budget.remaining > 0) or grace_call:
    if interrupt_requested: break
    response = client.chat.completions.create(model, messages, tools)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

**关键特性**:
- 同步执行，带中断检查和预算跟踪
- 一轮优雅调用机制（grace call）
- 推理内容存储在 `assistant_msg["reasoning"]`
- 支持思维链标签剥离（`` 等）

---

### 2.2 CLI 界面 (`cli.py`)

**文件大小**: ~490KB (11,131 行)

**主要职责**:
- 实现 `HermesCLI` 类，提供交互式命令行界面
- 使用 Rich 进行美化输出，prompt_toolkit 实现输入自动补全
- 管理 KawaiiSpinner 动画（API 调用时的可爱表情）
- 处理斜杠命令（slash commands）分发

**技术栈**:
- **Rich**: 横幅/面板渲染
- **prompt_toolkit**: 固定输入区域 TUI、历史记录、自动补全
- **KawaiiSpinner** (`agent/display.py`): 动画表情和工具结果活动流

**核心功能**:
- `load_cli_config()`: 合并硬编码默认值 + 用户配置 YAML
- `process_command()`: 通过 `resolve_command()` 解析并分发斜杠命令
- 技能斜杠命令：扫描 `~/.hermes/skills/`，作为用户消息注入（而非系统提示）以保持提示缓存

**皮肤引擎** (`hermes_cli/skin_engine.py`):
- 数据驱动的 CLI 主题定制
- 从 `display.skin` 配置键初始化
- 自定义横幅颜色、spinner 表情/动词/翅膀、工具前缀、响应框、品牌文本

---

### 2.3 工具系统 (`model_tools.py` + `tools/`)

**model_tools.py** (~27KB, 671 行):
- 工具注册表的薄编排层
- 触发所有工具模块的发现（导入时自动注册）
- 提供公共 API 供 `run_agent.py`, `cli.py`, `batch_runner.py` 使用

**核心函数**:
```python
def get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode) -> list
def handle_function_call(function_name, function_args, task_id, user_task) -> str
def check_toolset_requirements() -> dict
def get_available_toolsets() -> dict
```

**异步桥接**:
- `_get_tool_loop()`: 为主线程提供持久事件循环
- `_get_worker_loop()`: 为工作线程提供每线程持久循环
- `_run_async(coro)`: 从同步上下文运行异步协程的单源真相

**tools/registry.py** (~19KB):
- 工具自注册机制
- 每个 `tools/*.py` 文件在导入时调用 `registry.register()`
- 无需手动维护导入列表

**添加新工具的步骤**:
1. 创建 `tools/your_tool.py`，调用 `registry.register()`
2. 在 `toolsets.py` 中添加工具集定义

**工具示例**:
```python
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param"), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

---

### 2.4 工具集定义 (`toolsets.py`)

**文件大小**: ~24KB

**主要职责**:
- 定义所有可用的工具集及其包含的工具
- 管理 `_HERMES_CORE_TOOLS` 列表（所有平台可用）
- 提供工具集验证和解析功能

**核心数据结构**:
```python
_HERMES_CORE_TOOLS = [
    "terminal", "read_file", "write_file", "patch", 
    "search_files", "web_search", "web_extract", ...
]

TOOLSETS = {
    "core": {"tools": [...], "description": "..."},
    "web": {"tools": ["web_search", "web_extract"], ...},
    "terminal": {"tools": ["terminal"], ...},
    ...
}
```

---

### 2.5 状态管理 (`hermes_state.py`)

**文件大小**: ~68KB

**主要职责**:
- 实现 `SessionDB` 类，基于 SQLite 的会话存储
- 支持 FTS5 全文搜索
- 管理会话历史、消息存储、元数据

**核心功能**:
- 会话 CRUD 操作
- 消息追加和检索
- 全文搜索（标题、内容、工具调用）
- 会话导出/导入

---

### 2.6 常量与路径 (`hermes_constants.py`)

**文件大小**: ~10KB

**主要职责**:
- 提供 profile-aware 的路径管理
- `get_hermes_home()`: 获取 Hermes 主目录（支持自定义 profile）
- `display_hermes_home()`: 显示友好的路径字符串

**用途**:
- 所有工具 schema 中的路径引用应使用 `display_hermes_home()`
- 持久状态文件应使用 `get_hermes_home()` 作为基础目录
- 确保每个 profile 有独立的状态空间

---

### 2.7 日志系统 (`hermes_logging.py`)

**文件大小**: ~13KB

**主要职责**:
- `setup_logging()`: 配置日志记录器
- 管理三个日志文件：
  - `agent.log`: INFO 级别及以上
  - `errors.log`: WARNING 级别及以上
  - `gateway.log`: 网关运行时专用

**特性**:
- Profile-aware 日志路径
- 可通过 `hermes logs [--follow] [--level ...] [--session ...]` 浏览

---

## 三、Agent 内部组件 (`agent/`)

### 3.1 提供商适配器

| 文件 | 大小 | 用途 |
|------|------|------|
| `anthropic_adapter.py` | ~71KB | Anthropic Claude API 适配 |
| `bedrock_adapter.py` | ~47KB | AWS Bedrock 适配 |
| `codex_responses_adapter.py` | ~44KB | OpenAI Codex Responses API |
| `gemini_cloudcode_adapter.py` | ~33KB | Google Gemini Cloud Code |
| `gemini_native_adapter.py` | ~33KB | Google Gemini 原生 API |
| `auxiliary_client.py` | ~145KB | 辅助客户端（多提供商支持） |

**职责**:
- 统一不同提供商的 API 差异
- 处理认证、请求格式、响应解析
- 错误处理和重试逻辑

---

### 3.2 上下文管理

| 文件 | 大小 | 用途 |
|------|------|------|
| `context_compressor.py` | ~59KB | 上下文压缩引擎 |
| `context_engine.py` | ~8KB | 上下文引擎协调器 |
| `context_references.py` | ~17KB | 上下文引用管理 |
| `prompt_builder.py` | ~48KB | 提示词构建器 |
| `prompt_caching.py` | ~2KB | 提示缓存应用 |

**核心功能**:
- 智能上下文压缩，保持关键信息
- 动态构建系统提示和用户提示
- 支持 Anthropic prompt caching
- 管理上下文窗口限制

---

### 3.3 记忆系统

| 文件 | 大小 | 用途 |
|------|------|------|
| `memory_manager.py` | ~15KB | 记忆管理器协调器 |
| `memory_provider.py` | ~10KB | 记忆提供者抽象基类 |

**架构**:
- ABC (Abstract Base Class) 设计模式
- 插件式记忆后端（honcho, mem0, supermemory, byterover, hindsight, holographic, openviking, retaindb）
- 生命周期钩子：`sync_turn()`, `prefetch()`, `shutdown()`, `post_setup()`

---

### 3.4 凭据管理

| 文件 | 大小 | 用途 |
|------|------|------|
| `credential_pool.py` | ~59KB | 凭据池管理 |
| `credential_sources.py` | ~16KB | 凭据来源抽象 |

**功能**:
- 多凭据轮换和故障转移
- 支持多种凭据来源（环境变量、配置文件、OAuth）
- 凭据健康检查和自动切换

---

### 3.5 其他核心组件

| 文件 | 大小 | 用途 |
|------|------|------|
| `display.py` | ~38KB | 显示工具（KawaiiSpinner、工具预览等） |
| `error_classifier.py` | ~34KB | API 错误分类和故障转移 |
| `model_metadata.py` | ~58KB | 模型元数据获取和 token 估算 |
| `usage_pricing.py` | ~25KB | 使用量定价估算 |
| `insights.py` | ~38KB | 洞察和分析 |
| `rate_limit_tracker.py` | ~8KB | 速率限制跟踪 |
| `retry_utils.py` | ~2KB | 重试工具（抖动退避） |
| `skill_commands.py` | ~14KB | 技能命令扫描和注入 |
| `title_generator.py` | ~4KB | 会话标题生成 |
| `trajectory.py` | ~2KB | 轨迹转换和保存 |

---

## 四、CLI 子系统 (`hermes_cli/`)

### 4.1 核心文件

| 文件 | 大小 | 用途 |
|------|------|------|
| `main.py` | ~352KB | CLI 入口点和主命令分发 |
| `config.py` | ~167KB | 配置加载和管理 |
| `commands.py` | ~57KB | 斜杠命令注册表 |
| `models.py` | ~110KB | 模型管理和切换 |
| `auth.py` | ~163KB | 认证系统 |
| `gateway.py` | ~168KB | 网关启动和管理 |
| `setup.py` | ~129KB | 设置向导 |
| `plugins.py` | ~45KB | 插件管理器 |
| `skin_engine.py` | ~42KB | 皮肤引擎 |
| `skills_hub.py` | ~51KB | 技能中心 |

---

### 4.2 斜杠命令系统 (`commands.py`)

**核心数据结构**:
```python
COMMAND_REGISTRY = [
    CommandDef("mycommand", "Description", "Category", 
               aliases=("mc",), args_hint="[arg]"),
    ...
]
```

**CommandDef 字段**:
- `name`: 不带斜杠的规范名称
- `description`: 人类可读的描述
- `category`: 类别（"Session", "Configuration", "Tools & Skills", "Info", "Exit"）
- `aliases`: 替代名称元组
- `args_hint`: 帮助中显示的参数占位符
- `cli_only`: 仅在交互式 CLI 中可用
- `gateway_only`: 仅在消息平台中可用
- `gateway_config_gate`: 配置门控（dotpath）

**自动同步的消费者**:
- **CLI**: `process_command()` 通过 `resolve_command()` 解析别名
- **Gateway**: `GATEWAY_KNOWN_COMMANDS` frozenset 用于钩子发射
- **Gateway help**: `gateway_help_lines()` 生成 `/help` 输出
- **Telegram**: `telegram_bot_commands()` 生成 BotCommand 菜单
- **Slack**: `slack_subcommand_map()` 生成 `/hermes` 子命令路由
- **Autocomplete**: `COMMANDS` 平铺字典供给 `SlashCommandCompleter`
- **CLI help**: `COMMANDS_BY_CATEGORY` 字典供给 `show_help()`

**添加斜杠命令的步骤**:
1. 在 `COMMAND_REGISTRY` 中添加 `CommandDef`
2. 在 `HermesCLI.process_command()` 中添加处理程序
3. 如果在网关中可用，在 `gateway/run.py` 中添加处理程序
4. 对于持久设置，使用 `save_config_value()`

---

### 4.3 配置系统 (`config.py`)

**加载器类型**:

| 加载器 | 使用者 | 位置 |
|--------|--------|------|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | `hermes tools`, `hermes setup` 等 | `hermes_cli/config.py` |
| 直接 YAML 加载 | 网关运行时 | `gateway/run.py` + `gateway/config.py` |

**DEFAULT_CONFIG**:
- 包含所有配置的默认值
- 深度合并用户 YAML 配置
- 版本迁移支持（`_config_version`）

**添加配置项**:
1. 添加到 `DEFAULT_CONFIG` 中的适当部分
2. 如果需要主动迁移/转换现有用户配置，增加 `_config_version`
3. 非秘密设置放在 `config.yaml`，秘密设置（API 密钥）放在 `.env`

---

### 4.4 插件系统 (`plugins.py`)

**PluginManager**:
- 从 `~/.hermes/plugins/`, `./.hermes/plugins/`, pip 入口点发现插件
- 每个插件暴露 `register(ctx)` 函数

**插件能力**:
- 注册 Python 回调生命周期钩子：
  - `pre_tool_call`, `post_tool_call`
  - `pre_llm_call`, `post_llm_call`
  - `on_session_start`, `on_session_end`
- 通过 `ctx.register_tool(...)` 注册新工具
- 通过 `ctx.register_cli_command(...)` 注册 CLI 子命令

**重要规则** (Teknium, 2026年5月):
> 插件**不得**修改核心文件（`run_agent.py`, `cli.py`, `gateway/run.py`, `hermes_cli/main.py` 等）。如果插件需要框架未暴露的能力，应扩展通用插件表面（新钩子、新 ctx 方法），而不是将插件特定逻辑硬编码到核心中。

---

### 4.5 皮肤引擎 (`skin_engine.py`)

**架构**:
```
hermes_cli/skin_engine.py    # SkinConfig 数据类、内置皮肤、YAML 加载器
~/.hermes/skins/*.yaml       # 用户安装的自定义皮肤（drop-in）
```

**核心函数**:
- `init_skin_from_config()`: CLI 启动时调用，从配置读取 `display.skin`
- `get_active_skin()`: 返回当前皮肤的缓存 `SkinConfig`
- `set_active_skin(name)`: 运行时切换皮肤（`/skin` 命令使用）
- `load_skin(name)`: 先加载用户皮肤，再加载内置皮肤，最后回退到默认

**皮肤定制元素**:

| 元素 | 皮肤键 | 使用者 |
|------|--------|--------|
| 横幅面板边框 | `colors.banner_border` | `banner.py` |
| Spinner 表情（等待） | `spinner.waiting_faces` | `display.py` |
| Spinner 表情（思考） | `spinner.thinking_faces` | `display.py` |
| Spinner 动词 | `spinner.thinking_verbs` | `display.py` |
| Spinner 翅膀（可选） | `spinner.wings` | `display.py` |
| 工具输出前缀 | `tool_prefix` | `display.py` |
| 每工具表情 | `tool_emojis` | `display.py` → `get_tool_emoji()` |
| Agent 名称 | `branding.agent_name` | `banner.py`, `cli.py` |
| 欢迎消息 | `branding.welcome` | `cli.py` |
| 响应框标签 | `branding.response_label` | `cli.py` |
| 提示符号 | `branding.prompt_symbol` | `cli.py` |

**内置皮肤**:
- `default`: 经典 Hermes 金色/kawaii
- `ares`: 深红/青铜战神主题，自定义 spinner 翅膀
- `mono`: 简洁灰度单色
- `slate`: 冷蓝色开发者聚焦主题

---

## 五、工具实现 (`tools/`)

### 5.1 核心工具文件

| 文件 | 大小 | 用途 |
|------|------|------|
| `registry.py` | ~19KB | 工具注册表（无依赖，被所有工具文件导入） |
| `terminal_tool.py` | ~88KB | 终端执行工具 |
| `browser_tool.py` | ~103KB | 浏览器自动化工具 |
| `delegate_tool.py` | ~100KB | 任务委托工具 |
| `mcp_tool.py` | ~118KB | MCP (Model Context Protocol) 工具 |
| `web_tools.py` | ~85KB | Web 搜索和提取工具 |
| `file_operations.py` | ~50KB | 文件操作工具 |
| `file_tools.py` | ~44KB | 文件读写工具 |
| `code_execution_tool.py` | ~60KB | 代码执行工具 |
| `send_message_tool.py` | ~64KB | 消息发送工具 |
| `tts_tool.py` | ~57KB | 文本转语音工具 |
| `vision_tools.py` | ~30KB | 视觉分析工具 |
| `image_generation_tool.py` | ~37KB | 图像生成工具 |
| `approval.py` | ~46KB | 审批流程工具 |
| `todo_tool.py` | ~10KB | 待办事项工具 |
| `memory_tool.py` | ~23KB | 记忆工具 |

---

### 5.2 浏览器工具集

| 文件 | 大小 | 用途 |
|------|------|------|
| `browser_tool.py` | ~103KB | 主浏览器工具（导航、点击、输入等） |
| `browser_supervisor.py` | ~55KB | 浏览器监督器 |
| `browser_camofox.py` | ~21KB | CamoFox 浏览器支持 |
| `browser_cdp_tool.py` | ~22KB | Chrome DevTools Protocol 工具 |
| `browser_dialog_tool.py` | ~5KB | 对话框处理工具 |

**浏览器提供者** (`browser_providers/`):
- Playwright
- Selenium
- CamoFox
- 其他自定义提供者

---

### 5.3 环境后端 (`environments/`)

| 目录/文件 | 用途 |
|-----------|------|
| `local/` | 本地终端环境 |
| `docker/` | Docker 容器环境 |
| `ssh/` | SSH 远程环境 |
| `modal/` | Modal 云环境 |
| `daytona/` | Daytona 沙箱环境 |
| `singularity/` | Singularity 容器环境 |
| `agent_loop.py` | Agent 循环抽象 |
| `hermes_base_env.py` | 基础环境类 |
| `tool_context.py` | 工具上下文管理 |

---

### 5.4 其他重要工具

| 文件 | 大小 | 用途 |
|------|------|------|
| `skills_tool.py` | ~53KB | 技能管理工具 |
| `skills_hub.py` | ~109KB | 技能中心工具 |
| `cronjob_tools.py` | ~25KB | 定时任务工具 |
| `discord_tool.py` | ~33KB | Discord 集成工具 |
| `homeassistant_tool.py` | ~18KB | Home Assistant 集成 |
| `feishu_doc_tool.py` | ~4KB | 飞书文档工具 |
| `feishu_drive_tool.py` | ~13KB | 飞书云盘工具 |
| `rl_training_tool.py` | ~56KB | 强化学习训练工具 |
| `checkpoint_manager.py` | ~24KB | 检查点管理 |
| `process_registry.py` | ~56KB | 进程注册表 |

---

## 六、消息网关 (`gateway/`)

### 6.1 核心文件

| 文件 | 大小 | 用途 |
|------|------|------|
| `run.py` | ~515KB | 网关主循环和事件处理 |
| `session.py` | ~54KB | 网关会话管理 |
| `config.py` | ~59KB | 网关配置 |
| `stream_consumer.py` | ~40KB | 流式响应消费者 |
| `status.py` | ~26KB | 状态监控 |
| `channel_directory.py` | ~10KB | 频道目录管理 |
| `delivery.py` | ~8KB | 消息投递 |
| `hooks.py` | ~8KB | 钩子系统 |
| `pairing.py` | ~11KB | 设备配对 |

---

### 6.2 平台适配器 (`platforms/`)

**支持的平台** (27 个适配器):

| 平台 | 文件 | 状态 |
|------|------|------|
| Telegram | `telegram.py` | ✅ |
| Discord | `discord.py` | ✅ |
| Slack | `slack.py` | ✅ |
| WhatsApp | `whatsapp.py` | ✅ |
| Signal | `signal.py` | ✅ |
| Matrix | `matrix.py` | ✅ |
| Mattermost | `mattermost.py` | ✅ |
| Email | `email.py` | ✅ |
| SMS | `sms.py` | ✅ |
| 钉钉 | `dingtalk.py` | ✅ |
| 企业微信 | `wecom.py` | ✅ |
| 微信 | `weixin.py` | ✅ |
| 飞书 | `feishu.py` | ✅ |
| QQ Bot | `qqbot.py` | ✅ |
| BlueBubbles | `bluebubbles.py` | ✅ |
| Home Assistant | `homeassistant.py` | ✅ |
| Webhook | `webhook.py` | ✅ |
| API Server | `api_server.py` | ✅ |
| ... | ... | ... |

**添加新平台的指南**:
参见 `ADDING_A_PLATFORM.md`（如果存在）或参考现有平台适配器的实现模式。

---

### 6.3 内置钩子 (`builtin_hooks/`)

- 始终注册的网关节点钩子
- 例如：boot-md（启动 Markdown 渲染）

---

## 七、TUI 系统 (`ui-tui/` + `tui_gateway/`)

### 7.1 架构概览

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ 渲染转录、composer、提示、活动
```

- **TypeScript** 拥有屏幕渲染
- **Python** 拥有会话、工具、模型调用和斜杠命令逻辑
- 传输层：stdio 上的换行分隔 JSON-RPC

---

### 7.2 UI-TUI (TypeScript/React)

**目录结构**:
```
ui-tui/
├── src/
│   ├── entry.tsx          # 入口点
│   ├── app.tsx            # 主应用组件
│   ├── gatewayClient.ts   # 网关客户端
│   ├── components/        # React 组件
│   ├── hooks/             # 自定义钩子
│   └── lib/               # 工具库
├── packages/
│   └── hermes-ink/        # Ink 组件库
├── package.json
├── tsconfig.json
└── vitest.config.ts       # 测试配置
```

**关键表面**:

| 表面 | Ink 组件 | 网关方法 |
|------|----------|----------|
| 聊天流式传输 | `app.tsx` + `messageLine.tsx` | `prompt.submit` → `message.delta/complete` |
| 工具活动 | `thinking.tsx` | `tool.start/progress/complete` |
| 审批 | `prompts.tsx` | `approval.respond` ← `approval.request` |
| Clarify/sudo/secret | `prompts.tsx`, `maskedPrompt.tsx` | `clarify/sudo/secret.respond` |
| 会话选择器 | `sessionPicker.tsx` | `session.list/resume` |
| 斜杠命令 | 本地处理 + 回退 | `slash.exec` → `_SlashWorker`, `command.dispatch` |
| 自动补全 | `useCompletion` 钩子 | `complete.slash`, `complete.path` |
| 主题 | `theme.ts` + `branding.tsx` | `gateway.ready` 带皮肤数据 |

**开发命令**:
```bash
cd ui-tui
npm install       # 首次安装
npm run dev       # 监听模式（重建 hermes-ink + tsx --watch）
npm start         # 生产模式
npm run build     # 完整构建（hermes-ink + tsc）
npm run type-check # 类型检查（tsc --noEmit）
npm run lint      # eslint
npm run fmt       # prettier
npm test          # vitest
```

---

### 7.3 TUI Gateway (Python)

**目录**: `tui_gateway/`

**职责**:
- Python JSON-RPC 后端
- 与 `AIAgent` 交互
- 处理来自 Ink 的请求并发出事件

**关键方法/事件**:
参见 `tui_gateway/server.py` 获取完整的方法/事件目录。

---

### 7.4 Dashboard 中的 TUI (`hermes dashboard` → `/chat`)

**架构**:
- Dashboard 嵌入真实的 `hermes --tui` — **不是**重写
- 见 `hermes_cli/pty_bridge.py` + `@app.websocket("/api/pty")` 端点在 `hermes_cli/web_server.py`

**工作流程**:
1. 浏览器加载 `web/src/pages/ChatPage.tsx`，挂载 xterm.js 的 `Terminal`
2. WebGL 渲染器 + `@xterm/addon-fit`（容器驱动调整大小）+ `@xterm/addon-unicode11`（现代宽字符宽度）
3. `/api/pty?token=…` 升级为 WebSocket；认证使用与 REST 相同的临时 `_SESSION_TOKEN`
4. 服务器通过 `ptyprocess`（POSIX PTY — WSL 可用，原生 Windows 不可用）生成 `hermes --tui`
5. 帧：双向原始 PTY 字节；通过 `\x1b[RESIZE:<cols>;<rows>]` 调整大小，在服务器上拦截并应用 `TIOCSWINSZ`

**重要原则**:
> **不要在 React 中重新实现主要聊天体验。** 主要转录、composer/输入流（包括斜杠命令行为）和 PTY 支持的终端属于嵌入式 `hermes --tui` — 你添加到 Ink 的任何新功能都会自动出现在 dashboard 中。如果你发现自己为 dashboard 重建转录或 composer，请停止并改为扩展 Ink。

**允许的结构化 React UI**:
围绕 TUI 的结构化 React UI 是允许的，当它不是第二个聊天表面时。侧边栏小部件、检查器、摘要、状态面板和类似的支撑视图（例如 `ChatSidebar`, `ModelPickerDialog`, `ToolCall`）在补充嵌入式 TUI 而非替代表录/composer/终端时是可以的。保持它们的状态独立于 PTY 子进程的会话，并以非破坏性的方式呈现它们的失败，以便终端窗格继续正常工作。

---

## 八、技能系统 (`skills/` + `optional-skills/`)

### 8.1 内置技能 (`skills/`)

**组织方式**: 按类别目录组织

**技能类别**:
- `apple/`: Apple 生态集成
- `autonomous-ai-agents/`: 自主 AI 代理
- `creative/`: 创意工具
- `data-science/`: 数据科学
- `devops/`: DevOps
- `diagramming/`: 图表绘制
- `dogfood/`: 内部测试
- `domain/`: 域名管理
- `email/`: 电子邮件
- `feeds/`: RSS/Atom feeds
- `gaming/`: 游戏
- `gifs/`: GIF 制作
- `github/`: GitHub 集成
- `index-cache/`: 索引缓存
- `inference-sh/`: 推理 shell
- `mcp/`: MCP 协议
- `media/`: 媒体处理
- `mlops/`: MLOps
- `note-taking/`: 笔记
- `productivity/`: 生产力
- `red-teaming/`: 红队测试
- `research/`: 研究
- `smart-home/`: 智能家居
- `social-media/`: 社交媒体
- `software-development/`: 软件开发

---

### 8.2 可选技能 (`optional-skills/`)

**特点**:
- 更重或更小众的技能
- 随仓库一起发布但默认不激活
- 需要手动启用

**技能类别**:
- `autonomous-ai-agents/`
- `blockchain/`
- `communication/`
- `creative/`
- `devops/`
- `dogfood/`
- `email/agentmail/`
- `health/`
- `mcp/`
- `migration/`
- `mlops/` (25 个子目录)
- `productivity/`
- `research/`
- `security/`
- `web-development/`

参见 `optional-skills/DESCRIPTION.md` 获取详细说明。

---

## 九、插件目录 (`plugins/`)

### 9.1 记忆提供者 (`memory/`)

**当前内置提供者**:
- honcho
- mem0
- supermemory
- byterover
- hindsight
- holographic
- openviking
- retaindb

**架构**:
- 实现 `MemoryProvider` ABC（见 `agent/memory_provider.py`）
- 由 `agent/memory_manager.py` 协调
- CLI 命令通过 `plugins/memory/<name>/cli.py` 注册

**规则**:
框架只为**当前活跃**的记忆提供者暴露 CLI 命令（从 config.yaml 中的 `memory.provider` 读取），因此禁用的提供者不会 clutter `hermes --help`。

---

### 9.2 其他插件

| 插件 | 目录 | 用途 |
|------|------|------|
| Context Engine | `context_engine/` | 上下文引擎插件 |
| Disk Cleanup | `disk-cleanup/` | 磁盘清理 |
| Example Dashboard | `example-dashboard/` | 仪表板示例 |
| Image Gen | `image_gen/` | 图像生成提供者 |
| Spotify | `spotify/` | Spotify 集成 |
| Strike Freedom Cockpit | `strike-freedom-cockpit/` | 控制面板 |

---

## 十、ACP 适配器 (`acp_adapter/`)

**用途**: VS Code / Zed / JetBrains IDE 集成

**文件**:
- `__init__.py`: 初始化
- `__main__.py`: 入口点
- `auth.py`: 认证
- `entry.py`: 入口处理
- `events.py`: 事件处理
- `permissions.py`: 权限管理
- `server.py`: ACP 服务器
- `session.py`: 会话管理
- `tools.py`: 工具集成

**注册表** (`acp_registry/`):
- `agent.json`: Agent 配置
- `icon.svg`: 图标

---

## 十一、Cron 调度器 (`cron/`)

**文件**:
- `jobs.py`: 作业定义
- `scheduler.py`: 调度器逻辑

**功能**:
- 定时任务调度
- 周期性执行
- 与 Agent 集成

---

## 十二、RL 训练环境 (`environments/`)

**用途**: Atropos 强化学习训练环境

**关键文件**:
- `agent_loop.py`: Agent 循环抽象
- `hermes_base_env.py`: 基础环境类
- `tool_context.py`: 工具上下文管理
- `agentic_opd_env.py`: Agentic OPD 环境
- `web_research_env.py`: Web 研究环境
- `patches.py`: 补丁应用

**工具调用解析器** (`tool_call_parsers/`):
- 12 个解析器文件
- 用于解析不同模型的 tool call 格式

---

## 十三、Web 界面 (`web/` + `website/`)

### 13.1 Web 应用 (`web/`)

**技术栈**: React + TypeScript

**用途**: 
- Dashboard 前端
- 聊天界面
- 管理面板

---

### 13.2 文档网站 (`website/`)

**技术栈**: Docusaurus

**用途**:
- 官方文档
- 用户指南
- API 参考

---

## 十四、测试套件 (`tests/`)

**规模**: ~15k 测试，~700 文件

**组织**:
- 单元测试
- 集成测试
- E2E 测试
- 工具测试
- 平台适配器测试

**运行测试**:
```bash
scripts/run_tests.sh
```

该脚本首先探测 `.venv`，然后是 `venv`，最后是 `$HOME/.hermes/hermes-agent/venv`（适用于与工作树共享 venv 的主检出）。

---

## 十五、脚本和工具 (`scripts/`)

**关键脚本**:
- `run_tests.sh`: 运行测试套件
- `release.py`: 发布自动化
- 其他辅助脚本

---

## 十六、Nix 配置 (`nix/`)

**文件**:
- `checks.nix`: Nix 检查
- `devShell.nix`: 开发 shell
- `packages.nix`: 包定义
- `python.nix`: Python 环境
- `tui.nix`: TUI 配置
- `web.nix`: Web 配置
- `lib.nix`: 库函数
- `configMergeScript.nix`: 配置合并脚本
- `nixosModules.nix`: NixOS 模块

---

## 十七、Docker 支持

**文件**:
- `Dockerfile`: Docker 镜像构建
- `docker-compose.yml`: Docker Compose 编排
- `.dockerignore`: Docker 忽略文件
- `docker/entrypoint.sh`: 容器入口点
- `docker/SOUL.md`: Docker 相关说明

---

## 十八、Homebrew 打包 (`packaging/homebrew/`)

**文件**:
- `hermes-agent.rb`: Homebrew formula
- `README.md`: 打包说明

---

## 十九、依赖管理

### 19.1 Python 依赖

**文件**:
- `pyproject.toml`: 项目配置和依赖声明
- `uv.lock`: uv 锁文件（快速 Python 包管理器）
- `constraints-termux.txt`: Termux 约束

---

### 19.2 Node.js 依赖

**文件**:
- `package.json`: Node.js 依赖（TUI）
- `package-lock.json`: npm 锁文件
- `ui-tui/package.json`: TUI 前端依赖
- `ui-tui/package-lock.json`: TUI npm 锁文件

---

## 二十、文件依赖链

```
tools/registry.py  (无依赖 — 被所有工具文件导入)
       ↑
tools/*.py  (每个调用 registry.register() 在导入时)
       ↑
model_tools.py  (导入 tools/registry + 触发自定义工具发现)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## 二十一、关键设计原则

### 21.1 模块化

- 每个组件职责单一
- 清晰的接口边界
- 易于替换和扩展

### 21.2 插件化

- 核心不硬编码插件逻辑
- 通过钩子和注册表扩展
- 用户可自定义插件

### 21.3 Profile-Aware

- 所有路径通过 `get_hermes_home()` 获取
- 支持多 profile 隔离
- 配置、状态、日志都 profile-aware

### 21.4 自动发现

- 工具自动注册（无需手动维护列表）
- 插件自动发现
- 技能自动扫描

### 21.5 向后兼容

- 保留旧 API 签名
- 配置版本迁移
- 渐进式增强

---

## 二十二、开发工作流

### 22.1 环境设置

```bash
# 激活虚拟环境
source .venv/bin/activate   # 或: source venv/bin/activate

# 安装依赖
pip install -e .
```

### 22.2 运行 CLI

```bash
python cli.py                          # 交互式模式
python cli.py --toolsets web,terminal  # 指定工具集
python cli.py -q "your question"       # 单次查询
```

### 22.3 运行 TUI

```bash
hermes --tui                           # 启动 TUI
HERMES_TUI=1 hermes                    # 通过环境变量
```

### 22.4 运行网关

```bash
hermes gateway                         # 启动消息网关
```

### 22.5 运行测试

```bash
scripts/run_tests.sh                   # 运行所有测试
pytest tests/                          # 直接运行 pytest
```

### 22.6 构建 TUI

```bash
cd ui-tui
npm install
npm run dev                            # 开发模式
npm run build                          # 生产构建
```

---

## 二十三、常见问题和陷阱

### 23.1 配置加载器混淆

**问题**: CLI 看到配置但网关看不到（或反之）

**原因**: 使用了错误的加载器

**解决**: 检查 `DEFAULT_CONFIG` 覆盖范围，确保在正确的上下文中使用正确的加载器

### 23.2 插件发现时机

**问题**: 读取插件状态但未导入 `model_tools.py`

**原因**: `discover_plugins()` 仅作为导入 `model_tools.py` 的副作用运行

**解决**: 显式调用 `discover_plugins()`（它是幂等的）

### 23.3 事件循环关闭

**问题**: "Event loop is closed" 错误

**原因**: `asyncio.run()` 创建并关闭循环，但缓存的 httpx/AsyncOpenAI 客户端仍绑定到已死亡的循环

**解决**: 使用持久事件循环（`_get_tool_loop()`, `_get_worker_loop()`）

### 23.4 路径硬编码

**问题**: 使用 `Path.home() / ".hermes"` 而非 `get_hermes_home()`

**后果**: 破坏 profile 隔离

**解决**: 始终使用 `get_hermes_home()` 和 `display_hermes_home()`

---

## 二十四、贡献指南

参见 `CONTRIBUTING.md` 和 `AGENTS.md` 获取详细的贡献指南。

**关键原则**:
1. 遵循现有代码风格
2. 添加测试
3. 更新文档
4. 不要硬编码插件特定逻辑到核心
5. 使用 profile-aware 路径
6. 保持向后兼容

---

## 二十五、版本发布

**发布说明文件**:
- `RELEASE_v0.2.0.md` 到 `RELEASE_v0.11.0.md`
- 记录每个版本的主要变更

**发布自动化**:
- `scripts/release.py`: 发布脚本

---

## 总结

Hermes Agent 是一个高度模块化、可扩展的 AI 代理系统，具有以下特点：

1. **多界面支持**: CLI、TUI、消息网关（27+ 平台）
2. **丰富的工具集**: 自动发现、插件化、多后端支持
3. **技能系统**: 内置 + 可选技能，按需加载
4. **插件架构**: 通用插件 + 专用插件（记忆、上下文引擎、图像生成等）
5. **Profile 隔离**: 完全 profile-aware 的路径和配置管理
6. **现代化技术栈**: Python + TypeScript/React + Ink
7. **完善的测试**: ~15k 测试覆盖
8. **活跃的社区**: 持续开发和版本迭代

这个项目展示了如何构建一个大型、复杂但 maintainable 的 AI 代理系统，值得深入学习其架构设计和工程实践。
