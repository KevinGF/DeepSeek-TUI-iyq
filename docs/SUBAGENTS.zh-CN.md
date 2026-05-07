# 子智能体

> **English**：[SUBAGENTS.md](SUBAGENTS.md)

子智能体是智能体循环的后台实例。父智能体为聚焦任务生成一个子体，立即拿回 `agent_id`，并在子体运行至完成的同时继续工作。子体默认继承父体的工具注册表，且使用 `CancellationToken::child_token()` 运行——**取消父体会取消所有后代**。

本文说明角色分类。编排工具面（`agent_spawn` / `agent_wait` / `agent_result` / `agent_cancel` / `agent_list` / `agent_send_input` / `agent_resume` / `agent_assign`）见 `prompts/base.md` 中「Sub-Agent Strategy」及内联工具描述。

## 角色分类

`agent_spawn` 上的 `agent_type` 字段为子体选择系统提示姿态。每种角色是对工作的**不同立场**——不仅是标签不同。

| 角色 | 立场 | 写文件？ | 跑 shell？ | 典型用途 |
|---------------|----------------------------------------|---------|-------------|----------------------------------------------|
| `general` | 灵活；按父体指示执行 | 是 | 是 | 默认；多步任务 |
| `explore` | 只读；快速摸清相关代码 | 否 | 是（只读） | 「找出所有 `Foo` 调用点」 |
| `plan` | 分析并产出策略 | 极少 | 极少 | 「设计迁移；不要执行」 |
| `review` | 阅读并打分（严重度） | 否 | 否 | 「审计此 PR 的缺陷」 |
| `implementer` | 以最小改动落地指定修改 | 是 | 是 | 「把 `bar.rs::Foo::bar` 改成做 X」 |
| `verifier` | 跑测试/校验并报告结果 | 否 | 是（测试） | 「跑 cargo test --workspace 并汇报」 |
| `custom` | 显式窄工具白名单 | 视配置 | 视配置 | 手工挑选工具的受限分发 |

各角色完整系统提示位于 `crates/tui/src/tools/subagent/mod.rs`（搜索 `*_AGENT_PROMPT`）。前缀在子体启动时自动加载；父体的 spawn 提示成为首轮用户消息。

### 何时选哪种角色

- **`general`** —— 任务是「把整件事做完」，而非「去看」「设计」或「验证」。这是正确默认；仅当姿态重要时才换更细角色。
- **`explore`** —— 父体需要先取证再决策。探索体便宜又快；可对独立区域并行起 2–3 个。
- **`plan`** —— 有目标但无可执行分解。规划者写产物（`update_plan` 行、`checklist_write` 条目）但不执行。
- **`review`** —— 已有改动，父体要评分。评审者不打补丁——在 finding 里描述修复，若结论为「要修」再由父体派 Implementer。
- **`implementer`** —— 改动已规格化，只需落地。实现者范围要小：最小编辑、不顺手重构，交回前做快速验证。
- **`verifier`** —— 父体需要对测试套件或其它校验的权威通过/失败。验证者不修失败；捕获失败断言与栈，把修复候选放在 RISKS。
- **`custom`** —— 仅当父体需显式约束工具集。通过 `agent_spawn` 的 `allowed_tools` 字段传白名单。

### 别名

模型可用多种拼写指同一角色：

| 规范名 | 别名 |
|---------------|------------------------------------------------------------------|
| `general` | `worker`, `default`, `general-purpose` |
| `explore` | `explorer`, `exploration` |
| `plan` | `planning`, `awaiter` |
| `review` | `reviewer`, `code-review` |
| `implementer` | `implement`, `implementation`, `builder` |
| `verifier` | `verify`, `verification`, `validator`, `tester` |
| `custom` | （无；必须提供显式 `allowed_tools` 数组） |

匹配**大小写不敏感**。未知值返回类型化错误并列出可接受集合，模型可在下一轮自纠。

## 并发上限

调度器默认将并发子智能体上限设为 10（可在 `~/.deepseek/config.toml` 的 `[subagents].max_concurrent` 配置，**硬顶 20**）。父体达到上限时 `agent_spawn` 返回含上限值的错误；应先 `agent_wait` 等待完成或 `agent_cancel` 释放槽位再重试。

计数仅包含**运行中**的智能体——已完成/失败/取消的记录可查看但不占槽位。丢失 `task_handle` 的智能体（如跨进程重启）也不计入上限。

## 生命周期

每次 spawn 产生一条记录，状态迁移：

```
Pending → Running → (Completed | Failed(reason) | Cancelled | Interrupted(reason))
```

当管理器发现 `Running` 智能体的任务句柄已消失时触发 `Interrupted`——常见于进程重启后从 `~/.deepseek/subagents.v1.json` 加载智能体。父体可 `agent_resume` 尝试继续，或视为终态。

### 会话边界（#405）

每个 `SubAgentManager` 实例构造时分配新的 `session_boot_id`。每次 spawn 将该 id 打在智能体上；持久化状态文件跨重启携带该 id。

`agent_list` 默认**仅当前会话**：非仍运行的旧会话智能体会被过滤。传 `include_archived=true` 可列出全部记录，并带 `from_prior_session: true` 标志以便区分归档与活跃。

从 #405 前持久化文件加载且无 `session_boot_id` 的记录归类为先前会话——管理器无法将其与当前启动匹配。

## 输出约定

每个子智能体最终产出字符串，按顺序含五段：

```
SUMMARY:    一段：做了什么、结果如何
CHANGES:    修改的文件及一行说明；只读则写 "None."
EVIDENCE:   path:line-range 引用与要点；每条一 bullet
RISKS:      可能出问题处 / 父体应复核的内容
BLOCKERS:   阻碍因素；顺利完成则 "None."
```

确切格式见 `crates/tui/src/prompts/subagent_output_format.md`。父体将 `EVIDENCE` 作为下一轮工作集，故探索者与评审者应写精确。

## 记忆与 `remember` 工具（#489）

启用记忆时（`[memory] enabled = true` 或 `DEEPSEEK_MEMORY=on`），子体继承父体的记忆文件。可通过 `remember` 工具追加持久笔记——适合探索者发现值得跨会话保留的约定，或验证者记录「某测试不稳定」。

记忆写入限定在用户自有 `memory.md`，**不**走标准写文件审批流。

## 实现说明

- 源码：`crates/tui/src/tools/subagent/mod.rs`（约 3500 行）。
- 持久化：`~/.deepseek/subagents.v1.json`。Schema 版本 `1`（向前兼容——新可选字段用 `#[serde(default)]`）。
- `is_running` 忽略 `task_handle` 为 `None` 的智能体；避免将已持久化但脱接的记录计入并发上限（#509）。
- `SharedSubAgentManager` 为 `Arc<RwLock<...>>` —— 读路径用读锁，使 `/agents` 与侧栏投影在多智能体扇出时不阻塞主循环（#510）。
