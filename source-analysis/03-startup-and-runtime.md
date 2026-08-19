# 03 · 启动与运行时

本文说明进程如何启动，以及单次请求如何进入 `Runtime` 的 8 阶段编排。

---

## 1. 进程入口

```
python -m qwenpaw
  → qwenpaw/__main__.py
    → cli.main.cli()
```

包导入时（`__init__.py`）：

1. 尝试 `envs.load_envs_into_environ()`（持久化环境变量）
2. `setup_logger(QWENPAW_LOG_LEVEL)`

### CLI 行为摘要（`cli/main.py`）

| 调用 | 行为 |
|------|------|
| `qwenpaw`（无子命令） | 启动 TUI |
| `qwenpaw app` | 启动 FastAPI / uvicorn |
| `qwenpaw .` / 路径参数 | 视为项目目录进入 TUI |
| 其他子命令 | `init` / `doctor` / `agents` / `skills` / `plugin` / `daemon` … |

`cli/app_cmd.py` 固定：

```python
uvicorn.run("qwenpaw.app._app:app", host=..., port=..., workers=1)
```

单 worker 保证进程内状态（Workspace、通道队列、审批）一致。

---

## 2. FastAPI 启动两阶段（`app/_app.py`）

### 阶段 A — 同步快速启动（目标 <100ms）

1. 备份清理、认证中间件、遥测
2. 遗留配置迁移（旧 workspace → default agent 等）
3. 创建 `ProviderManager`、`LocalModelManager` 单例
4. 创建 `AppServiceManager` + `WorkspaceRegistry`
5. 收集 bootstrap 材料：
   - 内置工具
   - `@api_action` 路由与 slash 命令
   - 生命周期 Hook
   - Prompt Contributor
   - Modes：`CodingMode` / `MissionMode` / `GoalMode`
6. 暴露 `app.state.multi_agent_manager`（WorkspaceRegistry）
7. 注册 `DynamicMultiAgentRunner` 作为全局 runner

### 阶段 B — 后台异步初始化

1. 加载插件（channel 类插件优先）
2. 各 Agent `Workspace.start()`（通道、记忆、Driver、Cron）
3. 启动通道消费者循环

---

## 3. 请求如何落到 Runtime

统一收敛路径：

```
HTTP / WebSocket / TUI / Channel / Cron / ACP
  → DynamicMultiAgentRunner.stream_query(request)
    → WorkspaceRegistry.get_agent(agent_id)   # ContextVar 或 X-Agent-Id
      → Workspace.stream_query(request)
        → Runtime(workspace, app_services).run(request)
```

`Workspace.stream_query` 伪代码：

```python
async def stream_query(self, request, ...):
    from ...runtime import Runtime
    rt = Runtime(workspace=self, app_services=self._app_services)
    async for item in rt.run(request):
        yield item
```

---

## 4. Runtime 八阶段

定义见 `runtime/phases.py`：

```
PRE_DISPATCH
  → [固定] Slash 命令分发
POST_DISPATCH
PRE_AGENT_BUILD
  → [固定] AgentBuilder.build()
POST_AGENT_BUILD
PRE_EXECUTE
  → [固定] AgentExecutor.run()  # reply_stream
POST_RESPONSE
ON_ERROR          # 异常/取消
FINALLY           # 幂等清理
```

### 各阶段典型职责

| 阶段 | 典型 Hook / 动作 |
|------|------------------|
| `PRE_DISPATCH` | ContextVar 设置、媒体预处理；可 `SHORT_CIRCUIT` |
| Slash 分发 | `/stop`、`/goal`、内置与插件命令；命中则跳过 Agent |
| `POST_DISPATCH` | 未命中命令后的后续处理 |
| `PRE_AGENT_BUILD` | `SessionLoadHook` 恢复会话状态 |
| `AgentBuilder.build` | 装 model、toolkit、prompt、middleware、governor、gates |
| `POST_AGENT_BUILD` | Mode 上下文注入 |
| `PRE_EXECUTE` | Bootstrap、Skill 环境变量、prompt 刷新 |
| `AgentExecutor.run` | `agent.reply_stream()` + 心跳 + SSE Envelope |
| `POST_RESPONSE` | `SessionSaveHook`、Cron 回写 |
| `ON_ERROR` | 错误归一化、取消时保全部分响应 |
| `FINALLY` | `agent.close()`、清理 ContextVar、关闭临时资源 |

### Hook 动作（`HookAction`）

| 动作 | 含义 |
|------|------|
| `CONTINUE` | 继续流水线 |
| `SHORT_CIRCUIT` | 直接以给定 Msg 结束并产出 SSE |
| `SKIP_AGENT` | 跳过 build/execute，仍跑后续 POST/FINALLY |

---

## 5. AgentBuilder / AgentExecutor

### AgentBuilder（`runtime/builder.py`）

每请求组装一次（或按策略复用会话中的 agent 状态），主要步骤：

1. `load_agent_config(agent_id)`
2. `ProviderManager` → `create_model_and_formatter`
3. `ToolRegistry.filter(...)` → `PolicyGuardedTool` 包装
4. 附加：Coding 工具、Driver 工具、Memory 工具、Scroll recall 工具
5. `PromptManager` 组装 system prompt
6. 创建 `QwenPawAgent` + MemoryManager + ContextManager + Governor
7. 挂载 middlewares（ToolCoordinator、结果裁剪、Memory、Langfuse…）

### AgentExecutor（`runtime/executor.py`）

- 驱动 `agent.reply_stream(input_msgs)`
- 心跳保活（长工具调用期间）
- 将 AgentScope 事件翻译为 QwenPaw `Envelope` / SSE 对象

---

## 6. Envelope 与 schemas

`schemas.py` 定义对外流式契约（与 AgentScope 内部事件解耦）：

- `AgentRequest` / `AgentResponse`
- `Message` / `Content`（text/image/audio/video/file/data/refusal）
- `MessageType`：`message`、`function_call`、`mcp_tool_call`、`progress`…
- `RunStatus`：`created` / `in_progress` / `completed` / `failed` / `cancelled`

`runtime/envelope.py` 负责把内部事件收成稳定的 SSE 状态机，供 Console、通道 Renderer、TUI 统一消费。

---

## 7. 取消与错误路径

Runtime 对 `CancelledError` 有专门处理：

- 尽量注入 partial response，避免前端“空响应”
- 使用 `asyncio.shield` 保护 session save（`_try_save_on_cancel`）
- `ON_ERROR` + `FINALLY` 仍会执行，保证资源释放

下一篇：[Agent 执行引擎](./04-agent-engine.md)
