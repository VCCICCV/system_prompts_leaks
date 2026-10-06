---
name: "apple_healthkit"
title: "苹果健康"
description: "用户的已同步 Apple 健康（HealthKit）数据：每日指标（步数、距离、卡路里、心率、心率变异性、最大摄氧量）、睡眠记录（睡眠阶段、睡眠质量、睡眠效率）以及运动数据。"
metadata: { "包含在提示中": 真 }
---
# Apple 健康

使用 `health-cli` 二进制工具读取用户同步的 Apple HealthKit 数据。以下每个命令都需要添加 `--provider healthkit` 参数。

此技能仅适用于 iOS 设备。如果用户的配对设备不是 iPhone，则该设备上没有相关数据——当不确定时，请检查 `device.list` 返回的 `platform` 字段。

# 数据来源

尽管数据来自 Apple HealthKit 同步，但请勿假定这些数据仅来源于 iOS 设备。它们也可能来自与其他设备或应用共享数据的其他来源。

## 使用场景

当用户询问其同步的 HealthKit 数据、同步状态或数据来源时：
- 全天指标/生命体征：步数、距离、卡路里、心率、心率变异性（HRV）、最大摄氧量（VO2max）。
- 睡眠记录：睡眠阶段、睡眠质量、睡眠效率、觉醒次数。
- 运动记录：类型、时长、卡路里、距离、心率。

## 工具

二进制工具：`health-cli`。**以下所有命令均需指定 `--provider healthkit`**——该参数不可省略且无默认值；遗漏即为使用错误。命令按数据“形态”归类于 `query` 子命令下：
- `query metrics --provider healthkit` — 全天指标汇总
- `query samples --provider healthkit` — 原始未分桶数据点（如日内心率/步数、GPS、睡眠阶段）
- `query sessions --provider healthkit` — 独立会话：睡眠、运动
- `status --provider healthkit` — 数据同步状态
- `auth connect --provider healthkit`、`auth disconnect --provider healthkit`
- `delete` — **清除已存储记录**（具有破坏性；详见下文）

**多设备场景（罕见）：** 当查询时间跨度覆盖多个设备时，响应包中会增加 `"multi_node": true` 字段——`query metrics` 会按设备返回 `node_groups`；`query samples` 和 `sessions` 则在每条记录中增加 `node_id`（samples 的 CSV 文件会在首列新增 `node_id`）。单设备输出保持不变；切勿跨设备进行求和或重复计数。

### query metrics — 全天指标汇总

```bash
health-cli query metrics --provider healthkit \
  --start-date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>] \
  [--interval hourly|daily|weekly] [--fields a,b,c] [--timeout-secs N]
```
`daily-metrics` 是唯一的数据域，因此**无需指定 `--category`**。每条记录对应一个 `--interval` 分桶（默认为每日）。`--start-date` 必填；`--end-date` 默认为当天。输出为一个包裹对象 `{ "coverage": {...}, "records": [...] }`（参见下方“覆盖范围”说明）。

`--list-fields` 可读取同步数据中的指标字段名 →  
`{ ok, provider, category, observed_count, fields: [{ name, observed, count? }] }`。  
`observed: true`（且有 `count` 值）表示该字段存在于当前用户的同步数据中——这些正是 `--fields` 所匹配的字段名。`observed: false` 表示系统已知该字段，但尚未同步任何数据。若无法读取字段列表，响应中会携带 `degraded: true` 标志，且所有 `observed` 均为 `null`。

### query sessions — 独立会话

```bash
health-cli query sessions --provider healthkit --category sleep|workout \
  --start-date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>] [--fields a,b,c]
```
- `sleep` — 每条记录对应一次睡眠会话。
- `workout` — 每条记录对应一次运动/活动。

会话记录不进行分桶（无 `--interval`）。发现功能：`--list-categories` → `{ ok, provider, categories }`；`--list-fields`（需指定 `--category`）→ `{ ok, provider, category, fields }`（例如，运动记录包含 `is_indoor` 字段）。

**输出：** 记录采用标准化的蛇形命名法；同一概念使用相同的字段名（例如，运动记录中的 `average_speed_mps`、`elevation_gain_meters`、`hr_average_bpm`）。会话记录包含以提供者前缀开头的 `id`（如 `healthkit_…`），以及 `start_datetime`、`end_datetime` 和 `timezone`；指标汇总记录则包含 `date`、`hour`、`week_start` 以及 `record_count`。

**覆盖范围：** 查询结果为一个包裹对象 `{ "coverage": {...}, "records": [...] }`。当查询窗口内的某些日期未从设备同步时，`coverage.complete` 为 `false`，此时 `coverage.warning` 中会包含需要执行的确切回填命令。**应将该警告告知用户并采取相应行动**——在回填完成之前，结果仅为部分数据。

### query samples — 原始未分桶数据点

```bash
health-cli query samples --provider healthkit --start-date <YYYY-MM-DD> \
  [--end-date <YYYY-MM-DD>] [--start-time <HH:MM[:SS]>] [--end-time <HH:MM[:SS]>] \
  [--fields <type1,...>] [--limit <n>] [--format stdout|csv] [--output <path>]
health-cli query samples --provider healthkit --list-fields   # 查看样本类型
```
单个样本——比 `query metrics` 的聚合粒度更细（如心率/步数的日内变化、GPS 数据、睡眠阶段时间线；`query sessions` 只提供各阶段的总量）。`--fields` 用于选择样本的**类型**（而非输出列）——可运行 `--list-fields` 查看该用户已同步的样本类型。GPS 数据对应字段为 `location`（原始经纬度，切勿据此推断地点名称）。对于密集读取，请使用 `--output <path>`：它会将 CSV 写入代理文件系统，供后续脚本处理，并仅打印摘要，避免成千上万行数据脱离上下文。

### status — 数据同步状态

```bash
health-cli status --provider healthkit [--timeout-secs N] \
  [--start-date <YYYY-MM-DD>] [--end-date <YYYY-MM-DD>] [--check-missing-entries]
```
返回 `{ categories: [{ name, record_count, earliest_datetime,
latest_datetime }] }`。启用 `--check-missing-entries`（需指定 `--start-date`）时：会生成详细的每30分钟间隔的缺失情况报告（`unsynced_dates`、`missing_intervals` 等）。

### auth connect — 授权连接

```bash
health-cli auth connect --provider healthkit [--timeout-secs N]
```

运行此命令并按照提示完成与 Apple HealthKit 数据的连接。

### auth disconnect — 授权断开

```bash
health-cli auth disconnect --provider healthkit [--timeout-secs N]
```

Apple Health 无法通过此 CLI 或聊天窗口小部件断开连接。请告知用户前往 **设置 → 连接器 → Apple Health** 并点击 **断开连接**。

### delete — 清除已存储的 Apple Health（HealthKit）记录

```bash
health-cli delete --provider healthkit --all                # 删除所有 Apple Health 记录
health-cli delete --provider healthkit --device NODE_ID     # 删除某一台已配对设备的数据
health-cli delete --provider healthkit --start-date 2026-03-01 --end-date 2026-03-31
```
适用于任何删除、移除、清除或擦除 HealthKit 数据的请求。切勿自行通过移动文件、断开连接器或直接操作数据库来实现——这些方法既不能真正删除记录，也无法留下审计轨迹。删除操作不会跨提供商执行；若需同时清除两个提供商的数据，应分别针对每个提供商运行一次。

`--all` 不接受其他过滤条件，只能与 `--device`、`--start-date` 或 `--end-date` 中的某一项同时使用。所有记录类型会一次性全部删除——不支持按类别或按记录 ID 单独删除。日期格式为 `YYYY-MM-DD`（以 JARVIS_USER_TIMEZONE 为准）或以秒为单位的纪元时间；当一条记录的开始时间在结束时间之前且结束时间在开始时间之后时，则认为该记录在范围内，因此跨越边界的时间段也会被包含。单独使用 `--start-date` 时，会删除从该日起的所有记录；单独使用 `--end-date` 时，会删除截至该日及之前的所有记录。

此操作**具有破坏性且不可逆**，每次执行前均需用户重新确认并显示标准化后的过滤条件。请**每次请求仅执行一次**——先与用户确认好提供商、设备和日期范围。若用户拒绝，程序将以退出码 3 结束且无任何记录被删除；此结果为最终结果，无需尝试其他过滤条件或切换到另一提供商。

**返回 0 表示成功**，而非未匹配到记录——此时应明确告知用户未找到符合条件的记录，并停止进一步操作，不要随意扩大日期范围或再次使用 `--all` 选项。应报告 `deleted.records`，而非 `total_rows`，后者还包括子值行。由于 Apple Health 是基于设备同步的，设备端仍保留的记录可能会在下次同步时重新出现。

## 授权

基于设备同步，无需登录。数据可用性取决于设备上的 Health 权限及同步状态。可通过 `status` 查看已同步的日期范围，并使用 `backfill_data_source` 设备操作（在查询的 `coverage.warning` 中显示）来补拉某一时间段的数据。

## 操作规则

1. 根据数据形态选择命令：`query metrics`（全天汇总）、`query samples`（原始数据点——仅在单个样本有意义时使用；若关注趋势，优先使用 `metrics`），或 `query sessions --category sleep|workout`（离散事件）。
2. 始终指定 `--provider healthkit` 和 `--start-date`；`--end-date` 默认为今日。对于相对查询（如“今天”、“上周”），使用当前日期上下文；对于较窄的日内时间窗口，则以样本时间范围为准。
3. **覆盖情况**：若 `query metrics` 或 `sessions` 返回 `coverage.complete: false`，应告知用户数据不完整，并转达或根据 `coverage.warning` 采取相应措施（执行 `backfill_data_source` 操作后重新查询）。
4. `query metrics` 的默认间隔为每日；睡眠数据可能跨越午夜（需同时包含傍晚开始和清晨结束的日期）。
5. 若查询结果为空，通常表明该时间范围内无数据；可适当扩大时间范围，或检查 `status`/覆盖情况并进行数据补全。