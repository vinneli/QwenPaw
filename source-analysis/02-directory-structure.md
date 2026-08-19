# 02 · 目录结构

本文档说明 `src/qwenpaw/` 顶层包与关键子目录的职责，便于按图索骥。

---

## 顶层一览

| 路径 | 职责 |
|------|------|
| `__main__.py` | `python -m qwenpaw` → 委托 `cli.main.cli` |
| `__init__.py` | 包初始化：加载持久化 env、配置日志 |
| `__version__.py` | 版本号 |
| `api_action.py` | `@api_action`：HTTP 路由 + slash 命令自动注册 |
| `constant.py` | 全局常量、工作目录、环境变量名 |
| `exceptions.py` | 统一异常层次 |
| `schemas.py` | `AgentRequest` / `AgentResponse` / Content / SSE 契约 |
| `_compat/` | 跨版本兼容垫片 |
| `agents/` | Agent 本体、工具、技能、记忆、上下文、ACP |
| `app/` | FastAPI 应用、Workspace、通道、路由、Cron、审批 |
| `backup/` | 备份与恢复 |
| `cli/` | Click CLI + Textual TUI |
| `config/` | Pydantic 配置模型与读写缓存 |
| `drivers/` | 外部能力驱动（MCP 等）、凭证、策略 |
| `envs/` | 持久化环境变量 |
| `governance/` | 资源治理、策略、`PolicyGuardedTool` |
| `hooks/` | 内置生命周期 Hook 实现 |
| `local_models/` | 本地模型管理（Ollama / LM Studio 等） |
| `loop/` | ReAct 停止门控（Iteration / DoomLoop / Rubric…） |
| `market/` | 插件/技能市场相关 |
| `modes/` | Coding / Goal / Mission 模式包 |
| `observability/` | Langfuse 等可观测性 |
| `plugins/` | 插件发现、加载、注册、运行时 API |
| `providers/` | LLM Provider 抽象与管理器 |
| `runtime/` | 请求编排核心（Runtime / Builder / Executor） |
| `sandbox/` | 跨平台命令沙箱 |
| `security/` | Tool Guard、Skill Scanner、密钥存储 |
| `services/` | 通用服务（如 workspace_manager） |
| `tauri/` | 桌面端 Tauri 集成 |
| `token_usage/` | Token 用量统计 |
| `tokenizer/` | Token 计数 |
| `tool_calls/` | 工具调用协调、超时、offload、限流 |
| `tunnel/` | 内网穿透 |
| `utils/` | 日志、HTTP、端口等工具 |
| `agent_stats/` | Agent 运行统计 |

---

## `agents/` — Agent 核心

```
agents/
├── react_agent.py          # QwenPawAgent（主类）
├── model_factory.py        # 创建 ChatModel + Formatter
├── routing_chat_model.py   # 多 slot 模型路由
├── prompt_builder.py       # 系统提示词组装
├── prompt.py / templates.py
├── middlewares.py          # MemoryMiddleware 等
├── offloader.py            # 上下文/工具结果归档
├── command_handler.py
├── tools/                  # 内置工具（file/shell/browser/search…）
├── skills/                 # 内置 Skill 包（docx/pptx/pdf…）
├── skill_system/           # Skill 注册、池、工作区服务、远程安装
├── memory/                 # ReMe / ADBPG / Noop 等记忆后端
├── context/                # ContextManager；scroll 策略
├── acp/                    # Agent Communication Protocol
├── md_files/               # 人格/引导 Markdown 模板
└── utils/                  # 消息处理、文件处理等
```

要点：`QwenPawAgent` **不自行组装依赖**，构造参数由 `runtime/builder.py` 注入。

---

## `app/` — HTTP 服务与 Workspace

```
app/
├── _app.py                 # FastAPI 应用与 lifespan
├── workspace/              # Workspace、LocalWorkspace、插件容器
├── workspace_registry.py   # 多 Agent 注册表
├── multi_agent_manager.py  # 兼容/路由层（DynamicMultiAgentRunner）
├── channels/               # 18+ IM/语音通道
├── routers/                # REST API
├── approvals/              # 人工审批服务
├── crons/                  # 定时任务
├── chats/                  # 会话相关
├── agent_context.py        # ContextVar（agent_id / session_id）
└── migration.py            # 遗留配置迁移
```

`Workspace.stream_query()` 是对旧 `Runner.stream_query` 的替换：内部创建 `Runtime` 并 `run(request)`。

---

## `runtime/` — 请求编排

```
runtime/
├── runtime.py              # Runtime.run() — 8 阶段
├── phases.py               # Phase 枚举
├── hooks.py                # HookContext / HookResult
├── builder.py              # AgentBuilder
├── executor.py             # AgentExecutor（reply_stream + 心跳）
├── envelope.py             # SSE 状态机
├── tool_registry.py        # 工具注册与过滤
├── slash_command_registry.py
├── prompt_manager.py
└── builtin_commands.py
```

---

## `loop/` 与 `modes/`

```
loop/
├── react_gates.py          # 注册默认 Iteration/DoomLoop/Rubric Gate
└── gates/
    ├── loop_gate.py        # LoopGate 基类（session 隔离）
    ├── handler.py          # StopHandler
    ├── runner.py           # run_stop_handlers + scope 过滤
    ├── doom_loop.py
    └── ...

modes/
├── base.py                 # AgentMode / ModeGatedHook
├── coding/                 # CodingMode + CodingModeMixin
├── goal/                   # GoalMode + Goal*Gate
└── mission/                # MissionMode + MissionGate
```

---

## `providers/` / `drivers/` / `security/` / `sandbox/`

| 包 | 关键文件 | 说明 |
|----|----------|------|
| `providers/` | `provider.py`, `provider_manager.py`, `retry_chat_model.py` | LLM 抽象、重试、限流、能力探测 |
| `drivers/` | `manager.py`, `handler.py`, `handlers/`, `adapters/` | MCP 等外部能力统一入口 |
| `governance/` | `resource_governor.py`, `tool_adapter.py` | 策略评估与工具包装 |
| `security/` | `tool_guard/`, `skill_scanner/`, `secret_store.py` | 预检、技能扫描、密钥加密 |
| `sandbox/` | `macos_sandbox.py`, `bubblewrap_sandbox.py`, … | 按平台隔离执行 |

---

## `cli/` — 命令行与 TUI

- `cli/main.py`：`LazyGroup` 懒加载子命令；无子命令时启动 TUI
- `cli/app_cmd.py`：`uvicorn.run("qwenpaw.app._app:app", workers=1)`
- `cli/tui/`：基于 Textual 的全屏终端界面
- 其他：`init` / `doctor` / `agents` / `skills` / `plugin` / `daemon` 等

---

## `config/` — 配置体系

源码级拆解见 [11-Config 模块](./11-config-module.md)。

| 文件 | 作用 |
|------|------|
| 全局 `config.json`（`WORKING_DIR`） | channels、agents 列表、security、plugins、mcp |
| `{workspace}/agent.json` | 单 Agent：model、skills、running、approval_level |
| `governance/{ws_hash}/policy.yaml` | 治理策略（存 workspace 外，防 Agent 自改） |
| `config/config.py` | Pydantic 模型 + `load_agent_config` |
| `config/utils.py` | 根配置 mtime 缓存的 load/save |
| `config/context.py` | 工具可见的请求期 ContextVar |
| `config/timezone.py` | IANA 时区探测（避免循环导入） |

---

## `plugins/` — 插件系统

```
plugins/
├── architecture.py   # PluginType: TOOL|PROVIDER|HOOK|COMMAND|CHANNEL|…
├── loader.py         # 扫描、import、依赖安装
├── registry.py       # 全局插件注册
├── api.py            # PluginApi 注入点
└── runtime.py        # 插件运行时目录
```

插件目录通常位于 `WORKING_DIR/plugins/`，清单为 `plugin.json`。

---

## 仓库其他相关目录（非本分析主体）

| 路径 | 说明 |
|------|------|
| `console/` | Web 控制台（React + Tauri） |
| `website/` | 官网与用户文档 |
| `tests/` | 单元/集成测试 |
| `e2e/` | 端到端测试 |
| `plugins/`（仓库根） | 官方捆绑插件样例 |

下一篇：[启动与运行时](./03-startup-and-runtime.md)
