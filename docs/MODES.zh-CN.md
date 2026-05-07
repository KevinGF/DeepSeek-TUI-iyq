# 模式与审批

> **English**：[MODES.md](MODES.md)

DeepSeek TUI 有两个相关概念：

- **TUI 模式**：当前处于哪种可见交互（Plan / Agent / YOLO）。
- **审批模式**：执行工具前 UI 询问的积极程度。

## TUI 模式

在合成器（Composer）空闲时按 `Tab` 可补全菜单、在回合运行中将草稿排队为下一轮跟进，或在无其它操作时循环可见模式：**Plan → Agent → YOLO → Plan**。  
按 `Shift+Tab` 切换推理强度。

- **Plan**：设计优先。只读调查类工具仍可用；shell 与补丁执行关闭。适合先把思路摊开、产出计划交给人类（未来的自己或评审）。
- **Agent**：多步工具使用。shell 与付费类工具需审批（文件写入通常无需每次确认）。
- **YOLO**：启用 shell + 信任模式并自动批准所有工具。**仅在可信仓库使用。**

三种模式均可使用 `rlm` 工具；其 Python REPL 内 `llm_query_batched` 可并行发起 1–16 个固定在 `deepseek-v4-flash` 的子调用。工作在可分解时模型会倾向使用它。

## 兼容说明

- `/normal` 为隐藏兼容别名，等价切换到 `Agent`。
- 旧配置中 `default_mode = "normal"` 仍会加载为 `agent`；保存时会写回规范化值。

## Esc 键行为

`Esc` 是**取消栈**，不是切换模式。

- 先关闭斜杠菜单或临时 UI。
- 若有进行中的回合，则取消当前请求。
- 若合成器为空，丢弃已排队的草稿。
- 若有输入文本，清空当前输入。
- 否则无操作。

## 审批模式

可在运行时覆盖审批行为：

```text
/config
# 将 approval_mode 行改为：suggest | auto | never
```

遗留说明：`/set approval_mode ...` 已废弃，请用 `/config`。

- `suggest`（默认）：按上文各模式规则执行。
- `auto`：自动批准所有工具（类似 YOLO 的审批效果，但不强制 YOLO 模式）。
- `never`：阻止任何非安全/非只读工具。

## 小屏状态栏行为

终端高度不足时，优先压缩状态区，保证页眉/对话/合成器/页脚仍可见：

- 加载与排队状态行按可用高度预算。
- 完整预览放不下时，排队预览折叠为紧凑摘要。
- `/queue` 流程仍可用；仅影响渲染密度。

## 工作区边界与信任模式

默认文件工具限制在 `--workspace` 目录。启用信任模式可访问工作区外路径：

```text
/trust
```

YOLO 模式会自动启用信任模式。

## MCP 行为

MCP 工具以 `mcp_<server>_<tool>` 暴露，与内置工具走同一审批流。只读 MCP 在 suggest 类审批下可能自动运行；可能有副作用的 MCP 工具需审批。

见 `MCP.md`（[简体](MCP.zh-CN.md)）。

## 相关 CLI 标志

完整列表见 `deepseek --help`。常用项：

- `-p, --prompt <TEXT>`：一次性提示（打印后退出）
- `--model <MODEL>`：使用 `deepseek` 门面时向 TUI 转发模型覆盖
- `--workspace <DIR>`：文件工具的工作区根
- `--yolo`：以 YOLO 模式启动
- `-r, --resume <ID|PREFIX|latest>`：恢复已保存会话
- `-c, --continue`：恢复当前工作区最近一次会话
- `--max-subagents <N>`：限制在 `1..=20`
- `--no-alt-screen`：不使用备用屏幕缓冲区（内联运行）
- `--mouse-capture` / `--no-mouse-capture`：是否启用内部鼠标滚动、转录区选择与右键菜单。非 Windows 终端默认开启，拖拽选择仅复制用户/助手转录文本；按住 Shift 拖拽或 `--no-mouse-capture` 可恢复终端原生选择。Windows 默认关闭（避免 CMD/终端将鼠标转义序列插入提示）。JetBrains JediTerm（PyCharm/IDEA/CLion 等）内默认关闭——终端宣称支持鼠标但会把 SGR 鼠标事件当纯文本转发（#878、#898）。在默认关闭处可用 `--mouse-capture` 显式开启。
- `--profile <NAME>`：选择配置 profile
- `--config <PATH>`：配置文件路径
- `-v, --verbose`：详细日志
