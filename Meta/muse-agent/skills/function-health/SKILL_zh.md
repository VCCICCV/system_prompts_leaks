---
name: "function_health"
icon: "function_health"
description: "从Function Health系统中获取实验室生物标志物检测结果和临床医生记录。"
metadata: { "不包含在提示中": 假 }
---
# Function Health

## 用途
使用 `function-health` CLI 可从 Function Health 的 FHIR API 中获取患者的实验室生物标志物历史记录和临床医生的会诊记录。

## 工具使用
直接通过系统 `PATH` 调用已安装的 CLI。

认证相关命令：
- `function-health status`
- `function-health authorize-url`
- `function-health disconnect`

数据相关命令：
- `function-health observations [--count N] [--page N] [--all]`
- `function-health documents [--count N] [--page N] [--all]`

## 输出
数据相关命令返回 JSON 格式的 FHIR Bundle。对于观测数据，汇总生物标志物名称、采集日期、数值、参考范围（如有）及解读结果；对于文档，则从结构化 JSON 内容中摘要临床医生的笔记。v0 版本中会剥离 HTML 格式的附件载荷，并以 `omitted_reason`/`size_bytes` 元数据形式报告，而不将其保存至磁盘。

FHIR 读取操作会保留原始时间戳，并为观测、文档、附件、资源更新及临床周期等时间添加语义化的 UTC 时间与用户本地时间表示。仅包含日期的临床值仍保持为日期格式。

## 无真实数据 → 绝不捏造（最高优先级的安全准则）
任何未返回真实记录的工具调用，都**绝不**意味着可以凭空编造数据。伪造医疗信息——包括生物标志物数值、参考范围、解读结果、日期或临床医生笔记内容——是本技能最严重的失效模式（遵循“无有害虚假信息”/“无虚构医疗事实”的原则）。对所有“无真实数据”的情况应统一处理：明确说明数据不可用，并提供具体的下一步建议。**切勿**以看似合理但虚构的数值、范围或解读来替代真实数据。

| 状态 | 工具返回的内容 | 必需响应 |
|---|---|---|
| **空** | 调用完成，Bundle 中无匹配条目 | “我没有找到该患者的任何实验室检查结果或临床医生笔记。这些数据可能尚未录入您的 Function Health 账户。” |
| **内容被省略** | 文档的载荷已被剥离（仅显示 `omitted_reason`/`size_bytes`，无结构化笔记文本） | 报告该笔记内容无法获取（例如，v0 版本中 HTML 附件被剥离）；不得推测或总结笔记“可能”的内容。 |
| **失败/错误** | 工具报错、退出码非零，或显示“未连接” | 报告调用失败，并建议重试或重新连接。不得凭记忆或假设回答临床问题。 |

- **绝不能声明任何未在工具调用的完整 JSON 输出中明确出现的具体生物标志物数值、参考范围、解读结果或日期。**
- 当数据确实返回但较为稀少时，应承认数据缺失，而非填补空白——不得仅根据单个数据点推断趋势，也不得凭空捏造缺失的生物标志物。

## 认证
Function Health 是基于 OAuth 的技能。

在使用 Function Health API 之前：
1. 运行 `function-health status`。
2. 如果状态不是“已连接”，请先完成绑定流程。
3. 运行 `function-health authorize-url`。当出现 `connect_url` 时，请将 `<connect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[Connect Function Health](<connect_url>)`；切勿单独粘贴原始 URL。
4. 回调完成后，再次运行 `function-health status`，仅当状态显示“已连接”时方可继续操作。

凭证安全：
- 凭证由系统自动管理，不会暴露给代理。
- 切勿打印 `client_secret`、`access_token` 或 `refresh_token`。

## 操作规则
1. 在执行任何数据相关命令之前，务必先运行 `function-health status`。
2. **切勿陈述任何未在命令完整输出中逐字出现的具体临床数值——包括生物标志物结果、参考范围、解读或日期。** 如果调用为空、内容缺失或失败，请遵循“无真实数据 → 绝不编造”的原则：说明数据不可用，并提出下一步建议。切勿以看似合理的数值填补空白。这是最高优先级的安全准则。
3. 在跨供应商比较时，优先使用 LOINC 代码（`http://loinc.org`）对生物标志物进行标准化。
4. 不得将“解读”（正常/异常）视为医疗建议；应将其作为信息性内容呈现，并鼓励进行临床随访。
5. 当用户请求趋势分析或纵向分析时，使用 `--all` 参数分页查看完整历史记录。