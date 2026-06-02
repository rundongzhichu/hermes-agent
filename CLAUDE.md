# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Hermes Agent** — the self-improving AI agent by Nous Research. Python 3.11+ monolith with a TypeScript TUI, messaging gateway, plugin system, and cron scheduler. See `AGENTS.md` for exhaustive architecture documentation; this file is the condensed field manual.

## Development Setup

```bash
source .venv/bin/activate   # or: source venv/bin/activate
./setup-hermes.sh           # first-time: creates venv, installs .[all,dev], symlinks hermes
uv pip install -e ".[all,dev]"  # manual path
npm install                 # optional: browser tools
```

## Build, Lint, Test

```bash
# Testing — ALWAYS use the wrapper, not bare pytest
scripts/run_tests.sh                                    # full suite (per-file isolation)
scripts/run_tests.sh tests/gateway/                     # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x    # single test
scripts/run_tests.sh --no-isolate tests/foo/            # faster, for debugging (skip subprocess isolation)
scripts/run_tests.sh -v --tb=long                       # pass-through pytest flags

# Linting
ruff check .                                            # only PLW1514 (unspecified-encoding) is enforced

# TUI development
cd ui-tui && npm run dev                                # watch mode
cd ui-tui && npm run build && npm test                  # full build + vitest
```

## High-Level Architecture

### Entry Points

| Entry point | Module | Purpose |
|-------------|--------|---------|
| `hermes` CLI | `hermes_cli/main.py` → `cli.py` (`HermesCLI`) | Interactive terminal (prompt_toolkit + Rich) |
| `hermes --tui` | `tui_gateway/server.py` + `ui-tui/src/` | Ink/React TUI over JSON-RPC stdio |
| `hermes gateway` | `gateway/run.py` | Multi-platform messaging (Telegram, Discord, Slack, etc.) |
| `hermes-agent` | `run_agent.py:main()` | Headless agent entry point |
| `hermes-acp` | `acp_adapter/entry.py` | VS Code / Zed / JetBrains ACP integration |

### Core Loop (`run_agent.py` — `AIAgent` class, ~12k LOC)

Synchronous `run_conversation()` loop: iterate API calls → dispatch tool calls via `model_tools.py:handle_function_call()` → return tool results → repeat. Messages follow OpenAI format with reasoning stored in `assistant_msg["reasoning"]`. Respects `max_iterations`, `iteration_budget`, interrupt checks, and prompt-cache preservation.

### Tool System (`tools/` + `model_tools.py` + `toolsets.py`)

```
tools/registry.py         # Registry singleton — imported by all tool files
tools/*.py                # Each calls registry.register() at import time
model_tools.py            # discover_builtin_tools(), handle_function_call(), get_tool_definitions()
toolsets.py               # TOOLSETS dict + _HERMES_CORE_TOOLS — wiring tools into named bundles
```

Tools auto-discover via importing `tools/*.py` files with `registry.register()` calls. But a tool is only exposed to agents if its name appears in a toolset in `toolsets.py`. All handlers must return JSON strings. Tool schema descriptions must not hardcode cross-tool references (tools may be unavailable).

### Slash Command Registry (`hermes_cli/commands.py`)

Central `COMMAND_REGISTRY` list of `CommandDef` objects. CLI, gateway, Telegram bot menu, Slack subcommands, help text, and autocomplete all derive from this single registry. Adding a command = one `CommandDef` entry + one handler in the dispatcher.

### Plugin System (`plugins/`)

Two surfaces: **general plugins** (lifecycle hooks, tools, CLI subcommands) discovered by `PluginManager` in `hermes_cli/plugins.py`; **memory-provider plugins** (`plugins/memory/<name>/`) implementing the `MemoryProvider` ABC, orchestrated by `agent/memory_manager.py`. **No new in-tree memory providers** — new ones ship as standalone repos installed into `~/.hermes/plugins/`.

### Gateway (`gateway/`)

`run.py` + `session.py` + `platforms/<adapter>.py`. Each platform adapter handles connect/disconnect, message routing, and platform-specific commands. Two sequential message guards: base adapter queues during active sessions, gateway runner intercepts control commands before interrupt.

### Other Key Modules

| Module | Purpose |
|--------|---------|
| `hermes_state.py` | SQLite session store with FTS5 search |
| `hermes_constants.py` | `get_hermes_home()`, `display_hermes_home()` — profile-aware paths |
| `hermes_logging.py` | Profile-aware logging (agent.log, errors.log, gateway.log) |
| `batch_runner.py` | Parallel batch trajectory generation |
| `trajectory_compressor.py` | Compress trajectories for training |
| `cron/` | Scheduler — `jobs.py` (store) + `scheduler.py` (tick loop) |
| `agent/` | Provider adapters, memory, caching, compression, curator, skills |

## Critical Conventions

### NEVER break prompt caching
Do not alter past context, change toolsets, or rebuild system prompts mid-conversation. State-mutating slash commands use deferred invalidation (takes effect next session) with an opt-in `--now` flag.

### NEVER hardcode `~/.hermes` paths
Use `get_hermes_home()` for code paths, `display_hermes_home()` for user-facing messages. Hardcoding breaks profiles (multi-instance support via `HERMES_HOME`).

### Dependency pinning
All deps must have upper bounds (`>=floor,<next_major`). Core deps use exact `==X.Y.Z` pins. Run `uv lock` after changes. Never commit bare `>=X.Y.Z` without a ceiling.

### Config: secrets in .env, settings in config.yaml
API keys → `.env` (`OPTIONAL_ENV_VARS` in `hermes_cli/config.py`). Settings → `config.yaml` (`DEFAULT_CONFIG` in `hermes_cli/config.py`). Three config loaders exist: `load_cli_config()` (CLI mode), `load_config()` (subcommands), direct YAML load (gateway). Know which path you're in.

### Skills vs Tools
New capabilities should almost always be **skills** (instructions + shell commands). Make it a **tool** only when it needs custom Python integration, API key management, or binary/streaming data handling. Bundled skills in `skills/`; heavier/niche skills in `optional-skills/`.

### Tests: no change-detectors
Tests that snapshot current data (model names, config version numbers, enumeration counts) will be rejected. Write invariants (relationships, contracts), not snapshots.

### Gateway message guards
Control commands (`/stop`, `/new`, `/approve`, `/deny`, etc.) must bypass BOTH the base adapter queue guard AND the gateway runner guard to reach the runner during active agent sessions.

## Key Config Sections
`model`, `agent`, `terminal`, `compression`, `display`, `stt`, `tts`, `memory`, `security`, `delegation`, `smart_model_routing`, `checkpoints`, `auxiliary`, `curator`, `skills`, `gateway`, `logging`, `cron`, `profiles`, `plugins`, `honcho`.
