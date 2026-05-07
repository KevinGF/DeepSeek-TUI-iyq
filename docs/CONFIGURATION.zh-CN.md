# 配置

> **English**：[CONFIGURATION.md](CONFIGURATION.md)

DeepSeek TUI 从 TOML 配置文件与环境变量读取配置。进程启动时若存在工作区本地 `.env` 也会加载。以仓库内 `.env.example` 为模板；复制为 `.env` 后只改提供商与安全相关项。

## 查找位置

默认配置路径：

- `~/.deepseek/config.toml`

覆盖方式：

- CLI：`deepseek --config /path/to/config.toml`
- 环境：`DEEPSEEK_CONFIG_PATH=/path/to/config.toml`

若同时设置，`--config` 优先。环境变量覆盖在文件加载之后应用。

### 按项目叠加（#485）

当 TUI 在包含 `<workspace>/.deepseek/config.toml` 的工作区启动时，该文件中的值会合并到全局配置之上。这样仓库可锁定自己的提供商、模型、沙箱策略或审批策略，而无需改用户 `~/.deepseek/config.toml`。单次启动可用 `--no-project-config` 跳过叠加。

项目叠加支持的键（仅顶层字段）：

| 键 | 作用 |
|---|---|
| `provider` | 切换后端（如企业仓库用 `"nvidia-nim"`） |
| `model` | 覆盖 `default_text_model` |
| `api_key` | 每仓库密钥（通常从 `.env` 读，**勿提交**） |
| `base_url` | 指向自建端点 |
| `reasoning_effort` | 对复杂仓库强制 `"high"` / `"max"` |
| `approval_policy` | 仓库观点：`"never"` / `"on-request"` / `"untrusted"` |
| `sandbox_mode` | `"read-only"` / `"workspace-write"` / `"danger-full-access"` |
| `mcp_config_path` | 每仓库 MCP 服务器集合 |
| `notes_path` | 笔记放在仓库内 |
| `max_subagents` | 限制受限仓库并发（限制在 1..=20） |
| `allow_shell` | `false` 时门控 shell 工具 |

叠加有意保持窄——覆盖维护者最可能在贡献者间统一的字段。其它设置（`skills_dir`、hooks、capacity、retry 等）仍为用户全局。若需更多，请开 issue 说明用例。

`deepseek` 门面与 `deepseek-tui` 二进制共享同一配置文件做 DeepSeek 认证与模型默认。`deepseek auth set --provider deepseek`（及旧别名 `deepseek login --api-key ...`）把密钥写入 `~/.deepseek/config.toml`，`deepseek --model deepseek-v4-flash` 会转发为 TUI 进程的 `DEEPSEEK_MODEL`。

对托管或自建 DeepSeek V4 提供商，设 `provider = "nvidia-nim"`、`"fireworks"`、`"sglang"` 或 `"vllm"`，或传 `deepseek --provider <name>`。门面把提供商凭据写入共享用户配置，并将解析后的密钥、base URL、提供商、模型转发给 TUI。可用 `deepseek auth set --provider nvidia-nim --api-key "YOUR_NVIDIA_API_KEY"` 或 `deepseek auth set --provider fireworks --api-key "YOUR_FIREWORKS_API_KEY"` 经门面保存托管密钥。SGLang 与 vLLM 为自建，默认可无 API key。

需要额外请求头的 OpenAI 兼容网关可在顶层或 `[providers.deepseek]` 等提供商表设 `http_headers = { "X-Model-Provider-Id" = "your-model-provider" }`。配置后 DeepSeek TUI 在模型 API 请求上发送这些自定义头。等价环境覆盖为 `DEEPSEEK_HTTP_HEADERS`，逗号分隔 `name=value`，如 `X-Model-Provider-Id=your-model-provider,X-Gateway-Route=dev`。`Authorization` 与 `Content-Type` 由客户端管理，不被此设置覆盖。

要在解析路径初始化 MCP 与 skills 目录，运行 `deepseek-tui setup`。仅脚手架 MCP 用 `deepseek-tui mcp init`。

注意：setup、doctor、mcp、features、sessions、resume/fork、exec、review、eval 是 `deepseek-tui` 的子命令。`deepseek` 门面暴露另一套命令（`auth`、`config`、`model`、`thread`、`sandbox`、`app-server`、`mcp-server`、`completion`），并把纯提示转发给 `deepseek-tui`。

## Profile

可在同一文件定义多个 profile：

```toml
api_key = "PERSONAL_KEY"
default_text_model = "deepseek-v4-pro"

[profiles.work]
api_key = "WORK_KEY"
base_url = "https://api.deepseek.com"

[profiles.nvidia-nim]
provider = "nvidia-nim"
api_key = "NVIDIA_KEY"
base_url = "https://integrate.api.nvidia.com/v1"
default_text_model = "deepseek-ai/deepseek-v4-pro"

[profiles.fireworks]
provider = "fireworks"
default_text_model = "accounts/fireworks/models/deepseek-v4-pro"

[profiles.sglang]
provider = "sglang"
base_url = "http://localhost:30000/v1"
default_text_model = "deepseek-ai/DeepSeek-V4-Pro"

[profiles.vllm]
provider = "vllm"
base_url = "http://localhost:8000/v1"
default_text_model = "deepseek-ai/DeepSeek-V4-Pro"
```

选择 profile：

- CLI：`deepseek --profile work`
- 环境：`DEEPSEEK_PROFILE=work`

若选中 profile 但缺失，DeepSeek TUI 会列出可用 profile 并带错误退出。

## 环境变量

以下覆盖配置值：

- `DEEPSEEK_API_KEY`
- `DEEPSEEK_BASE_URL`
- `DEEPSEEK_HTTP_HEADERS`（自定义模型请求头，逗号分隔 `name=value`）
- `DEEPSEEK_PROVIDER`（`deepseek|nvidia-nim|openrouter|novita|fireworks|sglang|vllm`）
- `DEEPSEEK_MODEL` 或 `DEEPSEEK_DEFAULT_TEXT_MODEL`
- `NVIDIA_API_KEY` 或 `NVIDIA_NIM_API_KEY`（`nvidia-nim` 时优先；回退 `DEEPSEEK_API_KEY`）
- `NVIDIA_NIM_BASE_URL`、`NIM_BASE_URL` 或 `NVIDIA_BASE_URL`
- `NVIDIA_NIM_MODEL`
- `FIREWORKS_API_KEY`
- `FIREWORKS_BASE_URL`
- `SGLANG_BASE_URL`
- `SGLANG_MODEL`
- `SGLANG_API_KEY`（可选；许多本地 SGLang 不需认证）
- `VLLM_BASE_URL`
- `VLLM_MODEL`
- `VLLM_API_KEY`（可选；许多本地 vLLM 不需认证）
- `DEEPSEEK_LOG_LEVEL` 或 `RUST_LOG`（`info`/`debug`/`trace` 开启轻量详细日志）
- `DEEPSEEK_SKILLS_DIR`
- `DEEPSEEK_MCP_CONFIG`
- `DEEPSEEK_NOTES_PATH`
- `DEEPSEEK_MEMORY`（`1|on|true|yes|y|enabled` 打开用户记忆）
- `DEEPSEEK_MEMORY_PATH`
- `DEEPSEEK_ALLOW_SHELL`（`1`/`true` 启用）
- `DEEPSEEK_APPROVAL_POLICY`（`on-request|untrusted|never`）
- `DEEPSEEK_SANDBOX_MODE`（`read-only|workspace-write|danger-full-access|external-sandbox`）
- `DEEPSEEK_MANAGED_CONFIG_PATH`
- `DEEPSEEK_REQUIREMENTS_PATH`
- `DEEPSEEK_MAX_SUBAGENTS`（限制在 `1..=20`）
- `DEEPSEEK_TASKS_DIR`（运行时任务队列/产物存储，默认 `~/.deepseek/tasks`）
- `DEEPSEEK_ALLOW_INSECURE_HTTP`（`1`/`true` 允许非本地 `http://` base URL；默认拒绝）
- `DEEPSEEK_CAPACITY_ENABLED`
- `DEEPSEEK_CAPACITY_LOW_RISK_MAX`
- `DEEPSEEK_CAPACITY_MEDIUM_RISK_MAX`
- `DEEPSEEK_CAPACITY_SEVERE_MIN_SLACK`
- `DEEPSEEK_CAPACITY_SEVERE_VIOLATION_RATIO`
- `DEEPSEEK_CAPACITY_REFRESH_COOLDOWN_TURNS`
- `DEEPSEEK_CAPACITY_REPLAN_COOLDOWN_TURNS`
- `DEEPSEEK_CAPACITY_MAX_REPLAY_PER_TURN`
- `DEEPSEEK_CAPACITY_MIN_TURNS_BEFORE_GUARDRAIL`
- `DEEPSEEK_CAPACITY_PROFILE_WINDOW`
- `DEEPSEEK_CAPACITY_PRIOR_CHAT`
- `DEEPSEEK_CAPACITY_PRIOR_REASONER`
- `DEEPSEEK_CAPACITY_PRIOR_V4_PRO`
- `DEEPSEEK_CAPACITY_PRIOR_V4_FLASH`
- `DEEPSEEK_CAPACITY_PRIOR_FALLBACK`
- `NO_ANIMATIONS`（`1|true|yes|on` 在启动时强制 `low_motion = true` 与 `fancy_animations = false`，无论已存设置；见 [`docs/ACCESSIBILITY.md`](./ACCESSIBILITY.md)（[简体](ACCESSIBILITY.zh-CN.md)））
- `SSL_CERT_FILE` —— 企业代理 / TLS 检查 MITM 用户指向 PEM 包（或单 DER 证书），证书会与平台系统信任库一并加载。失败记警告并继续——系统根仍生效。

### 指令来源（`instructions = [...]`，#454）

附加系统提示来源列表，按声明顺序与自动加载的 `AGENTS.md` 拼接：

```toml
instructions = [
    "./AGENTS.md",
    "~/.deepseek/global.md",
    "~/team/agents-shared.md",
]
```

规则：

- 路径经 `expand_path` 展开，`~` 与环境变量可用。
- 每文件上限 100 KiB；超大文件截断并带 `[…elided]` 标记而非跳过。
- 缺失文件跳过并打 tracing 警告，避免陈旧项导致启动失败。
- 项目配置（`<workspace>/.deepseek/config.toml`）对用户数组为**整表替换**而非合并。若两者都要，在项目数组中列出 `~/global.md`。项目设 `instructions = []` 可清除该仓库的用户列表。

### `/hooks` 列表

在 TUI 内运行 `/hooks`（或 `/hooks list`）可按事件分组查看每个已配置生命周期 hook 的名称、命令预览、超时与条件。顶部显示 `[hooks].enabled` 状态，全局抑制时一目了然。Hook 在 `[[hooks.hooks]]` 下配置——完整 schema 见现有 hook 系统文档。

### Composer stash（`/stash`，Ctrl+S）

在 composer 按 **Ctrl+S** 将当前草稿暂存到 `~/.deepseek/composer_stash.jsonl`。`/stash list` 显示带一行预览与时间戳的条目；`/stash pop` 恢复最近一条（LIFO）；`/stash clear` 清空文件。上限 200 条；多行草稿可完整往返。

## 设置文件（持久 UI 偏好）

DeepSeek TUI 还将用户偏好存于：

- `~/.config/deepseek/settings.toml`

显著项含 `auto_compact`（默认 `false`），仅在接近活动模型上限时选择替换式摘要。默认 V4 路径保留稳定消息前缀以利缓存复用；仅当你明确要自动替换压缩时才开 `auto_compact`，否则用手动 `/compact`。可在 TUI 用 `/settings` 与 `/config`（交互编辑器）查看或修改。

常见键：

- `theme`（default、dark、light、whale）
- `auto_compact`（开/关，默认关）
- `paste_burst_detection`（开/关，默认开）：无 bracketed-paste 事件的终端上备用快速按键粘贴检测，与终端 bracketed-paste 模式独立。
- `show_thinking`（开/关）
- `show_tool_details`（开/关）
- `locale`（`auto`、`en`、`ja`、`zh-Hans`、`pt-BR`；默认 `auto`）：UI 外壳语言。`auto` 查 `LC_ALL`、`LC_MESSAGES`、`LANG`；不支持或缺失则回退英语。**不**强制模型输出语言。
- `cost_currency`（`usd`、`cny`；默认 `usd`）：页脚、上下文面板、`/cost`、`/tokens` 与长回合通知摘要所用货币。别名 `rmb`、`yuan` 规范为 `cny`。
- `default_mode`（agent、plan、yolo；旧值 `normal` 接受并规范为 `agent`）
- `max_history`（已提交输入历史条数；清除的草稿仍保留在本地供 composer 历史搜索）
- `default_model`（模型名覆盖）

UI 可见模式仅 `agent`、`plan`、`yolo`。兼容旧文件 `default_mode = "normal"` 仍加载为 `agent`，隐藏斜杠 `/normal` 可切到 `Agent`。

本地化范围见 [LOCALIZATION.md](LOCALIZATION.md)（[简体](LOCALIZATION.zh-CN.md)）。v0.7.6 核心包仅覆盖高可见 TUI 外壳；提供商/工具 schema、人格提示与完整文档除非另行翻译仍为英语。

可读性语义：

- 选择样式在 transcript、composer 菜单与模态间统一。
- 页脚提示使用专用语义角色（`FOOTER_HINT`），保证各主题下可读。
- 页脚含紧凑 `coherence` 芯片，描述当前会话稳定与专注程度。可能状态：`healthy`、`crowded`、`refreshing`、`verifying`、`resetting`；由容量与压缩事件推导，正常 UI 不暴露内部公式。

### Token 数量与驱动因素

DeepSeek V4 前缀缓存使 token 标签重要。以下量分开维护：

| 量 | 含义 | 允许驱动 |
|---|---|---|
| 活动请求输入估计 | 下一请求 live 系统提示与 transcript 负载的保守估计。 | 页眉/页脚上下文百分比、硬循环触发、可选 Flash 接缝触发、紧急溢出预检。 |
| 预留响应余量 | 请求的 `max_tokens` 预算加安全余量。v0.7.5 普通回合 `262144` 输出 token，上下文窗口检查另加 `1024` 安全 token。 | 仅硬循环与紧急溢出预算检查。 |
| 累计 API 用量 | 已完成 API 调用输入加输出 token 之和；多工具回合可能对同一稳定前缀重复计数。 | 仅会话用量与近似成本遥测。 |
| 提示缓存 命中/未命中 | 提供商缓存遥测（最近调用，若可用）。 | 仅缓存命中展示与成本估计；永不驱动压缩、接缝或循环触发。 |
| 上下文百分比 | 活动请求输入估计除以模型上下文窗口。 | 仅展示；镜像上下文防护所用的活动输入基准。 |
| 成本估计 | 由提供商用量与配置的 DeepSeek 费率近似花费。 | 仅展示。 |

默认 V4 路径下，当活动输入达到（配置的循环阈值 `768000`）与（模型窗口减预留响应余量）的**较小值**时触发硬循环。替换压缩仍为可选（默认 `auto_compact = false`），Flash 接缝管理器仍为可选（`[context].enabled = false`），容量控制器除非配置否则关闭。

### 命令迁移说明

从旧版本升级时：

- 旧：`/deepseek` → 新：`/links`（别名：`/dashboard`、`/api`）
- 旧：`/set model deepseek-reasoner` → 新：`/config` 编辑 `model` 行为 `deepseek-v4-pro` 或 `deepseek-v4-flash`
- 旧：可见 `Normal` 模式或 `default_mode = "normal"` → 新：用 `Agent` / `default_mode = "agent"`；旧 `normal` 仍映射 `agent`
- 旧：在斜杠 UX/帮助里发现 `/set` → 新：编辑用 `/config`，只读查看用 `/settings`

## 关键参考

### 核心键（TUI/引擎使用）

- `provider`（string，可选）：`deepseek`（默认）、`deepseek-cn`、`nvidia-nim`、`openrouter`、`novita`、`fireworks`、`sglang` 或 `vllm`。`deepseek-cn` 使用中国大陆端点（`https://api.deepseeki.com`）；`nvidia-nim` 经 `https://integrate.api.nvidia.com/v1` 指向 NVIDIA NIM 托管 DeepSeek；`fireworks` 为 `https://api.fireworks.ai/inference/v1`；`sglang` 为自建 OpenAI 兼容端点，默认 `http://localhost:30000/v1`；`vllm` 为自建 vLLM OpenAI 兼容端点，默认 `http://localhost:8000/v1`。
- `api_key`（string，必填）：须非空（或设 `DEEPSEEK_API_KEY`）。
- `base_url`（string，可选）：DeepSeek OpenAI 兼容 Chat Completions 默认 `https://api.deepseek.com`；`provider = "deepseek-cn"` 时为 `https://api.deepseeki.com`；托管/自建用各提供商端点。`https://api.deepseek.com/v1` 为 SDK 兼容也可接受；仅 DeepSeek beta 功能（严格工具模式、chat prefix、FIM 等）用 `https://api.deepseek.com/beta`。
- `default_text_model`（string，可选）：DeepSeek 默认 `deepseek-v4-pro`；NVIDIA NIM 默认 `deepseek-ai/deepseek-v4-pro`；Fireworks 默认 `accounts/fireworks/models/deepseek-v4-pro`；SGLang 默认 `deepseek-ai/DeepSeek-V4-Pro`。当前公开 DeepSeek ID 为 `deepseek-v4-pro` 与 `deepseek-v4-flash`，均 1M 上下文且默认开启 thinking。旧别名 `deepseek-chat`、`deepseek-reasoner` 仍兼容映射到 `deepseek-v4-flash`。各提供商在支持处将 `deepseek-v4-pro` / `deepseek-v4-flash` 映射为自身模型 ID。用 `/models` 或 `deepseek models` 从已配置端点发现实时 ID。`DEEPSEEK_MODEL` 单进程覆盖此项。
- `reasoning_effort`（string，可选）：`off`、`low`、`medium`、`high`、`max`；默认随已配置 UI 档位。DeepSeek 平台收顶层 `thinking` / `reasoning_effort`。NVIDIA NIM 经 `chat_template_kwargs` 传等价设置。
- `allow_shell`（bool，可选）：默认 `true`（沙箱内）。
- `approval_policy`（string，可选）：`on-request`、`untrusted`、`never`。运行时 `/config` 中 `approval_mode` 编辑也接受 `on-request` 与 `untrusted` 别名。
- `sandbox_mode`（string，可选）：`read-only`、`workspace-write`、`danger-full-access`、`external-sandbox`。
- `managed_config_path`（string，可选）：在用户/环境配置之后加载的托管配置文件。
- `requirements_path`（string，可选）：用于强制允许的审批/沙箱值的 requirements 文件。
- `max_subagents`（int，可选）：默认 `10`，限制在 `1..=20`。
- `subagents.*`（可选）：`agent_spawn` 等子智能体工具按角色/类型的模型默认。显式工具 `model` 优先，其次角色/类型覆盖，再父运行时模型。便捷键：`default_model`、`worker_model`、`explorer_model`、`awaiter_model`、`review_model`、`custom_model`、`max_concurrent`。`[subagents] max_concurrent` 覆盖顶层 `max_subagents`，亦限制在 `1..=20`。`[subagents.models]` 接受小写角色或类型键如 `worker`、`explorer`、`general`、`explore`、`plan`、`review`。spawn 前值须规范到支持的 DeepSeek 模型 id。
- `skills_dir`（string，可选）：默认 `~/.deepseek/skills`（每 skill 为含 `SKILL.md` 的目录）。若存在则优先工作区 `.agents/skills` 或 `./skills`；运行时还发现全局 agentskills.io 兼容 `~/.agents/skills` 与更广 Claude 生态 `~/.claude/skills`。
- `mcp_config_path`（string，可选）：默认 `~/.deepseek/mcp.json`。在 `/config` 可见并可从 TUI 修改。新路径立即被 `/mcp` 使用，但重建模型可见 MCP 工具池需重启 TUI。
- `notes_path`（string，可选）：默认 `~/.deepseek/notes.txt`，供 `note` 工具使用。
- `[memory].enabled`（bool，可选）：默认 `false`。为 `true` 时 TUI 将用户记忆文件载入 `<user_memory>` 提示块、在 composer 启用 `# foo` 快速捕获、暴露 `/memory` 斜杠命令并注册 `remember` 工具。亦可用 `DEEPSEEK_MEMORY=on`。
- `memory_path`（string，可选）：默认 `~/.deepseek/memory.md`。启用用户记忆时使用——完整功能见 [`MEMORY.md`](MEMORY.md)（[简体](MEMORY.zh-CN.md)）（`# foo` 前缀、`/memory`、`remember`、可选开关）。
- `snapshots.*`（可选）：文件回滚用 side-git 工作区快照：
  - `[snapshots].enabled`（bool，默认 `true`）
  - `[snapshots].max_age_days`（int，默认 `7`）
  - 快照位于 `~/.deepseek/snapshots/<project_hash>/<worktree_hash>/.git`，**从不**使用工作区自身 `.git`
- `context.*`（可选）：追加式 Flash 接缝管理器，当前可选。阈值使用**活动请求输入估计**，非终身累计 API 用量：
  - `[context].enabled`（bool，默认 `false`）
  - `[context].verbatim_window_turns`（int，默认 `16`）
  - `[context].l1_threshold`（int，默认 `192000`）
  - `[context].l2_threshold`（int，默认 `384000`）
  - `[context].l3_threshold`（int，默认 `576000`）
  - `[context].cycle_threshold`（int，默认 `768000`）
  - `[context].seam_model`（string，默认 `deepseek-v4-flash`）
- `retry.*`（可选）：API 请求重试/退避：
  - `[retry].enabled`（bool，默认 `true`）
  - `[retry].max_retries`（int，默认 `3`）
  - `[retry].initial_delay`（float 秒，默认 `1.0`）
  - `[retry].max_delay`（float 秒，默认 `60.0`）
  - `[retry].exponential_base`（float，默认 `2.0`）
- `capacity.*`（可选）：运行时上下文容量控制器。可选因其主动干预可能改写 live transcript。
  - `[capacity].enabled`（bool，默认 `false`）
  - `[capacity].low_risk_max`（float，默认 `0.50`）
  - `[capacity].medium_risk_max`（float，默认 `0.62`）
  - `[capacity].severe_min_slack`（float，默认 `-0.25`）
  - `[capacity].severe_violation_ratio`（float，默认 `0.40`）
  - `[capacity].refresh_cooldown_turns`（int，默认 `6`）
  - `[capacity].replan_cooldown_turns`（int，默认 `5`）
  - `[capacity].max_replay_per_turn`（int，默认 `1`）
  - `[capacity].min_turns_before_guardrail`（int，默认 `4`）
  - `[capacity].profile_window`（int，默认 `8`）
  - `[capacity].deepseek_v3_2_chat_prior`（float，默认 `3.9`）
  - `[capacity].deepseek_v3_2_reasoner_prior`（float，默认 `4.1`）
  - `[capacity].deepseek_v4_pro_prior`（float，默认 `3.5`）
  - `[capacity].deepseek_v4_flash_prior`（float，默认 `4.2`）
  - `[capacity].fallback_default_prior`（float，默认 `3.8`）
- `[notifications].method`（string，可选）：`auto`、`osc9`、`bel`、`off`。默认 `auto`。TUI 在**成功完成**且耗时满足 `threshold_secs` 的回合触发；失败与取消回合静默。`auto` 对 `iTerm.app`、`Ghostty`、`WezTerm`（经 `$TERM_PROGRAM`）解析为 `osc9`；否则 macOS/Linux 回退 `bel`，**Windows 回退 `off`**（因 BEL 映射系统错误提示音——见下文「Notifications」小节与 #583）。
- `[notifications].threshold_secs`（int，可选）：默认 `30`。仅耗时达到或超过此的已完成回合发通知。
- `[notifications].include_summary`（bool，可选）：默认 `false`。为 `true` 时通知正文含耗时与回合成本（显示货币）。
- `tui.alternate_screen`（string，可选）：`auto`、`always`、`never`。`auto` 在 Zellij 中禁用备用屏幕；`--no-alt-screen` 强制行内模式。需要真实终端滚回时设 `never` 或带 `--no-alt-screen` 运行。
- `tui.mouse_capture`（bool，可选；非 Windows 且备用屏幕激活时默认 `true`；Windows 与 JetBrains JediTerm——PyCharm/IDEA/CLion 等——默认 `false`，因鼠标事件转义会漏进输入流成乱码，见 #878 / #898）：启用内部滚轮、transcript 选择与右键上下文。TUI 拖拽选择仅复制用户/助手 transcript 文本。要原始终端选择设 `false` 或 `--no-mouse-capture`；在默认关闭处要显式打开设 `true` 或 `--mouse-capture`。
- `tui.terminal_probe_timeout_ms`（int，可选，默认 `500`）：启动时终端模式探测超时毫秒。限制在 `100..=5000`；超时打警告并中止启动而非无限挂起。
- `tui.osc8_links`（bool，可选，默认 `true`）：在 transcript URL 周围发 OSC 8 转义，支持终端（iTerm2、Terminal.app 13+、Ghostty、Kitty、WezTerm、Alacritty、较新 gnome-terminal/konsole）可 Cmd+点链。不支持则显示纯 URL 并忽略转义。误渲染的终端设 `false`；选择/剪贴板输出始终剥除转义。
- `hooks`（可选）：生命周期 hook 配置（见 `config.example.toml`）。
- `features.*`（可选）：特性开关覆盖（见下）。

### 用户记忆

用户记忆分一个顶层路径与一个可选开关表：

```toml
memory_path = "~/.deepseek/memory.md"

[memory]
enabled = true
```

说明：

- `memory_path` 与 `notes_path`、`skills_dir` 同在顶层；**不**嵌在 `[memory]` 下。
- `DEEPSEEK_MEMORY_PATH` 从环境覆盖文件路径。
- `DEEPSEEK_MEMORY=on`（及 `1`、`true`、`yes`、`y`、`enabled`）可在不编辑 `config.toml` 时打开功能。
- 关闭时功能惰性：不注入文件、`# foo` 仍作普通提交、模型不见 `remember` 工具。
- 示例与完整 `/memory` 界面见 [`MEMORY.md`](MEMORY.md)（[简体](MEMORY.zh-CN.md)）。

### Notifications（通知）

TUI 可在回合**成功完成**且耗时超过阈值时发桌面通知（OSC 9 转义或纯 BEL），便于长时间任务时切走标签页。失败或取消回合**有意静默**——通知表示「任务就绪」，非泛用 ping。配置在 `[notifications]`：

```toml
[notifications]
method          = "auto"  # auto | osc9 | bel | off
threshold_secs  = 30      # 仅当回合耗时 >= 此秒数时通知
include_summary = false   # 是否在正文中含耗时 + 成本
```

`method` 语义：

- `auto`（默认）——对 `iTerm.app`、`Ghostty`、`WezTerm`（`$TERM_PROGRAM`）选 `osc9`。macOS/Linux 否则回退 `bel`。**Windows 回退 `off` 而非 `bel`**，因 Windows 音频栈把 `\x07` 映射为 `SystemAsterisk` / `MB_OK` 提示音——与应用错误弹窗相同，成功回合通知听起来像错误（#583）。
- `osc9` —— 发 `\x1b]9;<msg>\x07`。tmux 内用 DCS 透传包裹以到达外层终端。
- `bel` —— 单字节 `\x07`。仅当你**主动**要在 Windows 上要提示音时用。
- `off` —— 完全关闭回合后通知。

在已知 OSC-9 终端（如 Windows 上 WezTerm）运行的 Windows 用户仍可得 OSC-9；`off` 回退仅当未识别 `TERM_PROGRAM` 时。

### 已解析但当前未使用（预留给未来版本）

以下键配置加载器接受，但交互 TUI 与内置工具当前不使用：

- `tools_file`

## 特性开关

特性在 `[features]` 表，跨 profile 合并。内置工具默认开启，只需写要强制开/关的项。

```toml
[features]
shell_tool = true
subagents = true
web_search = true # 启用规范 web.run 及兼容别名 web_search
apply_patch = true
mcp = true
exec_policy = true
```

单次运行也可覆盖：

- `deepseek-tui --enable web_search`
- `deepseek-tui --disable subagents`

用 `deepseek-tui features list` 查看已知标志与有效状态。

## 本地媒体附件

在 composer 用 `@path/to/file` 为下一条消息添加本地文本文件或目录上下文。本地图/视频用 `/attach <path>`，或 `Ctrl+V` 从剪贴板贴图。DeepSeek 公共 Chat Completions API 当前接受文本消息内容，故媒体附件以显式本地路径引用发送，而非原生图/视频负载。附件行在 composer 上方；移到 composer 开头按 `↑` 选附件行，再 `Backspace` 或 `Delete` 移除，无需手改占位文本。

## 托管配置与 Requirements

DeepSeek TUI 支持策略分层：

1. 用户配置 + profile + 环境覆盖
2. 托管配置（若存在）
3. requirements 校验（若存在）

Unix 默认：

- 托管：`/etc/deepseek/managed_config.toml`
- requirements：`/etc/deepseek/requirements.toml`

Requirements 文件形状：

```toml
allowed_approval_policies = ["on-request", "untrusted", "never"]
allowed_sandbox_modes = ["read-only", "workspace-write"]
```

若配置违反 requirements，启动失败并给出描述性错误。

公式、干预行为与遥测见 `docs/capacity_controller.md`（[简体](capacity_controller.zh-CN.md)）。

## 关于 `deepseek-tui doctor`

`deepseek-tui doctor` 与其余 TUI 使用相同配置解析。故尊重 `--config` / `DEEPSEEK_CONFIG_PATH`，MCP/skills 检查用解析后的 `mcp_config_path` / `skills_dir`（含环境覆盖）。

缺路径可运行 `deepseek-tui setup --all`。也可 `deepseek-tui setup --skills --local` 创建工作区本地 `./skills` 目录。

`deepseek-tui doctor --json` 打印机器可读报告并跳过实时 API 连通性探测。顶层键：`version`、`config_path`、`config_present`、`workspace`、`api_key.source`、`base_url`、`default_text_model`、`mcp`、`skills`、`tools`、`plugins`、`sandbox`、`platform`、`api_connectivity`、`capability`。CI 应依赖 `api_key.source`（`env`/`config`/`missing`）而非解析人类可读 `doctor` 文本。

`capability` 键含由各提供商静态知识（release 文档、API 指南）推导的能力信息，非实时探测。顶层子键：`resolved_provider`、`resolved_model`、`context_window`、`max_output`、`thinking_supported`、`cache_telemetry_supported`、`request_payload_mode`、`deprecation`。若解析模型为已知旧别名（如 `deepseek-chat`、`deepseek-reasoner`），`deprecation` 子对象含 `alias`、`replacement`、`notice`。

脚本做上下文窗口预算可用 `capability.context_window` 与 `capability.max_output`。是否配置 reasoning effort 看 `capability.thinking_supported`。旧模型别名警告用 `capability.deprecation`。

## setup 状态、clean 与扩展目录

`deepseek-tui setup` 除既有 `--mcp`、`--skills`、`--local`、`--all`、`--force` 外还支持：

- `--status` —— 打印紧凑一屏状态（api key、base URL、model、MCP/skills/tools/plugins 数量、沙箱、`.env` 是否存在）。只读、无网络；CI 安全。若缺 `.env` 且工作区有 `.env.example`，状态输出提示 `cp .env.example .env`。
- `--tools` —— 脚手架 `~/.deepseek/tools/`，含 `README.md` 说明自描述 frontmatter 约定（`# name:` / `# description:` / `# usage:`）及遵循它的 `example.sh`。目录**不**自动加载；经 MCP、hooks 或 skills 将各脚本接入智能体。
- `--plugins` —— 脚手架 `~/.deepseek/plugins/`，含 `README.md` 与 `example/PLUGIN.md` 占位，frontmatter 形状同 `SKILL.md`。插件也不自动加载；要在 skill 或 MCP 包装中引用才激活。
- `--all` 现同时脚手架 MCP + skills + tools + plugins。
- `--clean` —— 列出 `~/.deepseek/sessions/checkpoints/latest.json` 与 `offline_queue.json`（若存在）。传 `--force` 才真正删除。**从不**碰真实会话历史或任务队列。

`--status` 与 `--clean` 与脚手架类标志互斥。

## 引擎为何剥离 XML/`[TOOL_CALL]` 文本

DeepSeek TUI 仅经 API 工具通道收发工具调用（结构化 `tool_use` / `tool_call` 项）。`crates/tui/src/core/engine.rs` 中流式循环识别一组假封装起始标记——`[TOOL_CALL]`、`<deepseek:tool_call`、`<tool_call`、`<invoke `、`<function_calls>`——并从可见助手文本中清除，**从不**将其变为结构化工具调用。剥离封装时循环每回合发一条紧凑 `status` 通知，便于用户知悉可见文本缩短原因。任何重新启用基于文本的工具执行都应视为回归；`crates/tui/tests/protocol_recovery.rs` 中的协议恢复测试锁定该契约。

