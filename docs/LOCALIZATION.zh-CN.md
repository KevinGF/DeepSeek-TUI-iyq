# 本地化矩阵

> **English**：[LOCALIZATION.md](LOCALIZATION.md)

状态日期：2026-04-29

本文仅跟踪 **UI 本地化**。不改变模型输出语言、提供商行为或 DeepSeek 载荷能力。除非另行增加原生媒体载荷，媒体附件仍为本地路径文本引用。

## 来源审计

v0.7.6 对等检查使用实时 GitHub 源与 `/opt/homebrew/bin/gh`。

| 项目 | Ref | 证据 | 结果 |
|---|---:|---|---|
| Codex CLI | `openai/codex@df966996a75333add031fca47b72655e9ee504fd` | `gh repo view openai/codex`；递归扫描 `locale`、`i18n`、`l10n`、`translation`、`messages`；README 语言扫描 | 审计树中未发现已提交的 CLI UI 本地化注册表。将 Codex CLI 对等视为英文优先终端 UI，而非已发布语言标签来源。 |
| opencode | `anomalyco/opencode@00bb9836a60f1dcdd0ce5078b05d12f749fdde66` | `packages/console/app/src/lib/language.ts` 等 | opencode 提供应用/文档语言基础设施、检测、标签、文档语言别名、阿拉伯语 RTL 及针对性键的 parity 测试。 |

## v0.7.6 已发布核心包

以下语言由 `settings.toml` 的 `locale` 与 `LANG` / `LC_ALL` 自动检测支持。

| Locale | 显示名 | 文字 | 方向 | 回退 | 优先级 | v0.7.6 范围 | 说明 |
|---|---|---|---|---|---|---|---|
| `en` | English | Latin | LTR | `en` | 基线 | 源字符串仍为规范 | 英语始终可用。 |
| `ja` | Japanese | Jpan | LTR | `en` | v0.7.6 必备 | 核心 TUI 外壳 | 覆盖合成器占位、历史搜索、帮助与 `/config` 外壳。 |
| `zh-Hans` | 简体中文 | Hans | LTR | `en` | v0.7.6 必备 | 核心 TUI 外壳 | `zh`、`zh-CN`、`zh-Hans` 解析到此。**未**提供繁体。 |
| `pt-BR` | 巴西葡语 | Latin | LTR | `en` | v0.7.6 必备 | 核心 TUI 外壳 | `pt` 与 `pt-PT` 当前回退到巴葡；**未**单独提供欧葡。 |

选择：

```toml
locale = "auto"     # 默认；检查 LC_ALL、LC_MESSAGES、再 LANG
locale = "ja"
locale = "zh-Hans"
locale = "pt-BR"
```

回退：

- 缺失或不支持的配置语言回退英语。
- `auto` 在未检测到支持的环境语言时回退英语。
- **UI 语言与模型提示语言分离**。用户仍可在提示中指定回答语言。

## 计划中的全球南方 QA 矩阵

以下**除非**后续变更新增完整消息覆盖与 QA 证据，**不**声称在 v0.7.6 已发布翻译。

| Locale | 显示名 | 文字 | 方向 | 优先级 | 覆盖状态 | 回退 | QA 状态 | 版式风险 |
|---|---|---|---|---|---|---|---|---|
| `ar` | Arabic | Arab | RTL | 跟进 | 计划 | `en` | 仅自动化渲染样本；发布前需母语审阅 | RTL 顺序、标点、键位混合 |
| `hi` | Hindi | Deva | LTR | 跟进 | 计划 | `en` | 仅自动化窄宽样本；发布前偏好母语审阅 | 结合标记、光标宽度、截断 |
| `bn` | Bengali | Beng | LTR | 跟进 | 计划 | `en` | 仅矩阵；发布前需母语审阅 | 结合标记、换行 |
| `id` | Indonesian | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需自动化窄宽快照与审阅 | 标签长于英文 |
| `vi` | Vietnamese | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需自动化宽度快照与审阅 | 变音符号与换行标签 |
| `sw` | Swahili | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需母语或流利审阅 | 译文质量、更长命令描述 |
| `ha` | Hausa | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需母语或流利审阅 | 变音与术语 |
| `yo` | Yoruba | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需母语或流利审阅 | 声调与术语 |
| `fil` | Filipino/Tagalog | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；发布前需源字符串 | 术语一致性 |
| `es-419` | Spanish (Latin America) | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需审阅 | 区域术语 |
| `fr` | French | Latin | LTR | 跟进 | 计划 | `en` | 仅矩阵；需审阅 | 非洲区域法语术语差异 |

## 消息覆盖

首轮注册表覆盖高可见终端外壳的稳定消息 ID：

- 合成器占位
- 合成器历史搜索标题、占位、提示与无匹配状态
- `/config` 标题、过滤占位、无匹配、过滤计数与页脚提示
- 帮助浮层标题、过滤占位、无匹配、分区标签与页脚提示

v0.7.6 **尚未**翻译：

- 模型/系统提示与人格
- 提供商或工具 schema
- 全部斜杠命令描述及每条状态/toast/错误路径
- 除本配置说明外的 README/文档正文

## 译者注意事项

除非后续词汇表明确修改，以下技术术语保持稳定：`Plan`、`Agent`、`YOLO`、`/config`、`/mcp`、`@path`、`/attach`、`DeepSeek`、`MCP`、`CLI`、`TUI`，以及 `Enter`、`Esc`、`Tab`、`PgUp`、`PgDn` 等键位写法。

## QA 清单

将计划中的语言提升为「已发布」前：

1. 在 `crates/tui/src/localization.rs` 中补全消息覆盖。
2. 增加语言解析测试与缺键测试。
3. 至少为合成器、帮助、`/config` 增加窄宽渲染覆盖。
4. 验证 CJK 宽度、RTL 标点、结合标记与截断。
5. 记录母语/流利审阅状态，或标记为仅自动化 QA。
