---
name: "opentable"
title: "OpenTable"
description: "在OpenTable上查找餐厅、查看空位情况，并进行预订、修改或取消。用于餐厅预订及获取实时预订信息。"
icon: "opentable"
metadata: { "不包含在提示中": 假 }
---
# OpenTable

对于面向用户的餐厅搜索或预订功能，请先阅读  
`/opt/hatch/skills/booking/SKILL.md`、  
`/opt/hatch/skills/booking/references/restaurants.md`，以及  
`/opt/hatch/skills/booking/references/presentation.md`。这些文件定义了端到端的预订流程、回退机制和展示规则。本文件用于定义 OpenTable CLI 的接口规范。预订选项请以纯 Markdown 格式呈现。在餐厅预订过程中，请勿调用 `create_options` 或任何比较/列表组件。

## 连接
在任何命令返回数据之前，OpenTable 需要用户在聊天中进行一次性的授权同意。请运行 `opentable status`。如果是直接的预订请求，若状态为 `not_connected`，请建议用户将 OpenTable 作为首选的低门槛方式，并原样展示返回的 `connect_url`，切勿自行生成链接。简要说明此举可实现实时空位查询、使用已保存的个人资料，并使预订流程更加顺畅。不要因等待连接而停滞不前，也不应将其设为必要条件：可在有可用时，继续通过商家官方预订链接、其他信誉良好的平台，或使用 `phone.place_call` 完成预订。如果用户在已启动回退路径后完成连接，请在恢复 OpenTable 预订前遵循 `/opt/hatch/skills/booking/references/restaurants.md` 中的重复预订保护规则。若状态为 `unavailable`，则跳过连接提示，立即采用上述回退路径。若请求仅是连接 OpenTable 而非预订，则应立即展示链接并等待用户完成连接。可通过 `opentable disconnect` 断开连接，但请勿引导用户前往设置页面。

## 常见流程

### 寻找座位
使用 `lookup-rid` 解析指定名称的餐厅，或按城市、菜系和价格筛选餐厅。当城市已知时，请添加 `--city` 参数。若未找到明确匹配项，可尝试调整餐厅名称或筛选条件，例如检查常见的空格或标点差异，仅向 `--city` 传递城市名称，或省略 `--country-code`。同时保持所请求的地点不变。

随后使用 `search-availability` 查询指定时间和人数的空余座位。

### 预订
在执行 `search-availability` 后，根据用户需求确定具体的时段。如已有授权存储的联系方式，请优先使用；结账时仅补充姓名、邮箱或电话等仍需填写的字段。然后使用 `book-reservation` 完成预订（该命令一步锁定席位并完成预订）。

`book-reservation` 返回确认信息后，可主动提供在预订前 30 分钟提醒用户的服务。若用户接受或选择其他提前时间，可使用 `cron.add` 为其设定一次性的提醒任务，时间基于已确认的预订时间自动计算。

### 修改
使用 `search-availability` 查找新的时间，确定用户要求的具体时段，然后使用 `modify-reservation-with-lock` 进行修改。此操作需要提供预订的确认 ID。

### 取消或查询预订
分别使用 `cancel-reservation` 或 `get-reservation`，两者均需提供预订的确认 ID。

### 预订体验活动
先使用 `list-experiences` 查找活动，再通过 `search-availability --include-experiences true` 确认可选时间，确定用户要求的活动及具体时段，最后使用 `book-reservation --experience-json` 完成预订（需提供活动的 `id` 和 `version`）。

## 其他命令
以上流程涵盖了常见场景。如需了解餐厅政策、座位及用餐区域选项，或解除被占用的座位锁等其他操作，请运行 `opentable --help` 查看完整命令列表，或使用 `opentable <command> --help` 查看各命令的参数选项。

## 规则
- 在执行 `book-reservation` 或 `modify-reservation-with-lock` 之前，务必明确餐厅、日期、时间和用餐人数。在执行 `cancel-reservation` 之前，需确认用户指的是哪一笔预订。对于预订操作，应从已授权的用户档案或直接向用户获取姓名、邮箱和电话号码。请直接调用已解析的命令，以便连接器策略能够呈现任何必要的审批流程；切勿重复发送聊天确认。若取消请求清晰且无歧义，则可无需额外确认直接执行。切勿自行推测缺失的信息。
- 当 OpenTable 提供带有时区偏移的预订或保留时间戳时，应在返回结果中添加 UTC 时间字段及用户本地语义字段，并由运行时生成 `retrieved_at` 字段。若日期/时间未带时区偏移，则视为餐厅所在地的当地时间；切勿猜测时区或进行时区转换。
- `book-reservation` 命令在报告成功前会先与 OpenTable 核实预订信息。只有当结果明确显示预订已确认时，才可将其标记为“已确认”；仅凭确认编号本身并不能作为确认的依据。一旦确认，应直接告知用户，避免使用模糊表述、提及服务提供商的内部状态或预测后续状态变更。若 OpenTable 需要其他操作，请说明所需步骤，而非反复重试。若预订未确认或无法核实，应如实告知，并保留确认参考号，切勿自动重试。仅提供命令返回的 OpenTable 续订或恢复链接；该链接本身不能证明预订已确认，且不得自行伪造。
- 对于预订查询，应以自然语言概括返回的状态（例如：“您的预订已确认”）。切勿引用结果字段名或服务提供商内部的状态字段。对于已取消、已完成、未到店或未知状态，均应使用相同的简洁语言描述。
- 当 OpenTable 返回预订管理链接时，应在最终回复中仅包含该链接一次，并将其置于最后一行。切勿同时附上餐厅简介链接；多个链接会导致 Hatch 无法正确渲染管理卡片。
- 只有在命令返回确认信息时，才可告知用户预订、变更或取消操作已成功。若操作失败或返回空值，应如实告知，切勿虚构确认信息或提出变通方案。
- 若结果中携带 `resource_authorization` 且其 `persisted: false`，表示 OpenTable 操作已成功，但主代理未能确认其授权已被保存。此时不应自动重试相关操作，而应将确认编号告知用户，并说明主代理可能无法后续管理该预订。除因限流返回的结果外，读取操作可安全重试。
- 每次仅发起一次 OpenTable 连接器调用。不得批量调用，也不得并行或在后台执行，更不得将其放入 Shell/Python 循环或重试封装中。
- 若 OpenTable 命令返回 HTTP 429 状态码或提示已达到速率限制，请停止针对该任务的所有 OpenTable 调用，并报告当前部分结果。切勿在限流后重复执行 `book-reservation`、`modify-reservation-with-lock` 或 `cancel-reservation`，因为相关变更可能已生效。若结果包含确认参考号，则后续任务可在返回的 `retry_after` 时间之后，或在响应中无该字段时等待 60 秒后再尝试执行 `get-reservation`。
- 对于非主代理所预订的订单，可能无法通过 `get-reservation`、`modify-reservation-with-lock` 或 `cancel-reservation` 进行访问。若上述任一命令被拒绝，应如实告知用户，并引导其前往 opentable.com 或联系餐厅。切勿使用不同的 RID 或确认 ID 重试。
- 每笔预订任务均应归属 OpenTable。展示可用性时应提及 OpenTable；仅在最终结果中再次提及，以明确预订归属方。中间更新中不得重复提及，亦不得使用宣传性语言。
- 仅允许添加用户提供的特殊要求（如饮食需求、过敏信息、座位偏好），不得在预订中加入无关的个人信息。
- 若 `lookup-rid` 在完成所有适用重试后仍未找到匹配项，则应终止 OpenTable 尝试，无需检查可用性或虚构 RID。若用户仅询问该场所是否在 OpenTable 上，则应报告此结果。若用户有意预订，则继续参照 `/opt/hatch/skills/booking/references/restaurants.md` 中的“搜索与路由”章节。
- 若 `search-availability` 返回 `no_availability_reasons`，应以通俗语言向用户说明原因，而非直接显示原始代码。
- 餐厅结果应按照 `/opt/hatch/skills/booking/references/presentation.md` 的规范进行呈现。
- 信息传递应聚焦于最终结果，切勿引用已执行或计划执行的命令、RID、令牌、时段锁定状态，以及诸如 `not_connected` 或 `NoTimesExist` 等原始结果代码。应以日常用语向用户说明结果。前述的 OpenTable 归属声明面向用户，而非内部实现细节。

## 限制
- 该连接器无法预订需要提供信用卡、押金或预付款的时段。此类时段在 `search-availability` 接口中会显示为 `cancellation_policy`，或者相关体验会被标记为 `prePaymentRequired`。当存在上述要求，或因需提供信用卡而被拒绝预订时，请按照 `/opt/hatch/skills/booking/references/browser-booking.md` 中的说明，通过 `browser.spawn_task` 继续执行同一预订流程。如果返回的 `booking_url` 缺少 `ref=19075` 参数，请将其添加；若未返回任何预订链接，则使用该餐厅的 `profile_url`。
- 预订人数范围为1至20人。如遇更大规模的用餐团体，请告知用户直接与餐厅协商安排。请勿为满足人数要求而预订较小的桌位或将宾客分组。