# 04 · Agent 执行引擎

本文聚焦 `QwenPawAgent` 及其协作子系统：Tools、Skills、Memory、Context、ToolCoordinator。

---

## 1. QwenPawAgent

**路径：** `agents/react_agent.py`  
**继承：** `CodingModeMixin` + AgentScope 2.0 `Agent`

### 设计约束

- 构造函数接收已组装好的 `model` / `system_prompt` / `toolkit` / `middlewares` / `governor` 等
- **不在 Agent 内部**创建 Provider 或扫描技能目录（由 `AgentBuilder` 完成）
- 绕过 AgentScope 内置权限引擎，改用 `PolicyGuardedTool`：

```python
from agentscope.permission import PermissionMode
self.state.permission_context.mode = PermissionMode.BYPASS
```

### 关键覆写点

| 方法 | 作用 |
|------|------|
| `_reply` | 注入 ToolCoordinator 后台完成 hint，再进入父类 reply |
| `_reasoning` | 多模态预处理 → LLM → StopHandler（Gate）介入 |
| `compress_context` | 委托 `ContextManager` 或原生压缩 |
| `_save_to_context` | Scroll 写穿等 |
| `close` | 关闭 governor / scroll / offloader |

### `_reasoning` 一轮伪代码

```python
async def _reasoning(...):
    # 1) 消费上轮 deferred gate 决策
    pending = check_pending_gates(self)
    if pending.TERMINATE:
        yield stop_text; return

    # 2) 按模型能力剥离不支持的 media blocks
    maybe_strip_media()

    # 3) 父类推理（可能产出 tool_call）
    final_msg = None
    async for evt in super()._reasoning(...):
        ...

    # 4) 每轮跑 StopHandler
    stop_result = await run_stop_handlers(...)

    if final_msg is None:  # 本轮是 tool-call
        defer_stop_result(...)  # 延迟到下一轮开头
        return

    if stop_result.INTERRUPT_AND_CONTINUE:
        context.append(user_continuation)
        return  # 继续外层 ReAct，不结束

    yield final_msg
```

**Deferred Gate：** tool-call 轮次的 TERMINATE/CONTINUE 挂到 `agent._gate_pending_*`，避免 tool result 尚未写入 context 就中断。

---

## 2. 工具系统

### 内置工具（`agents/tools/`）

常见能力：

- 文件：`read_file` / `write_file` / `edit_file`
- Shell：`execute_shell_command`
- 搜索：`grep_search` / `glob_search`
- 浏览器：`browser_use` / `browser_snapshot`
- 网络：`web_search` / `web_fetch`
- Agent：`spawn_subagent` / `delegate_external_agent`
- 媒体：`view_media`
- Coding：`lsp` / `ast_search`（Coding Mode）

通过 `@tool_descriptor` 注册到 `ToolRegistry`，再按 mode/skill/feature 过滤。

### 装配与执行链

```
ToolDescriptor
  → ToolRegistry.filter(active_modes, skills, features, allow/deny)
    → PolicyGuardedTool（governance）
      → ToolGuardEngine.guard()（安全预检）
        → ToolCoordinatorMiddleware.on_acting
          → 实际函数 或 Sandbox.execute
```

### ToolCoordinator（`tool_calls/`）

统一管理：

- 并发与会话级协调
- 超时 → 后台 offload → 下轮 `_reply` 注入 hint
- 结果限流（`_result_limiter`）
- Graceful cancel

假设运行在单 asyncio loop；文档要求勿从 `asyncio.to_thread` 无同步地回呼 coordinator。

---

## 3. Skills

### 两层存储

| 层 | 说明 |
|----|------|
| Skill Pool | 全局只读共享库（`skill_system/pool_service.py`） |
| Workspace Skills | 单 Agent 可启用/安装的技能（`workspace_service.py`） |

内置技能包在 `agents/skills/`（如 `docx-zh`、`pdf-en`、`cron-zh`），含 `SKILL.md` 与脚本。

### 注入双通道

1. **LLM 可读：** `Toolkit(skills_or_loaders=[...])` 加载 SKILL.md
2. **运行时元数据：** `toolkit._qp_skills[name] = {dir}`，供 `/skill_name` slash 使用
3. **工具门控：** `ToolRegistry` 的 `requires_skills` 过滤

远程安装：`skill_system/hub.py`（skills.sh / GitHub / ModelScope 等）。

---

## 4. Memory

**抽象：** `agents/memory/base_memory_manager.py` → `BaseMemoryManager`

| 实现 | 用途 |
|------|------|
| `ReMeLightMemoryManager` | ReMe 轻量长期记忆 |
| `ADBPGMemoryManager` | 向量/ADBPG 后端 |
| `NoopMemoryManager` | 关闭记忆 |

### 协作双轨

1. **工具轨：** `list_memory_tools()` → 进入 toolkit（如检索工具）
2. **Middleware 轨：** `MemoryMiddleware`
   - `on_system_prompt`：注入记忆指引
   - `on_model_call`：回复前自动检索
   - `post_reply`：周期性提取写入

另有 `memory/proactive/`：主动触发检索/写入循环。

---

## 5. Context（Scroll）

**协议：** `ContextManager` — `compress(agent)` + `on_save(agent, blocks)`

| 策略 | 行为 |
|------|------|
| **native** | AgentScope 原生压缩 + `QwenPawOffloader` 归档 |
| **scroll** | 每轮写穿 `history.db`；超阈值驱逐中间段并建索引；按需 `recall_history` |

设计目标：**不摘要压缩、不丢信息**——滚出窗口的轮次可召回。

Scroll 组件（`agents/context/scroll/`）：

- `HistoryStore`（SQLite）
- `EvictionIndex`
- Recall 工具（可能要求沙箱或显式允许 unsandboxed recall）
- `ToolResultCapMiddleware`：大工具输出落库

`agents/offloader.py` 同时服务原生压缩的 `offload_context` 与工具结果裁剪。

---

## 6. Middleware 洋葱（Agent 内）

由 `AgentBuilder._build_middlewares` 组装，典型顺序：

```
ToolCoordinatorMiddleware
  → ToolResultPruningMiddleware
    → MemoryMiddleware
      → LangfuseToolSpanMiddleware（可选）
        → 插件 middleware
```

与 Runtime Hook 的边界：

| | Runtime Hook | Agent Middleware |
|--|--------------|------------------|
| 粒度 | 整次 HTTP/SSE 请求 | 单次 reply 循环 |
| 阶段 | 8 Phase | `on_acting` / `on_model_call` / `post_reply`… |

---

## 7. ACP 智能体

`agents/acp/` 提供 `QwenPawACPAgent`：通过完整 Workspace 生命周期暴露与 Console 相同的能力（MCP、memory、sub-agent）。Coding 项目目录可经 ACP meta 传入，触发 Builder 动态启用 Coding Mode。

下一篇：[Loop 与 Modes](./05-loop-and-modes.md)
