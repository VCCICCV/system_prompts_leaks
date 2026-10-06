# 托管代理 - 定时部署

**定时部署**会按照指定的 Cron 表达式周期性地运行代理——每次触发都会自动创建一个会话。适用于具有固定频率的工作场景，例如：每晚的故障排查、每周的合规性扫描、每小时的监控任务。

需要使用 `managed-agents-2026-04-01` 测试版标头（SDK 会为 `client.beta.deployments.*` 和 `client.beta.deployment_runs.*` 调用自动设置该标头）。

## 创建部署

部署将会话所需的一切（代理、环境、可选的文件/GitHub/内存存储/保险库）与调度计划及每次运行的初始事件打包在一起：

- `agent` 和 `environment_id` 是必填字段，其结构与 `sessions.create` 相同（参见 `shared/managed-agents-core.md`）。针对**自托管**环境的部署可以附加 `memory_store` 资源（需要 SDK Worker —— 参见 `shared/managed-agents-self-hosted-sandboxes.md` § 内存存储）；而 `file` 和 `github_repository` 资源则需要云环境。控制台的部署表单不支持为自托管环境添加内存存储，请通过 API 或 SDK 进行配置。
- `initial_events` 至少必须包含一个起始事件——可以是 `user.message` 或 `user.define_outcome`。默认行为与会话相同：对于那些会产生交付物（如周报、合规性扫描结果文件或数据集）的定时运行，应以 `user.define_outcome` 开头，并附带一份预设的评分标准（参见 `shared/managed-agents-outcomes.md`）；只有在运行本质上是对话型且无明确可验证输出时，才使用 `user.message`。（部署的 `initial_events` 还支持 `system.message`，而会话则不支持。）
- `schedule` 需要提供一个 Cron 表达式和一个 IANA 时区。时间粒度最高为分钟级。

```bash
curl -fsSL https://api.anthropic.com/v1/deployments \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d @- <<EOF
{
  "name": "每周合规性扫描",
  "agent": "$AGENT_ID",
  "environment_id": "$ENVIRONMENT_ID",
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "执行每周合规性扫描。"}]}
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
EOF
```

```python
deployment = client.beta.deployments.create(
    name="每周合规性扫描",
    agent=agent.id,
    environment_id=environment.id,
    initial_events=[
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "执行每周合规性扫描。"}],
        },
    ],
    schedule={
        "type": "cron",
        "expression": "0 20 * * 5",
        "timezone": "America/New_York",
    },
)
```

响应是一个部署对象（ID 前缀为 `depl_`）。可通过查看 `schedule.upcoming_runs_at` 来确认下一次触发时间是否符合预期：

```json
{
  "id": "depl_01xyz",
  "status": "active",
  "paused_reason": null,
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York",
    "last_run_at": null,
    "upcoming_runs_at": ["2026-05-09T00:00:00Z", "2026-05-16T00:00:00Z", "2026-05-23T00:00:00Z"]
  }
}
```

`upcoming_runs_at` 反映的是您所配置的确切调度时间，但**实际执行时间会有一定的抖动以均衡负载：最大允许偏差为两次运行间隔的 15%，最小为 5 秒，最大为 9 分钟。** 因此，每小时触发的部署可能会延迟最多 9 分钟；请勿基于列出的时间戳设定下游任务的截止期限。每个组织最多可创建 **1000 个定时部署**（如需更多，请联系 Anthropic 支持团队）。

### Cron 表达式与时区语义

- **表达式：** 标准 POSIX Cron 格式（分钟 小时 月份中的日期 月份 星期）。
- **时区：** IANA 时区标识符（例如 `"America/Los_Angeles"`）。
- **夏令时：** 按照当地实际时间计算——在 `"America/New_York"` 时区，“`0 20 * * *`” 会在当地时间晚上 8 点触发，无论当前是 EST 还是 EDT。> 警告：**夏令时边缘情况**：在春季调整时不存在的时钟时间（例如凌晨2点）会被**跳过**；而在秋季调整时重复出现的时间则会**触发两次**。当无法容忍错过或重复执行时，请将调度安排在本地时间凌晨1点至3点之外，或使用UTC时间。

## 部署预算

部署接受与会话相同的`budget`对象（`{type: "limit", max_list_cost: {amount, currency}}`——以最小单位分表示的字符串，仅支持`USD`；详见`shared/managed-agents-core.md`第“会话预算”节）。该限额会在每次触发时**复制到每个会话**中，之后该会话的行为就完全等同于任何已设定预算的会话。

部署预算的更新语义与会话有所不同：

- `budget`可在**创建和更新时**设置，并非仅限创建时。
- 更新时将`budget`设为`null`会将其**清空**，且清空后的预算**可随后重新添加**——不存在不可逆的操作。
- 更改自**下一次触发的会话**起生效——已在运行的会话仍沿用其创建时的限额（可通过各自的会话更新来修改）。

## 部署运行记录

每次触发尝试——无论成功与否——都会写入一条**部署运行记录**（前缀为`drun_`），因此您可以独立于会话生命周期审计失败情况。成功的运行会携带所创建的`session_id`；您可按常规方式通过事件流（参见`shared/managed-agents-events.md`）或Webhook（参见`shared/managed-agents-webhooks.md`）跟踪该会话。失败的运行则会携带一个`error`字段，其中的`type`会说明会话创建被拒绝的原因。

```python
# 查询某个部署的所有运行记录
for run in client.beta.deployment_runs.list(deployment_id=deployment.id):
    print(run.created_at, run.session_id or run.error.type)

# 仅查询失败的运行记录
for run in client.beta.deployment_runs.list(deployment_id=deployment.id, has_error=True):
    print(run.created_at, run.error.type, run.error.message)
```

```typescript
for await (const run of client.beta.deploymentRuns.list({
  deployment_id: deployment.id,
  has_error: true,
})) {
  console.log(run.created_at, run.error?.type, run.error?.message);
}
```

原始HTTP请求：`GET /v1/deployment_runs?deployment_id=...&has_error=true`。若要按ID获取单个运行记录，可使用`GET /v1/deployment_runs/{deployment_run_id}`（SDK调用：`client.beta.deployment_runs.retrieve(run_id)`）——`deployment_run.*`类型的Webhook事件会将运行ID作为其`data.id`字段传递。

失败的运行记录示例如下：

```json
{
  "type": "deployment_run",
  "id": "drun_01abc124",
  "deployment_id": "depl_01xyz",
  "trigger_context": { "type": "schedule", "scheduled_at": "2026-05-09T00:00:00Z" },
  "session_id": null,
  "error": { "type": "environment_archived", "message": "环境 `env_01abc` 已归档" },
  "agent": { "type": "agent", "id": "agent_01ghi789", "version": 3 },
  "created_at": "2026-05-09T00:00:01Z"
}
```

错误类型包括`environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited`以及`service_unavailable`。

每次**定时**运行的结果（启动/成功/失败）以及每次部署生命周期的变化（创建/更新/暂停/恢复/归档/删除）也会作为Webhook事件发出——具体请参阅`shared/managed-agents-webhooks.md`中的`deployment.*`和`deployment_run.*`事件类型——这样您无需轮询即可作出响应。手动运行**不会**触发`deployment_run.*`类型的Webhook事件。

## 生命周期操作：暂停/恢复/归档

| 操作 | SDK调用 | 效果 |
|---|---|---|
| 暂停 | `client.beta.deployments.pause(id)` | 自此之后，定时触发将被抑制。已在运行的会话将继续执行。**暂停期间仍允许手动触发。** 设置`paused_reason: {"type": "manual"}`。 |
| 恢复 | `client.beta.deployments.unpause(id)` | 自下一次计划触发起恢复执行。**不会补发错过的触发。** 清除`paused_reason`。 |
| 归档 | `client.beta.deployments.archive(id)` | **最终状态**——计划停止，且该部署不再可修改。对于任何可逆操作，请使用暂停。 |

原始 HTTP 请求：`POST /v1/deployments/{deployment_id}/pause`（同理适用于 `/unpause` 和 `/archive`）。

### 失败行为

- **被限流**：立即记录为 `session_rate_limited` 类型的运行，**不重试**——调度将在下一次触发时重新尝试。（会话内部的 API 调用限流由会话自身处理。）
- **其他失败的运行**（如 `environment_archived`、`vault_not_found`、`service_unavailable`）：运行会记录 `error.type`——请监控相关运行并修复所引用的资源，或暂停该部署。
- **代理已归档**：部署会在同一操作中自动**归档**（终止状态）。**代理已被删除**：下次计划触发时会检测到代理缺失，并随后归档该部署。无论哪种情况，都不会记录部署运行，也不会再创建新的会话。

## 手动运行

`POST /v1/deployments/{deployment_id}/run`（SDK：`client.beta.deployments.run(id)`）会立即创建一个会话，并生成一条 `trigger_context.type: "manual"` 的运行记录。可用于在正式启用调度前**测试部署**——请注意，即使部署处于暂停状态，此功能也仍然可用。