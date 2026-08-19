# 07 · 安全与治理

QwenPaw 的信任主干可概括为：

```
工具调用意图
  → ToolGuard（静态/规则预检）
    → ResourceGovernor（策略：ALLOW / DENY / ASK / SANDBOX）
      → Approval（人工确认，可选）
        → Sandbox（隔离执行，可选）
          → 实际副作用
```

策略文件刻意放在 Workspace **之外**（`governance/{ws_hash}/policy.yaml`），降低 Agent 改写自身权限的风险。

---

## 1. Tool Guard（`security/tool_guard/`）

`ToolGuardEngine.guard(tool_name, args)` 在治理之前做规则预检，例如：

- 危险 shell 模式
- 路径穿越 / 敏感文件访问
- 其他 YAML/规则定义的签名

与 File Guard 等配置项一起由 `SecurityConfig` 控制。

---

## 2. ResourceGovernor（`governance/`）

**核心：** `resource_governor.py`

```python
class ResourceGovernor:
    def assert_policy(self, tool_call: ToolCallSpec) -> GovernanceDecision
    def compile_sandbox_config(self, decision) -> SandboxConfig
```

### 决策类型（概念）

| 决策 | 含义 |
|------|------|
| ALLOW | 直接执行 |
| DENY | 拒绝 |
| ASK | 进入人工审批 |
| SANDBOX / SANDBOX_FALLBACK | 在沙箱中执行或降级到沙箱 |

另有审计（`audit.py`）、检测器（`detectors.py`）、策略模型（`policy.py`）。

---

## 3. PolicyGuardedTool（`governance/tool_adapter.py`）

每个暴露给 LLM 的工具外包一层：

```python
class PolicyGuardedTool:
    async def check_permissions(...) -> GovernanceDecision
    async def __call__(...) -> ToolChunk  # 执行 + 沙箱违规重试
```

审批级别解析优先级大致为：

1. `request_context["approval_level"]`
2. `agent.json` 的 `approval_level`
3. 默认 AUTO

Agent 侧将 AgentScope `PermissionMode` 设为 `BYPASS`，统一走此路径。

---

## 4. 人工审批（`app/approvals/` + security）

HITL 审批可通过：

- Web Console 审批 UI
- 通道卡片（部分 IM）
- Driver 侧 `ApprovalGate`（MCP 调用）

与 Tool Guard 的 `approval.py`、应用层 `approvals/service.py` 协同。

---

## 5. Sandbox（`sandbox/`）

### 模式

| 模式 | 平台 |
|------|------|
| `SEATBELT` | macOS |
| `BUBBLEWRAP` / `LANDLOCK` | Linux |
| `APPCONTAINER` | Windows |
| `NONE` | 不隔离（开发/显式关闭） |

### 生命周期

**按次工具调用创建与销毁**（per-tool-call），不是长期驻留容器。

```python
async with create_sandbox(SandboxConfig(...)) as sandbox:
    result = await sandbox.execute("echo hello")
```

`ResourceGovernor.compile_sandbox_config()` 根据策略生成挂载路径、网络/端口规则等。

---

## 6. Skill Scanner（`security/skill_scanner/`）

安装或启用 Skill 前扫描：

- 命令注入
- 数据外泄
- 社会工程类规则（YAML signatures）

策略见 `scan_policy.py` 与 `rules/signatures/`。

---

## 7. 密钥存储（`security/secret_store.py`）

Provider / Driver 敏感字段加密存储，配合系统 keyring 等机制，避免明文落盘。

---

## 8. 与 Scroll Recall 的交叉约束

Scroll 召回工具在无沙箱且未设置允许 unsandboxed recall 的环境变量时，Builder 可能整包降级到 native context 策略。配置预期与实际行为需对照 `runtime/builder.py` 中的 `_scroll_recall_runnable` 逻辑。

下一篇：[扩展机制](./08-extensibility.md)
