# MCP（外部工具服务器）

> **English**：[MCP.md](MCP.md)

DeepSeek TUI 可通过 MCP（Model Context Protocol）加载额外工具。MCP 服务器是由 TUI 启动、经 stdio 通信的本地进程。

浏览说明：

- `web.run` 是内置浏览的**规范**工具。
- `web_search` 仍作为兼容别名，服务旧提示与集成。

服务器模式说明：

- `deepseek-tui serve --mcp` 运行 MCP stdio 服务器。
- `deepseek-tui serve --http` 运行运行时 HTTP/SSE API（**另一模式**）。
- `deepseek` 调度器提供等价的 `deepseek mcp-server` stdio 入口（拆分 CLI 使用）。

## 引导 MCP 配置

在解析到的 MCP 路径创建入门配置：

```bash
deepseek-tui mcp init
```

`deepseek-tui setup --mcp` 在 skills 安装同时完成相同 MCP 引导。

常用管理命令：

```bash
deepseek-tui mcp list
deepseek-tui mcp tools [server]
deepseek-tui mcp add <name> --command "<cmd>" --arg "<arg>"
deepseek-tui mcp add <name> --url "http://localhost:3000/mcp"
deepseek-tui mcp enable <name>
deepseek-tui mcp disable <name>
deepseek-tui mcp remove <name>
deepseek-tui mcp validate
```

## TUI 内管理器

交互 TUI 中 `/mcp` 打开解析路径下 MCP 配置的紧凑管理器。展示各服务器启用/禁用、传输方式、命令或 URL、超时、连接错误，以及发现工具/资源/提示（若已跑发现）。

TUI 内支持的操作：

```text
/mcp init
/mcp init --force
/mcp add stdio <name> <command> [args...]
/mcp add http <name> <url>
/mcp enable <name>
/mcp disable <name>
/mcp remove <name>
/mcp validate
/mcp reload
```

`/mcp validate` 与 `/mcp reload` 会重连以做 UI 发现并刷新管理器快照。从 TUI 写的配置立即落盘，但**模型可见的 MCP 工具池不会热重载**；管理器会标为需重启，直到重启 TUI。

## 配置文件位置

默认路径：

- `~/.deepseek/mcp.json`

覆盖：

- 配置：`mcp_config_path = "/path/to/mcp.json"`
- 环境：`DEEPSEEK_MCP_CONFIG=/path/to/mcp.json`

`deepseek-tui mcp init`（及 `setup --mcp`）写入上述解析路径。

交互 `/config` 也可编辑 `mcp_config_path`。修改后 `/mcp` 使用新路径；**重建模型可见 MCP 工具池仍需重启 TUI**。

编辑文件或更改 `mcp_config_path` 后请重启 TUI。

## 工具命名

发现的 MCP 工具对模型暴露为：

- `mcp_<server>_<tool>`

示例：名为 `git` 的服务器、工具 `status` → `mcp_git_status`。

命令面板按服务器分组列出 MCP 项；会显示已禁用与失败的服务器而非隐藏，并使用与模型一致的运行时工具名。

## 资源与提示辅助工具

启用 MCP 时 CLI 还暴露辅助工具：

- `list_mcp_resources`（可选 `server` 过滤）
- `list_mcp_resource_templates`（可选 `server` 过滤）
- `mcp_read_resource` / `read_mcp_resource`（别名）
- `mcp_get_prompt`

## 最小示例

```json
{
  "timeouts": {
    "connect_timeout": 10,
    "execute_timeout": 60,
    "read_timeout": 120
  },
  "servers": {
    "example": {
      "command": "node",
      "args": ["./path/to/your-mcp-server.js"],
      "env": {},
      "disabled": false
    }
  }
}
```

也可使用 `mcpServers` 代替 `servers` 以兼容其它客户端。

## 将 DeepSeek 作为 MCP 服务器运行

可把本地 DeepSeek 二进制注册为 MCP 服务器，供其它 DeepSeek 会话（或任意 MCP 客户端）调用其工具。

### 快速设置

```bash
deepseek-tui mcp add-self
```

会解析当前二进制路径，生成运行 `deepseek-tui serve --mcp` 的配置项并写入 MCP 配置。默认服务器名为 `deepseek`。

选项：

- `--name <NAME>` —— 自定义服务器名（默认 `deepseek`）
- `--workspace <PATH>` —— 服务器工作区目录

### 手动配置

`~/.deepseek/mcp.json` 中等价条目：

```json
{
  "servers": {
    "deepseek": {
      "command": "/path/to/deepseek",
      "args": ["serve", "--mcp"],
      "env": {}
    }
  }
}
```

`deepseek-tui` 二进制直接支持 `serve --mcp`。`deepseek` 调度器提供等价 `deepseek mcp-server` stdio 入口。使用 `PATH` 上任意一个（`which deepseek` / `which deepseek-tui`）。`mcp add-self` 会自动解析正确二进制。

### 前提

- `command` 指向的二进制必须存在且可执行。
- MCP 服务器经 stdio 作为子进程运行——**无需**网络端口。
- 每个 MCP 客户端会话会各自 spawn 服务器进程。

### 工具命名

自托管 DeepSeek 服务器的工具遵循常规命名：

- `mcp_deepseek_<tool>`（若服务器名为 `deepseek`）

例如 `shell` 工具变为 `mcp_deepseek_shell`。

### MCP 服务器 vs HTTP/SSE API vs ACP

| | `deepseek-tui serve --mcp` | `deepseek-tui serve --http` | `deepseek-tui serve --acp` |
|---|---|---|---|
| **协议** | MCP stdio | HTTP/SSE JSON-RPC | ACP stdio |
| **用途** | MCP 客户端的工具服务器 | 应用直连的运行时 API | Zed/自定义 ACP 客户端的编辑器智能体 |
| **配置** | `~/.deepseek/mcp.json` 条目 | 直连 URL | 编辑器 `agent_servers` 自定义命令 |
| **生命周期** | 每客户端会话 spawn | 长驻守护进程 | 每编辑器智能体会话 spawn |

希望其它 MCP 客户端能调用 DeepSeek 工具时用 `mcp add-self`。构建直接消费 API 的应用用 `serve --http`。编辑器要以 ACP 智能体连 DeepSeek 用 `serve --acp`。

### 验证

添加后测试连接：

```bash
deepseek-tui mcp validate
deepseek-tui mcp tools deepseek
```

## 服务器字段

每服务器设置：

- `command`（字符串，必需）
- `args`（字符串数组，可选）
- `env`（对象，可选）
- `connect_timeout`、`execute_timeout`、`read_timeout`（秒，可选）
- `disabled`（布尔，可选）
- `enabled`（布尔，可选，默认 `true`）
- `required`（布尔，可选）：该服务器无法初始化则启动/连接校验失败。
- `enabled_tools`（数组，可选）：该服务器工具白名单。
- `disabled_tools`（数组，可选）：在 `enabled_tools` 之后应用的黑名单。

## 安全说明

MCP 工具现已与内置工具走**同一套**工具审批框架。只读 MCP 辅助（资源/提示列表与读取）在 suggest 类审批下可无提示运行；有副作用的 MCP 工具需审批。

仍应**只配置你信任**的 MCP 服务器，并将 MCP 配置视为等同于在本机运行代码。

## 故障排查

- 运行 `deepseek-tui doctor` 确认解析到的 MCP 配置路径及文件是否存在。
- TUI 内运行 `/mcp validate` 刷新可见服务器/工具快照。
- 若 MCP 配置缺失，运行 `deepseek-tui mcp init --force` 重新生成。
- 若工具不出现，在 shell 中验证服务器命令可运行，且服务器支持 MCP `tools/list`。
