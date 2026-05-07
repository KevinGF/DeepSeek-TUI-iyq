# v0.7.6 遗留 Rust 审计

> **English**：[LEGACY_RUST_AUDIT_0_7_6.md](LEGACY_RUST_AUDIT_0_7_6.md)

状态日期：2026-04-29

本审计**有意非破坏性**。除非测试证明公共 CLI、已保存会话、工具 schema 与文档化命令路径不再依赖，v0.7.6 **不**移除兼容代码。

## 摘要

| 表面 | 所属模块 | 当前消费者 | 引用检查 | 兼容原因 | 当前警告 | 建议动作 |
|---|---|---|---|---|---|---|
| 遗留 MCP 同步 API（`McpServerInput`、`list`、`add`、`remove`、`call_tool`、`load_legacy`） | `crates/tui/src/mcp.rs` | 未接入当前 `/mcp` 命令路径；保留在 `#[allow(dead_code)]` 后 | 已检查直接 Rust 引用与当前 MCP 命令路径；已保存/配置 JSON 兼容仍需专门冒烟 | 在异步 MCP 管理器为活跃路径时保留旧 JSON 形状（含 `mcpServers` 别名）与同步调用辅助 | 仅代码 TODO | 在显式遗留模块后设闸，或在 CLI/运行时 parity 测试证明无调用者后移除。#218 |
| 遗留提示常量/函数（`AGENT_PROMPT`、`YOLO_PROMPT`、`PLAN_PROMPT`、`base_system_prompt`、`normal_system_prompt` 等） | `crates/tui/src/prompts.rs` | 测试与仍直接导入提示常量的旧调用方 | 直接 Rust 引用仍存在；未证明公共 crate 与旧 harness 导入已消失 | 分层提示 API 取代单体提示，但旧调用点可能仍针对常量编译 | 无 | v0.7.6 保留；内部调用迁移后再加 deprecation。 #219 |
| `/compact` 斜杠命令定位 | `crates/tui/src/commands/mod.rs` | 公共斜杠注册表与帮助浮层 | 公共命令注册表/文档路径仍引用 | 当前 cycle/seam 策略偏好重启/cycle 流，但用户仍可手动运行 `/compact` | 描述标为 legacy 并指向 cycle 重启 | 作为手动兼容命令保留；上下文/token 问题未解决前勿删。 |
| `todo_*` 兼容工具 | `crates/tui/src/tools/todo.rs` | 仍使用 `todo_add`、`todo_update`、`todo_list`、`todo_write` 的工具注册表/模型调用 | 工具注册兼容与已保存工具调用风险仍在 | `checklist_*` 为规范名，但旧名可能出现在已保存提示、追踪或模型先验中 | 元数据 `compat_alias: true`；描述写明兼容别名 | 加显式 deprecation 元数据与目标版本，仅在工具 schema 迁移证据充分后移除。#220 |
| 已弃用子智能体别名工具（`spawn_agent`、`send_input`、委托别名等） | `crates/tui/src/tools/subagent/mod.rs` | 工具注册表与模型/工具调用兼容 | 同上 | 规范名为 `agent_spawn`、`agent_send_input` 等；别名保留旧工具调用兼容 | `_deprecation` 元数据与 tracing 警告；移除目标 `v0.8.0` | 贯穿 v0.7.x 保留；移除已编码元数据，需单独跟踪测试。#221 |
| 遗留根/提供商 TOML `api_key` 兼容 | `crates/tui/src/config.rs`、`crates/config/src/lib.rs` | 配置解析器；已有 `api_key` 的用户 | 公共配置加载与文档仍描述迁移行为 | 偏好钥匙圈迁移，但破坏现有配置会阻断启动/认证 | tracing 警告指向 `deepseek auth set` / `deepseek auth migrate` | 保留；警告可指导用户。移除需迁移命令与发行说明窗口。 |
| 模型别名规范化（`deepseek-chat`、`deepseek-reasoner`、旧 V3/R1 别名） | `crates/tui/src/config.rs`、`crates/config/src/lib.rs` | 配置/环境/模型选择器规范化 | 公共文档与现有配置仍可能使用别名 | 保留旧文档化 DeepSeek 别名并映射到 `deepseek-v4-flash` | 按设计静默别名 | 保留；移除别名会无意义地破坏配置。 |
| 已弃用调色板常量与别名 | `crates/tui/src/palette.rs`、`crates/tui/tests/palette_audit.rs` | 现有调用点与审计测试 | 调色板审计强制执行剩余允许列表 | 倾向语义别名，但旧常量存在以避免大范围样式变动 | 调色板审计阻止允许列表外直接使用已弃用符号 | 保留别名；继续择机将调用点迁到语义角色。 |

## 后续移除候选

v0.7.6 **不宜**移除：

1. #218 遗留 MCP 同步 API：需调用图检查及对 `/mcp`、`deepseek mcp`、MCP 校验流的显式 CLI/运行时 parity 测试。
2. #219 遗留提示常量/函数：需证明无公共 crate 或旧测试 harness 导入。
3. #220 `todo_*` 工具别名：需 deprecation 元数据与已保存追踪/工具 schema 迁移窗口。
4. #221 已弃用子智能体别名工具：移除目标已编码为 `v0.8.0`，实际移除应单独跟踪与测试。

## 验证清单

移除任何兼容表面前：

1. 用 `rg` 搜索直接 Rust 引用。
2. 搜索文档与 README 命令示例。
3. 以全部 feature 运行工作区测试。
4. 若表面影响工具 schema 或持久历史，运行已保存会话/工具调用兼容冒烟。
5. 保留发行说明条目；对用户可见的配置/工具变更，至少一个次版本保留迁移提示。
