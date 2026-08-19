# 05 · Loop 与 Modes

Loop Engineering 把「何时停止 / 何时继续」从 ReAct 硬编码中抽离为可组合的 **StopGate**；**Modes** 则以包的形式注入命令、工具、Hook 与 Prompt。

---

## 1. Gate 类层次

```
StopGate (ABC)
 └── LoopGate          # 按 session_id 隔离 _sessions
      ├── IterationGate
      ├── DoomLoopGate
      ├── BudgetGate
      ├── FileLoopGate
      ├── StandaloneRubricGate
      ├── GoalTurnGate / GoalBudgetGate / RubricGate
      └── MissionGate
```

**路径：** `loop/gates/`

### StopAction 语义

| 动作 | 含义 |
|------|------|
| `BYPASS` | 本 Gate 不干预 |
| `TERMINATE` | 结束 ReAct 循环 |
| `INTERRUPT_AND_CONTINUE` | 注入 continuation 用户消息，强制再跑一轮 |

`StopHandler`（`loop/gates/handler.py`）按 priority 组合多个 Gate；`run_stop_handlers`（`runner.py`）在 `_reasoning` 末尾调用。

---

## 2. 默认 ReAct Gates

`loop/react_gates.py::register_react_gates` 在无特殊 Mode 时也生效：

| Gate | 典型 priority | 作用 |
|------|---------------|------|
| `IterationGate` | 10 | 硬迭代上限（`AgentsRunningConfig.loop.iteration`） |
| `DoomLoopGate` | 5 | 滑动窗口内重复 tool_call 签名检测 |
| `StandaloneRubricGate` | 90 | 纯文本无 tool call 时可能注入 continuation，防过早停止 |

每轮用户 turn 入口会 `_reset_gates_for_new_turn`。

### DoomLoopGate 要点（`doom_loop.py`）

- 签名：`tool_name:args_hash[:8]`
- 相似度随窗口内重复升高
- 分 stage：警告（`modify_prompt`）或停止（`stop`）

---

## 3. Scope 过滤（关键）

`runner.py::_filter_by_scope`：

1. 若存在**已激活**的非 default scope handler（如 `mission`），则**只跑该 scope**
2. `scope="default"` 的默认 ReAct gates 在此时被跳过
3. `scope=""` 的 universal handler（如 Goal 注册的 gates）**始终运行**

因此：

- **Goal** 与默认 gates **可叠加**（Goal 用 universal）
- **Mission** 与默认 gates **互斥**（Mission 用独立 scope）

---

## 4. AgentMode 基类

**路径：** `modes/base.py`

```python
class AgentMode:
    name: str
    def setup(workspace) -> None      # 注册到四个 registry
    def commands() -> list[CommandSpec]
    def tools() -> list[ToolDescriptor]
    def hooks() -> list[HookBase]
    def prompt_contributors() -> list[PromptContributor]
    def is_active(ctx: HookContext) -> bool  # 子类必须覆写
```

`ModeGatedHook`：当 `owner_mode.is_active(ctx)` 为 False 时自动跳过，防止模式泄漏。

启动时（`app/_app.py`）注册：

```python
builtin_mode_clses = [CodingMode, MissionMode, GoalMode]
```

---

## 5. Coding Mode

| 项 | 说明 |
|----|------|
| 激活 | `agent_config.coding_mode.enabled` 或 ACP 传入项目目录 |
| 能力 | LSP（`python-lsp-server`）、`ast_search`（ast-grep）、Inline Diff |
| 实现 | `modes/coding/` + `CodingModeMixin` 混入 `QwenPawAgent` |
| Gate | 无专属 StopHandler；仍走默认 ReAct gates |

控制台提供三面板 Web IDE（文件树 / Diff / 对话），后端工具与 mixin 负责编码语义。

---

## 6. Goal Mode

| 项 | 说明 |
|----|------|
| 激活 | `/goal <描述>` 创建 `GoalSession` |
| 工具 | `get_goal` / `create_goal` / `update_goal`（`requires_modes=("goal",)`） |
| Gates | `GoalTurnGate`（跨请求外层轮次）、`GoalBudgetGate`（token）、`RubricGate`（LLM 评分） |
| Scope | 注册到 **universal**（`scope=""`），与默认 gates 共存 |

区分两层循环：

- **内层：** `ReActConfig.max_iters` — 单次请求内的 tool 迭代
- **外层：** `GoalTurnGate` — 跨请求的 goal 轮次

---

## 7. Mission Mode

| 项 | 说明 |
|----|------|
| 激活 | `/mission` + `session_state.mission_active` |
| 状态 | 文件状态机：`prd.json`、`progress.txt`、`loop_config.json` 等 |
| Gate | `MissionGate`（`scope="mission"`） |
| Hook | `MissionStateLoadHook` / `SaveHook` 持久化 |

与 Goal 的不对称：Mission 偏**文件持久化 + 独立 scope**；Goal 偏**内存会话 + universal gates**。

---

## 8. Mode 对工具可见性的影响

`WorkspacePlugins.active_mode_names(ctx)` 在 `AgentBuilder.build` 时计算；`ToolRegistry.filter(active_modes=...)` 只暴露匹配 `requires_modes` 的工具。Prompt Contributor 同样按 `is_active` 注入系统提示片段。

---

## 9. 阅读建议

1. 先读 `loop/gates/handler.py` + `runner.py` 理解决策合并
2. 再读 `loop/react_gates.py` 看默认行为
3. 对照 `modes/goal/` 与 `modes/mission/` 理解 scope 差异
4. 最后看 `agents/react_agent.py` 中 Gate 的 deferred 挂载点

下一篇：[通道与模型](./06-channels-and-providers.md)
