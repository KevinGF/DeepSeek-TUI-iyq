# 无障碍

> **English**：[ACCESSIBILITY.md](ACCESSIBILITY.md)

DeepSeek-TUI 运行在终端中，因此平台自身的无障碍栈（读屏、放大镜、终端级主题）承担主要工作。TUI 提供少量开关，用于减轻动效与密度，服务读屏与低动效用户。

## 速查

| 开关 | 默认 | 效果 |
| --- | --- | --- |
| 环境变量 `NO_ANIMATIONS=1` | 未设置 | 启动时强制 `low_motion = true`、`fancy_animations = false`，覆盖 `settings.toml` 已存值。 |
| `low_motion` | `false` | 抑制转圈动画、转录淡入、页脚漂移与活动单元脉冲。帧率限制器亦降低空闲重绘，减轻光标闪烁。 |
| `fancy_animations` | `false` | 页脚水柱带与子智能体计数脉冲。默认关闭。 |
| `calm_mode` | `false` | 默认折叠工具输出细节并裁剪状态消息。适合读屏每次重绘都播报的场景。 |
| `show_thinking` | `true` | 设为 `false` 可完全隐藏模型 `reasoning_content` 块。 |
| `show_tool_details` | `true` | 设为 `false` 可将工具调用渲染为单行，不展开载荷。 |

## 标准环境变量

在 shell 配置中设置以便每次会话生效：

```bash
# 强制低动效 + 关闭花哨动画
export NO_ANIMATIONS=1

# 可选：遵循更广泛的终端颜色约定
export NO_COLOR=1            # 由底层 ratatui 后端识别
```

`NO_ANIMATIONS` 接受 `1`、`true`、`yes`、`on`（大小写不敏感）。其它值（含 `0`、`false`、空、未设置）不改动已保存设置。

覆盖仅在**启动时**应用一次。会话中途改环境变量无效——需下次启动才重读设置。

## 通过 `/settings` 配置

命令面板可访问相同开关：

* `/settings set low_motion on`
* `/settings set fancy_animations off`
* `/settings set calm_mode on`

如此写入会持久化到 `~/.config/deepseek/settings.toml`。若设置了 `NO_ANIMATIONS` 环境变量，启动时仍优先生效；要尊重已保存选择需取消该环境变量。

## 读屏用户说明

* `low_motion` 将空闲重绘循环放慢到约每帧 120ms，减少光标反复重定位。与 `calm_mode` 合用可使 VoiceOver / Orca 的播报与模型输出线性对应，而非每次 tick 重读整屏。
* 转录为纯文本——无图片或画布——因此与平台无障碍集成的终端（如 macOS Terminal.app、iTerm2、Ghostty、Windows Terminal）会把渲染内容直接交给读屏。
* 若 `low_motion = true` 仍有动效表面，请向 [`PRIOR: Screen-reader / accessibility flag`](https://github.com/Hmbown/DeepSeek-TUI/issues/450) 提 issue，附截图或录屏。

## 相关 issue / 历史

* [#450](https://github.com/Hmbown/DeepSeek-TUI/issues/450) — 记录现有标志、增加 `NO_ANIMATIONS` 启动覆盖、撰写本页。
* [#449](https://github.com/Hmbown/DeepSeek-TUI/issues/449) — 页脚状态行改用活动主题对比色对，而非独立调色板。
