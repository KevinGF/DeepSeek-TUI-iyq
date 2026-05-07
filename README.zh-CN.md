# DeepSeek TUI

> 面向 DeepSeek V4 的终端编程智能体：通过 `deepseek` 命令运行，流式展示推理块，在带审批门禁的前提下编辑本地工作区，并提供可在每轮自动选择模型与思考档位的 **Auto 模式**。

[English README](README.md)

## 安装

`deepseek` 以 Rust 二进制分发：调度器命令（`deepseek`）与配套的 TUI 运行时（`deepseek-tui`）。任选你习惯的安装方式，它们都会把同名命令放到 `PATH` 上。npm 包是发布版二进制的安装器/封装，**不是**智能体运行时本身。

```bash
# 1. npm —— 若已使用 Node 最省事。包会从 GitHub Releases 下载匹配平台的预编译二进制。
npm install -g deepseek-tui

# 2. Cargo —— 无需 Node。
cargo install deepseek-tui-cli --locked   # `deepseek`（推荐入口）
cargo install deepseek-tui     --locked   # `deepseek-tui`（TUI 二进制）

# 3. Homebrew —— macOS 包管理器。
brew tap Hmbown/deepseek-tui
brew install deepseek-tui

# 4. 直接下载 —— 无需包管理器或工具链。
#    https://github.com/Hmbown/DeepSeek-TUI/releases
#    提供 Linux x64/ARM64、macOS x64/ARM64、Windows x64 预编译包。
```

> 中国大陆访问 npm 较慢时，可使用 `--registry=https://registry.npmmirror.com`，或见下方 [Cargo 镜像](#中国大陆--镜像友好安装)。

[![CI](https://github.com/Hmbown/DeepSeek-TUI/actions/workflows/ci.yml/badge.svg)](https://github.com/Hmbown/DeepSeek-TUI/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/deepseek-tui)](https://www.npmjs.com/package/deepseek-tui)
[![crates.io](https://img.shields.io/crates/v/deepseek-tui-cli?label=crates.io)](https://crates.io/crates/deepseek-tui-cli)
[DeepWiki 项目索引](https://deepwiki.com/Hmbown/DeepSeek-TUI)

![DeepSeek TUI 截图](assets/screenshot.png)

---

## 这是什么？

DeepSeek TUI 是在终端里运行的编程智能体：可读写文件、执行 shell、搜索网页、管理 git，并通过键盘驱动的 TUI 协调子智能体。

面向 **DeepSeek V4**（`deepseek-v4-pro` / `deepseek-v4-flash`）构建，具备约 100 万 token 上下文、流式推理块，以及前缀缓存感知的成本展示。

### 主要功能

- **Auto 模式** —— `--model auto` / `/model auto` 在每轮同时选择模型与思考档位
- **思考模式流式输出** —— 实时查看 DeepSeek 推理块
- **完整工具集** —— 文件、shell、git、网页搜索/浏览、apply-patch、子智能体、MCP 服务器
- **100 万 token 上下文** —— 上下文跟踪、手动或可配置的压缩、前缀缓存遥测
- **三种模式** —— Plan（只读探索）、Agent（交互 + 审批）、YOLO（自动批准）
- **推理强度档位** —— `Shift+Tab` 在 `off → high → max` 间切换
- **会话保存/恢复** —— 检查点与长会话续跑
- **工作区回滚** —— side-git 每轮前后快照，`/restore` 与 `revert_turn`，不碰你仓库自己的 `.git`
- **持久化任务队列** —— 后台任务可跨重启保留
- **HTTP/SSE 运行时 API** —— `deepseek serve --http` 用于无头智能体流程
- **MCP 协议** —— 连接 Model Context Protocol 服务器扩展工具；见 [docs/MCP.md](docs/MCP.md) / [简体](docs/MCP.zh-CN.md)
- **原生 RLM**（`rlm_query`）—— 通过同一 API 客户端调度低成本 `deepseek-v4-flash` 子调用做批处理分析
- **LSP 诊断** —— 每次编辑后由 rust-analyzer、pyright、typescript-language-server、gopls、clangd 等提供行内错误/警告
- **用户记忆** —— 可选持久化笔记注入系统提示，跨会话保留偏好
- **本地化 UI** —— `en`、`ja`、`zh-Hans`、`pt-BR`，支持自动检测
- **实时成本** —— 按轮次与会话的用量与费用估算，含缓存命中/未命中拆分
- **技能系统** —— 可从 GitHub 安装的组合式指令包，无需后端服务

---

## 架构说明

`deepseek`（调度器 CLI）→ `deepseek-tui`（伴随二进制）→ ratatui 界面 ↔ 异步引擎 ↔ OpenAI 兼容流式客户端。工具调用经类型化注册表（shell、文件、git、网络、子智能体、MCP、RLM）路由，结果流回对话记录。引擎管理会话状态、轮次追踪、持久化任务队列，以及 LSP 子系统——在下一轮推理前把编辑后的诊断喂回模型上下文。

完整说明见 [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)（[简体](docs/ARCHITECTURE.zh-CN.md)）。

---

## 快速开始

```bash
npm install -g deepseek-tui
deepseek --version
deepseek --model auto
```

预编译二进制支持 **Linux x64**、**Linux ARM64**（v0.8.8+）、**macOS x64**、**macOS ARM64**、**Windows x64**。其它目标（musl、riscv64、FreeBSD 等）见 [从源码安装](#从源码安装) 或 [docs/INSTALL.md](docs/INSTALL.md)（[简体](docs/INSTALL.zh-CN.md)）。

首次启动会提示输入 [DeepSeek API 密钥](https://platform.deepseek.com/api_keys)。密钥写入 `~/.deepseek/config.toml`，任意目录可用，一般不会弹系统凭据框。

也可提前配置：

```bash
deepseek auth set --provider deepseek   # 写入 ~/.deepseek/config.toml

export DEEPSEEK_API_KEY="YOUR_KEY"      # 环境变量方式；非交互 shell 建议写入 ~/.zshenv
deepseek

deepseek doctor                         # 验证安装
```

若 `deepseek doctor` 提示被拒绝的密钥来自 `DEEPSEEK_API_KEY`，请从 shell 启动文件里删掉过期的 `export`，开一个新 shell，再执行 `deepseek auth set --provider deepseek`。**已保存的配置密钥优先于环境变量**，且更易轮换。

> 轮换或删除已保存密钥：`deepseek auth clear --provider deepseek`。

### Auto 模式

若希望由 DeepSeek TUI 决定每轮用多大模型、多高推理强度，使用 `deepseek --model auto` 或 `/model auto`。

Auto 同时控制两项：

- 模型：`deepseek-v4-flash` 或 `deepseek-v4-pro`
- 思考：`off`、`high` 或 `max`

在发送正式请求前，应用会先发起一次小的 `deepseek-v4-flash` 路由调用（思考关闭）。路由器根据最新请求与近期上下文，为真实请求选定具体模型与思考档位。短/简单轮次可留在 Flash + 思考关；编码、排障、发版、架构、安全审查或模糊多步任务可升到 Pro 和/或更高思考。

`auto` 仅存在于 DeepSeek TUI 本地；上游 API **不会**收到 `model: "auto"`，只会收到该轮选定的具体模型与思考设置。TUI 会显示所选路由，费用按实际运行的模型计费。若路由调用失败或返回无效结果，会回退到本地启发式。子智能体默认继承 Auto，除非你为它们指定显式模型。

若要做可重复基准测试、严格费用上限或固定供应商/模型映射，请改用固定模型或固定思考档位。

### Linux ARM64（树莓派、Asahi、Graviton、HarmonyOS PC 等）

自 v0.8.8 起，`npm i -g deepseek-tui` 支持 glibc 系 ARM64 Linux。也可从 [Releases](https://github.com/Hmbown/DeepSeek-TUI/releases) 下载预编译二进制并放入 `PATH`。

### 中国大陆 / 镜像友好安装

若 GitHub 或 npm 下载较慢，可使用 Cargo 注册表镜像：

```toml
# ~/.cargo/config.toml
[source.crates-io]
replace-with = "tuna"

[source.tuna]
registry = "sparse+https://mirrors.tuna.tsinghua.edu.cn/crates.io-index/"
```

然后安装两个二进制（运行时由调度器调用 TUI）：

```bash
cargo install deepseek-tui-cli --locked   # 提供 `deepseek`
cargo install deepseek-tui     --locked   # 提供 `deepseek-tui`
deepseek --version
```

也可从 [GitHub Releases](https://github.com/Hmbown/DeepSeek-TUI/releases) 下载预编译包。镜像资源可使用 `DEEPSEEK_TUI_RELEASE_BASE_URL`。

### Windows（Scoop）

[Scoop](https://scoop.sh) 是 Windows 包管理器。安装 Scoop 后执行：

```bash
scoop install deepseek-tui
```

<details id="install-from-source">
<summary>从源码安装</summary>

适用于任意 Tier-1 Rust 目标，含 musl、riscv64、FreeBSD 及较老 ARM64 发行版。

```bash
# Linux 构建依赖（Debian/Ubuntu/RHEL）：
#   sudo apt-get install -y build-essential pkg-config libdbus-1-dev
#   sudo dnf install -y gcc make pkgconf-pkg-config dbus-devel

git clone https://github.com/Hmbown/DeepSeek-TUI.git
cd DeepSeek-TUI

cargo install --path crates/cli --locked   # 需 Rust 1.88+；提供 `deepseek`
cargo install --path crates/tui --locked   # 提供 `deepseek-tui`
```

两个二进制都需要。交叉编译与平台说明见 [docs/INSTALL.md](docs/INSTALL.md)（[简体](docs/INSTALL.zh-CN.md)）。

</details>

### 其它 API 提供商

```bash
# NVIDIA NIM
deepseek auth set --provider nvidia-nim --api-key "YOUR_NVIDIA_API_KEY"
deepseek --provider nvidia-nim

# Fireworks
deepseek auth set --provider fireworks --api-key "YOUR_FIREWORKS_API_KEY"
deepseek --provider fireworks --model deepseek-v4-pro

# 自托管 SGLang
SGLANG_BASE_URL="http://localhost:30000/v1" deepseek --provider sglang --model deepseek-v4-flash

# 自托管 vLLM
VLLM_BASE_URL="http://localhost:8000/v1" deepseek --provider vllm --model deepseek-v4-flash
```

---

## v0.8.15 新特性

面向社区的一次稳定化更新，聚焦认证恢复、Windows 终端、Zed/ACP 兼容、降低上手摩擦，以及更清晰的花费展示。[完整变更日志](CHANGELOG.md)。

- **更友好的认证恢复** —— 运行时 API 密钥失败时会说明：当前密钥是否仅来自 `DEEPSEEK_API_KEY`、是否没有已保存的配置密钥
- **Zed / ACP 适配器** —— `deepseek serve --acp` 暴露本地 stdio Agent Client Protocol 服务器，供 Zed 等编辑器使用
- **Windows 终端修复** —— UTF-8 控制台、调度器 resume、剪贴板回退、Ctrl+E 编辑区行为、更安全的 Windows 鼠标默认策略
- **人民币成本展示** —— 设置 `cost_currency = "cny"`（或 `yuan` / `rmb`），页脚、`/cost`、`/tokens` 与通知摘要以人民币展示
- **安装与技能打磨** —— 工作区信任全局持久化；纯 Markdown `SKILL.md` 可正确加载；发现全局 Agents/Cursor 技能路径；斜杠补全展示技能
- **可靠性** —— 工作区范围的 `resume --last`、 capped API `max_tokens`、`deepseek doctor` 端点诊断、npm `--version` 回退、带当前日期的轮次元数据

---

## 使用方式

```bash
deepseek                                         # 交互式 TUI
deepseek "explain this function"                 # 一次性提示
deepseek --model deepseek-v4-flash "summarize"   # 指定模型
deepseek --model auto "fix this bug"             # 自动选模型 + 思考
deepseek --yolo                                  # 自动批准工具
deepseek auth set --provider deepseek            # 保存 API 密钥
deepseek doctor                                  # 检查安装与连通性
deepseek doctor --json                           # 机器可读诊断
deepseek setup --status                          # 只读安装状态
deepseek setup --tools --plugins                 # 脚手架工具/插件目录
deepseek models                                  # 列出线上 API 模型
deepseek sessions                                # 列出已保存会话
deepseek resume --last                           # 恢复当前工作区最近一次会话
deepseek resume <SESSION_ID>                     # 按 UUID 恢复指定会话
deepseek fork <SESSION_ID>                       # 在指定轮次分叉会话
deepseek serve --http                            # HTTP/SSE API 服务
deepseek serve --acp                             # 面向 Zed/自定义智能体的 ACP stdio 适配器
deepseek pr <N>                                  # 拉取 PR 并预填审查提示
deepseek mcp list                                # 列出已配置 MCP 服务器
deepseek mcp validate                            # 校验 MCP 配置与连通性
deepseek mcp-server                              # 运行调度器 MCP stdio 服务器
deepseek update                                  # 检查并应用二进制更新
```

### Zed / ACP

DeepSeek 可作为 **Agent Client Protocol** 自定义服务器，供在 stdio 上启动本地 ACP 智能体的编辑器使用。在 Zed 中添加自定义 agent server：

```json
{
  "agent_servers": {
    "DeepSeek": {
      "type": "custom",
      "command": "deepseek",
      "args": ["serve", "--acp"],
      "env": {}
    }
  }
}
```

当前 ACP 切片支持通过既有 DeepSeek 配置/API 密钥创建新会话与提示回复。**基于工具的编辑与检查点回放尚未通过 ACP 暴露。**

### 常用快捷键

| 按键 | 功能 |
|---|---|
| `Tab` | 补全 `/` 或 `@`；运行中则将草稿排队为后续轮次；否则切换模式 |
| `Shift+Tab` | 推理强度：off → high → max |
| `F1` | 可搜索帮助层 |
| `Esc` | 返回 / 关闭 |
| `Ctrl+K` | 命令面板 |
| `Ctrl+R` | 恢复较早会话 |
| `Alt+R` | 搜索提示历史并恢复已清空草稿 |
| `Ctrl+S` | 暂存当前草稿（`/stash list`、`/stash pop` 取回） |
| `@path` | 在输入区附加文件/目录上下文 |
| `↑`（在输入区行首） | 选择附件行以便删除 |

完整快捷键目录：[docs/KEYBINDINGS.md](docs/KEYBINDINGS.md)（[简体](docs/KEYBINDINGS.zh-CN.md)）。

---

## 模式

| 模式 | 行为 |
| --- | --- |
| **Plan** 🔍 | 只读调查 —— 模型先探索并给出计划（`update_plan` + `checklist_write`），再修改 |
| **Agent** 🤖 | 默认交互 —— 多步工具调用带审批；模型通过 `checklist_write` 列出工作 |
| **YOLO** ⚡ | 在可信工作区自动批准全部工具；仍维护计划与清单以便可见 |

---

## 配置

用户配置：`~/.deepseek/config.toml`。项目覆盖：`<workspace>/.deepseek/config.toml`（禁止：`api_key`、`base_url`、`provider`、`mcp_config_path`）。全部选项见 [config.example.toml](config.example.toml)。

常用环境变量：

| 变量 | 用途 |
|---|---|
| `DEEPSEEK_API_KEY` | API 密钥 |
| `DEEPSEEK_BASE_URL` | API 根 URL |
| `DEEPSEEK_HTTP_HEADERS` | 可选自定义模型请求头，例如 `X-Model-Provider-Id=your-model-provider` |
| `DEEPSEEK_MODEL` | 默认模型 |
| `DEEPSEEK_PROVIDER` | `deepseek`（默认）、`nvidia-nim`、`fireworks`、`sglang`、`vllm` |
| `DEEPSEEK_PROFILE` | 配置 profile 名 |
| `DEEPSEEK_MEMORY` | 设为 `on` 启用用户记忆 |
| `NVIDIA_API_KEY` / `FIREWORKS_API_KEY` / `SGLANG_API_KEY` / `VLLM_API_KEY` | 各提供商认证 |
| `SGLANG_BASE_URL` | 自托管 SGLang 端点 |
| `VLLM_BASE_URL` | 自托管 vLLM 端点 |
| `NO_ANIMATIONS=1` | 启动时强制无障碍模式 |
| `SSL_CERT_FILE` | 企业代理自定义 CA 包 |

UI 语言与模型输出语言无关 —— 在 `settings.toml` 设置 `locale`、使用 `/config locale zh-Hans`，或依赖 `LC_ALL`/`LANG`。详见 [docs/CONFIGURATION.md](docs/CONFIGURATION.md)（[简体](docs/CONFIGURATION.zh-CN.md)）与 [docs/MCP.md](docs/MCP.md)（[简体](docs/MCP.zh-CN.md)）。

### 切换为简体中文界面

1. 在 Composer 输入 `/config`，用 Tab 或 Enter 打开配置面板。  
2. 选择 **Edit locale**，在 `New:` 填入 `zh-Hans`，回车应用。

可选语言：`auto` | `en` | `ja` | `zh-Hans` | `pt-BR`。

也可在 `~/.deepseek/config.toml` 中设置：

```toml
[tui]
locale = "zh-Hans"
```

或通过环境变量（中文系统常已生效）：

```bash
LANG=zh_CN.UTF-8 deepseek
```

更多说明见 [docs/LOCALIZATION.md](docs/LOCALIZATION.md)（[简体](docs/LOCALIZATION.zh-CN.md)）。

---

## 模型与价格

| 模型 | 上下文 | 输入（缓存命中） | 输入（缓存未命中） | 输出 |
|---|---|---|---|---|
| `deepseek-v4-pro` | 1M | $0.003625 / 1M* | $0.435 / 1M* | $0.87 / 1M* |
| `deepseek-v4-flash` | 1M | $0.0028 / 1M | $0.14 / 1M | $0.28 / 1M |

旧别名 `deepseek-chat` / `deepseek-reasoner` 映射到 `deepseek-v4-flash`。NVIDIA NIM 变体以你的 NVIDIA 账号条款为准。

*DeepSeek Pro 当前为限时 75% 折扣价，有效期至 **2026-05-31 15:59 UTC**；之后 TUI 成本估算将回退到 Pro 基础价。*

> [!Note]
> DeepSeek-V4-Pro 最新定价（含当前 75% 折扣，至 2026-05-31 15:59 UTC 有效）请以官方 [定价页](https://api-docs.deepseek.com/zh-cn/quick_start/pricing) 为准。README 中列出的费率与官方发布一致。

---

## 发布你自己的技能

DeepSeek TUI 从工作区目录（`.agents/skills` → `skills` → `.opencode/skills` → `.claude/skills` → `.cursor/skills`）以及全局目录（`~/.agents/skills` → `~/.claude/skills` → `~/.deepseek/skills`）发现技能。每个技能是包含 `SKILL.md` 的目录：

```text
~/.agents/skills/my-skill/
└── SKILL.md
```

需要 YAML frontmatter：

```markdown
---
name: my-skill
description: 当 DeepSeek 应遵循我的自定义工作流时使用。
---

# My Skill
写给智能体的指令。
```

命令：`/skills`（列表）、`/skill <name>`（激活）、`/skill new`（脚手架）、`/skill install github:<owner>/<repo>`（社区）、`/skill update` / `uninstall` / `trust`。从 GitHub 安装无需后端。已安装技能出现在模型可见的会话上下文中；任务匹配描述时，智能体可通过 `load_skill` 自动选用。

---

## 文档

| 文档 | 主题 |
|---|---|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) · [简体](docs/ARCHITECTURE.zh-CN.md) | 代码库内部结构 |
| [CONFIGURATION.md](docs/CONFIGURATION.md) · [简体](docs/CONFIGURATION.zh-CN.md) | 完整配置参考 |
| [MODES.md](docs/MODES.md) · [简体](docs/MODES.zh-CN.md) | Plan / Agent / YOLO 模式 |
| [MCP.md](docs/MCP.md) · [简体](docs/MCP.zh-CN.md) | Model Context Protocol |
| [RUNTIME_API.md](docs/RUNTIME_API.md) · [简体](docs/RUNTIME_API.zh-CN.md) | HTTP/SSE 运行时 API |
| [INSTALL.md](docs/INSTALL.md) · [简体](docs/INSTALL.zh-CN.md) | 各平台安装指南 |
| [MEMORY.md](docs/MEMORY.md) · [简体](docs/MEMORY.zh-CN.md) | 用户记忆功能 |
| [SUBAGENTS.md](docs/SUBAGENTS.md) · [简体](docs/SUBAGENTS.zh-CN.md) | 子智能体角色与生命周期 |
| [KEYBINDINGS.md](docs/KEYBINDINGS.md) · [简体](docs/KEYBINDINGS.zh-CN.md) | 完整快捷键 |
| [TOOL_SURFACE.md](docs/TOOL_SURFACE.md) · [简体](docs/TOOL_SURFACE.zh-CN.md) | 模型可见工具面 |
| [ACCESSIBILITY.md](docs/ACCESSIBILITY.md) · [简体](docs/ACCESSIBILITY.zh-CN.md) | 无障碍与低动效 |
| [LOCALIZATION.md](docs/LOCALIZATION.md) · [简体](docs/LOCALIZATION.zh-CN.md) | UI 语言矩阵与切换 |
| [DOCKER.md](docs/DOCKER.md) · [简体](docs/DOCKER.zh-CN.md) | Docker |
| [capacity_controller.md](docs/capacity_controller.md) · [简体](docs/capacity_controller.zh-CN.md) | 容量控制器 |
| [RELEASE_RUNBOOK.md](docs/RELEASE_RUNBOOK.md) · [简体](docs/RELEASE_RUNBOOK.zh-CN.md) | 发布流程 |
| [OPERATIONS_RUNBOOK.md](docs/OPERATIONS_RUNBOOK.md) · [简体](docs/OPERATIONS_RUNBOOK.zh-CN.md) | 运维与恢复 |
| [COMPETITIVE_ANALYSIS.md](docs/COMPETITIVE_ANALYSIS.md) · [简体](docs/COMPETITIVE_ANALYSIS.zh-CN.md) | 竞品分析（参考） |
| [LEGACY_RUST_AUDIT_0_7_6.md](docs/LEGACY_RUST_AUDIT_0_7_6.md) · [简体](docs/LEGACY_RUST_AUDIT_0_7_6.zh-CN.md) | 旧版 Rust 审计备忘 |
| [V0_7_5_IMPLEMENTATION_PLAN.md](docs/V0_7_5_IMPLEMENTATION_PLAN.md) · [简体](docs/V0_7_5_IMPLEMENTATION_PLAN.zh-CN.md) | v0.7.5 实现计划（已归档索引） |
| [archive/V0_7_5_IMPLEMENTATION_PLAN.md](docs/archive/V0_7_5_IMPLEMENTATION_PLAN.md) · [简体](docs/archive/V0_7_5_IMPLEMENTATION_PLAN.zh-CN.md) | v0.7.5 实现计划（归档正文） |
| [v0.8.8-coordinator-prompt.md](docs/v0.8.8-coordinator-prompt.md) · [简体](docs/v0.8.8-coordinator-prompt.zh-CN.md) | v0.8.8 协调器提示（内部参考） |

完整变更历史：[CHANGELOG.md](CHANGELOG.md)。

---

## 致谢

本项目离不开社区贡献者：

- **[merchloubna70-dot](https://github.com/merchloubna70-dot)** — 28 个 PR，功能、修复与 VS Code 扩展脚手架 (#645–#681)
- **[WyxBUPT-22](https://github.com/WyxBUPT-22)** — Markdown 表格、粗体/斜体、水平线 (#579)
- **[loongmiaow-pixel](https://github.com/loongmiaow-pixel)** — Windows 与中国安装文档 (#578)
- **[20bytes](https://github.com/20bytes)** — 用户记忆文档与帮助文案 (#569)
- **[staryxchen](https://github.com/staryxchen)** — glibc 兼容性预检 (#556)
- **[Vishnu1837](https://github.com/Vishnu1837)** — glibc 兼容性改进 (#565)
- **[shentoumengxin](https://github.com/shentoumengxin)** — Shell `cwd` 边界校验 (#524)
- **[toi500](https://github.com/toi500)** — Windows 粘贴问题报告
- **[xsstomy](https://github.com/xsstomy)** — 终端启动重绘报告
- **[melody0709](https://github.com/melody0709)** — 斜杠前缀回车激活报告
- **[lloydzhou](https://github.com/lloydzhou)** 与 **[jeoor](https://github.com/jeoor)** — 压缩成本报告
- **[Agent-Skill-007](https://github.com/Agent-Skill-007)** — README 清晰度改进 (#685)
- **[woyxiang](https://github.com/woyxiang)** — Windows Scoop 安装文档 (#696)
- **[wangfeng](mailto:wangfengcsu@qq.com)** — 定价/折扣信息 (#692)
- **[zichen0116](https://github.com/zichen0116)** — CODE_OF_CONDUCT.md (#686)
- **[dfwqdyl-ui](https://github.com/dfwqdyl-ui)** — 模型 ID 大小写兼容报告 (#729)
- **[Oliver-ZPLiu](https://github.com/Oliver-ZPLiu)** — `working...` 卡死与 Windows 剪贴板回退 (#738, #850)
- **[reidliu41](https://github.com/reidliu41)** — resume 提示与工作区信任持久化 (#863, #870)
- **[xieshutao](https://github.com/xieshutao)** — 纯 Markdown 技能回退 (#869)
- **[GK012](https://github.com/GK012)** — npm wrapper `--version` 回退 (#885)
- **Hafeez Pizofreude** — `fetch_url` SSRF 保护与 Star History 图表
- **Unic (YuniqueUnic)** — 基于 schema 的配置 UI（TUI + web）
- **Jason** — SSRF 安全加固

---

## 贡献

请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。欢迎 PR —— 参见 [开放 issues](https://github.com/Hmbown/DeepSeek-TUI/issues)。

支持作者：[Buy me a coffee](https://www.buymeacoffee.com/hmbown)。

> [!Note]
> *与 DeepSeek Inc. 无隶属关系。*

## 许可证

[MIT](LICENSE)

## Star 历史

[![Star History Chart](https://api.star-history.com/chart?repos=Hmbown/DeepSeek-TUI&type=date&legend=top-left)](https://www.star-history.com/?repos=Hmbown%2FDeepSeek-TUI&type=date&logscale=&legend=top-left)
