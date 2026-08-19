# 06 · 通道与模型

本文覆盖三类「对外连接」：**Channels**（人找 Agent）、**Providers**（LLM）、**Drivers**（Agent 找外部系统，如 MCP）。

---

## 1. Channels — 多通道消息入口

**路径：** `app/channels/`

### 架构

```
外部 IM / 语音 / Console
  → Channel 适配器
    → ChannelManager.enqueue(channel_id, payload)
      → UnifiedQueueManager（按 channel+session+priority 串行）
        → BaseChannel.consume_one(payload)
          → AgentRequest
            → workspace.stream_query → Runtime.run
              → Event 流
            → MessageRenderer → 回发通道
```

### 关键类

| 类 | 职责 |
|----|------|
| `BaseChannel` | ACL、debounce、payload→`AgentRequest`、消费与回发钩子 |
| `ChannelManager` | 队列与消费者；`from_config` / `from_env` |
| `registry.py` | 内置通道 + 插件通道 |
| `unified_queue_manager.py` | 同 session 顺序保证 |
| `renderer.py` | Event → 通道特定格式 |
| `access_control.py` | DM/群组访问策略 |
| `command_registry.py` | 通道级控制命令（如 `/stop`） |

### 内置通道（`registry._BUILTIN_SPECS`）

`imessage`, `discord`, `dingtalk`, `feishu`, `qq`, `telegram`, `mattermost`, `mqtt`, `console`, `matrix`, `slack`, `voice`, `sip`, `wecom`, `xiaoyi`, `yuanbao`, `wechat`, `onebot`

- **必需：** `console`（加载失败会抛错）
- **可选：** 依赖缺失时跳过，不阻断 CLI 启动

### 消费流程要点（`BaseChannel._consume_one_request`）

1. ACL gate
2. 控制命令 → 直接处理（可绕过 TaskTracker）
3. 普通消息 → `TaskTracker` 注册（支持 `/stop` 取消）
4. `self._process(request)` 迭代 Event 流
5. `on_reply_sent(...)` 回调

---

## 2. Providers — LLM 抽象

**路径：** `providers/`

### 抽象

```python
class Provider(ABC):
    def create_model(config) -> ChatModelBase
    def list_models() -> list[ModelInfo]
```

`ModelInfo` 描述多模态能力、thinking、上下文窗口等。

### ProviderManager

单例，管理：

- 内置：OpenAI、Anthropic、DashScope、Gemini、Ollama、OpenRouter、LM Studio 等
- Custom provider
- 密钥加密（`security/secret_store`）
- `get_chat_model(slot: ModelSlotConfig)`

### 包装层

| 包装 | 作用 |
|------|------|
| `RetryChatModel` | 重试 + QPM 限流 |
| `TokenRecordingModelWrapper` | Token 用量记录 |
| `CappingFormatter` | 内联媒体大小限制 |
| `model_capability_cache` / `multimodal_prober` | 能力探测与缓存 |

`agents/model_factory.py` 按 slot 创建 model + formatter，并套上上述包装。  
`agents/routing_chat_model.py` 支持多 slot 路由。

### 本地模型

`local_models/` 管理本机运行时（如 llama.cpp / Ollama 集成路径），与云端 Provider 并列可选。

---

## 3. Drivers — 外部能力运行时

**路径：** `drivers/`

Drivers 是**协议中立**的连接器层（当前实现以 **MCP** 为主），与 Channel 对称：

| | Channel | Driver |
|--|---------|--------|
| 方向 | 人 → Agent | Agent → 外部系统 |
| 例子 | 钉钉、飞书 | MCP Server 工具 |

### 核心类型

```python
class DriverHandler(ABC):
    async def list_capabilities(request_context) -> list[DriverCapability]
    async def invoke_capability(invocation) -> DriverInvocationResult
    # 内含策略授权 + 审批门
```

`DriverManager`：

- `DriverCard` 存储、凭证（`AsyncCredentialStore`）、`ApprovalGate`
- `register_handler_type("mcp", MCPHandler)`
- 生命周期 `init()` / `shutdown()`

### 进入 Agent 的路径

`drivers/adapters/agentscope_tool.py::build_driver_agent_tools`  
把 capability 转成 AgentScope 工具，在 `AgentBuilder` 阶段注入 `extra_tools`，再经 `PolicyGuardedTool` 包装。

MCP **不再**走独立旁路，统一经 Driver，避免双重暴露。

---

## 4. 配置关联

| 配置 | 文件/字段 |
|------|-----------|
| 通道开关与凭证 | 全局 `config.json` → `channels` |
| 模型 slot | `agent.json` → model / slots |
| MCP / Driver | `config.json` → `mcp`；Driver 卡与加密凭据分存 |
| 安全 | `config.json` → `security.tool_guard` 等 |

下一篇：[安全与治理](./07-security-and-governance.md)
