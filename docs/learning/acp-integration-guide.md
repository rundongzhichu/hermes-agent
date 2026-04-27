# ACP (Agent Communication Protocol) 集成指南

## 概述

ACP 是 Hermes Agent 与代码编辑器（VS Code、Zed、JetBrains IDE 等）之间的通信协议适配器。它允许开发者在 IDE 中直接调用 Hermes Agent 的强大功能，实现 AI 辅助编程的无缝集成。

**核心定位**：将 Hermes Agent 的完整能力通过标准化的 ACP 协议暴露给编辑器客户端，提供会话管理、工具执行、实时流式响应等功能。

---

## 架构设计

### 组件结构

```
acp_adapter/
├── __main__.py          # Python 模块入口点
├── entry.py             # CLI 启动器，环境加载和日志配置
├── server.py            # ACP 服务器主实现 (HermesACPAgent)
├── session.py           # 会话管理器 (SessionManager + SessionState)
├── tools.py             # 工具映射和事件构建器
├── events.py            # 回调工厂（桥接 AIAgent 事件到 ACP 通知）
├── auth.py              # 认证提供者检测
├── permissions.py       # 权限审批回调
└── acp_registry/        # ACP 注册表元数据
    ├── agent.json       # Agent 描述和分发配置
    └── icon.svg         # 图标资源
```

### 运行时流程

```
编辑器客户端 (ACP Client)
    ↓ JSON-RPC over stdio
acp_adapter.entry.main()
    ↓
HermesACPAgent (server.py)
    ↓ 会话管理
SessionManager (session.py)
    ↓ 代理实例化
AIAgent (run_agent.py)
    ↓ 工具执行
tools/*.py (hermes-acp toolset)
    ↓ 事件回调
events.py → ACP 协议更新
    ↓
编辑器客户端接收实时更新
```

---

## 核心功能模块

### 1. 会话管理 (Session Management)

**文件**: `acp_adapter/session.py`

#### SessionState 数据结构

每个 ACP 会话维护以下状态：

- **session_id**: UUID 唯一标识符
- **agent**: AIAgent 实例（包含模型配置、工具集、历史记录）
- **cwd**: 当前工作目录（影响终端命令执行路径）
- **model**: 当前使用的 LLM 模型
- **history**: 对话历史消息列表（OpenAI 格式）
- **cancel_event**: 线程安全的中断信号

#### SessionManager 核心方法

| 方法 | 功能 | 说明 |
|------|------|------|
| `create_session(cwd)` | 创建新会话 | 生成 UUID，初始化 AIAgent，持久化到数据库 |
| `get_session(session_id)` | 获取会话 | 优先内存查找，未命中则从数据库恢复 |
| `remove_session(session_id)` | 删除会话 | 清理内存和数据库记录 |
| `fork_session(session_id, cwd)` | 克隆会话 | 深度复制历史，创建新会话 ID |
| `list_sessions()` | 列出所有会话 | 合并内存和数据库中的会话 |
| `update_cwd(session_id, cwd)` | 更新工作目录 | 同步到工具环境变量覆盖 |
| `save_session(session_id)` | 持久化会话 | 写入 SessionDB（state.db） |
| `cleanup()` | 清理所有会话 | 用于测试或重置场景 |

#### 持久化机制

- **存储位置**: `~/.hermes/state.db` (SQLite)
- **会话来源标记**: `source="acp"`
- **自动恢复**: 进程重启后，`load_session` / `resume_session` 自动从数据库重建会话
- **元数据存储**: model_config 包含 cwd、provider、base_url、api_mode

---

### 2. ACP 协议实现

**文件**: `acp_adapter/server.py`

#### 生命周期方法

##### Initialize (初始化)

```python
async def initialize(protocol_version, client_capabilities, client_info) -> InitializeResponse
```

- 返回协议版本兼容性信息
- 声明 Agent 能力：`load_session`, `fork`, `list`, `resume`
- 检测并通告可用的认证方法（基于 `.env` 配置的 provider）

##### Authenticate (认证)

```python
async def authenticate(method_id) -> AuthenticateResponse | None
```

- 验证运行时凭证是否可用
- 支持 OpenRouter、Anthropic 等 Provider 的自动检测

#### 会话操作方法

##### new_session

```python
async def new_session(cwd, mcp_servers) -> NewSessionResponse
```

- 创建全新会话
- 注册 MCP 服务器（如果提供）
- 广播可用命令列表

##### load_session / resume_session

```python
async def load_session(cwd, session_id, mcp_servers) -> LoadSessionResponse
async def resume_session(cwd, session_id, mcp_servers) -> ResumeSessionResponse
```

- 从数据库恢复会话（如果存在）
- 不存在则创建新会话（容错处理）
- 更新工作目录和 MCP 服务器

##### fork_session

```python
async def fork_session(cwd, session_id, mcp_servers) -> ForkSessionResponse
```

- 深度复制原会话的历史记录
- 生成新的 session_id
- 保持独立的 agent 实例

##### cancel

```python
async def cancel(session_id)
```

- 设置取消事件标志
- 调用 `agent.interrupt()` 中断正在执行的对话

##### list_sessions

```python
async def list_sessions(cursor, cwd) -> ListSessionsResponse
```

- 返回所有活跃会话的轻量级信息
- 支持游标分页（当前未使用）

#### 核心交互方法

##### prompt (用户提示处理)

```python
async def prompt(prompt_blocks, session_id) -> PromptResponse
```

**处理流程**:

1. **提取文本**: 从 ACP 内容块中提取纯文本
2. **斜杠命令拦截**: 本地处理 `/help`, `/model`, `/tools` 等命令
3. **设置回调**: 绑定工具进度、思考过程、步骤完成、消息流式输出回调
4. **权限审批**: 注入危险命令审批回调（terminal 工具）
5. **线程池执行**: 在专用线程池中运行同步的 `agent.run_conversation()`
6. **持久化历史**: 保存更新后的对话历史到数据库
7. **返回结果**: 包含停止原因和 Token 使用统计

**关键特性**:

- **异步桥接**: ACP 是异步的，但 AIAgent 是同步的，通过 `ThreadPoolExecutor` 桥接
- **回调系统**: 实时推送工具执行、思考过程、最终响应到编辑器
- **错误隔离**: 异常捕获确保不会崩溃整个 ACP 服务

#### 模型切换方法

##### set_session_model

```python
async def set_session_model(model_id, session_id) -> SetSessionModelResponse
```

- 动态切换会话使用的 LLM 模型
- 重新创建 AIAgent 实例（保留历史）
- 支持 provider 自动检测

##### set_session_mode

```python
async def set_session_mode(mode_id, session_id) -> SetSessionModeResponse
```

- 持久化编辑器请求的模式（如 "edit", "ask"）
- Hermes 暂不使用模式概念，但需要响应以避免协议错误

##### set_config_option

```python
async def set_config_option(config_id, session_id, value) -> SetSessionConfigOptionResponse
```

- 接受通用配置选项更新
- 存储在 session 的 config_options 字典中

---

### 3. 斜杠命令系统

**文件**: `acp_adapter/server.py` (HermesACPAgent 类内部)

ACP 适配器实现了轻量级的斜杠命令处理器，无需调用 LLM 即可快速响应用户操作。

#### 支持的命令

| 命令 | 功能 | 实现方法 |
|------|------|----------|
| `/help` | 显示可用命令列表 | `_cmd_help()` |
| `/model [name]` | 查看或切换模型 | `_cmd_model()` |
| `/tools` | 列出可用工具 | `_cmd_tools()` |
| `/context` | 显示对话上下文统计 | `_cmd_context()` |
| `/reset` | 清空对话历史 | `_cmd_reset()` |
| `/compact` | 压缩对话上下文 | `_cmd_compact()` |
| `/version` | 显示 Hermes 版本 | `_cmd_version()` |

#### 命令广告机制

- **AvailableCommandsUpdate**: 会话创建时自动向编辑器发送支持的命令列表
- **UnstructuredCommandInput**: 为 `/model` 等需要参数的命令提供输入提示
- **容错设计**: 未知命令传递给 LLM 处理（用户可能输入 `/something` 作为普通文本）

---

### 4. 工具映射与事件系统

**文件**: `acp_adapter/tools.py`, `acp_adapter/events.py`

#### 工具类型映射 (ToolKind)

将 Hermes 工具名称映射到 ACP 的标准工具分类：

```python
TOOL_KIND_MAP = {
    # 文件操作
    "read_file": "read",
    "write_file": "edit",
    "patch": "edit",
    "search_files": "search",
    
    # 终端/执行
    "terminal": "execute",
    "process": "execute",
    "execute_code": "execute",
    
    # Web 访问
    "web_search": "fetch",
    "web_extract": "fetch",
    "browser_navigate": "fetch",
    
    # 浏览器自动化
    "browser_click": "execute",
    "browser_type": "execute",
    "browser_snapshot": "read",
    
    # Agent 内部
    "delegate_task": "execute",
    "vision_analyze": "read",
    "_thinking": "think",
}
```

#### 事件回调工厂

##### make_tool_progress_cb

监听 `tool.started` 事件，发送 `ToolCallStart` 到编辑器：

- 生成唯一的 `tool_call_id`
- 维护 FIFO 队列处理同名工具的并行调用
- 提取文件位置信息（path, line）
- 构建人类可读的工具标题

##### make_thinking_cb

流式推送模型的思考过程（reasoning content）：

- 调用 `acp.update_agent_thought_text(text)`
- 编辑器可显示为折叠的思考块

##### make_step_cb

在每次 API 调用完成后触发：

- 从队列中弹出对应的 `tool_call_id`
- 发送 `ToolCallProgress` (status="completed")
- 包含工具执行结果的截断预览

##### make_message_cb

流式推送最终响应文本：

- 增量更新编辑器的 agent message 区域
- 支持 Markdown 渲染

#### 跨线程通信

由于 AIAgent 在线程池中运行，而 ACP 连接在主事件循环上，所有回调使用：

```python
asyncio.run_coroutine_threadsafe(conn.session_update(...), loop).result(timeout=5)
```

确保线程安全的异步调用。

---

### 5. MCP 服务器集成

**文件**: `acp_adapter/server.py` (`_register_session_mcp_servers`)

#### 功能

允许编辑器通过 ACP 协议动态注册 MCP (Model Context Protocol) 服务器到 Hermes Agent。

#### 支持的服务器类型

1. **Stdio 服务器**:
   ```json
   {
     "command": "npx",
     "args": ["@modelcontextprotocol/server-example"],
     "env": {"API_KEY": "..."}
   }
   ```

2. **HTTP/SSE 服务器**:
   ```json
   {
     "url": "http://localhost:3000/sse",
     "headers": {"Authorization": "Bearer ..."}
   }
   ```

#### 注册流程

1. 解析 ACP 提供的服务器配置
2. 调用 `tools.mcp_tool.register_mcp_servers()` 注册到全局 MCP 客户端
3. 刷新工具定义：`get_tool_definitions(enabled_toolsets=["hermes-acp"])`
4. 更新 agent 的 `tools` 和 `valid_tool_names`
5. 使系统提示失效，触发下次对话时重新生成

---

### 6. 权限审批系统

**文件**: `acp_adapter/permissions.py`

#### 危险命令审批

当 Hermes Agent 尝试执行危险终端命令时：

1. `make_approval_callback()` 创建审批回调
2. 调用 `conn.request_permission()` 向编辑器发送权限请求
3. 编辑器显示确认对话框（允许/拒绝/始终允许）
4. 用户选择后，回调返回布尔值
5. terminal 工具根据返回值决定是否执行

#### 集成点

- 在 `prompt()` 方法中临时替换 `terminal_tool._approval_callback`
- 执行完成后恢复原始回调（避免影响其他会话）

---

## 启动与配置

### 启动方式

#### 1. 命令行直接启动

```bash
python -m acp_adapter
# 或
python -m acp_adapter.entry
```

#### 2. 通过 hermes CLI

```bash
hermes acp
```

#### 3. 作为独立可执行文件

```bash
hermes-acp
```

### 环境准备

**必需**:
- `~/.hermes/.env`: 包含 LLM provider 的 API 密钥
- 至少一个有效的 provider 配置（OpenRouter、Anthropic 等）

**可选**:
- `~/.hermes/config.yaml`: Hermes 配置文件（模型默认值、工具集等）

### 日志配置

- **日志目标**: stderr（stdout 专用于 ACP JSON-RPC 协议）
- **日志级别**: INFO（可通过修改 `_setup_logging()` 调整）
- **静默库**: httpx, httpcore, openai 设置为 WARNING

---

## ACP 注册表

**文件**: `acp_registry/agent.json`

```json
{
  "schema_version": 1,
  "name": "hermes-agent",
  "display_name": "Hermes Agent",
  "description": "AI agent by Nous Research with 90+ tools, persistent memory, and multi-platform support",
  "icon": "icon.svg",
  "distribution": {
    "type": "command",
    "command": "hermes",
    "args": ["acp"]
  }
}
```

### 作用

- **编辑器发现**: VS Code、Zed 等编辑器扫描此文件识别可用的 ACP Agent
- **启动配置**: 指定如何启动 Hermes Agent（`hermes acp` 命令）
- **元数据展示**: 在编辑器的 Agent 选择界面显示名称、描述、图标

---

## 技术细节

### 线程模型

```
主线程 (Async Event Loop)
├── ACP JSON-RPC 服务器
├── 接收客户端请求
├── 发送 session_update 通知
└── 管理 WebSocket/stdin 连接

线程池 (ThreadPoolExecutor, max_workers=4)
├── Worker 1: Session A 的 AIAgent.run_conversation()
├── Worker 2: Session B 的 AIAgent.run_conversation()
├── Worker 3: Session C 的 AIAgent.run_conversation()
└── Worker 4: 备用
```

**关键点**:
- ACP 协议层是异步的（asyncio）
- AIAgent 是同步的（阻塞式 LLM 调用）
- 通过线程池桥接，避免阻塞事件循环
- 最多同时处理 4 个并发会话

### 数据持久化

#### SessionDB 集成

- **数据库路径**: `~/.hermes/state.db`
- **表结构**:
  - `sessions`: 会话元数据（id, source, model, model_config）
  - `messages`: 对话消息（session_id, role, content, tool_calls）
  - FTS5 索引：支持全文搜索

#### 持久化时机

1. **会话创建**: `create_session()` 后立即写入
2. **提示完成**: `prompt()` 结束后保存完整历史
3. **斜杠命令**: `/reset`, `/compact`, `/model` 修改状态后保存
4. **工作目录更新**: `update_cwd()` 时同步

### 工具集限制

ACP 模式默认启用 `hermes-acp` 工具集，而非完整的 Hermes 工具集。

**原因**:
- 编辑器环境中某些工具不适用（如 Discord 集成）
- 减少模型混淆，聚焦开发相关工具
- 可通过 `enabled_toolsets` 参数自定义

**核心工具**:
- 文件操作: read_file, write_file, patch, search_files
- 终端执行: terminal, process
- Web 搜索: web_search, web_extract
- 浏览器自动化: browser_*
- 代码执行: execute_code
- Agent 委托: delegate_task

---

## 扩展开发

### 添加新的斜杠命令

1. 在 `HermesACPAgent._SLASH_COMMANDS` 中添加命令描述
2. 在 `_ADVERTISED_COMMANDS` 中添加广告配置
3. 实现处理方法 `_cmd_xxx(args, state)`
4. 在 `_handle_slash_command()` 的 dispatch 表中注册

### 自定义工具映射

修改 `acp_adapter/tools.py` 中的 `TOOL_KIND_MAP`：

```python
TOOL_KIND_MAP["my_custom_tool"] = "execute"  # 或其他 ToolKind
```

### 增强事件回调

在 `acp_adapter/events.py` 中添加新的回调工厂：

```python
def make_custom_cb(conn, session_id, loop):
    def _custom(data):
        update = acp.custom_update(...)
        _send_update(conn, session_id, loop, update)
    return _custom
```

然后在 `prompt()` 中绑定到 agent 的对应回调属性。

---

## 故障排查

### 常见问题

#### 1. 会话无法恢复

**症状**: 重启后 `load_session` 返回 None

**检查**:
- 确认 `~/.hermes/state.db` 存在且可写
- 验证会话的 `source` 字段为 `"acp"`
- 查看日志中是否有数据库错误

#### 2. 工具执行无响应

**症状**: 编辑器显示工具开始但从未完成

**检查**:
- 确认 `tool_call_ids` 队列正确维护
- 验证 `step_callback` 被正确调用
- 检查线程池是否饱和（max_workers=4）

#### 3. 日志污染 stdout

**症状**: ACP 客户端收到无效的 JSON-RPC 帧

**原因**: 某些库输出到 stdout

**解决**: 
- 确保 `_acp_stderr_print` 被正确设置
- 检查第三方库的日志配置
- 所有 print 语句应重定向到 stderr

#### 4. MCP 服务器注册失败

**症状**: 编辑器注册的 MCP 工具不可用

**检查**:
- 验证 MCP 服务器命令可执行
- 确认 `tools.mcp_tool` 模块导入成功
- 查看 `_register_session_mcp_servers` 的异常日志

---

## 最佳实践

### 1. 会话生命周期管理

- **及时清理**: 长时间不用的会话应手动删除，避免数据库膨胀
- **合理 Fork**: 分支实验时使用 `fork_session`，而非创建多个独立会话
- **CWD 一致性**: 确保 `cwd` 反映实际项目根目录，影响文件操作和终端命令

### 2. 性能优化

- **上下文压缩**: 定期使用 `/compact` 减少历史长度
- **模型选择**: 简单任务使用轻量模型，复杂推理使用强大模型
- **工具缓存**: MCP 服务器注册后会被缓存，避免重复注册

### 3. 安全性

- **权限审批**: 始终保持 `request_permission` 回调启用
- **环境隔离**: 不同项目使用不同的 `cwd`，避免跨项目文件访问
- **API 密钥保护**: `.env` 文件不应提交到版本控制

---

## 与编辑器集成示例

### VS Code 配置

在 VS Code 的 ACP 扩展设置中：

```json
{
  "acp.agents": [
    {
      "name": "hermes-agent",
      "command": "hermes",
      "args": ["acp"],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

### Zed 配置

在 `~/.config/zed/settings.json` 中：

```json
{
  "context_servers": [
    {
      "command": {
        "path": "hermes",
        "args": ["acp"]
      }
    }
  ]
}
```

---

## 未来改进方向

1. **流式工具输出**: 支持长运行工具（如 terminal）的实时输出流
2. **多模态支持**: 完善 ImageContentBlock 和 AudioContentBlock 的处理
3. **会话标签**: 允许用户为会话添加标签和描述，便于管理
4. **批量操作**: 支持同时对多个会话执行命令
5. **性能监控**: 暴露 Token 使用、响应时间等指标

---

## 参考资源

- **ACP 规范**: https://github.com/agent-client-protocol/agent-client-protocol
- **Hermes Agent 文档**: 项目根目录 README.md
- **MCP 协议**: https://modelcontextprotocol.io/
- **SessionDB 实现**: `hermes_state.py`

---

**最后更新**: 2026-04-12  
**维护者**: Hermes Agent Team
