# 11 · Config：分层与载入

`config/` 回答三件事：**有哪些 Agent**、**每个 Agent 怎么跑**、**这一跳工具该看哪份工作区**。它是磁盘 JSON 的契约，不是密钥库，也不是治理策略。

密钥在 `SECRET_DIR/providers.json`（`ProviderManager`），沙箱策略在 `governance/{ws_hash}/policy.yaml`（`ResourceGovernor`）。配置模块故意不碰这两类数据：前者要进密钥目录，后者要放在 Workspace 外面，防止 Agent 改自己的权限。

**路径：** `src/qwenpaw/config/`

| 文件 | 角色 |
|------|------|
| `config.py` | 模型 + `load_agent_config` |
| `utils.py` | 根 `config.json` 的 load/save/缓存 |
| `context.py` | 请求期 ContextVar，给工具读 |
| `timezone.py` | 时区探测（独立出来避免循环导入） |

字段手册见官网 [`config.zh.md`](../website/public/docs/config.zh.md)。

---

## 1. 设计从哪来

QwenPaw 从单 Agent 变成「一台机器托管多个隔离 Agent」。如果所有设置仍挤在一份 `config.json` 里：

- 备份 / 删除一个 Agent 会碰到别人的通道密钥和工具开关
- 根文件无限膨胀，mtime 缓存几乎总失效
- Workspace 不再是自包含边界

于是配置按**生命周期**切开，而不是按功能域切开：

```
进程级（安装只有一份）     →  谁在、听哪个端口、插件、时区
Agent 级（随工作区走）     →  这个 Agent 用哪套模型 / 工具 / Loop
请求级（不落盘）           →  这一跳 shell/文件 的 cwd 与超时
```

对应三套机制：`load_config` → `load_agent_config` → `config.context` 的 ContextVar。

热路径（`AgentBuilder`、通道、Cron）读 **agent.json**。根配置只做目录、回退和进程级状态。

---

## 2. 三层分别装什么

```
WORKING_DIR/config.json                 Config          进程级
WORKING_DIR/workspaces/{id}/agent.json  AgentProfileConfig
（请求内 ContextVar）                   config.context  不落盘
```

`WORKING_DIR` 来自 `constant.py`：环境变量 → 遗留 `~/.copaw` → 默认 `~/.qwenpaw`。

### 进程级：`Config`

根文件要保持小。`agents.profiles` 只存引用：

```
AgentProfileRef: id + workspace_dir + enabled
```

其余是「整个进程共享、不属于某一个 Agent」的东西：`last_api`、`user_timezone`、`plugins`、`skill_paths`、`show_tool_details`。

根上仍留着 `channels` / `mcp` / `tools` / `security` / `agents.running`。这是**有意的双写**：

- 旧版本还能读新写出的 `config.json`（降级兼容）
- `agent.json` 缺失时，`build_fallback_agent_profile_config` 从根拷一份出来

真正生效的运行参数在 agent.json。改根上的 `running` 却期望当前 Agent 变行为，是最常见的误读。

### Agent 级：`AgentProfileConfig`

一个工作区一份，移动 / 备份 / 删除都跟着走。核心是 `running: AgentsRunningConfig`——Loop Gates、上下文（native/scroll）、记忆后端、LLM 重试与 QPM、shell 超时。旁边挂 `active_model`、`tools`、`channels`、`mcp`、`approval_level`、`coding_mode`。

同一份 schema 会在根、Agent 两边出现（通道、MCP、工具、安全）。语义不是「全局覆盖 Agent」，而是「Agent 为主，根作遗留与 fallback」。

### 请求级：ContextVar

工具是普通函数，拿不到 `Workspace`。`ContextVarsSetupHook`（`PRE_DISPATCH`）把本跳的 `workspace_dir`、shell 超时、输出裁剪上限写入 `config.context`。HTTP / 通道用的 `agent_id`、`user_id` 在另一套 `app.agent_context` 里。两套不要混：一套给**副作用**，一套给**路由**。

---

## 3. 模型怎么长

不必背字段。记住四条组织原则：

**1. 容器合并默认，不覆盖用户值。**  
升级加了新内置工具或 ACP agent 时，`model_validator` 把缺的 key 补进已保存的 JSON（`ToolsConfig._merge_default_tools`、`ACPConfig._merge_default_agents`）。用户关掉的项保持关掉。

**2. 通道对外开，运行时对内收。**  
`ChannelConfig(extra="allow")`：插件通道的未知字段要保住。`AgentsRunningConfig` 等子模型 `extra="ignore"`：拼错的新字段静默丢失，避免脏 JSON 撑爆模型。

**3. 别名在进模型前消掉。**  
MCP 的 `isActive` / `baseUrl` / `streamable-http` 在 `mode="before"` 里收成内部字段，校验（stdio 要 command，HTTP 要 url）在 `mode="after"`。非法 MCP client 加载时直接丢掉，不让一个坏条目卡死整个 Agent。

**4. 缺省写在代码里，磁盘可以不存在。**  
`load_config` 找不到文件就返回 `Config()`；缺 `agent.json` 就从根 fallback 再写盘。首次启动和 `doctor fix` 走同一条 `build_fallback_agent_profile_config`，避免两套默认值。

---

## 4. 载入逻辑

设计目标可以压成一句话：**热路径要快，坏文件也要能开机，旧安装不要被升级踢下线。**

### 根：`load_config`

```
WORKING_DIR/config.json
    │  无文件 → Config() 默认值
    ▼
mtime 未变？ → 返回进程缓存
    ▼
读盘：json.loads，失败则 json_repair；再失败 → 备份 .bak，用默认值
    ▼
内存改写 ~/.copaw 路径；折叠 last_api_host/port
    ▼
Config.model_validate
    │  失败 → 删出错字段再试
    │  仍失败 → 备份，用默认值
    ▼
一次性磁盘迁移（如 weixin→wechat）
    ▼
写入 (_config_cache, _config_mtime)
```

`save_config` 写盘后把缓存置空。`strict_validate_config_file` 给 doctor 用：同样读盘，但**不删字段**，把错误列表交出去。

### Agent：`load_agent_config`

必须先能在根 `profiles` 里解析出 `workspace_dir`，否则这个 id 不存在。

```
load_config() → 取 AgentProfileRef
    │  无 agent.json → 从根 fallback，写盘，返回
    ▼
mtime 命中 _agent_config_cache → 返回
    ▼
json.load
    → 一次性迁移（通道 key、ACL 字段）可回写磁盘
    → ~/.copaw 路径只改内存，不回写（尊重用户文件）
    → sanitize_mcp_clients：坏条目丢掉
    ▼
AgentProfileConfig(**data) → 缓存
```

根配置自愈（repair / 删字段），agent.json **不**同等自愈：除 MCP skip 外，校验失败即抛。根文件坏了进程起不来；单个 Agent 配坏了只应打掉这一个。

`last_api` 额外有一份进程内缓存。桌面端随机端口时磁盘可能被迁移写乱，`read_last_api` 优先信内存。

### 迁移放在加载路径上

兼容代码不做成独立「升级向导」，而是 **load 时顺手做一次**：

| 问题 | 策略 |
|------|------|
| JSON 语法脏 | repair；不行就备份 + 默认 |
| 字段改名 | validator / 读时改磁盘 |
| 单 Agent → 多 Agent | `migrate_legacy_config_to_multi_agent()` 拆出 `workspaces/default/` |
| 缺 agent.json | 根配置生成一份 |

所以「载入」同时是「校验 + 升级 + 缓存」。读配置的代码不必先问版本号。

---

## 5. 读的时候认哪一层

```
load_config()            进程：Agent 列表、插件、时区、last_api、鉴权白名单
load_agent_config(id)    热路径：模型、工具、Loop、通道、MCP
config.context           工具：cwd、shell、裁剪上限
```

`AgentBuilder.build` 每请求 `load_agent_config`。mtime 不变则是内存命中，所以「每请求读配置」成本可接受，也因此外部改 JSON 只要碰到保存（mtime 变）就会生效；进程内改了 Pydantic 对象却不 `save_*`，下次 load 仍是旧缓存。

---

## 6. 用这张图做决策

加开关时先问「它跟谁一起死」：

- 整个安装一份 → 根 `Config`
- 跟某个 Agent 走 → `AgentProfileConfig` / `AgentsRunningConfig`
- 只在这一跳有效 → ContextVar，不要写进 JSON

改已有字段时记住：热路径读 agent.json；根上的同名段是 fallback 和降级兼容。两边都改、只改一边、或以为根覆盖 Agent，都会和运行时看到的不一致。

---

返回 [文档索引](./README.md)
