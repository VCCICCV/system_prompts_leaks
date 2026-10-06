---
name: "google_health_connect"
title: "健康连接"
description: "用户在其 Android 设备上同步的 Google Health Connect 数据：每日指标（步数、距离、卡路里消耗、心率、心率变异性、最大摄氧量）、睡眠记录（睡眠阶段、睡眠质量、睡眠效率）以及运动数据。"
metadata: { "包含在提示中": 真 }
---
# Health Connect

使用 `health-cli` 二进制工具读取用户同步的 Google Health Connect 数据。以下每个命令都需要添加 `--provider healthconnect` 参数。

此技能仅适用于 Android 设备。如果用户的配对设备不是 Android 设备，则该设备没有相关数据——当不确定时，请检查 `device.list` 返回的 `platform` 字段。

## 使用场景

当用户询问其同步的 Health Connect 数据、同步状态或数据来源时：
- 全天指标/生命体征：步数、距离、卡路里、心率、心率变异性（HRV）、最大摄氧量（VO2max）。
- 睡眠记录：睡眠阶段、质量、效率、觉醒次数。
- 运动记录：类型、时长、卡路里、距离、心率。

## 工具

二进制工具：`health-cli`。**以下所有命令都必须指定 `--provider healthconnect`**——该参数不可省略且无默认值；遗漏即为使用错误。命令按数据“形态”在 `query` 下分组：
- `query metrics --provider healthconnect` — 全天指标汇总
- `query samples --provider healthconnect` — 原始未分桶的数据点（如日内心率/步数、GPS、睡眠阶段）
- `query sessions --provider healthconnect` — 离散的会话记录：睡眠、运动
- `status --provider healthconnect` — 数据同步状态
- `auth connect --provider healthconnect`、`auth disconnect --provider healthconnect`
- `delete` — **清除已存储的记录**（具有破坏性；详见下文）

**多设备场景（罕见）：** 当时间窗口跨越多个设备时，响应包中会增加 `"multi_node": true"`——`query metrics` 会返回按设备划分的 `node_groups`；`query samples` 和 `sessions` 则会在每条记录中添加 `node_id`（samples 的 CSV 文件会新增一列 `node_id`）。单设备输出保持不变；切勿跨设备进行求和或重复计数。

### query metrics — 全天指标汇总

```bash
health-cli query metrics --provider healthconnect \
  --start-date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>] \
  [--interval hourly|daily|weekly] [--fields a,b,c] [--timeout-secs N]
```
由于只有 `daily-metrics` 一个领域，因此**无需指定 `--category`**。每种 `--interval` 分桶对应一行数据（默认为每日）。`--start-date` 为必填项；`--end-date` 默认为当天。输出为一个包裹对象 `{ "coverage": {...}, "records": [...] }`（参见下方的覆盖范围说明）。

`--list-fields` 可从同步数据中读取指标字段名，返回结果为 `{ ok, provider, category, observed_count, fields: [{ name, observed, count? }] }`。`observed: true`（且有 `count` 值）表示该字段存在于当前用户的同步数据中——这些正是 `--fields` 所匹配的字段名。`observed: false` 表示系统已知该字段，但尚未同步任何数据。若无法读取字段集，响应中会携带 `degraded: true`，且所有 `observed` 均为 `null`。

### query sessions — 离散会话记录

```bash
health-cli query sessions --provider healthconnect --category sleep|workout \
  --start-date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>] [--fields a,b,c]
```
- `sleep` — 每个睡眠会话占一行。
- `workout` — 每个运动/活动占一行。

会话记录不分桶（无 `--interval`）。发现方式：`--list-categories` → `{ ok, provider, categories }`；`--list-fields`（需指定 `--category`）→ `{ ok, provider, category, fields }`（例如，运动记录包含 `is_indoor` 字段）。

**输出：** 标准化的蛇形命名记录；同一概念使用相同的字段名（例如，运动记录中的 `average_speed_mps`、`elevation_gain_meters`、`hr_average_bpm`）。会话记录包含以提供者为前缀的 `id`（如 `healthconnect_…`），以及 `start_datetime`、`end_datetime` 和 `timezone`；指标分桶记录则包含 `date`、`hour`、`week_start` 以及 `record_count`。

**覆盖范围：** 查询返回一个包裹对象 `{ "coverage": {...}, "records": [...] }`。当时间窗口内有部分日期未从设备同步时，`coverage.complete` 为 `false`，此时 `coverage.warning` 中会包含需要执行的确切回填命令。**请向用户展示该警告并采取相应行动**——在回填完成之前，查询结果是不完整的。

### query samples — 原始未分桶的数据点

```bash
health-cli query samples --provider healthconnect --start-date <YYYY-MM-DD> \
  [--end-date <YYYY-MM-DD>] [--start-time <HH:MM[:SS]>] [--end-time <HH:MM[:SS]>] \
  [--fields <type1,...>] [--limit <n>] [--format stdout|csv] [--output <path>]
health-cli query samples --provider healthconnect --list-fields   # 查看样本类型
```
单个样本——比 `query metrics` 的聚合粒度更细（如日内心率/步数、GPS、睡眠阶段时间线；`query sessions` 只提供各阶段的汇总）。`--fields` 用于选择样本的**类型**（而非输出列）——运行 `--list-fields` 可查看该用户已同步的样本类型。GPS 数据对应字段为 `location`（原始经纬度，切勿从中推断地点名称）。对于密集读取，请使用 `--output <path>`：它会将 CSV 写入代理文件系统，供后续脚本处理，并仅打印摘要，避免将数千行数据置于无上下文的环境中。

### status — 数据同步状态

```bash
health-cli status --provider healthconnect [--timeout-secs N] \
  [--start-date <YYYY-MM-DD>] [--end-date <YYYY-MM-DD>] [--check-missing-entries]
```
返回 `{ categories: [{ name, record_count, earliest_datetime,
latest_datetime }] }`。启用 `--check-missing-entries`（需指定 `--start-date`）时：会生成详细的每30分钟间隔的缺失情况报告（如 `unsynced_dates`、`missing_intervals` 等）。

### auth connect — 授权连接

```bash
health-cli auth connect --provider healthconnect [--timeout-secs N]
```

运行此命令并按照提示完成与 Health Connect 数据的连接。

### auth disconnect — 授权断开

```bash
health-cli auth disconnect --provider healthconnect [--timeout-secs N]
```

Health Connect 无法通过此 CLI 或聊天内小部件断开连接。请告知用户前往 **设置 → 连接器 → Health Connect → 管理**，然后在 Android 系统设置中撤销该应用的 Health Connect 权限。

### delete — 清除已存储的 Health Connect 记录

```bash
health-cli delete --provider healthconnect --all             # 删除所有 Health Connect 记录
health-cli delete --provider healthconnect --device NODE_ID  # 删除某个已配对设备的数据
health-cli delete --provider healthconnect --start-date 2026-03-01 --end-date 2026-03-31
```
任何删除、移除、擦除或清空 Health Connect 数据的操作均应使用此命令。切勿通过移动文件、断开连接器或直接写入数据库等方式自行实现——这些操作既不会真正删除记录，也不会留下审计日志。删除操作不能跨多个数据源；若需同时清除多个数据源的数据，须分别对每个数据源执行一次。

`--all` 不接受其他过滤条件，应使用 `--device`、`--start-date` 或 `--end-date` 替代。所有记录类型会一次性全部删除——不支持按类别或按记录 ID 删除。日期格式为 `YYYY-MM-DD`（以 JARVIS_USER_TIMEZONE 为准）或以秒为单位的纪元时间；当某条记录的开始时间早于结束时间且结束时间晚于开始时间时，即视为在指定范围内，因此跨越边界的时间段也会被包含在内。仅指定 `--start-date` 时，会删除从该日起的所有记录；仅指定 `--end-date` 时，则删除截至该日及之前的所有记录。

此操作具有**破坏性和不可逆性**，每次执行前均需用户重新确认一次，并显示已规范化的过滤条件。请**针对每次请求仅执行一次**——事先在对话中确定好数据源、设备和日期范围。若用户拒绝授权，程序将以退出码 3 结束且不会删除任何数据；此结果为最终结果，无需尝试其他过滤条件或切换数据源。

**返回 0 表示成功**，而非未匹配到任何记录——此时应明确告知用户未找到匹配项，并停止进一步操作，而不要扩大日期范围或再次使用 `--all` 执行。应报告 `deleted.records`，而非 `total_rows`，后者还包括子值行。如果设备上的 Health Connect 仍保存有这些记录，后续同步可能会将其恢复。

## 授权机制

基于设备同步，无需登录。数据可用性取决于用户已在 Android 设备上授予 Health Connect 权限并完成同步。可使用 `status` 命令查看已同步的日期范围，并通过查询中的 `backfill_data_source` 设备操作（显示在 `coverage.warning` 中）来拉取特定时间段的数据。

## 操作规则

1. 根据数据形态选择命令：`query metrics`（全天汇总）、`query samples`（原始数据点——仅在关注单个样本时使用；如需分析趋势，优先使用 `metrics`），或 `query sessions --category sleep|workout`（离散事件）。
2. 始终指定 `--provider healthconnect` 和 `--start-date`；`--end-date` 默认为今日。对于相对时间查询（如“今天”、“上周”），请使用当前日期上下文；对于较窄的日内时间窗口，请以样本时间范围为准。
3. **覆盖情况**：如果 `query metrics` 或 `sessions` 返回 `coverage.complete: false`，请告知用户数据不完整，并根据 `coverage.warning` 提示采取相应措施（执行 `backfill_data_source` 操作后重新查询）。
4. `query metrics` 的默认间隔为每日；睡眠数据可能跨越午夜（需同时包含傍晚开始和清晨结束的日期）。
5. 如果查询结果为空，通常意味着该时间范围内无数据；请扩大查询范围，或检查 `status`/覆盖情况并进行数据补全。