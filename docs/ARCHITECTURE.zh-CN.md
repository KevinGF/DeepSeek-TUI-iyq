# DeepSeek TUI 架构

> **English**：[ARCHITECTURE.md](ARCHITECTURE.md)

本文面向开发者与贡献者，概述 DeepSeek TUI 架构。

当前边界说明（v0.8.6）：

- `crates/tui` 仍是 TUI、运行时 API、任务管理器与工具执行循环的**活**端到端运行时。
- 其它 workspace crate 在逐步拆分，但**尚未**单独成为唯一真相源。
- LSP 子系统（`crates/tui/src/lsp/`）已完整接入引擎**工具执行后**路径（`core/engine/lsp_hooks.rs`），在每次 `edit_file`/`apply_patch`/`write_file` 后提供行内诊断。
- v0.8.5 移除 swarm，改为子智能体（`agent_spawn`）与 RLM（`rlm_query`）。活跃代码库中**无**模型可见的 swarm 工具。

## 高层概览

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │   TUI (ratatui) │  │  One-shot Mode  │  │  Config/CLI    │  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬───────┘  │
└───────────┼─────────────────────┼────────────────────┼──────────┘
            │                     │                    │
            ▼                     ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Core Engine                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Agent Loop (core/engine.rs)           │   │
│  │  ┌─────────┐  ┌─────────────┐  ┌──────────────────────┐ │   │
│  │  │ Session │  │ Turn Mgmt   │  │ Tool Orchestration   │ │   │
│  │  └─────────┘  └─────────────┘  └──────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
            │                     │                    │
            ▼                     ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Tool & Extension Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐  ┌────────────────┐   │
│  │  Tools   │  │  Skills  │  │  Hooks  │  │  MCP Servers   │   │
│  │ (shell,  │  │ (plugins)│  │ (pre/   │  │  (external)    │   │
│  │  file)   │  │          │  │  post)  │  │                │   │
│  └──────────┘  └──────────┘  └─────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
            │                     │                    │
            ▼                     ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Runtime API + Task Management                  │
│  ┌─────────────────────────────┐  ┌──────────────────────────┐  │
│  │ HTTP/SSE Runtime API        │  │ Persistent Task Manager  │  │
│  │ (runtime_api.rs)            │  │ (task_manager.rs)        │  │
│  └─────────────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
            │                     │
            ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                        LLM Layer                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              LLM Client Abstraction (llm_client.rs)       │  │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐    │  │
│  │  │  DeepSeek Client │  │  Compatible Client (DeepSeek)│    │  │
│  │  │   (client.rs)   │  │       (client.rs)           │    │  │
│  │  └─────────────────┘  └─────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 模块组织

### 入口

- **`main.rs`** —— CLI 参数（clap）、配置加载、入口路由

### 核心组件

- **`core/`** —— 引擎主体
  - `engine.rs` —— 引擎状态、操作处理、消息处理
  - `engine/turn_loop.rs` —— 流式回合循环与工具编排
  - `engine/capacity_flow.rs` —— 容量护栏检查点与干预
  - `session.rs` —— 会话状态
  - `turn.rs` —— 回合对话
  - `events.rs` —— UI 更新事件
  - `ops.rs` —— 核心操作

### 配置

- **`config.rs`** —— 配置加载、profile、环境变量
- **`settings.rs`** —— 运行时偏好设置

### Workspace Crates

- **`crates/tools`** —— 共享工具调用原语（结果/错误/能力类型等）
- **`crates/agent`** —— 模型/提供商注册表（`ModelRegistry`）
- **`crates/app-server`** —— 无头智能体工作流的 HTTP/SSE + JSON-RPC
- **`crates/config`** —— 配置加载、profile、环境优先级、CLI 覆盖
- **`crates/core`** —— 智能体循环、会话、回合编排、容量流护栏
- **`crates/execpolicy`** —— 工具执行审批/沙箱策略引擎
- **`crates/hooks`** —— 生命周期钩子（stdout、jsonl、webhook）
- **`crates/mcp`** —— MCP 客户端 + stdio 服务器
- **`crates/protocol`** —— 请求/响应帧与协议类型
- **`crates/secrets`** —— OS 钥匙串集成
- **`crates/state`** —— SQLite 线程/会话持久化
- **`crates/tui-core`** —— 事件驱动 TUI 状态机脚手架

### LLM 集成

- **`client.rs`** —— DeepSeek 文档化 OpenAI 兼容 Chat Completions HTTP 客户端
- **`llm_client.rs`** —— 抽象 LLM 客户端 trait 与重试
- **`models.rs`** —— API 请求/响应数据结构

#### DeepSeek API 端点

- `https://api.deepseek.com/v1/chat/completions` —— 常规与流式回合
- `https://api.deepseek.com/v1/models` —— 模型发现与健康检查

`https://api.deepseek.com/v1` 为 OpenAI SDK 兼容；`https://api.deepseek.com/beta` 可用于 beta 功能（严格工具模式、chat prefix、FIM 等）。公开文档未将 Responses API 作为本工作流路径；引擎通过 Chat Completions 驱动回合。

### 工具系统

- **`tools/`** —— 内置工具：`shell`、`file`、`todo`、`tasks`、`github`、`automation`、`plan`、`subagent`、`spec`、`rlm` 等

### 扩展

- **`mcp.rs`** —— MCP 客户端集成
- **`skills.rs`** —— Skill 加载与发现
- **`hooks.rs`** —— 生命周期 hook 集成

### 用户界面

- **`tui/`** —— 主 TUI 实现
  - `app.rs` —— 应用状态与事件循环
  - `widgets.rs` —— 可复用 UI 组件
  - `approval.rs` —— 工具审批 UI
  - `transcript.rs` —— 消息展示
  - `composer.rs` —— 输入处理
- **`ui.rs`** —— 旧版/简单 UI 工具

### LSP 集成

- **`lsp/`** —— 编辑后诊断注入（#136）
  - `mod.rs` —— `LspManager`，惰性按语言的传输池 + 配置
  - `client.rs` —— `StdioLspTransport`，经 stdio 的 JSON-RPC，`didOpen`/`didChange`/`publishDiagnostics`
  - `diagnostics.rs` —— 诊断类型、严重级别与 HTML 块渲染器
  - `registry.rs` —— 语言检测与默认服务器映射（rust-analyzer、pyright、gopls、clangd、typescript-language-server）
  - 经 `core/engine/lsp_hooks.rs` 接入引擎——每次成功编辑后调用

### 安全

- **`sandbox/`** —— macOS 沙箱支持
  - `mod.rs` —— 沙箱类型定义
  - `policy.rs` —— 沙箱策略配置
  - `seatbelt.rs` —— macOS Seatbelt profile 生成

### 工具模块

- **`utils.rs`** —— 通用工具
- **`logging.rs`** —— 日志基础设施
- **`compaction.rs`** —— 长对话上下文压缩
- **`pricing.rs`** —— 成本估计
- **`prompts.rs`** —— 系统提示模板
- **`project_doc.rs`** —— 项目文档处理
- **`session.rs`** —— 会话序列化
- **`runtime_api.rs`** —— HTTP/SSE 运行时 API（`deepseek serve --http`）
- **`runtime_threads.rs`** —— 持久 thread/turn/item 存储 + 可回放事件时间线
- **`task_manager.rs`** —— 持久队列、worker 池、任务时间线与产物

## 数据流

### 交互会话

1. TUI 接收用户输入  
2. `core/engine.rs` 处理  
3. 经 `llm_client.rs` 发往 LLM  
4. 流式响应在 `client.rs` 解析  
5. 提取工具调用并由 `tools/` 执行  
6. 工具前后触发 hooks  
7. 聚合结果再送回 LLM  
8. 最终响应渲染于 TUI  

### 崩溃恢复与离线队列

1. 发送用户输入前，TUI 将检查点快照写入 `~/.deepseek/sessions/checkpoints/latest.json`
2. 默认启动仍为全新；先前会话经 `--resume`/`--continue`（或 TUI 内 `Ctrl+R`）显式恢复
3. 降级/离线时，新提示在内存排队并镜像到 `~/.deepseek/sessions/checkpoints/offline_queue.json`
4. 队列编辑（`/queue ...`）持续持久化，草稿与排队提示可 survive 重启
5. 回合成功完成会清除活动检查点并写入持久会话快照
6. Agent/YOLO 回合还会在 `~/.deepseek/snapshots/<project_hash>/<worktree_hash>/.git` 做回合前后 side-git 工作区快照；`/restore N` 与 `revert_turn` 恢复文件状态，**不**改对话历史与用户 `.git`

### 工具执行

1. LLM 经 `tool_use` 内容块请求工具
2. 工具注册表查找处理器
3. 执行前 hooks 运行
4. 若需要则请求审批（非 yolo 模式）
5. 执行工具（macOS 上可能沙箱）
6. 执行后 hooks 运行
7. 结果元数据保留在运行时 item 记录上
8. **LSP 编辑后 hook**（v0.8.6）：若工具为 `edit_file`/`apply_patch`/`write_file` 且 LSP 启用，引擎运行 `run_post_edit_lsp_hook()` 收集诊断
9. **诊断 flush**（v0.8.6）：下次 API 请求前，`flush_pending_lsp_diagnostics()` 将已收集错误以合成用户消息注入
10. 结果返回智能体循环

### 后台任务

1. 客户端入队任务（`/task add ...` 或 `POST /v1/tasks`）
2. `task_manager.rs` 在 `~/.deepseek/tasks` 下持久化任务 + 队列项
3. Worker（有界池）取排队任务，转为 `running`
4. 任务创建/使用运行时线程并启动运行时回合
5. `runtime_threads.rs` 持久化 thread/turn/item 记录 + 单调事件序列
6. 时间线/工具摘要/产物引用增量持久化
7. 清单状态、验证 gate、PR 尝试与受控 GitHub 事件从工具元数据应用到活动任务
8. 终态（`completed|failed|canceled`）可持久查询，经 TUI/API 访问

模型可见的持久任务工具是同一管理器的表面。它们**不**引入并行工作系统：`task_create` 入队普通任务，`checklist_*` 更新任务本地进度，`task_gate_run` 与完成的 `task_shell_wait` 挂验证证据，自动化运行入队普通持久任务。

### 运行时 Thread/Turn 时间线

1. API/TUI 创建或恢复线程（`/v1/threads*`）
2. 在线程上启动回合（`/v1/threads/{id}/turns`）
3. 引擎事件映射为 item 生命周期事件（`item.started|item.delta|item.completed`）
4. 中断/steer 仅作用于活动回合
5. 压缩（自动/手动）作为 `context_compaction` item 生命周期发出
6. 客户端回放历史并经 `/v1/threads/{id}/events?since_seq=<n>` 恢复

### 持久 Schema 门控

- `session_manager.rs`、`runtime_threads.rs` 与 `task_manager.rs` 在持久记录上嵌入 `schema_version`。
- 加载时，较新 schema 版本以显式错误拒绝，而非静默截断/覆盖数据。
- 允许安全向前迁移，并在二进制与存储状态不同步时防止损坏。

## 扩展点

### 新增工具

1. 在 `tools/` 创建处理器
2. 在 `tools/registry.rs` 注册
3. 添加工具规格（名称、描述、输入 schema）

### 新增 MCP 服务器

1. 在 `~/.deepseek/mcp.json` 配置
2. 启动时自动发现
3. 工具自动暴露给 LLM

### 创建 Skill

1. 建含 `SKILL.md` 的 skill 目录
2. 定义 skill 提示与可选脚本
3. 放到 `~/.deepseek/skills/`

### 添加 Hooks

在 `~/.deepseek/config.toml` 配置：

```toml
[[hooks]]
event = "tool_call_before"
command = "echo 'Running tool: $TOOL_NAME'"
```

## 关键设计决策

1. **流式优先**：所有 LLM 响应流式输出以保证响应性
2. **工具安全**：非 YOLO 模式对破坏性操作（含副作用 MCP 工具）需审批
3. **可扩展性**：MCP、skills、hooks 允许不改代码定制
4. **跨平台**：核心在 Linux/macOS/Windows 工作；沙箱仅 macOS
5. **依赖克制**：审慎选依赖以加快构建
6. **本地优先运行时 API**：HTTP/SSE 端点面向可信 localhost，当前由 `crates/tui` 运行时提供

## 配置文件路径

- `~/.deepseek/config.toml` —— 主配置
- `/etc/deepseek/managed_config.toml` —— 可选托管默认层（Unix）
- `/etc/deepseek/requirements.toml` —— 可选允许策略约束（Unix）
- `~/.deepseek/mcp.json` —— MCP 服务器配置
- `~/.deepseek/skills/` —— 用户 skills 目录
- `~/.deepseek/sessions/` —— 会话历史
- `~/.deepseek/sessions/checkpoints/` —— 崩溃检查点 + 离线队列持久化
- `~/.deepseek/snapshots/` —— `/restore` 与 `revert_turn` 的 side-git 回合前后工作区快照
- `~/.deepseek/tasks/` —— 后台任务记录、队列、时间线、产物
- `~/.deepseek/audit.log` —— 凭证 + 审批/提权操作的追加式审计事件
