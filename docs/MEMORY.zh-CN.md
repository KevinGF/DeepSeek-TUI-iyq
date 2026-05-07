# 用户记忆

> **English**：[MEMORY.md](MEMORY.md)

用户记忆功能为模型提供一小块**持久化笔记文件**，在每一轮注入系统提示。适合存放跨会话仍应生效的偏好与约定——例如「优先 pytest 而非 unittest」「本仓库四空格缩进」「提交前总是跑 `cargo fmt`」——而无需每轮重复。

记忆为**可选**。关闭时（默认）不加载、不拦截，模型也看不到 `remember` 工具。未启用该功能的用户零额外开销。

## 启用记忆

环境变量：

```bash
export DEEPSEEK_MEMORY=on
```

接受的真值：`1`、`on`、`true`、`yes`、`y`、`enabled`。

或在 `~/.deepseek/config.toml`：

```toml
[memory]
enabled = true
```

切换后重启 TUI。关闭则反向操作。

默认记忆文件为 `~/.deepseek/memory.md`；可在 `config.toml` 用 `memory_path` 或环境变量 `DEEPSEEK_MEMORY_PATH` 覆盖。**同时设置时 `DEEPSEEK_MEMORY_PATH` 优先于配置文件。**

## 快速示例

```text
# remember that this repo prefers cargo fmt before commits
/memory
/memory path
/memory edit
/memory help
```

- 在合成器输入 `# remember that this repo prefers cargo fmt before commits` 可追加带时间戳的条目，**不**触发回合。
- `/memory` 查看解析路径与当前内容。
- `/memory edit` 若要在编辑器中手工整理文件。

## 注入内容

启用且文件存在时，每轮系统提示会多一块：

```xml
<user_memory source="/Users/you/.deepseek/memory.md">
- (2026-05-03 22:14 UTC) prefer pytest over unittest
- (2026-05-03 22:31 UTC) this codebase uses 4-space indentation
…
</user_memory>
```

该块位于提示组装中易失内容边界**之上**，以便在 DeepSeek 前缀缓存中跨轮复用。文件在每次构建提示时读取——通过 `/memory` 或外部编辑器的修改**下一轮即生效**，无需重启。

大于 100 KiB 的文件仍会加载但截断，并附加标记标明截断点。

## 三种添加方式

### 1. 合成器 `# ` 前缀（#492）

在合成器输入以 `#` 开头的**单行**（但非 `##` 或 `#!`）：

```
# remember to use 4-space indentation in this repo
```

TUI 拦截输入并将带时间戳条目追加到记忆文件。**不发起回合**——输入被消费，状态行确认写入路径，你可继续输入真正问题。

多 `#` 前缀故意**不**拦截，以便粘贴 Markdown 标题时不出意料。

### 2. `/memory` 斜杠命令（#491）

检查、清空或获取编辑提示：

| 子命令 | 效果 |
|---------------------|--------------------------------------------------------|
| `/memory` | 内联显示解析路径与当前内容 |
| `/memory show` | 与无参形式相同 |
| `/memory path` | 仅打印解析路径 |
| `/memory clear` | 用空标记替换文件 |
| `/memory edit` | 打印 `${VISUAL:-${EDITOR:-vi}} <path>` 行 |
| `/memory help` | 显示命令帮助与当前路径 |

`/memory edit` **仅打印**命令而不在进程内启动编辑器——保持斜杠处理器简单，与所用编辑器无关。

也可从通用帮助发现：

- `/help memory` 显示斜杠摘要与用法。
- `/memory help` 打印记忆子命令与解析路径。

### 3. `remember` 工具（自动更新，#489）

启用记忆后模型获得 `remember` 工具，形状大致为：

```json
{
  "name": "remember",
  "description": "Append a durable note to the user memory file...",
  "input_schema": {
    "type": "object",
    "properties": {
      "note": { "type": "string", ... }
    },
    "required": ["note"]
  }
}
```

当模型注意到值得跨会话保留的偏好、约定或事实时会调用。工具**自动批准**——写入限定在用户自有记忆文件，若再走标准写审批会失去自动捕获意义。

若模型用 `remember` 记瞬时任务状态（「我正在改 foo.rs」）无害但浪费上下文。工具描述明确要求**不要**这样做——仅 durable、单句笔记。

## 文件格式

记忆为带时间戳条目的纯 Markdown：

```markdown
- (2026-05-03 22:14 UTC) prefer pytest over unittest
- (2026-05-03 22:31 UTC) this codebase uses 4-space indentation
- (2026-05-04 09:02 UTC) all PRs need 2 reviewers before merge
```

可任意编辑器手工修改——加载器不关心时间戳格式；整块文件作为记忆块读取。时间戳仅为整理文件时区分添加时间。

## 层级与导入

记忆有意设计为**用户级**而非仓库级。与 `AGENTS.md`、`.deepseek/instructions.md`、`instructions = [...]` 等项目指令源**并列**，而非嵌套在内。

- **记忆**：跨仓库、跨会话跟身的个人偏好。
- **项目指令**：应随代码库传播的仓库约定。

当前加载器只读取**一个**解析路径原文。**不支持** `@path` 导入/包含；若需要更大可复用指令包，请放入项目指令文件或 [skill](../crates/tui/src/skills.rs)。

## 不应写入记忆的内容

记忆用于**持久**信号。**不要**放：

- **密钥** —— 无 API key、token、密码。文件为磁盘明文并原样注入系统提示。
- **瞬时任务状态** —— 「正在解析器上干活」每会话都变，不属于跨会话记忆。
- **对话片段** —— 引用式笔记应用 `note` 工具，而非记忆。
- **长指令** —— 超过几句的应放 `AGENTS.md`（项目级）或 [skill](../crates/tui/src/skills.rs)（可复用指令包）。

## 隐私与范围

记忆文件完全在本地 `~/.deepseek/`，**不会**上传到任何云服务——仅当启用记忆时，TUI 将其内联进发给 LLM 提供商的系统提示。切换提供商（DeepSeek / NVIDIA NIM / Fireworks 等）仍用同一文件；与提供商无关。

文件按用户，**非**按项目。若需要项目级记忆，请用项目内 `AGENTS.md` 或 `.deepseek/instructions.md` —— 由 `project_context` 加载并可随仓库提交。

## 配置参考

```toml
# ~/.deepseek/config.toml
[memory]
enabled = true                    # 默认 false；或 DEEPSEEK_MEMORY=on
# 路径在顶层配置（与 skills_dir、notes_path 并列）：
memory_path = "~/.deepseek/memory.md"
```

| 设置 | 默认 | 覆盖 |
|-----------------------|-------------------------------|---------------------------------------|
| 启用记忆 | `false` | `[memory] enabled = true` 或 `DEEPSEEK_MEMORY=on` |
| 记忆文件路径 | `~/.deepseek/memory.md` | `memory_path = "..."` 或 `DEEPSEEK_MEMORY_PATH=` |
| 最大文件大小 | 100 KiB | （暂无配置；截断处有标记） |

## 相关

- `docs/SUBAGENTS.md`（[简体](SUBAGENTS.zh-CN.md)）—— 子智能体继承记忆，也可使用 `remember`。
- `docs/CONFIGURATION.md`（[简体](CONFIGURATION.zh-CN.md)）—— 完整配置参考。
- Issue [#489](https://github.com/Hmbown/DeepSeek-TUI/issues/489) —— 阶段一 EPIC 跟踪。
