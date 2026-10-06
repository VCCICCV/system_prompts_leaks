---
name: "withings"
description: "在链接 Withings 或读取 Withings 的身体测量、活动、睡眠、锻炼、心率及日内数据时使用。"
icon: "withings"
metadata: { "不包含在提示中": false }
---
# Withings

通过捆绑的 `withings` CLI 查询 Withings 的身体测量、活动、睡眠和锻炼数据。

## 使用场景

当用户询问其 Withings 数据时激活：
- 身体指标（体重、BMI、体脂、血压、心率）
- 日常活动（步数、卡路里、距离、活跃时长）
- 睡眠记录（评分、阶段、时长、呼吸频率）
- 锻炼记录（跑步/步行/骑行等）

## 工具

三个子命令可满足大部分需求：

| 子命令 | 用途 |
|---|---|
| `withings status` | 连接状态（`{ok, status, connect_url?, disconnect_url?, reason?}`）。 |
| `withings list-fields --category <CAT>` | 某类别的字段列表（蛇形命名，单位包含在名称中）。 |
| `withings query --category <CAT> --start-date <YYYY-MM-DD> [--end-date --interval --fields]` | 类型化的蛇形命名记录。返回 JSON 数组。 |

### 类别

- `daily-metrics` — 活动（步数/卡路里/距离/心率）与身体测量（体重/血压/体成分）的每日汇总。支持 `--interval hourly|daily|weekly`（默认为 `daily`）。聚合规则：计数器字段（`step_count`、`active_energy_burned_kcal`、`distance_walking_running_meters`）采用 **求和**，`hr_average_bpm` 采用 **平均值**，`hr_max_bpm` 采用 **最大值**，身体测量（体重/血压/体脂等）采用 **桶内最后值**。
- `sleep` — 每条睡眠记录一行（评分、阶段、时长、心率、呼吸）。
- `workout` — 每条锻炼记录一行（翻译后的 `workout_type` 名称、时长、距离、卡路里、心率）。

使用 `list-fields` 可查看各类别下的可用字段。

### 输出

JSON 输出至标准输出。日期时间为本地时间，格式为 `YYYY-MM-DD HH:MM:SS`，按每条记录的时区显示。会话记录包含 `id`（前缀为 `withings_<id>`）、`start_datetime`、`end_datetime` 和 `timezone`。`daily-metrics` 的分桶记录包含 `date` / `hour` / `week_start` 以及 `record_count`，而非 `id`。

### 示例

```bash
# 最近的锻炼记录
withings query --category workout --start-date 2026-05-01

# 体重趋势（每天的最新读数）
withings query --category daily-metrics --start-date 2026-04-01 --fields body_mass_kg

# 每周步数总计
withings query --category daily-metrics --start-date 2026-04-01 --interval weekly --fields step_count
```

## 认证

在进行任何 Withings API 调用之前：
1. 运行 `withings status`。
2. 如果 `status` 为 `not_connected`，请原样分享 `[Connect Withings](<connect_url>)`——切勿粘贴原始 URL。
3. 用户完成回调后，重新运行 `withings status`，仅当显示 `connected` 时方可继续操作。

如需断开连接，请运行 `withings disconnect`，并在存在时分享 `[Disconnect Withings](<disconnect_url>)`。

切勿打印令牌或凭据。

## 操作规则

1. 将链接视为一次性入职流程——除非调用持续失败，否则不要再次提示用户进行身份验证。
2. 优先使用 `query --category <CAT>` 命令，而不是旧版命令。字段名称采用蛇形命名法，并在名称中包含单位（如 `body_mass_kg`、`step_count`、`hr_average_bpm`、`sleep_total_duration_sec`），切勿向用户暴露 Withings 的数值型测量类型 ID。
3. **如果不确定某个字段名称，请在编写 `query --fields` 命令之前先运行 `withings list-fields --category <CAT>`。** Withings 原生名称（如 `calories`、`distance`、`hr_average`、`weight`）无效——它们会被翻译成带有单位的蛇形命名法（如 `energy_burned_kcal`、`distance_meters`、`hr_average_bpm`、`body_mass_kg`）。如果过滤结果为空，通常意味着字段名称错误，而不是数据缺失。
4. 当用户询问近期趋势但未指定日期时，默认使用最近 7 天的数据。
5. 对于宏观趋势，使用 `--category daily-metrics --interval weekly` 而不是逐条拉取所有记录。`--interval` 标志已经按照上述规则进行了聚合，客户端侧无需再对分桶后的输出进行额外的聚合（求和或平均）。
6. 如果 `query` 返回 `[]`，请明确告知用户——切勿凭空捏造数值。如果用户请求的是 `list-fields` 中未列出的指标，可退回到旧版兜底方案（如 `measures --meas-types <ID>` 或更具体的子命令，如 `heart-list`；测量类型 ID 表参见 [references/commands.md](references/commands.md)）。

## 旧版 / 高级命令

这些是针对每个端点的 `query` 前直通命令，返回原始的 `{ok, status, body: <Withings 原生>}` 封装。仅作为“兜底”手段使用，用于获取 `query` 未公开的数据——例如，翻译表中缺失的测量类型（如 `withings measures --meas-types 130` 用于获取房颤心电图结果）、高频睡眠传感器数据（`withings sleep`）或心电信号（`withings heart-list` / `heart-get`）。

其他直通命令的快速参考：
- `withings activity` — 原始每日活动记录。
- `withings sleep-summary` — 原始每晚睡眠摘要，使用 Withings 原生字段名称。
- `withings intraday` — 分钟级活动传感器数据。
- `withings devices` — 已配对的 Withings 设备列表。

完整的命令矩阵、测量类型 ID 以及 Withings 到 Muse 的字段映射：[references/commands.md](references/commands.md)。
- 权限受限的数据：当用户的“读取测量数据和心脏数据”权限为拒绝或询问时，`query --category daily-metrics` 会将记录包裹在 `{"withheld": {...}, "records": [...]}` 中，其中移除了体重、血压、血氧饱和度、心脏测量等身体指标，且仅请求这些字段的查询会因 `data_class_excluded` 而失败。以 `withheld` 标记为准：当该标记存在时，表示权限正在限制这些字段——应明确告知用户，而非说“无读数”。当用户确实需要被限制的指标，且权限要求批准时（即 `withheld.reason` 为 `requires_approval`，或者仅请求身体指标的查询因 `data_class_excluded` 而失败并提示需批准），可执行 `withings measures --category 1 --start-date <d> --end-date <d>`，该命令受相应权限约束，并会向用户显示权限审批提示。睡眠和运动结果从不被包裹；无该标记的响应即为普通结果。