# QwenPaw 源码分析文档

> 面向贡献者与二次开发者的源码级导读。基于 `src/qwenpaw/`（v2.0.0，AgentScope 2.0）分析整理。

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-架构总览](./01-architecture-overview.md) | Agent OS 分层、核心概念、模块依赖图 |
| [02-目录结构](./02-directory-structure.md) | `src/qwenpaw/` 顶层与关键子目录职责 |
| [03-启动与运行时](./03-startup-and-runtime.md) | CLI/FastAPI 启动、Runtime 8 阶段编排 |
| [04-Agent 执行引擎](./04-agent-engine.md) | QwenPawAgent、Builder、Tools/Skills/Memory/Context |
| [05-Loop 与 Modes](./05-loop-and-modes.md) | StopGate、默认 ReAct Gates、Coding/Goal/Mission |
| [06-通道与模型](./06-channels-and-providers.md) | Channels、Providers、Drivers（MCP） |
| [07-安全与治理](./07-security-and-governance.md) | Tool Guard、ResourceGovernor、Sandbox、Skill Scanner |
| [08-扩展机制](./08-extensibility.md) | Hooks、Plugins、Modes、`@api_action` |
| [09-数据流](./09-data-flow.md) | 端到端请求路径（Console / IM / Cron / ACP） |
| [10-设计模式与要点](./10-design-patterns.md) | 设计亮点、复杂度风险、阅读建议 |
| [11-Config 模块](./11-config-module.md) | 进程 / Agent / 请求三层，以及 load 时的校验、升级与缓存 |

## 快速定位

```
用户消息
  → Channel / Console / TUI / CLI
    → Workspace.stream_query()
      → Runtime.run()          # 8 阶段编排
        → AgentBuilder.build() # 组装 QwenPawAgent
        → AgentExecutor.run()  # reply_stream + SSE
          → QwenPawAgent ReAct 循环
            → LLM (Provider) / Tools (Governance + Sandbox)
            → Loop Gates 决定继续或停止
```

## 与官方用户文档的关系

- 官网文档（`website/public/docs/`）面向**使用者**：安装、配置、功能说明。
- 本目录（`docs/`）面向**开发者**：源码结构、类职责、调用链与扩展点。

## 分析范围

- 主包：`src/qwenpaw/`
- 基座依赖：`agentscope==2.0.4`、`reme-ai==0.4.0.9`
- 未展开：`console/`（前端）、`website/`、`plugins/` 第三方插件实现细节（仅描述加载契约）
