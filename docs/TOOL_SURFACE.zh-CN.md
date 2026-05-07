# 工具面

为何是这些具体工具、如何分组，以及各自相对可用 shell 的取舍。配套 `crates/tui/src/prompts/agent.txt`。

## 设计立场

- **只要专用工具能返回结构化输出，就优先于 `exec_shell`。** Bash 转义易错，平台行为不一（GNU 与 BSD `grep`、`rg` 未必安装）。结构化输出也避免模型再去解析自由文本。
- **其余一律 `exec_shell`。** 构建、测试、格式化、Lint、临时命令、平台相关——不试图封装长尾。
- **打不过 shell 等价物的工具就删。** 同一底层操作的两个别名是模型陷阱——LLM 会在其间切换，缓存命中率下降。

## 当前面（v0.7.5）

### 文件操作

| 工具 | 定位 |
|---|---|
| `read_file` | 读取 UTF-8 文件。若可用 `pdftotext`（poppler）则自动抽取 PDF；`pages: "1-5"` 可切片大文档。 |
| `list_dir` | 结构化、尊重 gitignore 的目录列表。优于 `exec_shell("ls")`。 |
| `write_file` | 创建或覆盖文件。 |
| `edit_file` | 单文件内查找替换。比整文件重写更省。 |
| `apply_patch` | 应用 unified diff。多 hunk 编辑的合适工具。 |

### 搜索

| 工具 | 定位 |
|---|---|
| `grep_files` | 工作区内正则搜索文件内容；结构化匹配 + 上下文行。纯 Rust（`regex` crate），不 shell 调 `rg`/`grep`。 |
| `file_search` | 文件名模糊匹配（非内容）。大致知道文件名时用。 |
| `web_search` | DuckDuckGo（Bing 回退）；排序摘要 + 引用用 `ref_id`。 |
| `fetch_url` | 对已知 URL 直接 HTTP GET。链接已知时比 `web_search` 更快。默认将 HTML 剥成文本。 |

### Shell

| 工具 | 定位 |
|---|---|
| `exec_shell` | 运行 shell 命令。前台运行可取消，但仅用于有界命令；超时杀进程并返回「后台重跑」提示。 |
| `exec_shell_wait` | 轮询后台任务的增量输出。取消回合会停止等待但不会杀任务。 |
| `exec_shell_interact` | 向运行中的后台任务发 stdin 并读增量输出。 |
| `exec_shell_cancel` | 按 id 取消单个后台 shell，或按显式请求取消全部运行中的后台 shell。 |
| `task_shell_start` | 后台启动长时命令并立即返回。诊断、测试、搜索、可能跑数分钟的服务等，优于前台 shell。 |
| `task_shell_wait` | 轮询后台命令。若完成后提供 `gate`，会在当前持久任务上记录结构化 gate 证据。 |

前台 shell 超时时，进程不会悄悄继续跑。工具结果会提示模型用 `task_shell_start` 或 `background = true` 的 `exec_shell` 重跑长任务，再用 `task_shell_wait` 或 `exec_shell_wait` 轮询。

交互式 shell 任务也可通过 `/jobs` 查看。TUI 任务中心与 `exec_shell`/`task_shell_start` 共用同一 shell 管理器，展示命令、cwd、已用时间、状态、输出尾部、进程局部 shell id，以及关联的持久任务 id（若有）。`/jobs show`、`/jobs poll`、`/jobs wait`、`/jobs stdin`、`/jobs cancel` 提供检查、轮询、stdin 与取消。任务为进程局部；重启后不会重连实时进程状态，任何记着的 detached 项须标为陈旧而非当作仍在运行。

### MCP 管理与调色板发现

MCP 服务器配置通过 TUI 的 `/mcp` 与 `/config` 中的 `mcp_config_path` 行展示。`/mcp` 显示解析后的配置路径、启用/禁用、传输、命令或 URL、超时、连接错误，以及发现的工具/资源/提示。支持较窄的管理动作：init、add、enable、disable、remove、validate、reload/reconnect。配置写入立即生效，但编辑后模型可见的 MCP 工具池需重启 TUI。

命令调色板含按服务器分组的 MCP 项。禁用与失败的服务仍可见；发现的工具/提示使用向模型展示的运行时名，如 `mcp_<server>_<tool>`。

### Git / 诊断 / 测试

| 工具 | 定位 |
|---|---|
| `git_status` | 不跑 shell 即可查看仓库状态。 |
| `git_diff` | 查看工作区或暂存区 diff。 |
| `diagnostics` | 工作区、git、沙箱、工具链信息一次调用。 |
| `run_tests` | `cargo test` 及可选参数。 |

### 任务管理与持久工作

| 工具 | 定位 |
|---|---|
| `update_plan` | 复杂多步工作的结构化清单。 |
| `task_create` | 通过 `TaskManager` 创建/入队持久后台任务。长时智能体工作的真实可执行对象。 |
| `task_list` | 列出持久任务及状态与关联运行时 id。 |
| `task_read` | 读持久任务详情：线程/回合关联、时间线、清单、gates、产物、PR 尝试、GitHub 事件。 |
| `task_cancel` | 取消排队或运行中的持久任务。需审批。 |
| `checklist_write` | 当前线程/任务下的细粒度进度。清单状态从属于持久任务。 |
| `checklist_add` / `checklist_update` / `checklist_list` | 单条清单操作。 |
| `todo_write` / `todo_add` / `todo_update` / `todo_list` | 清单工具的兼容别名。旧会话仍可用，新提示应使用 `checklist_*`。 |
| `note` | 一次性重要事实供后续使用。 |

### 验证 gate 与产物

| 工具 | 定位 |
|---|---|
| `task_gate_run` | 运行已批准的验证命令，并将结构化证据挂到当前持久任务：命令、cwd、退出码、时长、分类、摘要与日志产物。 |

大日志与命令输出应作为产物，在对话中有紧凑摘要。`task_gate_run` 对活跃持久任务会自动处理。

### GitHub 上下文与受控写入

| 工具 | 定位 |
|---|---|
| `github_issue_context` | 只读 issue 上下文（`gh issue view`）；大正文在可能时变为任务产物。 |
| `github_pr_context` | 只读 PR 上下文（`gh pr view`）；可选 `gh pr diff --patch` 捕获 diff；大正文/diff 在可能时变为任务产物。 |
| `github_comment` | 需审批的 issue/PR 评论与结构化证据。 |
| `github_close_issue` | 需审批的 issue 关闭。要求非空验收标准与证据；工作区脏时除非显式允许否则拒绝。绝不因智能体停止而关闭 issue。 |

### PR 尝试

| 工具 | 定位 |
|---|---|
| `pr_attempt_record` | 将当前 git diff 捕获为尝试元数据 + 持久任务上的 patch 产物。 |
| `pr_attempt_list` | 列出任务上记录的尝试。 |
| `pr_attempt_read` | 查看某次记录及其产物引用。 |
| `pr_attempt_preflight` | 对尝试 patch 运行 `git apply --check`。不修改工作区。 |

### 自动化

| 工具 | 定位 |
|---|---|
| `automation_create` | 创建定时自动化。需审批。 |
| `automation_list` / `automation_read` | 查看持久自动化与最近运行。 |
| `automation_update` | 更新提示、计划、cwd 或状态。需审批。 |
| `automation_pause` / `automation_resume` / `automation_delete` | 生命周期控制。需审批。 |
| `automation_run` | 立即运行自动化；运行会入队普通持久任务。需审批。 |

### 子智能体

`agent_spawn` 及配套工具（`agent_result` / `wait` / `send_input` / `agent_assign` / `agent_cancel` / `resume_agent` / `agent_list`）。委派协议见 `agent.txt`，角色分类见 [`SUBAGENTS.md`](SUBAGENTS.md)（[简体](SUBAGENTS.zh-CN.md)）（`general` / `explore` / `plan` / `review` / `implementer` / `verifier` / `custom`）。

### 并行扇出：成本类别上限

两个工具提供并行扇出，并发上限反映截然不同的成本类别：

| 工具 | 每个子任务做什么 | 墙钟 | Token 成本 | 上限 |
|---|---|---|---|---|
| `agent_spawn` | 完整子智能体循环（规划、工具调用、多回合流式，可再 spawn） | 分钟级 | 数千 token | 默认飞行中 10（`[subagents].max_concurrent`，硬顶 20） |
| RLM 辅助 `llm_query_batched` | 一次性非流式 Chat Completions，固定 `deepseek-v4-flash` | 秒级 | 约数百 token | 每次调用 16 |

上限写在各工具描述与错误信息中，便于模型（与用户）选型。若一个子智能体够用但需要并行查找，优先 `rlm` + `llm_query_batched`；若每项需要自带工具循环的智能体，用 `agent_spawn`（并取消已完成的以释放槽位）。

## 近期合并（v0.5.1）

从提示中移除的重复项（底层仍解析，旧会话不断——只是不再污染模型工具列表）：

- `spawn_agent` → 用 `agent_spawn`。
- `close_agent` → 用 `agent_cancel`。
- `assign_agent` → 用 `agent_assign`。

## 弃用时间表（v0.6.2 → v0.8.0）

以下别名仍可成功执行，但每次结果会附带 `_deprecation` 块。模型应在 v0.8.0 移除别名前迁到规范名。

| 弃用别名 | 规范名 | 自何版本警告 | 移除版本 |
|---|---|---|---|
| `spawn_agent` | `agent_spawn` | v0.6.2 | v0.8.0 |
| `delegate_to_agent` | `agent_spawn` | v0.6.2 | v0.8.0 |
| `close_agent` | `agent_cancel` | v0.6.2 | v0.8.0 |
| `send_input` | `agent_send_input` | v0.6.2 | v0.8.0 |

`_deprecation` 块形状：

```json
{
  "_deprecation": {
    "this_tool": "spawn_agent",
    "use_instead": "agent_spawn",
    "removed_in": "0.8.0",
    "message": "Tool 'spawn_agent' is deprecated; switch to 'agent_spawn' before v0.8.0."
  }
}
```

该块合并进工具结果的 `metadata`，与其它键（如 `status`、`timed_out`）并存。每次调用别名还会在审计日志以 `tracing::warn` 打一行弃用警告。

## 为何不单独提供 `bash` 工具

单一 `bash` 式智能体（如 Claude Code 设计）很强，但把 shell 脚本的全部坑交给模型：引号、平台差异、误读 cwd 的副作用、调用间 `cd` 不持久等。我们的文件工具在对话中渲染也更便宜（结构化 JSON 比 `ls -la` 大段文本更易折叠）。

缺什么时模型总可回退到 `exec_shell`。专用工具只是拿走常见 80%，shell 作逃生舱。
