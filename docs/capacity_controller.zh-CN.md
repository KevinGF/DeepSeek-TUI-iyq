# 容量控制器

> **English**：[capacity_controller.md](capacity_controller.md)

`deepseek-tui` 包含可选的、感知容量的上下文控制器。在默认 V4 路径下**关闭**，因为其主动干预可能改写实时提示并破坏前缀缓存亲和性。除非显式设置 `capacity.enabled = true`，否则将其视为遥测或实验性护栏。

## 策略概览

每个检查点计算：

- `H_hat`（运行时压力代理）
- `C_hat`（模型容量先验）
- `slack = C_hat - H_hat`
- 最近 `N=8` 次观测上的动态 slack 曲线

### 运行时压力代理（`H_hat`）

- `action_complexity_bits = log2(1 + action_count_this_turn)`
- `tool_complexity_bits = log2(1 + tool_calls_recent_window)`
- `ref_complexity_bits = log2(1 + unique_reference_ids_recent_window)`
- `context_pressure_bits = 6.0 * context_used_ratio`

公式：

`H_hat = 0.35*action_complexity_bits + 0.30*tool_complexity_bits + 0.20*ref_complexity_bits + 0.15*context_pressure_bits`

### 容量先验（`C_hat`）

按模型的先验：

- `deepseek_v3_2_chat = 3.9`
- `deepseek_v3_2_reasoner = 4.1`
- `deepseek_v4_pro = 3.5`
- `deepseek_v4_flash = 4.2`
- 回退 `3.8`（其它 DeepSeek ID 及未来版本）

### 失败概率

使用滚动曲线字段：

- `final_slack`
- `min_slack`
- `violation_ratio`
- `slack_volatility`
- `slack_drop`

公式：

`z = -1.65*final_slack -0.85*min_slack +1.35*violation_ratio +0.70*slack_volatility +0.28*slack_drop -0.12`

`p_fail = sigmoid(z)`，限制在 `[0,1]`。

风险带：

- low：`p_fail <= low_risk_max`
- medium：`p_fail <= medium_risk_max`
- high：其它

显式启用控制器时的动作映射：

- low → `NoIntervention`
- medium → `TargetedContextRefresh`
- high + 严重动态（`min_slack <= severe_min_slack` 或 `violation_ratio >= severe_violation_ratio`）→ `VerifyAndReplan`
- 否则 high → `VerifyWithToolReplay`

## 检查点

启用时，引擎在以下时机评估控制器策略：

1. 预请求检查点（`MessageRequest` 组装前）。
2. 后工具检查点（工具结果追加后）。
3. 错误升级检查点（工具错误连击路径）。

## 干预

干预**不是**默认 v0.7.5 V4 路径的一部分。默认路径：追加消息、保留前缀缓存复用、在真实模型压力附近建议手动 `/compact`、仅当请求将超过模型输入预算时使用溢出恢复。

### `TargetedContextRefresh`

- 可能时运行压缩（`compact_messages_safe`）。
- 压缩失败则回退本地裁剪。
- 持久化规范状态。
- 用紧凑规范提示 + 记忆指针替换长尾活跃上下文。

### `VerifyWithToolReplay`

- 重放最近回合上下文中的一个只读关键工具调用。
- 追加验证说明（通过/失败 + diff 摘要）。
- 重放冲突/错误时标记升级候选并禁用当前回合的重放。

### `VerifyAndReplan`

- 持久化规范快照。
- 清除易失提示尾部，保留最新用户请求与最新验证说明。
- 将规范 replan 指令注入系统提示。
- 从紧凑规范状态继续回合循环。

## 安全控制

- 每回合最多一次干预。
- 刷新与 replan 的冷却时间。
- 每回合重放预算（`max_replay_per_turn`）。
- 控制器输入不可用时的 fail-open 行为。
- 压缩/重放失败会记录日志；回合继续。

## 记忆存储

路径：

- `DEEPSEEK_CAPACITY_MEMORY_DIR`（若设置）
- 否则 `~/.deepseek/memory/<session_id>.jsonl`
- 回退：当 home 不可用/不可写时为 `<workspace>/.deepseek/memory/<session_id>.jsonl`

记录字段：

- `id`、`ts`、`turn_index`、`action_trigger`
- `h_hat`、`c_hat`、`slack`、`risk_band`
- `canonical_state`
- `source_message_ids`
- 可选 `replay_info`

加载工具支持取最后 `K` 个快照用于再水合。

## 配置

`[capacity]` 键：

- `enabled`（默认 `false`）
- `low_risk_max`（默认 `0.50`）
- `medium_risk_max`（默认 `0.62`）
- `severe_min_slack`（默认 `-0.25`）
- `severe_violation_ratio`（默认 `0.40`）
- `refresh_cooldown_turns`（默认 `6`）
- `replan_cooldown_turns`（默认 `5`）
- `max_replay_per_turn`（默认 `1`）
- `min_turns_before_guardrail`（默认 `4`）
- `profile_window`（默认 `8`）
- `deepseek_v3_2_chat_prior`（默认 `3.9`）
- `deepseek_v3_2_reasoner_prior`（默认 `4.1`）
- `deepseek_v4_pro_prior`（默认 `3.5`）
- `deepseek_v4_flash_prior`（默认 `4.2`）
- `fallback_default_prior`（默认 `3.8`）

等效环境覆盖使用 `DEEPSEEK_CAPACITY_*` 前缀。
