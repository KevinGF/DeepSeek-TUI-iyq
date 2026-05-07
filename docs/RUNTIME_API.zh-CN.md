# 运行时 API 与集成契约

> **English**：[RUNTIME_API.md](RUNTIME_API.md)

DeepSeek TUI 通过 `deepseek serve --http` 暴露本地运行时 API，通过 `deepseek doctor --json` 提供机器可读健康信息，并通过 `deepseek serve --acp` 为通过 stdio 使用 Agent Client Protocol 的编辑器客户端提供服务。本文档是本地 macOS 工作台应用（及其它本地监管进程）嵌入 DeepSeek 引擎、无需抓取终端输出的**稳定集成契约**。

## 架构

```
macOS 工作台（或任意本地监管进程）
        │
        ├─ deepseek doctor --json   → 机器可读健康与能力
        ├─ deepseek serve --http    → HTTP/SSE 运行时 API
        ├─ deepseek serve --acp     → 面向 Zed 等编辑器的 ACP stdio 智能体
        ├─ deepseek serve --mcp     → MCP stdio 服务器
        └─ deepseek [args]          → 交互式 TUI 会话
```

引擎以仅本地进程运行。默认所有 API 绑定 `localhost`。无托管中继、无代管提供商 token、无密钥泄漏。

## ACP stdio 适配器：`deepseek serve --acp`

`deepseek serve --acp` 通过换行分隔的 stdio 使用 JSON-RPC 2.0，面向兼容 ACP 的编辑器。初始适配器实现 ACP 基线：

- `initialize`
- `session/new`
- `session/prompt`
- `session/cancel`

提示请求经配置的 DeepSeek 客户端与当前默认模型路由。响应以 `session/update` 智能体消息块流式发出，随后 `session/prompt` 响应带 `stopReason: "end_turn"`。

适配器有意保守：尚未通过 ACP 暴露 shell 工具、文件写入工具、检查点回放或会话加载。需要完整本地运行时 API 用 `deepseek serve --http`；需要把 DeepSeek 工具当 MCP 工具给其它客户端用 `deepseek serve --mcp`。

## 能力端点：`deepseek doctor --json`

返回描述当前安装就绪状态的 JSON。适合 macOS 工作台轮询健康检查。

```bash
deepseek doctor --json
```

### 响应 schema（关键字段）

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | string | 安装版本（如 `"0.8.9"`） |
| `config_path` | string | 解析后的配置文件路径 |
| `config_present` | bool | 配置文件是否存在 |
| `workspace` | string | 默认工作区目录 |
| `api_key.source` | string | `env`、`config` 或 `missing` |
| `base_url` | string | API 基 URL |
| `default_text_model` | string | 默认模型 |
| `memory.enabled` | bool | 记忆功能是否开启 |
| `memory.path` | string | 记忆文件路径 |
| `memory.file_present` | bool | 记忆文件是否存在 |
| `mcp.config_path` | string | MCP 配置文件路径 |
| `mcp.present` | bool | MCP 配置是否存在 |
| `mcp.servers` | array | 每服务器健康：`{name, enabled, status, detail}` |
| `skills.selected` | string | 解析后的 skills 目录 |
| `skills.global.path` / `.present` / `.count` | — | DeepSeek 全局 skills（`~/.deepseek/skills`） |
| `skills.agents.path` / `.present` / `.count` | — | 工作区 `.agents/skills/` |
| `skills.agents_global.path` / `.present` / `.count` | — | agentskills.io 全局（`~/.agents/skills`） |
| `skills.local.path` / `.present` / `.count` | — | `skills/` 目录 |
| `skills.opencode.path` / `.present` / `.count` | — | `.opencode/skills/` |
| `skills.claude.path` / `.present` / `.count` | — | `.claude/skills/` |
| `tools.path` / `.present` / `.count` | — | 全局 tools 目录 |
| `plugins.path` / `.present` / `.count` | — | 全局 plugins 目录 |
| `sandbox.available` | bool | 当前 OS 是否支持沙箱 |
| `sandbox.kind` | string 或 null | 沙箱种类（如 `"macos_seatbelt"`） |
| `storage.spillover.path` / `.present` / `.count` | — | 工具输出溢出目录 |
| `storage.stash.path` / `.present` / `.count` | — | Composer stash |

### 示例

```json
{
  "version": "0.8.9",
  "config_path": "/Users/you/.deepseek/config.toml",
  "config_present": true,
  "workspace": "/Users/you/projects/deepseek-tui",
  "api_key": {
    "source": "env"
  },
  "base_url": "https://api.deepseek.com",
  "default_text_model": "deepseek-v4-pro",
  "memory": {
    "enabled": false,
    "path": "/Users/you/.deepseek/memory.md",
    "file_present": true
  },
  "mcp": {
    "config_path": "/Users/you/.deepseek/mcp.json",
    "present": true,
    "servers": [
      {"name": "filesystem", "enabled": true, "status": "ok", "detail": "ready"}
    ]
  },
  "sandbox": {
    "available": true,
    "kind": "macos_seatbelt"
  }
}
```

## HTTP/SSE 运行时 API：`deepseek serve --http`

```bash
deepseek serve --http [--host 127.0.0.1] [--port 7878] [--workers 2] [--auth-token TOKEN]
```

默认：主机 `127.0.0.1`，端口 `7878`，2 个 worker（限制在 1–8）。

服务器默认绑定 `localhost`。配置通过 CLI 标志——**无** `[app_server]` 配置段。

默认现有本地行为不变，`/v1/*` 路由**不**鉴权。若要对 `/v1/*` 要求 bearer token，传入 `--auth-token TOKEN` 或在启动前设置 `DEEPSEEK_RUNTIME_TOKEN=TOKEN`。`/health` 对本地进程监管与就绪检查保持公开。

已认证客户端可提供 `Authorization: Bearer TOKEN`、`X-DeepSeek-Runtime-Token: TOKEN`，或对无法设自定义头的 EventSource 式客户端使用 `?token=TOKEN`。

### 端点

**健康**
- `GET /health`

**会话**（旧会话管理器）
- `GET /v1/sessions?limit=50&search=<substring>`
- `GET /v1/sessions/{id}`
- `DELETE /v1/sessions/{id}`
- `POST /v1/sessions/{id}/resume-thread`

**线程**（持久运行时数据模型）
- `GET /v1/threads?limit=50&include_archived=false&archived_only=false`
- `GET /v1/threads/summary?limit=50&search=<optional>&include_archived=false&archived_only=false`
- `POST /v1/threads`
- `GET /v1/threads/{id}`
- `PATCH /v1/threads/{id}`（见下方 body）
- `POST /v1/threads/{id}/resume`
- `POST /v1/threads/{id}/fork`

`archived_only=true` 仅返回已归档线程（与 `include_archived` 互斥覆盖）。默认：`include_archived=false` 且 `archived_only=false` 返回活跃线程。v0.8.10（#563）新增。

`PATCH /v1/threads/{id}` body——字段均可选，缺省表示不改。至少提供一个字段。`title` 与 `system_prompt` 允许空字符串以清除先前值。v0.8.10（#562）新增：

```json
{
  "archived": true,
  "allow_shell": false,
  "trust_mode": false,
  "auto_approve": false,
  "model": "deepseek-v4-pro",
  "mode": "agent",
  "title": "User-set thread title",
  "system_prompt": "You are a useful assistant."
}
```

**回合**（线程内）
- `POST /v1/threads/{id}/turns`
- `POST /v1/threads/{id}/turns/{turn_id}/steer`
- `POST /v1/threads/{id}/turns/{turn_id}/interrupt`
- `POST /v1/threads/{id}/compact`（手动压缩）

**事件**（SSE 回放 + 实时流）
- `GET /v1/threads/{id}/events?since_seq=<u64>`

**兼容流**（一次性，向后兼容）
- `POST /v1/stream`

**任务**（持久后台工作）
- `GET /v1/tasks`
- `POST /v1/tasks`
- `GET /v1/tasks/{id}`
- `POST /v1/tasks/{id}/cancel`

**自动化**（定时重复工作）
- `GET /v1/automations`
- `POST /v1/automations`
- `GET /v1/automations/{id}`
- `PATCH /v1/automations/{id}`
- `DELETE /v1/automations/{id}`
- `POST /v1/automations/{id}/run`
- `POST /v1/automations/{id}/pause`
- `POST /v1/automations/{id}/resume`
- `GET /v1/automations/{id}/runs?limit=20`

**自省**
- `GET /v1/workspace/status`
- `GET /v1/skills`
- `GET /v1/apps/mcp/servers`
- `GET /v1/apps/mcp/tools?server=<optional>`

**用量**（跨线程 token/成本聚合）
- `GET /v1/usage?since=<rfc3339>&until=<rfc3339>&group_by=<day|model|provider|thread>`

`since` / `until` 为含端点的 RFC 3339 时间戳，可省略（无界）。`group_by` 默认 `day`。桶按键升序。空时间范围返回空 `buckets`（永不 404）。成本由模型→定价表计算；无定价条目的模型仍贡献 token 但 `cost_usd` 为 `0.0`。v0.8.10（#564）新增。

```json
{
  "since": "2026-04-01T00:00:00Z",
  "until": "2026-04-30T23:59:59Z",
  "group_by": "day",
  "totals": {
    "input_tokens": 12345,
    "output_tokens": 6789,
    "cached_tokens": 0,
    "reasoning_tokens": 0,
    "cost_usd": 0.012,
    "turns": 42
  },
  "buckets": [
    {
      "key": "2026-04-30",
      "input_tokens": 1234,
      "output_tokens": 678,
      "cached_tokens": 0,
      "reasoning_tokens": 0,
      "cost_usd": 0.001,
      "turns": 3
    }
  ]
}
```

## 运行时数据模型

运行时使用持久 Thread/Turn/Item 生命周期。

- **ThreadRecord** — `id`、`created_at`、`updated_at`、`model`、`workspace`、`mode`、`task_id`、`coherence_state`、`system_prompt`、`latest_turn_id`、`latest_response_bookmark`、`archived`
- **TurnRecord** — `id`、`thread_id`、`status`（`queued|in_progress|completed|failed|interrupted|canceled`）、时间戳、时长、用量、错误摘要
- **TurnItemRecord** — `id`、`turn_id`、`kind`（`user_message|agent_message|tool_call|file_change|command_execution|context_compaction|status|error`）、生命周期 `status`、`metadata`

事件为追加式，带全局单调 `seq` 供回放/恢复。

### 重启语义

- 若进程重启时回合或项仍为 `queued` 或 `in_progress`，恢复后的记录标为 `interrupted`，错误为 `"Interrupted by process restart"`。
- 任务执行在同一持久 thread/turn 存储之上另有自身恢复逻辑。

### 审批模型

- `auto_approve` 标志作用于运行时审批桥与引擎工具上下文。在线程/回合/任务上启用时，非交互路径中需审批工具自动批准，shell 安全检查以自动批准模式运行，spawn 的子智能体继承该设置。
- 省略时 `auto_approve` 默认为 `false`。

### SSE 事件流

SSE 事件 payload 形状：

```json
{
  "seq": 42,
  "timestamp": "2026-02-11T20:18:49.123Z",
  "thread_id": "thr_1234abcd",
  "turn_id": "turn_5678efgh",
  "item_id": "item_90ab12cd",
  "event": "item.delta",
  "payload": {
    "delta": "partial output",
    "kind": "agent_message"
  }
}
```

常见事件名：`thread.started`、`thread.forked`、`turn.started`、`turn.lifecycle`、`turn.steered`、`turn.interrupt_requested`、`turn.completed`、`item.started`、`item.delta`、`item.completed`、`item.failed`、`item.interrupted`、`approval.required`、`sandbox.denied`、`coherence.state`。

## 安全边界

- **仅 localhost**。服务器默认绑定 `127.0.0.1`。仅在有反向代理/VPN 鉴权时设 `--host 0.0.0.0`。运行时**不**提供用户隔离或 TLS。
- **可选 token 护栏**。`--auth-token` 或 `DEEPSEEK_RUNTIME_TOKEN` 要求 `/v1/*` 匹配 bearer。这是本地便利措施，**不能**替代公网上的 TLS、VPN 或可信反向代理。
- **不代管提供商 token**。服务器从不返回 API key。`api_key.source` 能力字段报告 `env`、`config` 或 `missing`——从不返回密钥本身。
- **无托管中继**。app-server 是用户控制下的本地进程，无云组件。
- **能力响应**不泄漏密钥、文件内容或会话正文，仅报告*元数据*：存在性、计数、状态标志。

### CORS 允许列表

运行时 API 内置开发源允许列表：`http://localhost:3000`、`http://127.0.0.1:3000`、`http://localhost:1420`、`http://127.0.0.1:1420`、`tauri://localhost`。要增加源（如在 Vite 默认 `:5173` 上开发 UI），可用：

- CLI（可重复）：`deepseek serve --http --cors-origin http://localhost:5173`
- 环境变量（逗号分隔）：`DEEPSEEK_CORS_ORIGINS="http://localhost:5173,http://localhost:8080"`
- 配置（`~/.deepseek/config.toml`）：
  ```toml
  [runtime_api]
  cors_origins = ["http://localhost:5173"]
  ```

用户提供的源**叠在**内置默认之上，不替换。不支持通配源——显式列表模型保留。v0.8.10（#561）新增。

## 会话生命周期（原生 UI 监管）

| 操作 | 端点 |
|---|---|
| 列出会话 | `GET /v1/sessions` |
| 获取会话 | `GET /v1/sessions/{id}` |
| 删除会话 | `DELETE /v1/sessions/{id}` |
| 恢复进线程 | `POST /v1/sessions/{id}/resume-thread` |
| 创建线程 | `POST /v1/threads` |
| 列出线程 | `GET /v1/threads` |
| 附着事件 | `GET /v1/threads/{id}/events?since_seq=0` |
| 发送消息 | `POST /v1/threads/{id}/turns` |
| Steer | `POST /v1/threads/{id}/turns/{turn_id}/steer` |
| 中断 | `POST /v1/threads/{id}/turns/{turn_id}/interrupt` |
| 压缩 | `POST /v1/threads/{id}/compact` |

## 兼容测试

契约快照位于 `crates/protocol/tests/`。运行：

```bash
cargo test -p deepseek-protocol --test parity_protocol --locked
```

验证 app-server 事件 schema 未偏离文档契约。CI 在每次推送到 `main` 与 release tag 上运行。
