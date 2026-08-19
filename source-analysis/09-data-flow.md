# 09 · 数据流

本文给出四条端到端路径，帮助从「用户输入」追到「工具副作用」与「回复落盘」。

---

## 路径 A：Web Console / HTTP API

```
1. 前端 POST /api/... 或 SSE/WebSocket 对话接口
2. AgentContextMiddleware 设置 ContextVar（agent_id, session_id, channel）
3. DynamicMultiAgentRunner.stream_query(AgentRequest)
4. Runtime.run():
   a. PRE_DISPATCH（ContextVars、媒体）
   b. Slash 分发 → 命中则 SHORT_CIRCUIT
   c. PRE_AGENT_BUILD → SessionLoad
   d. AgentBuilder.build：
        load_agent_config
        → ProviderManager.get_chat_model
        → ToolRegistry.filter → PolicyGuardedTool
        → PromptManager
        → QwenPawAgent + Memory + Context + Governor
   e. POST_AGENT_BUILD / PRE_EXECUTE（Mode、Bootstrap、SkillEnv）
   f. AgentExecutor.run：
        reply_stream 循环
          · LLM（RetryChatModel → Provider API）
          · tool_call → ToolGuard → Governor → Approval? → Sandbox?
          · StopHandler 判定继续/终止
        Event → Envelope → SSE
   g. POST_RESPONSE → SessionSave
   h. FINALLY → close / 清理 ContextVar
5. 前端渲染 text delta / tool call / completed
```

---

## 路径 B：IM 通道（飞书 / 钉钉 / …）

```
1. Webhook 或长连接收到原生 payload
2. Channel.enqueue → UnifiedQueueManager（同 session 串行）
3. BaseChannel._consume_one_request：
   - ACL
   - payload → AgentRequest（含 channel_meta）
   - TaskTracker 注册（可 /stop）
4. workspace.stream_query → 同路径 A 的 Runtime
5. MessageRenderer 将 Event 转为通道消息并发送
6. on_reply_sent 更新 last_dispatch
```

要点：通道只做适配与排队；**业务智能**全部在 Runtime/Agent。

---

## 路径 C：Cron / Heartbeat

```
CronManager 触发
  → 构造 AgentRequest
  → CronContextHook 注入上下文
  → CronMemoryIsolateHook（可选隔离记忆）
  → Runtime.run()
  → 回复写入指定通道或 Console
  → CronMemoryRestoreHook
```

与交互式对话共用同一 Runtime，保证行为一致。

---

## 路径 D：TUI / ACP

```
TUI（Textual）或外部 ACP Client
  → 本地 API / ACP Server（agents/acp）
  → 同一 Workspace + Runtime
  → 共享记忆、Skills、会话
```

Coding 项目可通过 ACP meta 启用 Coding Mode（项目目录注入）。

---

## 工具调用细流

```
LLM 产出 tool_call block
  → ToolCoordinatorMiddleware.on_acting
    → ToolCoordinator.execute
      → before_hook
      → PolicyGuardedTool.__call__
          → ToolGuardEngine.guard
          → ResourceGovernor.assert_policy
          → (ASK → 等待审批)
          → Sandbox 或本地执行
      → after_hook / result_limiter
  → tool result 写入 context
  → 若超时：offload 后台，下轮 _reply 注入 hint
  → 下一轮 _reasoning + Gates
```

---

## 会话与记忆写回

| 数据 | 时机 | 位置（概念） |
|------|------|----------------|
| Agent state / 对话 | POST_RESPONSE SessionSave | workspace 会话存储 |
| Scroll history | on_save / compress | `history.db` + eviction index |
| 长期记忆 | MemoryMiddleware post_reply / 工具 | ReMe / 后端存储 |
| Token 用量 | TokenRecording 包装 | `token_usage/` |
| 审计 | Governor | governance audit |

取消路径：Runtime 尽量保存 partial response，并用 shield 保护 save。

---

## 多 Agent

```
WorkspaceRegistry
  ├── agent-A Workspace（独立文件/记忆/通道）
  ├── agent-B Workspace
  └── ...
```

跨 Agent 通信需显式工具/协议（如 sub-agent、ACP、消息投递）；默认互不可见。

下一篇：[设计模式与要点](./10-design-patterns.md)
