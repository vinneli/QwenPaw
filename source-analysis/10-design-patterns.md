# 10 · 设计模式与要点

总结源码级设计亮点、复杂度风险，以及建议的阅读顺序。

---

## 1. 设计亮点

### 依赖注入 + 薄 Agent

`QwenPawAgent` 不拼装基础设施；`AgentBuilder` 负责装配。好处：单测可替换 model/toolkit；请求级配置变更无需重启进程内全局 agent 单例逻辑。

### 双循环解耦

| 循环 | 控制方 | 粒度 |
|------|--------|------|
| Runtime 8 Phase | HookRegistry | 一次用户请求 |
| ReAct + StopGate | QwenPawAgent + StopHandler | 单次请求内的推理/工具轮 |

请求编排与「智能体是否继续想」分开，Modes 可以只挂 Gate 而不改 Runtime。

### Composable Gate + Scope

priority 合并 + session 隔离 + scope 过滤，使 Goal（叠加）与 Mission（互斥）能共存于同一代码库而不互相污染默认 ReAct。

### 统一信任主干

所有工具（含 Driver/MCP）经 `PolicyGuardedTool`，避免「有的工具有沙箱、有的没有」的分裂实现。

### Context 策略可插拔

`ContextManager` Protocol：Scroll 失败可静默降级 native，保证可用性优先。

### Mode 作为 Bundle

命令 / 工具 / Hook / Prompt 四件套一次注册，配合 `ModeGatedHook`，降低「半开模式」泄漏。

### 取消路径状态保全

CancelledError 时注入 partial response + shield session save，兼顾 UX 与一致性。

### Channel / Driver 对称

人→Agent 与 Agent→外部系统分属两套注册表，协议演进互不绑架。

---

## 2. 复杂度与风险点

1. **StopHandler 依赖 ContextVar + agent_id**  
   ACP / 多 Workspace 下若 id 绑定错误，Gate 可能漏跑或错跑。

2. **Goal 与 Default gates 叠加**  
   需理解 priority 与 `reset_peers` 才能预测迭代计数与干预顺序。

3. **Deferred gate 状态挂在 agent 实例**  
   会话恢复或并发请求处理不当时可能残留 `_gate_pending_*`。

4. **Scroll 与 Governor/沙箱耦合**  
   无沙箱时可能整包降级 native，与用户配置预期不一致。

5. **两套 Hook 认知负担**  
   新人易混淆 Runtime Hook 与 Agent Middleware。

6. **Mission vs Goal 状态模型不对称**  
   文件状态机 vs 内存 GoalSession；持久化/恢复路径不同。

7. **部分 Rubric 占位**  
   如 `SubAgentRubric` 仍可能返回 grader error，扩展 ralph 类 loop 需补全。

8. **ToolCoordinator 单 loop 假设**  
   多 event loop 或错误地从线程回调会出问题。

---

## 3. 关键类速查

```python
# 编排
class Runtime:
    async def run(self, request) -> AsyncGenerator

class AgentBuilder:
    async def build(self, ctx) -> QwenPawAgent

class AgentExecutor:
    async def run(self, msgs) -> AsyncGenerator

# Agent
class QwenPawAgent(CodingModeMixin, Agent): ...

# Loop
class StopHandler:
    def register(self, gate: StopGate) -> None

# Modes
class AgentMode:
    def setup(self, workspace) -> None
    def is_active(self, ctx: HookContext) -> bool

# 通道
class BaseChannel:
    async def consume_one(self, payload) -> None

# 安全
class ResourceGovernor:
    def assert_policy(self, tool_call: ToolCallSpec) -> GovernanceDecision

class ToolGuardEngine:
    def guard(self, tool_name: str, args: dict) -> ToolGuardResult

# 配置
def load_config() -> Config
def load_agent_config(agent_id: str) -> AgentProfileConfig
```

---

## 4. 建议阅读顺序

### 第一周：跑通主路径

1. `__main__.py` → `cli/main.py` → `cli/app_cmd.py`
2. `config/config.py`（`Config` / `AgentProfileConfig`）+ `config/utils.py`（`load_config`）
3. `app/_app.py` lifespan
4. `app/workspace/workspace.py` → `runtime/runtime.py`
5. `runtime/builder.py` → `agents/react_agent.py`
6. `schemas.py` + `runtime/envelope.py`

### 第二周：工具与安全

1. `runtime/tool_registry.py`
2. `governance/tool_adapter.py` + `resource_governor.py`
3. `security/tool_guard/engine.py`
4. `sandbox/` 入口
5. `tool_calls/_coordinator.py`

### 第三周：记忆、上下文、模式

1. `agents/memory/` + `middlewares.py`
2. `agents/context/scroll/`
3. `loop/react_gates.py` + `loop/gates/runner.py`
4. `modes/coding/`、`modes/goal/`、`modes/mission/`

### 第四周：扩展与通道

1. `hooks/` 内置实现
2. `plugins/loader.py` + `api.py`
3. `app/channels/base.py` + `registry.py`
4. `providers/provider_manager.py` + `drivers/manager.py`

---

## 5. 调试切入点

| 现象 | 优先查看 |
|------|----------|
| 请求没进 Agent | Slash 是否命中、`SKIP_AGENT`、Channel ACL |
| 工具被拒 | ToolGuard 日志、Governor decision、approval 队列 |
| 无限循环 / 过早停止 | DoomLoopGate、RubricGate、deferred pending |
| 上下文丢失 | SessionLoad/Save、Scroll vs native、offloader |
| 通道乱序/堆积 | UnifiedQueueManager、TaskTracker |
| 模型报错 | RetryChatModel、能力缓存、media strip |

---

## 6. 与官方文档对照

| 主题 | 用户文档（官网） | 本文档 |
|------|------------------|--------|
| Agent OS 概念 | `website/public/docs/architecture.zh.md` | `01` / `03` |
| 记忆与 Scroll | `memory*.zh.md` / `context.zh.md` | `04` |
| Loop Engineering | `loop-engineering.zh.md` | `05` |
| 安全 | `security.zh.md` | `07` |
| 插件 | `plugins.zh.md` | `08` |
| 通道 | `channels.zh.md` | `06` |
| 配置 | `config.zh.md` | `11` |

用户文档描述**稳定产品行为**；本目录描述**当前源码结构与调用链**，类名可能随版本演进，以仓库为准。

---

返回 [文档索引](./README.md)
