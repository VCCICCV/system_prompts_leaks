# Withings CLI 命令

从 `PATH` 中使用已安装的 `withings` CLI。所有命令均返回 JSON 格式。

---

## 推荐路径 — healthkit-shape 表面

这三个子命令通过规范化的蛇形命名字段、在名称中包含单位以及按类别聚合，覆盖了大多数代理的需求。新代码应优先使用这些命令。

### `withings status`

返回当前活动认证后端的连接状态。

```json
{
  "ok": true,
  "status": "connected",
  "connect_url": null,
  "disconnect_url": "https://..."
}
```

未连接时：

```json
{"ok": true, "status": "not_connected", "connect_url": "https://...", "reason": "withings 未连接"}
```

### `withings list-fields --category CATEGORY`

返回指定类别的字段列表。`CATEGORY` 可取值为 `daily-metrics`、`sleep` 或 `workout`。

### `withings query --category CATEGORY --start-date YYYY-MM-DD [...]`

标志说明：

| 标志 | 必需 | 描述 |
|---|---|---|
| `--category` | 是 | `daily-metrics`、`sleep` 或 `workout` |
| `--start-date` | 是 | `YYYY-MM-DD`，包含该日期 |
| `--end-date` | 否 | `YYYY-MM-DD`，包含该日期，默认为今天 |
| `--interval` | 否 | `hourly`、`daily`、`weekly` — 仅适用于 `daily-metrics` |
| `--fields` | 否 | 逗号分隔的字段子集 |

`query` 返回一个 JSON **数组**，其中包含多个记录（不包裹在 `{ok, status, body}` 中），但有一个例外：当用户的“读取测量数据和心率数据”权限为拒绝或询问时，`query --category daily-metrics` 会返回 `{"withheld": {...}, "records": [...]}`（不含身体指标的行）。而睡眠和运动数据则始终直接返回普通数组。

### 输出约定（新表面）

- 字段名称采用蛇形命名，且在名称中包含单位（无 `measuregrps`，无整数形式的测量类型 ID）。
- 日期时间为本地格式的 `YYYY-MM-DD HH:MM:SS` 字符串——非 Unix 时间戳。
- 会话（睡眠、运动）包含以 `withings_` 为前缀的 `id`。
- 聚合后的 `daily-metrics` 分桶包含 `date` / `hour` / `week_start` 以及 `record_count`（无 `id`）。
- 使用 `--fields foo` 时，仅返回包含 `foo` 的记录，并保留始终存在的字段（`start_datetime`、`end_datetime`、`timezone`、`date`、`hour`、`week_start`、`record_count`、`id`）。

### 日常指标聚合规则

| 聚合方式 | 字段 |
|---|---|
| **求和** | `step_count`、`active_energy_burned_kcal`、`total_calories_kcal`、`distance_walking_running_meters`、`elevation_climbed_meters`、`soft_activity_duration_sec`、`moderate_activity_duration_sec`、`intense_activity_duration_sec` |
| **平均值** | `hr_average_bpm` |
| **最大值** | `hr_max_bpm` |
| **最后值**（取分桶内最新读数） | `body_mass_kg`、`body_fat_percentage`、`fat_free_mass_kg`、`fat_mass_kg`、`muscle_mass_kg`、`bone_mass_kg`、`hydration_kg`、`blood_pressure_systolic_mmhg`、`blood_pressure_diastolic_mmhg`、`vo2_max`、`spo2_percentage`、`body_temperature_celsius`、`skin_temperature_celsius`、`pulse_wave_velocity_meters_per_sec`、`basal_metabolic_rate_kcal`、`metabolic_age_years`、`visceral_fat`、`height_meters` |

### Withings → Muse 字段名映射（新表面）

此翻译表由 `query` 内部使用。代理只需关注右侧列（即 `query` 输出中的蛇形命名字段）。未映射的 Withings 测量类型 ID 将从 `query` 输出中移除——如需获取这些数据，请使用旧版的 `measures` 命令（见下文的应急方案）。| Withings 测量类型 ID | Muse 字段 |
|---|---|
| 1 | `body_mass_kg` |
| 4 | `height_meters` |
| 5 | `fat_free_mass_kg` |
| 6 | `body_fat_percentage` |
| 8 | `fat_mass_kg` |
| 9 | `blood_pressure_diastolic_mmhg` |
| 10 | `blood_pressure_systolic_mmhg` |
| 11 | `hr_average_bpm` |
| 12 | `temperature_celsius` |
| 35 | `co2_ppm` |
| 54 | `spo2_percentage` |
| 71 | `body_temperature_celsius` |
| 73 | `skin_temperature_celsius` |
| 76 | `muscle_mass_kg` |
| 77 | `hydration_kg` |
| 88 | `bone_mass_kg` |
| 91 | `pulse_wave_velocity_meters_per_sec` |
| 123 | `vo2_max` |
| 135 | `qrs_interval_ms` |
| 136 | `pr_interval_ms` |
| 137 | `qt_interval_ms` |
| 138 | `corrected_qt_interval_ms` |
| 139 | `atrial_fibrillation_ppg` |
| 155 | `vascular_age_years` |
| 167 | `nerve_health_score_conductance_feet` |
| 168 | `extracellular_water_kg` |
| 169 | `intracellular_water_kg` |
| 170 | `visceral_fat` |
| 174 | `fat_free_mass_segmental_kg` |
| 175 | `muscle_mass_segmental_kg` |
| 196 | `electrodermal_activity_feet` |
| 226 | `basal_metabolic_rate_kcal` |
| 227 | `metabolic_age_years` |

运动类型翻译（`query --category workout` 输出中的 `workout_type` 字段）：  
`1=步行, 2=跑步, 3=徒步, 4=滑冰, 5=小轮车, 6=骑行, 7=游泳, 8=冲浪, 9=风筝冲浪, 10=帆板, 12=网球, 13=乒乓球, 14=壁球, 15=羽毛球, 16=举重, 17=徒手健身, 18=椭圆机, 19=普拉提, 20=篮球, 21=足球, 22=橄榄球, 27=高尔夫, 28=瑜伽, 30=拳击, 34=滑雪, 35=单板滑雪, 36=划船, 42=攀岩, 45=室内步行, 46=室内跑步, 47=室内骑行, 187=拉伸, 188=综合训练, 191=健身, 195=其他`，等等（完整列表见源码：`withings/src/schema.rs::WITHINGS_WORKOUT_TYPE_TO_NAME`）。未知整数将从 `workout_type` 中剔除。

---

## 遗留 / 高级命令

这些透传命令会返回原始的 Withings 响应结构（`{ok, status, body: <Withings 原生>}`）。在以下情况下使用它们：
- 您需要一个不在上述翻译表中的测量类型（应急出口）。
- 您需要一天内的每一条单独记录，而不是每日汇总数据。
- 您需要高频的睡眠或心率传感器数据。

### 认证

- `withings status` — 返回连接器状态，包含 `connect_url` 和 `disconnect_url`（如有）。
- `withings authorize-url` — 返回当前后端的授权 URL。请解析 `authorize_url`。建议优先使用 `withings status`（它返回与 `connect_url` 相同的 URL）。
- `withings disconnect` — 断开与当前后端的连接。

### 数据读取（原始透传）

- `withings measures [--category 1] [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--meas-types 1,4,11] [--offset <n>] [--last-update <ts>]`
- `withings activity [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--offset <n>] [--last-update <ts>]`
- `withings sleep-summary [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--offset <n>] [--last-update <ts>]`
- `withings sleep [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--meas-types <csv>]`
- `withings workouts [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--offset <n>] [--last-update <ts>]`
- `withings intraday [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD]`
- `withings heart-list [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--offset <n>]`
- `withings heart-get --signal-id <id>`
- `withings devices`

### 测量类型 ID（用于旧版 `measures` 命令的 `--meas-types` 参数）

完整参考，包括未在新 `query` 接口中暴露的 ID（当代理需要原始访问权限时使用此表）：

| 编号 | 指标 |
|----|--------|
| 1 | 体重（kg） |
| 4 | 身高（m） |
| 5 | 去脂体重（kg） |
| 6 | 体脂率（%） |
| 8 | 脂肪重量（kg） |
| 9 | 舒张压（mmHg） |
| 10 | 收缩压（mmHg） |
| 11 | 心率（bpm） |
| 12 | 体温（℃） |
| 54 | 血氧饱和度（%） |
| 71 | 体温（℃） |
| 73 | 皮肤温度（℃） |
| 76 | 肌肉量（kg） |
| 77 | 体水分（kg） |
| 88 | 骨量（kg） |
| 91 | 脉搏波传导速度（m/s） |
| 123 | 最大摄氧量 |
| 130 | 房颤检测结果（心电图） |
| 135 | QRS间期（ms） |
| 136 | PR间期（ms） |
| 137 | QT间期（ms） |
| 138 | 校正后的QT间期（ms） |
| 139 | 房颤检测结果（光电容积描记法） |
| 155 | 血管年龄 |
| 167 | 神经健康评分 |
| 168 | 细胞外液量（kg） |
| 169 | 细胞内液量（kg） |
| 170 | 内脏脂肪 |
| 173 | 去脂体重（分部位） |
| 174 | 脂肪量（分部位） |
| 175 | 肌肉量（分部位） |
| 196 | 皮肤电活动 |
| 226 | 基础代谢率 |
| 227 | 代谢年龄 |
| 229 | 皮肤电导 |

### 旧版输出协议

- 读取命令返回顶层的 `ok`、`status` 和 `body`。
- `measures` 命令使用 `--category 1` 表示实际测量指标，`--category 2` 表示用户目标。
- 日期参数接受 `YYYY-MM-DD` 格式。当 API 需要时，CLI 会将日期转换为 Unix 时间戳。
- 数值缩放：`body.measuregrps[].measures[]` 中的每个指标以 `{value, unit, type}` 格式编码，其中实际值 = `value * 10^unit`。

### 分页

- 支持 `--offset` 的命令在还有更多页面时会返回 `body.more`（布尔值）和 `body.offset`。传递 `--offset <value>` 可获取下一页。
- 新的 `query` 子命令会在内部处理分页（每个类别最多 20 页）。