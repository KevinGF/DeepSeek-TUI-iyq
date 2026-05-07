# 竞品分析：DeepSeek TUI vs OpenCode vs Codex CLI

> **English**：[COMPETITIVE_ANALYSIS.md](COMPETITIVE_ANALYSIS.md)

三款 AI 编程智能体能力对比：OpenCode（`/Volumes/VIXinSSD/opencode`）、Codex CLI（`/Volumes/VIXinSSD/codex-main`）、DeepSeek TUI（`/Volumes/VIXinSSD/deepseek-tui`）。

## 工具矩阵

| 能力 | OpenCode | Codex CLI | DeepSeek TUI |
|---|---|---|---|
| 读文件 | ✅ Read | ✅ | ✅ file |
| 写文件 | ✅ Write | ✅ | ✅ file |
| 编辑文件 | ✅ Edit（字符串替换） | ✅ apply_patch（diff） | ✅ edit_file + apply_patch |
| 文件 glob | ✅ Glob | ✅ | ✅ file_search |
| 代码搜索 | ✅ Grep + CodeSearch（Exa） | ✅ | ✅ grep_files + search |
| Shell 执行 | ✅ Bash | ✅ exec/shell | ✅ shell |
| Web 拉取 | ✅ WebFetch | ✅ | ✅ fetch_url |
| Web 搜索 | ✅ WebSearch | ✅ WebSearchRequest | ✅ web_search |
| Web 浏览 | ❌ | ❌ | ✅ web_run |
| LSP | ✅ Lsp（实验） | ❌ | ❌ |
| 任务/todo | ✅ TodoWrite | ✅ | ✅ todo_write |
| 子智能体 spawn | ✅ Task | ✅ Collab/SpawnCsv | ✅ agent_spawn |
| Skill 系统 | ✅ Skill（多路径发现） | ✅ core-skills | ⚠️ 部分（`.deepseek/skills/`） |
| Plan 模式 | ✅ plan-enter/exit | ✅ Plan 模式 | ✅ Plan 模式 |
| 向用户提问 | ✅ Question | ✅ request_user_input | ✅ user_input |
| 应用补丁 | ✅ apply_patch（自定义格式） | ✅ apply_patch（diff） | ✅ apply_patch |
| 数据校验 | ❌ | ❌ | ✅ validate_data |
| 金融行情 | ❌ | ❌ | ✅ finance |
| Git 操作 | 经 Bash | ✅ git-utils | ✅ git 模块 |
| GitHub 操作 | 经 Bash（gh） | ✅ | ✅ github |
| 跑测试 | ❌ | ✅ | ✅ test_runner |
| 自动化 | ❌ | ❌ | ✅ automation |
| Code review | ❌ | ✅ GuardianApproval | ✅ review |
| 回忆/归档 | ❌ | ❌ | ✅ recall_archive |
| 诊断 | ❌ | ✅ | ✅ diagnostics |
| 回滚回合 | ❌ | ❌ | ✅ revert_turn |
| 图像生成 | ❌ | ✅ ImageGeneration | ❌ |
| 浏览器使用 | ❌ | ✅ BrowserUse | ❌（`web_run` 为无头） |
| Computer use | ❌ | ✅ ComputerUse | ❌ |
| 实时语音 | ❌ | ✅ RealtimeConversation | ❌ |

---

## 高优先级缺口

这些能力对 DeepSeek TUI 作为编程智能体的效用提升最直接。

### 1. LSP 集成

**是什么：** 模型可调用的工具，通过 Language Server Protocol 向语言服务器查询代码智能——跳转定义、查找引用、悬停（类型信息）、文档符号、工作区符号、调用层次与实现。

**为何重要：** 最大能力缺口。当前每次探索代码库都要 shell `rg` 与顺序读文件。有 LSP 后，智能体可一次调用跳到定义、找函数所有调用方、查看类型。对结构化代码库估计可减少 30–50% 探索回合。

**OpenCode 实现：** `packages/opencode/src/tool/lsp.ts` 暴露九种操作，带文件/行/列参数。提示在 `tool/lsp.txt`。LSP 服务器需按文件类型配置。

```
支持的操作：
- goToDefinition
- findReferences  
- hover
- documentSymbol
- workspaceSymbol
- goToImplementation
- prepareCallHierarchy
- incomingCalls
- outgoingCalls
```

**DeepSeek TUI 需要：** 在 `crates/tui/src/tools/` 新增 `lsp.rs` 工具，与 tower-lsp 或 lsp-server 集成，并按语言配置服务器。

### 2. 细粒度权限系统

**是什么：** 按 工具名 × 文件路径模式 的允许/拒绝/询问规则，支持通配符、家目录展开，并级联到待处理请求。

**为何重要：** 当前全有或全无的审批模型摩擦大。用户无法表达「`src/` 下读永远允许、`.env` 永远要问」。对模式永久批准可在一长会话中减少约 60–80% 审批疲劳。

**OpenCode 实现：** `packages/opencode/src/permission/index.ts`：

- `Action`：`allow | deny | ask`
- `Rule`：`{ permission: string, pattern: string, action: Action }`
- `Ruleset`：有序规则，最后匹配胜出
- `~/`、`$HOME/` 模式展开
- 权限名与路径模式上的通配
- 回复模式：`once`（仅本次）、`always`（永久按模式批准）、`reject`（拒绝本次）
- 自动级联：`always` 可自动消解同会话待定请求
- 区分错误类型：`DeniedError`（规则）、`RejectedError`（用户拒绝）、`CorrectedError`（用户带反馈拒绝）

智能体定义继承可被用户覆盖的权限规则集：
```typescript
build: {
  permission: merge(defaults, { question: "allow", plan_enter: "allow" }, user),
}
plan: {
  permission: merge(defaults, { edit: { "*": "deny" } }, user),
}
explore: {
  permission: merge(defaults, { "*": "deny", grep: "allow", read: "allow", ... }, user),
}
```

**DeepSeek TUI 需要：** 同维度（工具名 × 路径模式 × 动作）的权限规则引擎，持久化到磁盘，并与 hooks 集成使审批可级联。

### 3. 生命周期 Hooks

**是什么：** 用户在特定生命周期事件触发的 shell 命令或插件函数——工具执行前/后、请求权限时、会话开始、用户提交提示、会话停止。

**为何重要：** Hooks 是让用户在不污染系统提示的前提下强制不变量的逃生舱。「写完 `.rs` 永远跑 `cargo fmt`。」「任何 `rm -rf` 前警告我。」可组合、可审计，且不消耗上下文窗口 token。

**Codex CLI 实现：** `codex-rs/hooks/` 定义六种事件类型与类型化请求/响应载荷：

| 事件 | 触发时机 | 载荷 |
|---|---|---|
| `PreToolUse` | 工具执行前 | 工具名、输入参数、沙箱状态 |
| `PostToolUse` | 工具执行后 | 工具名、输入、成功/失败、时长、输出预览 |
| `PermissionRequest` | 模型请求权限 | 权限类型、理由 |
| `SessionStart` | 新会话开始 | 会话 ID、cwd、来源（新建/恢复） |
| `UserPromptSubmit` | 用户发消息 | 提示文本 |
| `Stop` | 会话结束 | 原因 |

每个 hook 处理器支持：
- `matcher`：可选正则，过滤触发该 hook 的调用
- `command`：要运行的 shell
- `timeout_sec`：最长运行时间
- `status_message`：hook 运行时向用户显示
- `source_path` + `source`：定义来源（项目 hooks.json、用户配置、插件）
- Hook 可返回 `Success`、`FailedContinue` 或 `FailedAbort`（阻塞操作）

**DeepSeek TUI 需要：** 扩展 `crates/hooks/` 支持完整事件面、基于 matcher 的过滤，以及类似 Codex CLI 的 `hooks.json` 发现机制。

### 4. 持久记忆

**是什么：** 从对话自动抽取用户偏好、项目惯例与历史决策，存为可检索记忆并注入新会话。

**为何重要：** 长调试会话里智能体反复发现同一事实：「本项目 Rust edition 2024」「测试用 `cargo test --workspace`」「用户偏好 4 空格缩进」。记忆系统复利——每会话建立在先前知识上而非从零开始。

**Codex CLI 实现：** 实验性 `MemoryTool`（`/experimental` 菜单下）支持：
- 记忆生成：模型从对话内容生成结构化记忆
- 记忆检索：相关记忆注入新对话上下文
- `Chronicle` 通过旁路进程增加被动屏幕上下文记忆
- 记忆存 SQLite，TUI 经 `/memories` 暴露

**DeepSeek TUI 需要：** 记忆抽取提示、向量或关键词检索，以及存入现有会话/状态基础设施。

### 5. Skill 自动发现

**是什么：** 自动扫描多路径下的 `SKILL.md`，提供领域说明、脚本与引用。通过 `skill` 工具按需注入对话。

**为何重要：** Skill 是社区打包专业知识的方式——Rust 重构、Docker 部署、GitHub Actions 等——在不膨胀主系统提示的情况下提供专精说明。OpenCode 多路径发现意味着 skill 可项目本地、用户全局或来自 URL。

**OpenCode 实现：** `packages/opencode/src/skill/index.ts` 扫描：

1. `~/.claude/skills/**/SKILL.md`（Claude Code 兼容）
2. `~/.agents/skills/**/SKILL.md`（Agents SDK 兼容）  
3. 从 cwd 到工作区根父目录链上的 `.claude/skills/` 与 `.agents/skills/`
4. 项目配置目录中 `{skill,skills}/**/SKILL.md`
5. 用户配置路径（`~/` 展开）
6. 用户配置 URL（经发现模块拉取）

Skill 解析 YAML frontmatter（`name`、`description`）与 Markdown 正文。重名警告但不报错。Skill 尊重智能体权限——只能加载其权限规则集允许的技能。

**DeepSeek TUI 需要：** 扩展现有 `~/.deepseek/skills/` 发现为父目录遍历、Claude Code 兼容路径与基于 URL 的 skill 源；增加 YAML frontmatter 解析。

---

## 中优先级缺口

能明显改善体验但紧迫性较低。

### 6. 带权限继承的智能体 Profile

**是什么：** 命名智能体类型（build、plan、general、explore）继承不同工具权限集。用户可定义自定义智能体，指定模型、温度、系统提示与权限规则。

**OpenCode 实现：** `packages/opencode/src/agent/agent.ts`：

- `build`：全访问，敏感路径询问
- `plan`：禁止所有编辑工具，允许 plan-exit，允许写 `.opencode/plans/`
- `general`：仅子智能体，禁止 todo-write
- `explore`：只读，允许 grep/glob/read/bash/webfetch/websearch
- 另有内部任务用隐藏智能体（压缩、标题生成、摘要）

每个智能体自带 `model`、`temperature`、`topP`、`prompt`、`permission` 规则集。`generate` 可从自然语言描述动态生成新智能体配置。

**DeepSeek TUI 需要：** 扩展 Plan/Agent/YOLO 模式，支持命名 profile 及每 profile 工具过滤与模型配置。

### 7. Shell 沙箱

**是什么：** 操作系统级强制执行 shell 命令沙箱——网络限制、文件系统只读挂载、允许/禁止路径。

**Codex CLI 实现：** `codex-rs/sandboxing/`：

- macOS：Seatbelt（`sandboxing/src/seatbelt.rs`）与 `.sbpl` 策略文件
- Linux：bubblewrap（默认）或 Landlock（旧回退）
- Windows：restricted token
- 每命令可配置沙箱策略
- 集成测试可检测在沙箱下运行并提前退出

**DeepSeek TUI 需要：** 扩展 `crates/execpolicy/` 支持平台特定沙箱强制执行。先从 macOS Seatbelt 开始（多数 DeepSeek TUI 用户在 macOS）。

### 8. 工具搜索 / 延迟暴露 MCP 工具

**是什么：** 不把全部 MCP 工具 dump 进系统提示（膨胀上下文），而是暴露 `tool_search`，让模型按名或描述发现相关工具。

**Codex CLI 实现：** `ToolSearch`（稳定默认开启）。`ToolSearchAlwaysDeferMcpTools` 更进一步——从不直接暴露 MCP，总需搜索。在 MCP 暴露数百工具时很关键。

**DeepSeek TUI 需要：** `tool_search_tool_regex` 与 `tool_search_tool_bm25` 已存在作延迟发现；扩展为按需搜索门控 MCP 工具暴露。

### 9. ExecPolicy / 命令审批规则

**是什么：** 策略引擎按用户定义规则评估 shell 命令——前缀允许列表、网络限制、模式匹配——并自动批准、拒绝或升级。

**Codex CLI 实现：** `codex-rs/execpolicy/src/`：

- `Policy`：有序 `Rule` 列表
- `Rule`：前缀模式（如允许 `cargo build*`、拒绝 `rm *`）
- `NetworkRule`：协议级网络限制
- `MatchOptions`：规则求值行为
- `Evaluation`：对某条命令的策略求值结果

规则可在运行时经 `blocking_append_allow_prefix_rule` 追加。

**DeepSeek TUI 需要：** 扩展 `crates/execpolicy/` 支持前缀规则、网络规则与运行时策略修正。

### 10. 动态生成智能体

**是什么：** 从自然语言描述即时生成新智能体配置。

**OpenCode 实现：** `agent.ts` 的 `generate` 接受如「只读文件并报告问题的代码审查员」的描述，经结构化 LLM 调用返回 `{ identifier, whenToUse, systemPrompt }`。生成智能体遵守现有名称冲突规则。

**DeepSeek TUI 需要：** 模型可调用的工具或斜杠命令，从描述生成智能体配置并注册到会话。

### 11. 流式 Patch 事件

**是什么：** 在模型生成 `apply_patch` 输入时流式输出结构化进度，让用户实时看到将改哪些文件。

**Codex CLI 实现：** 开发中的 `ApplyPatchStreamingEvents` 在模型产生 patch hunk 时流式输出文件级进度。`apply-patch/src/streaming_parser.rs` 的 `StreamingPatchParser` 做增量解析。

**DeepSeek TUI 需要：** 扩展 `apply_patch.rs` 在流式模型输出期间发出进度事件。

---

## 低优先级缺口

有价值但非核心编码工作流关键路径的专业功能。

| 能力 | 所在 | 备注 |
|---|---|---|
| 图像生成 | Codex CLI `ImageGeneration` | 编程场景小众；文档配图有用 |
| Browser Use | Codex CLI `BrowserUse` | 交互浏览器（点击、输入、截图）。DeepSeek TUI 有 `web_run` 无头 |
| Computer Use | Codex CLI `ComputerUse` | 完整桌面自动化，需桌面应用门控 |
| 实时语音 | Codex CLI `RealtimeConversation` | 语音对话，实验性 |
| 统一 PTY Exec | Codex CLI `UnifiedExec` | 单 PTY shell，跨回合状态快照 |
| Artifacts | Codex CLI `Artifact` | 原生产物渲染工具 |
| Goals | Codex CLI `Goals` | 持久线程目标， survive 压缩与会话重启 |
| Git 提交归属 | Codex CLI `CodexGitCommit` | 模型说明如何正确归属提交 |
| CSV 智能体 spawn | Codex CLI `SpawnCsv` | CSV 驱动的并行智能体任务分发 |
| Shell 快照 | Codex CLI `ShellSnapshot` | 跨回合保存/恢复 shell 状态 |
| 防止空闲睡眠 | Codex CLI `PreventIdleSleep` | 长任务期间保持机器唤醒 |

---

## 架构模式

### OpenCode

**客户端/服务器架构：** TUI 是一种客户端；服务器可由移动应用、桌面或 Web 控制台远程驱动。将智能体运行时与 UI 解耦。

**插件系统：** `packages/opencode/src/plugin/` 支持热加载 JS/TS 插件，增加工具、模型、认证提供者与聊天中间件。插件获得带工具执行、认证与文件系统访问的类型化上下文。

**多提供商：** 不绑定单一 AI 提供商。模型用 provider ID 配置，经 provider 注册表解析。`plugin/codex.ts` 中有 OpenAI Codex 的 OAuth（ChatGPT 订阅集成）。

**配置分层：** 配置从多源（全局、项目、环境变量）加载并按明确优先级合并。

### Codex CLI

**App-Server 协议：** `codex-rs/app-server-protocol/` 定义 TUI 前端与智能体后端之间的版本化 RPC（v2）。新 API 严格经 v2，命名约定（`*Params`/`*Response`/`*Notification`，`resource/method` RPC 名）。

**特性开关：** `codex-rs/features/` 集中 60+ 特性，生命周期阶段（UnderDevelopment、Experimental、Stable、Deprecated、Removed）。特性含元数据（菜单名、描述、公告文本）并可带自定义配置结构。

**Bazel + Cargo 双构建：** Codex CLI 开发与 CI/release 兼用 Cargo 与 Bazel。`find_resource!` 宏与 `cargo_bin()` 辅助抽象 runfile 差异。

**快照测试：** `codex-rs/tui/` 大量使用 `insta` 做 UI 快照测试。UI 变更需对应快照覆盖。

**核心模块化：** 明确抵制向 `codex-core` 堆代码。新功能进专用 crate（`codex-apply-patch`、`codex-memories`、`codex-sandboxing`）而非膨胀核心 crate。

### DeepSeek TUI

**RLM（递归语言模型）：** 该领域独特。沙箱化 Python REPL，子 LLM 可调用辅助函数（`llm_query`、`llm_query_batched`、`rlm_query`）做批处理、分块与递归批评。两竞品无等价物。

**持久任务：** 可重启的持久任务对象，带证据跟踪（gate 运行、PR 尝试、时间线）。为长时自主工作设计，可 survive 重启。

**自动化：** 带 cron 式 RRULE 的定时重复任务。三者中唯一。

---

## DeepSeek TUI 已突出之处

- **RLM** —— Python 沙箱中批量/大块 LLM 处理；竞品无等价
- **金融** —— 实时股票/加密行情；该领域独特
- **自动化** —— 带 cron 规则的定时重复任务
- **持久任务** —— 可重启，带证据跟踪与 gate 验证
- **回合回滚** —— 通过 side-git 快照按回合撤销工作区变更
- **数据校验** —— JSON/TOML 校验工具
- **Web run** —— 无头浏览器交互（Codex 有 Browser Use 但桌面门控）
- **并行工具执行** —— 明确建模为基础设施
- **Git/GitHub** —— 完整 git 模块（blame、log、diff、status）加经 gh 的 GitHub API
- **项目地图** —— 高层项目结构生成

---

## 建议实现顺序

1. **LSP 工具** —— 最大能力缺口。估计减少 30–50% 代码库探索回合。
2. **路径模式权限** —— 长会话减少约 60–80% 审批疲劳。
3. **持久记忆** —— 跨会话复利；长项目基础。
4. **工具使用前/后 hooks** —— 用户自定义护栏，不膨胀系统提示。
5. **Skill 自动发现** —— 社区 skill 生态与 Claude Code 兼容。
6. **智能体 profile** —— 命名类型与模型/权限继承。
7. **MCP 工具搜索** —— 连接超多工具 MCP 时保持上下文可控。
8. **Shell 沙箱** —— 安全改进，从 macOS Seatbelt 起步。
