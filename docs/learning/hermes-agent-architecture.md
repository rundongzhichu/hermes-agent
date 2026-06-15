# Hermes Agent 项目架构文档

> **版本**: v0.15.1 | **作者**: Nous Research | **语言**: Python 3.11+ / TypeScript
>
> 本文档对 Hermes Agent 项目的整体架构、核心模块、目录职责和关键设计模式进行全面描述。

---

## 一、项目概览

Hermes Agent 是一个**具备自我学习能力的 AI 编程助手**，由 Nous Research 开发。核心特性包括：

- **多模型支持**：可接入 30+ 模型提供商（OpenRouter、Anthropic、OpenAI、Google、xAI、DeepSeek 等）
- **多平台消息网关**：Telegram、Discord、Slack、WhatsApp、Signal、飞书、钉钉、微信、QQ 等 20+ 平台
- **技能系统**：从经验中创建技能，在使用中自我改进，支持 agentskills.io 开放标准
- **记忆系统**：基于 SQLite FTS5 的会话搜索、可插拔记忆后端（Honcho、mem0 等 8 种）
- **Cron 调度**：内置自然语言定时任务，可交付到任意消息平台
- **子代理委托**：隔离上下文的并行任务分发
- **终端和 TUI**：基于 prompt_toolkit 的 CLI + 基于 Ink/React 的终端 UI
- **编辑器集成**：VS Code / Zed / JetBrains ACP 协议支持
- **六种终端后端**：Local、Docker、SSH、Singularity、Modal、Daytona

### 关键规模指标

| 维度 | 数据 |
|------|------|
| Python 文件数 | ~500+ |
| 测试文件数 | ~900 |
| 测试用例数 | ~17,000 |
| 核心循环 (`run_agent.py`) | ~4,800 行 |
| CLI 入口 (`cli.py`) | ~15,800 行 |
| 网关入口 (`gateway/run.py`) | ~19,600 行 |
| Agent 内部模块 (`agent/`) | ~75 个模块 |
| 工具模块 (`tools/`) | ~82 个模块 |
| 消息平台适配器 | 20+ |
| 模型提供商插件 | 27 个 |

---

## 二、顶级目录结构

```
hermes-agent/
├── run_agent.py            # AIAgent 类 — 核心对话循环
├── cli.py                  # HermesCLI 类 — 交互式 CLI 编排器
├── model_tools.py          # 工具编排层 — get_tool_definitions(), handle_function_call()
├── toolsets.py             # 工具集定义 — TOOLSETS 字典 + _HERMES_CORE_TOOLS
├── hermes_state.py         # SessionDB — SQLite 会话持久化（FTS5 全文搜索）
├── hermes_constants.py     # get_hermes_home() — 多 profile 感知的路径管理
├── hermes_logging.py       # 日志系统 — agent.log / errors.log / gateway.log
├── batch_runner.py         # 批量并行轨迹生成
├── trajectory_compressor.py # 轨迹压缩（用于模型训练）
├── hermes_bootstrap.py     # 进程启动引导（Windows UTF-8 等）
├── hermes_time.py          # 时间工具
├── utils.py                # 通用工具函数
├── mcp_serve.py            # MCP 服务入口
├── mini_swe_runner.py      # Mini SWE-bench 运行器
├── toolset_distributions.py # 工具集分发配置
│
├── agent/                  # Agent 内部模块（75 个模块）
├── hermes_cli/             # CLI 子命令系统（97 个模块）
├── tools/                  # 工具实现（82 个模块，含子目录）
├── gateway/                # 消息网关（核心 + 30+ 平台适配器）
├── plugins/                # 插件系统（19 个子目录）
├── skills/                 # 内置技能（25 个类别目录）
├── optional-skills/        # 可选技能（18 个类别）
├── optional-mcps/          # MCP 目录
├── providers/              # 遗留 provider 注册
├── cron/                   # 定时任务调度器
├── ui-tui/                 # Ink/React TUI 前端
├── tui_gateway/            # Python JSON-RPC TUI 后端
├── acp_adapter/            # ACP 编辑器集成
├── web/                    # Web Dashboard（Vite/React）
├── website/                # Docusaurus 文档站点
├── apps/                   # 桌面应用 + 引导安装器
├── scripts/                # 构建/测试/发布脚本
├── tests/                  # 测试套件（~900 个文件，~17k 测试）
├── docker/                 # Docker 容器配置
├── nix/                    # Nix flake 配置
├── docs/                   # 文档
├── locales/                # 国际化翻译
└── packaging/               # Homebrew 打包
```

---

## 三、核心架构：Agent 循环

### 3.1 AIAgent 类 (`run_agent.py`)

`AIAgent` 是整个系统的核心编排器。其 `__init__` 接收 ~60 个参数：

```python
class AIAgent:
    def __init__(self,
        base_url, api_key, provider, api_mode, model,
        max_iterations=90,
        enabled_toolsets, disabled_toolsets,
        platform, session_id, user_id, chat_id,
        skip_context_files, skip_memory,
        credential_pool, fallback_model,
        iteration_budget, checkpoints_enabled,
        # ... 40+ 回调函数
    )
```

初始化逻辑已提取到 `agent/agent_init.py::init_agent()`（~1,400 行），负责：
- 提供商自动检测和凭据解析
- 上下文引擎（context engine）引导
- 模型元数据获取
- 记忆提供者（memory provider）设置
- 工具护栏（tool guardrail）配置
- 流式输出处理器初始化

### 3.2 对话循环 (`agent/conversation_loop.py`)

核心循环是 `run_conversation()` 函数（~3,900 行），执行流程：

```
用户消息
  → 构建系统提示（system_prompt.py + prompt_builder.py）
  → 加载上下文文件（AGENTS.md / .cursorrules）
  → 预取记忆（MemoryManager.prefetch）
  → 调用 LLM（通过 transports/ 层）
  → 解析响应（工具调用 vs 文本回复）
  → 如有工具调用：
      → 并行/串行执行工具（tool_executor.py）
      → 护栏检查（tool_guardrails.py）
      → 工具结果注入消息列表
      → 回到 LLM 调用步骤
  → 后处理：
      → 记忆同步（MemoryManager.sync_turn）
      → 背景审查（background_review.py）
      → 技能维护提示（curator.py）
      → 会话标题生成（title_generator.py）
  → 返回最终回复
```

### 3.3 传输层 (`agent/transports/`)

最新的抽象层，将不同提供商的响应标准化为统一的 `NormalizedResponse` / `ToolCall` / `Usage` 数据类：

```
agent/transports/
├── types.py              # NormalizedResponse, ToolCall, Usage
├── base.py               # ProviderTransport ABC
├── anthropic.py          # Anthropic 原生 API
├── chat_completions.py   # OpenAI Chat Completions（~16 个提供商）
├── codex.py              # OpenAI Responses API (Codex)
├── bedrock.py            # AWS Bedrock
├── hermes_tools_mcp_server.py  # MCP 工具服务
├── codex_app_server.py   # Codex 应用服务器
├── codex_app_server_session.py
└── codex_event_projector.py
```

---

## 四、工具系统

### 4.1 三层架构

```
tools/registry.py          # 注册表单例 — 被所有工具文件导入
       ↑
tools/*.py                 # 每个文件调用 registry.register() 自注册
       ↑
model_tools.py             # 发现工具 → get_tool_definitions() → handle_function_call()
       ↑
toolsets.py                # TOOLSETS 字典 — 将工具名绑定到命名工具集
       ↑
run_agent.py, cli.py, gateway/run.py  # 消费端
```

### 4.2 工具注册模式

每个工具文件遵循统一模式：

```python
# tools/example_tool.py
from tools.registry import registry

# 1. 定义 JSON Schema（OpenAI 函数调用格式）
EXAMPLE_SCHEMA = {
    "name": "example_tool",
    "description": "工具描述",
    "parameters": {...}
}

# 2. 处理函数
def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True})

# 3. 可用性检查
def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

# 4. 模块级自注册
registry.register(
    name="example_tool",
    toolset="example",
    schema=EXAMPLE_SCHEMA,
    handler=lambda args, **kw: example_tool(args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

### 4.3 工具发现机制

`discover_builtin_tools()` 使用 **AST 静态解析** 扫描 `tools/*.py` 文件，检测包含顶级 `registry.register(...)` 调用的模块，然后逐一导入触发自注册。无需手动维护导入列表。

### 4.4 核心工具集 (`_HERMES_CORE_TOOLS`)

```python
_HERMES_CORE_TOOLS = [
    # Web          → web_search, web_extract
    # 终端         → terminal, process
    # 文件         → read_file, write_file, patch, search_files
    # 视觉         → vision_analyze, image_generate
    # 技能         → skills_list, skill_view, skill_manage
    # 浏览器       → browser_navigate/snapshot/click/type/scroll/...
    # TTS          → text_to_speech
    # 规划/记忆    → todo, memory
    # 搜索历史     → session_search
    # 澄清         → clarify
    # 代码执行     → execute_code
    # 委托         → delegate_task
    # Cron         → cronjob
    # 消息         → send_message
    # Home Assistant → ha_list_entities/get_state/...
    # 看板         → kanban_show/list/complete/block/...
    # 电脑使用     → computer_use
]
```

### 4.5 工具集定义 (`toolsets.py`)

所有工具集在 `TOOLSETS` 字典中统一定义，支持组合（`includes`），共 30+ 工具集：

| 工具集 | 用途 |
|--------|------|
| `web` | Web 搜索和内容提取 |
| `browser` | 浏览器自动化 |
| `terminal` | 终端命令执行 |
| `file` | 文件读写/补丁/搜索 |
| `vision` / `image_gen` | 图像分析和生成 |
| `code_execution` | 沙盒代码执行 |
| `delegation` | 子代理分发 |
| `memory` / `todo` | 记忆和任务管理 |
| `skills` | 技能系统访问 |
| `messaging` | 跨平台消息发送 |
| `homeassistant` | 智能家居控制 |
| `kanban` | 多代理看板协作 |
| `hermes-cli` / `hermes-telegram` / ... | 各平台的完整工具集 |

### 4.6 终端后端 (`tools/environments/`)

终端执行支持六种后端，统一继承 `BaseEnvironment`：

```
tools/environments/
├── base.py         # BaseEnvironment ABC + ProcessHandle 协议
├── local.py        # subprocess.Popen（默认，最快）
├── docker.py       # docker exec（沙盒，安全加固）
├── ssh.py          # SSH ControlMaster 持久连接
├── modal.py        # Modal 无服务器 GPU 云
├── managed_modal.py # 通过托管网关的 Modal
├── singularity.py  # Singularity/Apptainer 容器
├── daytona.py      # Daytona 云开发环境
└── file_sync.py    # 远程后端文件同步（tar + mtime/size 追踪）
```

---

## 五、CLI 架构 (`hermes_cli/` + `cli.py`)

### 5.1 入口点

```
hermes                   # hermes_cli/main.py → cli.py (HermesCLI)
hermes <subcommand>      # main.py 的 argparse 树 → cmd_* 函数
hermes --tui             # tui_gateway/server.py + ui-tui/src/
hermes gateway           # gateway/run.py
hermes-acp               # acp_adapter/entry.py
```

### 5.2 子命令系统

`hermes_cli/main.py`（~15,000 行）构建了包含约 50 个子命令的 argparse 树：

```
hermes chat          # 交互式对话（默认）
hermes gateway       # 消息网关管理
hermes setup         # 交互式设置向导
hermes model         # 模型选择
hermes config        # 配置管理
hermes tools         # 工具集配置（curses UI）
hermes skills        # 技能市场
hermes plugins       # 插件管理
hermes mcp           # MCP 服务器管理
hermes kanban        # 看板系统
hermes curator       # 技能策展
hermes cron          # 定时任务
hermes sessions      # 会话管理
hermes backup/import # 备份恢复
hermes doctor        # 诊断
hermes status        # 状态查看
hermes logs          # 日志查看
hermes proxy         # 本地 API 代理
hermes dashboard     # Web 仪表盘
hermes profile       # 多 profile 管理
hermes acp           # 编辑器集成
# ... 等 30+ 子命令
```

### 5.3 斜杠命令注册表 (`hermes_cli/commands.py`)

所有斜杠命令定义在中心化的 `COMMAND_REGISTRY` 列表中，CLI 自动补全、网关分发、Telegram 菜单、Slack 子命令映射、帮助文本全部派生于此：

```python
CommandDef("new", "Start a new conversation", "Session",
           aliases=("reset",)),
CommandDef("model", "Show or change the current model", "Configuration",
           aliases=(), args_hint="[provider:model]"),
# ... 80+ 命令
```

### 5.4 皮肤/主题引擎 (`hermes_cli/skin_engine.py`)

数据驱动的 CLI 视觉定制系统：
- 内置皮肤：`default`（金色）、`ares`（暗红战纹）、`mono`（灰度）、`slate`（冷蓝）
- 用户可通过 `~/.hermes/skins/<name>.yaml` 安装自定义皮肤
- 自定义内容：颜色、微调器动画、品牌文字、工具前缀/图标

### 5.5 Web Dashboard (`hermes dashboard`)

`hermes_cli/web_server.py` 提供 FastAPI 后端，嵌入真正的 `hermes --tui`（通过 `pty_bridge.py` + WebSocket），浏览器端使用 xterm.js 渲染。

---

## 六、消息网关 (`gateway/`)

### 6.1 架构总览

```
gateway/
├── run.py              # GatewayRunner（~19,600 行）— 核心编排器
├── config.py           # GatewayConfig + PlatformConfig 数据类
├── session.py          # SessionStore + SessionContext
├── delivery.py         # DeliveryRouter — 消息路由
├── hooks.py            # 事件钩子系统
├── pairing.py          # DM 配对授权
├── status.py           # 运行时状态管理
├── channel_directory.py # 联系人频道缓存
├── stream_consumer.py  # 流式消息消费者
├── stream_events.py    # 流事件词汇表
├── stream_dispatch.py  # 事件分发器
├── display_config.py   # 按平台显示配置
├── session_context.py  # 线程安全的会话上下文
├── platform_registry.py # 插件适配器注册
├── assets/             # 静态资源
├── builtin_hooks/      # 内置钩子（当前为空）
└── platforms/          # 30+ 平台适配器
```

### 6.2 平台适配器

所有适配器继承 `BasePlatformAdapter`（`base.py`，~3,400 行）：

| 适配器 | 传输方式 | 库 |
|--------|----------|-----|
| `telegram.py` | 轮询 | `python-telegram-bot` |
| `discord.py` | WebSocket | `discord.py` |
| `slack.py` | Socket Mode | `slack-bolt` |
| `whatsapp.py` | WebSocket/HTTP | 多后端 |
| `signal.py` | SSE + JSON-RPC | `signal-cli` |
| `matrix.py` | WebSocket | `mautrix` |
| `email.py` | IMAP + SMTP | stdlib |
| `sms.py` | HTTP Webhook | Twilio |
| `feishu.py` | WebSocket | `lark-oapi` |
| `dingtalk.py` | WebSocket | `dingtalk-stream` |
| `wecom.py` | WebSocket/Callback | 自实现加密 |
| `weixin.py` | 长轮询 | iLink Bot API |
| `bluebubbles.py` | REST + Webhook | BlueBubbles |
| `qqbot/` | WebSocket | QQ Bot API v2 |
| `yuanbao.py` | WebSocket | 自实现协议 |
| `api_server.py` | HTTP REST + SSE | aiohttp |
| `webhook.py` | HTTP POST | 通用 Webhook |
| `homeassistant.py` | WebSocket | Home Assistant |
| `msgraph_webhook.py` | HTTP | Microsoft Graph |
| `mattermost.py` | WebSocket | Mattermost |

### 6.3 消息处理流程

```
入站消息
  → 平台适配器接收（WebSocket 事件 / HTTP webhook / 长轮询）
  → 构建 MessageEvent
  → GatewayRunner._handle_message():
      1. 插件钩子 pre_gateway_dispatch（可跳过/改写/放行）
      2. 授权检查（allowlist / DM 配对码）
      3. 斜杠命令检测（/new /reset /model 等）
      4. 会话解析（获取或创建 SessionEntry）
      5. 检查运行中的 agent，必要时中断
      6. 异步线程池中调用 agent.run_conversation()
      7. 流式响应通过 GatewayStreamConsumer 回传
  → 回复通过适配器 send() 投递到平台
```

---

## 七、Agent 内部模块 (`agent/`)

### 7.1 核心编排

| 模块 | 行数 | 职责 |
|------|------|------|
| `agent_init.py` | ~1,400 | `AIAgent.__init__` 的完整实现 |
| `conversation_loop.py` | ~3,900 | `run_conversation()` 主循环 |
| `agent_runtime_helpers.py` | ~2,000 | 运行时辅助函数 |
| `chat_completion_helpers.py` | ~2,300 | Chat Completions API 路径 |
| `system_prompt.py` | ~400 | 三层系统提示组装 |
| `prompt_builder.py` | ~1,400 | 无状态提示构建函数 |
| `tool_executor.py` | ~1,000 | 工具调用执行（串行/并行） |
| `tool_dispatch_helpers.py` | ~300 | 并行门控、文件变更追踪 |
| `display.py` | ~800 | CLI 展示层（KawaiiSpinner 等） |

### 7.2 记忆与上下文

| 模块 | 职责 |
|------|------|
| `memory_provider.py` | `MemoryProvider` ABC — 可插拔记忆后端接口 |
| `memory_manager.py` | `MemoryManager` — 编排所有 MemoryProvider |
| `context_engine.py` | `ContextEngine` ABC — 上下文压缩引擎接口 |
| `context_compressor.py` | `ContextCompressor` — 内置压缩实现 |
| `conversation_compression.py` | 独立的压缩辅助函数 |

### 7.3 提供商适配

| 模块 | 职责 |
|------|------|
| `anthropic_adapter.py` | Anthropic Messages API 转换 |
| `codex_responses_adapter.py` | OpenAI Responses API (Codex) 转换 |
| `gemini_native_adapter.py` | Google Gemini 原生 API |
| `gemini_cloudcode_adapter.py` | Google Cloud Code 适配 |
| `bedrock_adapter.py` | AWS Bedrock 适配 |
| `azure_identity_adapter.py` | Azure 身份认证 |
| `transports/` | 统一传输抽象层（10 个文件） |

### 7.4 凭据管理

| 模块 | 职责 |
|------|------|
| `credential_pool.py` | `CredentialPool` — 多凭据故障转移 |
| `credential_sources.py` | 凭据来源统一接口 |
| `credential_persistence.py` | 网关会话凭据追踪 |
| `google_oauth.py` | Google OAuth 流程 |

### 7.5 错误处理

| 模块 | 职责 |
|------|------|
| `error_classifier.py` | `FailoverReason` 枚举 + `classify_api_error()` |
| `retry_utils.py` | 抖动退避重试 |
| `rate_limit_tracker.py` | 速率限制状态追踪 |

### 7.6 后台任务

| 模块 | 职责 |
|------|------|
| `curator.py` | 技能生命周期管理（自动归档/审查） |
| `curator_backup.py` | 策展前技能快照 |
| `background_review.py` | 回合后记忆/技能审查 |
| `title_generator.py` | 会话标题自动生成 |

### 7.7 辅助 LLM 客户端 (`auxiliary_client.py` ~4,800 行)

`call_llm()` 函数提供辅助 LLM 调用路由，用于压缩、搜索、提取、视觉等辅助任务。解析回退链：主提供商 → OpenRouter → Nous Portal → 自定义 → Anthropic → 直接 API-key 提供商。每个辅助任务可在 `config.yaml` 中覆盖提供商/模型。

### 7.8 注册表模式（插件化子系统）

```
agent/browser_registry.py   → 浏览器提供商注册
agent/web_search_registry.py → 搜索提供商注册
agent/image_gen_registry.py  → 图像生成提供商注册
agent/video_gen_registry.py  → 视频生成提供商注册
agent/transcription_registry.py → 转录提供商注册
agent/tts_registry.py       → TTS 提供商注册
```

每个注册表都有一个对应的 `_provider.py` ABC 和多个 `plugins/<category>/<name>/` 下的实现。

### 7.9 安全模块

| 模块 | 职责 |
|------|------|
| `tool_guardrails.py` | 工具调用安全检查（重复调用、空闲循环、身份冲突） |
| `file_safety.py` | 敏感文件路径保护（SSH 密钥、.env 等） |
| `message_sanitization.py` | 消息/响应清理（代理、非 ASCII） |
| `redact.py` | API 密钥/令牌日志脱敏 |
| `think_scrubber.py` | 流式输出中去 `<thinking>` 块 |
| `prompt_builder.py` | 上下文文件中的提示注入检测 |

---

## 八、插件系统 (`plugins/`)

### 8.1 插件发现

`PluginManager`（`hermes_cli/plugins.py`）从四个来源发现插件：

1. **捆绑**：`<repo>/plugins/<name>/`
2. **用户**：`~/.hermes/plugins/<name>/`
3. **项目**：`./.hermes/plugins/<name>/`
4. **pip 入口点**：`hermes_plugins` 组

每个插件需要 `plugin.yaml` 清单和带 `register(ctx)` 函数的 `__init__.py`。

### 8.2 插件目录一览

插件分为两种形式：**类别插件**（category plugin，子目录中各有一个独立提供者）和 **独立插件**（standalone plugin，单个目录提供完整功能）。

```
plugins/
├── memory/               # 类别插件 — 记忆提供者（8 个后端）
│   ├── __init__.py       #   发现/加载器 (14 KB, discover_memory_providers)
│   ├── honcho/           #   Honcho 方言式用户建模 (+ cli.py 69 KB)
│   ├── mem0/             #   Mem0 记忆
│   ├── supermemory/      #   Supermemory
│   ├── byterover/        #   Byterover
│   ├── hindsight/        #   Hindsight
│   ├── holographic/      #   全息记忆 (store/retrieval 分离)
│   ├── openviking/       #   OpenViking
│   └── retaindb/         #   RetainDB
│
├── model-providers/      # 类别插件 — 模型提供商（29 个）
│   ├── openrouter/       #   OpenRouter (200+ 模型, 自定义 extra_body/build_api_kwargs)
│   ├── anthropic/        #   Anthropic (x-api-key + anthropic-version 头)
│   ├── gemini/           #   Google Gemini (双 profile: API key + CloudCode OAuth)
│   ├── openai-codex/     #   OpenAI Codex
│   ├── deepseek/         #   DeepSeek
│   ├── xai/              #   xAI Grok
│   ├── qwen-oauth/       #   阿里通义千问 (OAuth)
│   ├── bedrock/          #   AWS Bedrock (SDK, 非 REST)
│   ├── azure-foundry/    #   Azure AI Foundry
│   ├── huggingface/      #   Hugging Face
│   ├── ollama-cloud/     #   Ollama
│   ├── nous/             #   Nous Portal
│   ├── novita/           #   NovitaAI
│   ├── nvidia/           #   NVIDIA NIM
│   ├── gmi/              #   Google GenAI Media
│   ├── copilot/          #   GitHub Copilot
│   ├── copilot-acp/      #   Copilot ACP
│   ├── custom/           #   自定义端点
│   ├── alibaba/          #   阿里云
│   ├── alibaba-coding-plan/
│   ├── arcee/            #   Arcee
│   ├── kilocode/         #   Kilo Code
│   ├── kimi-coding/      #   Moonshot Kimi
│   ├── minimax/          #   MiniMax
│   ├── opencode-zen/     #   OpenCode Zen
│   ├── stepfun/          #   阶跃星辰
│   ├── xiaomi/           #   小米 MiMo
│   └── zai/              #   z.ai/GLM
│
├── context_engine/       # 类别插件 — 上下文压缩引擎（1 个内置实现）
├── browser/              # 类别插件 — 浏览器后端（3 个）
│   ├── browser_use/      #   browser-use 库
│   ├── browserbase/      #   Browserbase 云浏览器
│   └── firecrawl/        #   Firecrawl 云浏览器
│
├── web/                  # 类别插件 — 搜索引擎（8 个）
│   ├── exa/              #   Exa
│   ├── firecrawl/        #   Firecrawl
│   ├── ddgs/             #   DuckDuckGo
│   ├── brave_free/       #   Brave Search (免费)
│   ├── searxng/          #   SearXNG
│   ├── tavily/           #   Tavily
│   ├── parallel/         #   Parallel Web
│   └── xai/              #   xAI Search
│
├── image_gen/            # 类别插件 — 图像生成（5 个后端）
├── video_gen/            # 类别插件 — 视频生成（2 个后端）
├── observability/        # 类别插件 — 可观测性（langfuse）
│
├── platforms/            # 类别插件 — 网关平台适配器（8 个）
│   ├── discord/          #   Discord (280 KB adapter.py)
│   ├── google_chat/      #   Google Chat (146 KB + oauth)
│   ├── irc/              #   IRC
│   ├── line/             #   LINE
│   ├── mattermost/       #   Mattermost
│   ├── ntfy/             #   ntfy 推送
│   ├── simplex/          #   SimpleX
│   └── teams/            #   Microsoft Teams
│
├── kanban/               # 独立插件 — 看板系统 (dashboard UI + systemd 服务)
├── hermes-achievements/  # 独立插件 — 成就系统 (dashboard UI + 规范文档)
├── security-guidance/    # 独立插件 — 代码安全扫描 (25 条安全规则, 非阻塞式)
├── spotify/              # 独立插件 — Spotify (7 个工具, PKCE OAuth)
├── google_meet/          # 独立插件 — Google Meet 机器人 (5 个工具, 实时音频)
├── teams_pipeline/       # 独立插件 — Teams 会议管道管理
├── disk-cleanup/         # 独立插件 — 临时文件自动清理 (hooks + 斜杠命令)
├── dashboard_auth/       # 类别插件 — 仪表盘认证 (nous OAuth)
└── example-dashboard/    # 示例仪表盘
```

### 8.3 插件注册机制

每个插件的 `__init__.py` 通过 `register(ctx)` 函数在导入时自注册：

```python
# 独立插件示例 (如 spotify)
def register(ctx):
    ctx.register_tool(playback_tool)
    ctx.register_tool(search_tool)
    ctx.register_hook("post_tool_call", spotify_post_tool_hook)

# 类别插件示例 (如 web/ 下的搜索引擎)
def register(ctx):
    ctx.register_web_search_provider(ExaProvider())
```

### 8.4 模型提供商注册 (`providers/`)

独立的懒加载注册系统（3 个文件）：

```python
# providers/base.py — ProviderProfile 数据类
class ProviderProfile:
    name, api_mode, aliases, display_name, description
    base_url, models_url, auth_type
    env_vars, default_headers
    fixed_temperature, default_max_tokens
    # 钩子方法
    def build_extra_body()       # 自定义请求体
    def build_api_kwargs_extras() # 自定义 API 参数
    def fetch_models()            # 动态获取模型列表

# providers/__init__.py — 注册表
register_provider(profile)        # 注册
get_provider_profile(name)        # 通过名称/别名查找
list_providers()                  # 列出所有
```

发现顺序：捆绑插件 → 用户插件 → 遗留单文件。后注册者覆盖前者（last-writer-wins），允许用户替换内置配置。

### 8.5 记忆提供者接口

```python
class MemoryProvider(ABC):
    def initialize(self, config, hermes_home)         # 初始化
    def system_prompt_block(self) -> str               # 注入系统提示块
    def prefetch(self, query) -> str                   # 预取相关记忆
    def sync_turn(self, turn_messages) -> None         # 同步对话轮次
    def get_tool_schemas(self) -> list                 # 工具 Schema
    def handle_tool_call(self, name, args) -> str      # 处理工具调用
    def shutdown(self) -> None                          # 清理
```

一次只能激活一个记忆提供者。新记忆后端必须发布为独立插件仓库——不接受新的内置提供者 PR。

### 8.3 记忆提供者接口

```python
class MemoryProvider(ABC):
    def initialize(self, config, hermes_home)         # 初始化
    def system_prompt_block(self) -> str               # 注入系统提示块
    def prefetch(self, query) -> str                   # 预取相关记忆
    def sync_turn(self, turn_messages) -> None         # 同步对话轮次
    def get_tool_schemas(self) -> list                 # 工具 Schema
    def handle_tool_call(self, name, args) -> str      # 处理工具调用
    def shutdown(self) -> None                          # 清理
```

---

## 九、技能系统 (`skills/` + `optional-skills/`)

### 9.1 技能目录结构

```
skills/                         # 内置技能（默认可用，25 个类别，100+ 个子技能）
├── github/ (6)                 # codebase-inspection, github-auth, code-review, 
│                               #   github-issues, pr-workflow, repo-management
├── software-development/ (12)  # plan, spike, test-driven-development, 
│                               #   subagent-driven-development, writing-plans, ...
├── creative/ (20)              # architecture-diagram, ascii-art, excalidraw, 
│                               #   manim-video, p5js, pixel-art, comfyui, ...
├── mlops/ (7)                  # evaluation, huggingface-hub, inference, models, 
│                               #   research, training, vector-databases
├── research/ (8)               # arxiv, blogwatcher, research-paper-writing, 
│                               #   kanban-orchestrator, kanban-worker, ...
├── productivity/ (9)           # airtable, google-workspace, linear, notion, 
│                               #   powerpoint, nano-pdf, ocr, teams-meeting-pipeline
├── media/ (5)                  # gif-search, spotify, youtube-content, ...
├── apple/ (5)                  # apple-notes, reminders, findmy, imessage, macos
├── devops/                     # DevOps 自动化
├── data-science/               # 数据科学
├── email/                      # 邮件管理
├── social-media/               # 社交媒体
├── note-taking/                # 笔记管理
├── diagramming/                # 图表绘制
├── gaming/                     # 游戏相关
├── gifs/                       # GIF 制作
├── smart-home/                 # 智能家居
├── domain/                     # 域名管理
├── mcp/ (1)                    # native-mcp
├── inference-sh/               # 推理服务
├── red-teaming/                # 安全测试
├── dogfood/                    # 内部使用
├── index-cache/                # 索引缓存
├── yuanbao/                    # 腾讯元宝
└── autonomous-ai-agents/       # 自主 AI Agent

optional-skills/                # 可选技能（需显式安装，17 个类别）
├── mlops/ (25)                 # accelerate, chroma, clip, faiss, flash-attention,
│                               #   guidance, instructor, modal, peft, pinecone,
│                               #   stable-diffusion, tensorrt-llm, whisper, ...
├── research/ (11)              # bioinformatics, darwinian-evolver, drug-discovery,
│                               #   osint-investigation, gitnexus-explorer, ...
├── autonomous-ai-agents/ (5)   # antigravity-cli, blackbox, grok, honcho, openhands
├── creative/ (5)               # blender-mcp, concept-diagrams, meme-generation, ...
├── security/ (4)               # 1password, oss-forensics, sherlock, web-pentest
├── blockchain/                 # 区块链
├── communication/              # 通讯
├── devops/                     # DevOps
├── email/                      # 邮件
├── finance/                    # 金融
├── health/                     # 健康
├── mcp/                        # MCP
├── migration/                  # 迁移
├── productivity/               # 生产力
├── software-development/       # 软件开发
└── web-development/            # Web 开发
```

### 9.2 SKILL.md 格式

每个技能目录包含一个 `SKILL.md` 文件：

```markdown
---
name: skill-name
description: ≤60 字符的一句话描述。
version: 1.0.0
author: 作者名
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [tag1, tag2]
    category: devops
---

# Skill Name Skill

## When to Use
...

## Prerequisites
...

## How to Run
...

## Procedure
...

## Pitfalls
...

## Verification
...
```

### 9.3 策展系统 (Curator)

后台技能维护系统，自动追踪 agent 创建的技能使用情况：
- 不活跃技能自动标记为 `stale`
- 超期不活跃技能自动 `archive` 到 `~/.hermes/skills/.archive/`
- 仅处理 `created_by: "agent"` 来源的技能
- 内置/市场安装的技能不受影响
- 固定（pinned）技能豁免所有自动转换

---

## 十、Cron 调度系统 (`cron/`)

```
cron/
├── __init__.py     # 公开 API 导出
├── jobs.py         # 任务存储（~47 KB，JSON 文件 + croniter 解析）
└── scheduler.py    # 调度引擎（~87 KB，tick 循环）
```

### 10.1 调度流程

调度器通过 `GatewayRunner` 运行在网关进程内部，每 60 秒触发一次 `tick()`：

```
tick() 调用
  → 文件锁获取 (~/.hermes/cron/.tick.lock, fcntl/msvcrt)
  → get_due_jobs() 获取到期任务
  → advance_next_run() 推进调度（在锁内，保证 at-most-once）
  → ThreadPoolExecutor 并行执行
     ├── no_agent 快捷路径：直接执行 bash/python 脚本
     └── LLM 路径：
          ├── 构建 AIAgent（模型、工具集、MCP、凭据池）
          ├── 加载技能内容（skill_view）
          ├── 提示注入扫描（创建时 + 运行时双重检查）
          ├── 超时控制（默认 600s，HERMES_CRON_TIMEOUT 可配置）
          └── 结果投递（多平台：origin/telegram/discord/...)
```

### 10.2 调度格式

- **Duration**: `"30m"`, `"2h"`, `"1d"`
- **"every" 短语**: `"every 2h"`, `"every monday 9am"`
- **Cron 表达式**: `"0 9 * * *"`（通过 croniter 解析）
- **ISO 时间戳**: `"2026-06-01T09:00:00Z"` （一次性）

### 10.3 任务配置

每个任务可配置：
- 模型/提供商覆盖、推理配置
- 技能加载（`skills` 列表，运行前动态加载技能内容）
- 脚本预运行（`script` 字段，输出注入提示）
- `no_agent=True`：纯脚本，无 LLM 参与
- 上下文链：`context_from` 字段链接前后任务
- 工作目录和 profile 隔离
- 多平台交付（`deliver` 字段：`local` / `origin` / 特定平台 / 逗号分隔）

### 10.4 安全机制

- Cron 会话 3 分钟强制中断
- 文件锁防止重复 tick（跨进程安全）
- 提示注入双重扫描（创建时 + 运行时包含技能内容后）
- 脚本执行限制在 `HERMES_HOME/scripts/` 目录内
- 任务 ID 验证为文件系统安全路径组件
- 默认 `skip_memory=True`（记忆提供者不在 cron 中运行）

---

## 十一、看板系统 (Kanban)

持久化的 SQLite 多代理工作队列：

```
hermes_cli/kanban.py           # CLI 命令（~2,800 行）
hermes_cli/kanban_db.py        # SQLite 数据层（~7,500 行）
tools/kanban_tools.py          # Agent 工具接口
plugins/kanban/dashboard/      # Web UI
plugins/kanban/systemd/        # Systemd 服务
```

特性：
- 一个看板可服务多个配置文件和业务租户
- 调度程序每 60 秒回收过期声明、提升就绪任务
- `failure_limit` 后自动阻塞任务（默认 2 次连续失败）
- 租户级隔离（工作空间路径 + 记忆键隔离）

---

## 十二、测试架构 (`tests/`)

### 12.1 测试基础设施

```
tests/
├── conftest.py               # _isolate_hermes_home autouse fixture
├── _isolate_plugin.py        # 子进程隔离插件
├── fakes/                    # 模拟对象
├── agent/                    # Agent 模块测试
├── cli/                      # CLI 测试
├── gateway/                  # 网关测试
├── tools/                    # 工具测试
├── hermes_cli/               # CLI 子命令测试
├── plugins/                  # 插件测试
├── skills/                   # 技能测试
├── providers/                # 提供商测试
├── integration/              # 集成测试
├── e2e/                      # 端到端测试
├── stress/                   # 压力测试
├── run_agent/                # AIAgent 测试
├── hermes_state/             # 会话状态测试
├── acp/                      # ACP 测试
└── acp_adapter/              # ACP 适配器测试
```

### 12.2 测试运行

```bash
# 始终使用包装脚本（确保 CI 环境一致性）
scripts/run_tests.sh                          # 完整套件
scripts/run_tests.sh tests/gateway/           # 按目录
scripts/run_tests.sh tests/agent/test_foo.py  # 单个文件
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # 单个测试

# 环境保证
# - TZ=UTC, LANG=C.UTF-8, PYTHONHASHSEED=0
# - 所有凭证环境变量被清空
# - HOME/~/.hermes/ 重定向到临时目录
# - 每个测试文件独立子进程（spawn 模式，Linux/macOS/Windows 兼容）
```

### 12.3 测试原则

- 不写 **变更检测器测试**（快照当前数据、枚举计数的测试）
- 写 **契约不变性测试**（模块间的关系约束）
- 测试不得写入真实 `~/.hermes/` — `_isolate_hermes_home` fixture 重定向到临时目录

---

## 十三、关键设计模式与约定

### 13.1 自注册模式

工具、提供商、平台适配器都使用模块级的自注册调用。导入模块即完成注册，无需手动维护列表。

### 13.2 懒加载优化

- OpenAI SDK 使用代理对象延迟导入（节省 ~240ms 启动时间）
- 可选依赖通过 `tools/lazy_deps.py` 在首次使用时 pip install
- `[all]` 额外只包含无法懒安装的核心依赖

### 13.3 提示缓存保护

绝不中途更改上下文、工具集或重建系统提示。状态变更命令默认使用延迟失效（下个会话生效），可选 `--now` 标志立即失效。

### 13.4 Profile 安全

- 所有路径使用 `get_hermes_home()`（绝不硬编码 `~/.hermes`）
- 用户可见路径使用 `display_hermes_home()`
- ContextVar 提供进程内覆盖，避免修改 `os.environ`

### 13.5 依赖锁定策略

- 所有依赖必须有上界 (`>=floor,<next_major`)
- 核心依赖固定精确版本 (`==X.Y.Z`)
- CI Actions 使用 commit SHA + 版本注释
- 运行 `uv lock` 重新生成哈希锁文件

### 13.6 配置分离

- 设置 → `config.yaml`（`DEFAULT_CONFIG` in `hermes_cli/config.py`）
- 秘密 → `.env`（`OPTIONAL_ENV_VARS` in `hermes_cli/config.py`）
- 三种配置加载器：
  - `load_cli_config()` — CLI 模式
  - `load_config()` — 大多数子命令
  - 直接 YAML 加载 — 网关运行时

### 13.7 斜杠命令架构

添加命令 = 一个 `CommandDef` 条目 + 一个处理函数。CLI、网关、Telegram 菜单、Slack 子命令映射、帮助文本、自动补全全部自动更新。

---

## 十四、消息平台 (Gateway Platforms)

### 内置平台适配器完整列表

| 平台 | 适配器文件 | 协议 |
|------|-----------|------|
| Telegram | `telegram.py` | 轮询 (python-telegram-bot) |
| Discord | `discord.py` | WebSocket (discord.py) |
| Slack | `slack.py` | Socket Mode (slack-bolt) |
| WhatsApp | `whatsapp.py` | 多后端 (Baileys/WABA/WebJS) |
| Signal | `signal.py` | SSE + JSON-RPC (signal-cli) |
| Matrix | `matrix.py` | WebSocket (mautrix + E2EE) |
| Email | `email.py` | IMAP + SMTP |
| SMS | `sms.py` | HTTP Webhook (Twilio) |
| Feishu/Lark | `feishu.py` | WebSocket (lark-oapi) |
| DingTalk | `dingtalk.py` | WebSocket (dingtalk-stream) |
| WeCom | `wecom.py` | WebSocket + Callback |
| WeChat | `weixin.py` | 长轮询 (iLink Bot API) |
| BlueBubbles | `bluebubbles.py` | REST + Webhook (iMessage) |
| QQ | `qqbot/` | WebSocket (QQ Bot API v2) |
| Yuanbao | `yuanbao.py` | WebSocket (自实现协议) |
| Home Assistant | `homeassistant.py` | WebSocket |
| API Server | `api_server.py` | HTTP REST + SSE |
| Webhook | `webhook.py` | HTTP POST (通用) |
| Mattermost | `mattermost.py` | WebSocket |
| MS Graph | `msgraph_webhook.py` | HTTP Webhook |

---

## 十五、模型提供商系统

### 15.1 发现机制

提供商发现由 `providers/__init__.py` 中的 `_discover_providers()` 实现，采用**懒加载**模式（首次访问 `get_provider_profile()` 或 `list_providers()` 时触发）。

扫描顺序（后注册者覆盖前者）：
1. **捆绑**：`<repo>/plugins/model-providers/<name>/`
2. **用户**：`$HERMES_HOME/plugins/model-providers/<name>/`
3. **遗留**：`<repo>/providers/<name>.py`（向后兼容）

### 15.2 ProviderProfile 核心字段

```python
@dataclass
class ProviderProfile:
    name: str              # 规范名称 (如 "openrouter")
    api_mode: str          # "chat_completions" | "codex_responses"
    aliases: tuple         # 替代名称 (如 ("open-router",))
    display_name: str      # 在 UI 中显示的名称
    base_url: str          # API 基础 URL
    models_url: str        # 模型列表端点
    auth_type: str         # "api_key" | "oauth" | ...
    env_vars: dict         # 所需环境变量
    default_headers: dict  # 默认 HTTP 头
    fixed_temperature: float | None # 固定温度 (OMIT_TEMPERATURE 表示不发送)
    default_max_tokens: int
    # 钩子方法
    def build_extra_body()        # 自定义请求体 (如 OpenRouter provider preferences)
    def build_api_kwargs_extras() # 自定义 API kwargs (如推理配置传递)
    def get_max_tokens()          # 每模型动态 max_tokens
    def fetch_models()            # 动态模型列表获取 (None = 使用 models.py 中的静态列表)
```

### 15.3 完整提供商列表（29 个）

```
openrouter      anthropic       openai-codex      deepseek
xai             gemini          gmi               qwen-oauth
bedrock         azure-foundry   huggingface       ollama-cloud
nous            novita          nvidia            copilot
copilot-acp     custom          alibaba           alibaba-coding-plan
arcee           kilocode        kimi-coding       minimax
opencode-zen    stepfun         xiaomi            zai
```

每个提供商都是 `plugins/model-providers/<name>/` 下的独立插件，在模块导入时通过 `providers.register_provider(ProviderProfile(...))` 完成注册。

---

## 十六、数据流总结

```
┌─────────────────────────────────────────────────────────┐
│                      用户交互层                           │
│  CLI (prompt_toolkit)  TUI (Ink/React)  Web Dashboard    │
│  消息平台 (Telegram/Discord/Slack/WhatsApp/...)           │
│  编辑器 (VS Code/Zed/JetBrains via ACP)                   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   AIAgent (run_agent.py)                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ init_agent  │  │ system_prompt│  │ memory_manager │  │
│  └─────────────┘  └──────────────┘  └────────────────┘  │
│  ┌──────────────────────────────────────────────────┐   │
│  │  run_conversation() 主循环                        │   │
│  │  LLM 调用 → 工具分发 → 结果处理 → 循环/返回       │   │
│  └──────────────────────────────────────────────────┘   │
└───────┬──────────────────┬──────────────────────────────┘
        │                  │
        ▼                  ▼
┌───────────────┐  ┌───────────────────────┐
│  transports/  │  │  model_tools.py       │
│  提供商适配    │  │  → tools/registry.py  │
│  (标准化响应)  │  │  → tools/*.py (82个)  │
└───────────────┘  │  → tools/environments/ │
                   │  → MCP 工具            │
                   └───────────────────────┘
        │                  │
        ▼                  ▼
┌───────────────┐  ┌───────────────────────┐
│  credential_  │  │  toolsets.py          │
│  pool.py      │  │  (30+ 工具集)         │
│  (多凭据切换)  │  └───────────────────────┘
└───────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                     持久化层                              │
│  hermes_state.py (SQLite + FTS5)  记忆系统 (8 种后端)     │
│  ~/.hermes/config.yaml             ~/.hermes/.env          │
│  ~/.hermes/skills/                 ~/.hermes/sessions/     │
└─────────────────────────────────────────────────────────┘
```

---

## 十七、开发快速参考

```bash
# 环境搭建
source .venv/bin/activate
./setup-hermes.sh

# 测试
scripts/run_tests.sh                           # 全量测试
scripts/run_tests.sh tests/agent/test_foo.py   # 单文件
scripts/run_tests.sh --no-isolate tests/foo/   # 调试模式

# 代码检查
ruff check .

# TUI 开发
cd ui-tui && npm run dev

# 直接运行
./hermes                  # CLI 模式
./hermes --tui            # TUI 模式
./hermes gateway          # 启动网关
```

---

> **相关文档**:
> - `AGENTS.md` — AI 编码助手开发指南（1,157 行）
> - `CONTRIBUTING.md` — 贡献指南
> - `CLAUDE.md` — Claude Code 快速参考
> - `SECURITY.md` — 安全策略
> - `README.md` — 用户文档入口
