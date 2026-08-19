# 08 · 扩展机制

QwenPaw 的扩展面分为四层：**Runtime Hooks**、**Agent Middleware**、**Modes**、**Plugins**（外加 `@api_action` 与 Drivers）。

---

## 1. Runtime Hooks

**注册表：** 每 Workspace 一份 `HookRegistry`（`runtime/hooks.py`）  
**阶段：** 见 [03-启动与运行时](./03-startup-and-runtime.md)

### 内置 Hook 包（`hooks/`）

| 包 | 示例 |
|----|------|
| `session/` | SessionLoad / SessionSave |
| `bootstrap/` | 首次引导 |
| `skill_env/` | Skill 环境变量推入/清理 |
| `cron/` | Cron 上下文与记忆隔离/恢复 |
| `request_setup/` | ContextVars、媒体预处理 |
| `error/` | 错误归一化、取消清理 |
| `observability/` | Langfuse 等 |

自定义 Hook 实现 `HookBase`，按 `Phase` 注册，返回 `HookResult(action=CONTINUE|SHORT_CIRCUIT|SKIP_AGENT)`。

---

## 2. Agent Middleware

挂在单次 `reply_stream` 上（AgentScope middleware 协议），由 `AgentBuilder` 组装。适合：

- 工具执行包装（ToolCoordinator）
- 记忆自动检索/写入
- 工具结果裁剪
- 链路追踪 span

**不要**把「整次请求的 session load」放进 middleware——那属于 Runtime Hook。

---

## 3. Modes（能力包）

见 [05-Loop 与 Modes](./05-loop-and-modes.md)。一次 `setup(workspace)` 注册：

- Slash 命令
- 工具（可 `requires_modes`）
- Hook（建议包在 `ModeGatedHook` 内）
- Prompt Contributor

这是**一等公民**的产品功能扩展方式（Coding / Goal / Mission）。

---

## 4. Plugins

**路径：** `plugins/`

### 类型（`architecture.py`）

`TOOL | PROVIDER | HOOK | COMMAND | CHANNEL | FRONTEND | GENERAL`

### 加载流程（`loader.py`）

1. 扫描 `WORKING_DIR/plugins/`
2. 解析 `plugin.json` → `PluginManifest`
3. 动态 import `entry.backend`
4. 通过 `PluginApi` 写入 `PluginRegistry`
5. 依赖安装到 `plugin_runtime/`（桌面 frozen 构建有特殊处理）

### 注册时机

- **Channel 插件：** 后台启动阶段尽早加载
- **其余：** Workspace bootstrap 时应用到各 Agent 的 registry

仓库根目录 `plugins/` 含官方捆绑样例（如 `qwenpaw-pet`、channel 插件）。

---

## 5. `@api_action`（`api_action.py`）

装饰器同时生成：

- FastAPI 路由（进入 `app/_api_action_routes.py` 收集）
- 可选 slash 命令

适合把「控制面操作」暴露给 HTTP 与对话命令，保持单一实现。

---

## 6. Workspace 插件容器

`app/workspace/workspace_plugins.py` 中每个 Workspace 持有：

| Registry | 内容 |
|----------|------|
| `SlashCommandRegistry` | 命令 |
| `HookRegistry` | Runtime Hook |
| `ToolRegistry` | 工具描述符 |
| `PromptManager` | 提示词片段 |
| `modes` | `AgentMode` 列表 |
| `stop_handlers` | Loop Gate 注册 |

外部插件与内置 Modes 最终都汇入这些注册表。

---

## 7. Drivers 作为扩展

新增外部协议时：

1. 实现 `DriverHandler`
2. `DriverManager.register_handler_type(...)`
3. 通过 adapter 暴露为 Agent 工具

无需改 ReAct 核心循环。

---

## 8. 扩展选型指南

| 需求 | 推荐扩展点 |
|------|------------|
| 改请求生命周期（load/save、短路） | Runtime Hook |
| 改单次工具调用/模型调用 | Agent Middleware |
| 一整套产品模式（命令+工具+gate） | AgentMode |
| 第三方分发安装 | Plugin |
| 对接新 IM | Channel 插件 |
| 对接新 LLM 厂商 | Provider 插件或内置 Provider |
| 对接 MCP/外部 API | Driver |

下一篇：[数据流](./09-data-flow.md)
