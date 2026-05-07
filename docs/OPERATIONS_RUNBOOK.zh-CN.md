# DeepSeek TUI 运维手册

> **English**：[OPERATIONS_RUNBOOK.md](OPERATIONS_RUNBOOK.md)

本手册涵盖本地 CLI/TUI 运行时的实用排障与事件响应。

## 快速分诊

1. 确认二进制与配置：
   - `cargo run -- --version`
   - `cat ~/.deepseek/config.toml`（或检查所选 profile）
2. 启用详细日志：
   - `RUST_LOG=deepseek_cli=debug cargo run`
   - HTTP 重试/重连：`RUST_LOG=deepseek_cli::client=debug cargo run`
3. 抓取当前状态：
   - `ls ~/.deepseek/sessions`
   - `ls ~/.deepseek/sessions/checkpoints`
   - `ls ~/.deepseek/tasks`

## 事件：回合挂起或流停止

症状：

- TUI 一直处于加载
- 助手输出不完整且无结束

检查：

1. 查看重试/健康日志（`deepseek_cli::client`）
2. 验证端点连通：
   - `curl -sS https://api.deepseek.com/v1/models -H "Authorization: Bearer $DEEPSEEK_API_KEY"`
3. 确认工具输出中无本地沙箱/权限死锁

处置：

1. 若前台 shell 在跑，按 `Ctrl+B` 选择后台或取消当前回合。
2. 若命令在后台启动，让助手用 `exec_shell_cancel` 及返回的任务 id 取消。
3. 需要停止请求本身时用 `Esc` 或 `Ctrl+C` 中断当前回合。
4. 重试提示；仍失败则重启 TUI。
5. 重启后确认先前排队/进行中的运行时回合显示为已中断，而非一直运行。

## 事件：网络中断 / 离线行为

预期：

- 离线模式激活时新提示进入队列
- 队列状态持久化到 `~/.deepseek/sessions/checkpoints/offline_queue.json`

检查：

1. TUI 中打开队列：`/queue list`
2. 确认队列文件存在且时间戳更新

处置：

1. 恢复网络
2. 重新发送队列项（`/queue edit <n>` + Enter，或正常输入流）
3. 队列为空时确保队列文件被清空

## 事件：需要崩溃恢复

预期：

- 检查点位于 `~/.deepseek/sessions/checkpoints/latest.json`
- 除非提供 `--resume`/`--continue`，启动为新会话

处置：

1. 通过 `deepseek --resume <id>` 或 TUI 内 `Ctrl+R` 显式恢复
2. 若需检查检查点，查看 `latest.json` 的模式不匹配/细节
3. 若模式新于二进制支持，升级二进制或删除陈旧检查点

## 事件：持久化状态模式错误

症状：

- 如 `schema vX is newer than supported vY`

涉及存储：

- 会话（`~/.deepseek/sessions/*.json`）
- 运行时 thread/turn/item 记录
- 任务（`~/.deepseek/tasks/tasks/*.json`）

处置：

1. 确认二进制版本与迁移预期
2. 编辑前备份状态目录
3. 选择：
   - 使用更新的兼容二进制，或
   - 归档不兼容记录并重建状态

## 事件：MCP/工具执行失败

检查：

1. 校验 `~/.deepseek/mcp.json` schema 与服务器命令路径
2. 手动确认服务器进程可启动
3. 在 TUI 历史/日志中查看沙箱拒绝

处置：

1. 在需要时重试审批（或仅在合适场景使用 YOLO）
2. 暂时禁用故障 MCP 服务器以隔离问题
3. 用 `/mcp` 诊断验证后重新启用

## 事后清单

1. 保留日志与相关状态文件
2. 记录触发条件、影响与缓解措施
3. 添加或更新回归测试（重试/恢复/schema）
4. 若行为变更，更新本手册与架构文档
