# 托管代理 - 端点引用

所有端点都要求提供 `x-api-key` 和 `anthropic-version: 2023-06-01` 请求头。托管代理的端点还额外需要 `anthropic-beta` 请求头。

> 大多数用户应将代理和环境定义为通过 `ant apply` 同步的版本控制文件——参见 `shared/anthropic-cli.md`。以下端点是 CLI 和 SDK 背后的底层 API。

## Beta 请求头

```
anthropic-beta: managed-agents-2026-04-01
```

SDK 会自动为所有 `client.beta.{agents,environments,sessions,vaults,deployments,deployment_runs}.*` 调用添加此请求头。内存存储端点（`client.beta.memory_stores.*`）则使用 `agent-memory-2026-07-22`，SDK 也会自动设置；在内存存储请求中同时发送这两个请求头会导致返回 400 错误。文件和技能 API 已退出 Beta 阶段，无需 Beta 请求头。

---

## SDK 方法参考

所有资源均位于 `beta` 命名空间下。Python 和 TypeScript 的方法名称完全一致。

| 资源 | Python / TypeScript (`client.beta.*`) | Go (`client.Beta.*`) |
| --- | --- | --- |
| 代理 | `agents.create` / `retrieve` / `update` / `list` / `archive` | `Agents.New` / `Get` / `Update` / `List` / `Archive` |
| 代理版本 | `agents.versions.list` | `Agents.Versions.List` |
| 环境 | `environments.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Environments.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| 环境工作（自托管） | `environments.work.poller` / `stats` / `stop` | 参见 `shared/managed-agents-self-hosted-sandboxes.md` |
| 会话 | `sessions.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Sessions.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| 会话事件 | `sessions.events.list` / `send` / `stream` | `Sessions.Events.List` / `Send` / `StreamEvents` |
| 会话线程 | `sessions.threads.list` / `retrieve` / `archive`；`sessions.threads.events.list` / `stream` | `Sessions.Threads.List` / `Get` / `Archive`；`Sessions.Threads.Events.List` / `StreamEvents` |
| 会话资源 | `sessions.resources.add` / `retrieve` / `update` / `list` / `delete` | `Sessions.Resources.Add` / `Get` / `Update` / `List` / `Delete` |
| 部署 | `deployments.create` / `update` / `pause` / `unpause` / `archive` / `run` | 尚未文档化——请通过 Web 获取 SDK 仓库（参见 `shared/live-sources.md`） |
| 部署运行 | `deployment_runs.list` / `retrieve`（TS：`deploymentRuns.*`） | 尚未文档化——请通过 Web 获取 SDK 仓库（参见 `shared/live-sources.md`） |
| 保险库 | `vaults.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Vaults.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| 凭证 | `vaults.credentials.create` / `retrieve` / `update` / `list` / `delete` / `archive` / `mcp_oauth_validate` | `Vaults.Credentials.New` / `Get` / `Update` / `List` / `Delete` / `Archive` / `McpOauthValidate` |
| 内存存储 | `memory_stores.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `MemoryStores.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| 记忆 | `memory_stores.memories.create` / `retrieve` / `update` / `list` / `delete` | `MemoryStores.Memories.New` / `Get` / `Update` / `List` / `Delete` |
| 内存版本 | `memory_stores.memory_versions.list` / `retrieve` / `redact` | `MemoryStores.MemoryVersions.List` / `Get` / `Redact` |

**需要注意的命名细节：**
- 代理和会话线程**不可删除**，仅支持“归档”。归档是**永久性的**：代理变为只读状态，新会话无法引用该代理，且无法取消归档。在对生产环境中的代理执行归档操作前，请务必与用户确认。环境、会话、保险库、凭据和记忆存储同时支持“删除”和“归档”；会话资源、文件、技能和记忆仅支持“删除”；记忆版本则既不支持“删除”，也不支持“归档”，仅支持“编辑”。
- 会话资源使用`add`（而非`create`）。
- Go语言的事件流为`StreamEvents`（而非`Stream`）。
- 自托管工作进程类为`anthropic.lib.environments` / `@anthropic-ai/sdk/helpers/beta/environments` / `anthropic-sdk-go/lib/environments`中的`EnvironmentWorker`；`client.beta.environments.work.worker(...)`是一个工厂方法，返回同一类，并提供`environments.work.poller/stats/stop`等客户端方法。

**代理简写：** 在创建会话时，`agent`参数接受三种形式——纯字符串（如`agent="agent_abc123"`，表示最新版本）、固定引用（`{type: "agent", id, version}`），或`{type: "agent_with_overrides", id, version?, model?, system?, tools?, mcp_servers?, skills?}`，用于仅针对本次会话覆盖指定字段（参见`shared/managed-agents-core.md` -> 为会话覆盖代理配置）。

**模型简写：** 在创建代理时，`model`参数可接受纯字符串（如`model="claude-opus-5-5"`，默认使用`standard`速度）或完整配置对象，后者除`id`外还包含`speed`、`effort`和`inference_geo`：`{id: "claude-opus-5-5", speed: "fast"}`、`{id: "claude-opus-5-5", effort: "high"}`、`{id: "claude-opus-5-5", inference_geo: "us"}`。`effort`可接受级别字符串（`low`/`medium`/`high`/`xhigh`/`max`）或`{type: "<level>"}`；在会话级别的`model`覆盖中，它会设置该会话的努力等级（代理自身的`effort`不会被继承，且未指定`effort`的`model`覆盖将按该模型的默认努力等级运行）。`inference_geo`（`"us"` | `"global"`）用于锁定为代理模型请求提供服务的地域，并在会话级别的`model`覆盖中同样生效。详情请参阅`shared/managed-agents-core.md` -> 代理模型的努力等级 / 锁定推理地域。注意：`speed: "fast"`仅在Claude Opus 5.5、Claude Opus 5及Opus 4.8上受支持——仅限于Claude API，包括托管代理，但不适用于Amazon Bedrock、Google Cloud或Microsoft Foundry。Opus 4.7的快速模式已下线；在Opus 4.7上使用`speed: "fast"`将返回错误。

---

## 代理

**每个流程的第一步。** 会话必须预先创建一个代理——在`managed-agents-2026-04-01`中不存在内联代理配置。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/agents` | ListAgents | 列出所有代理 |
| `POST` | `/v1/agents` | CreateAgent | 创建并保存代理配置 |
| `GET` | `/v1/agents/{agent_id}` | GetAgent | 获取代理详情 |
| `POST` | `/v1/agents/{agent_id}` | UpdateAgent | 更新代理配置。`version`为**可选**：若提供（≥1），则启用乐观并发控制——版本不匹配时返回409；若省略，则执行无条件的“最后写入优先”更新。 |
| `POST` | `/v1/agents/{agent_id}/archive` | ArchiveAgent | 归档代理。使其变为**只读**状态；现有会话可继续，但新会话无法引用。此为最终状态，不可恢复。 |
| `GET` | `/v1/agents/{agent_id}/versions` | ListAgentVersions | 列出代理的所有版本 |

## 会话

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions` | ListSessions | 列出会话（分页） |
| `POST` | `/v1/sessions` | CreateSession | 创建新会话 |
| `GET` | `/v1/sessions/{session_id}` | GetSession | 获取会话详情 |
| `POST` | `/v1/sessions/{session_id}` | UpdateSession | 更新会话的 `metadata`/`title`、`agent.tools`/`agent.mcp_servers`（仅限会话本地覆盖；会话必须处于 `idle` 状态），或更新 `budget`——可上调或下调预算上限（新值必须高于已消耗的列表成本），也可将其设为 `null` 以移除；移除操作不可逆，会话创建后无法再添加预算。`vault_ids` 仅可在创建时设置（更新时会被拒绝）。详见 `shared/managed-agents-core.md` 中的“会话中更新代理配置”和“会话预算”部分。 |
| `DELETE` | `/v1/sessions/{session_id}` | DeleteSession | 删除会话 |
| `POST` | `/v1/sessions/{session_id}/archive` | ArchiveSession | 归档会话 |

## 事件

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/events` | ListEvents | 列出事件（轮询，分页） |
| `POST` | `/v1/sessions/{session_id}/events` | SendEvents | 发送事件（用户消息、工具结果） |
| `GET` | `/v1/sessions/{session_id}/events/stream` | StreamEvents | 通过 SSE 流式传输事件。可选参数 `event_deltas[]=agent.message` / `agent.thinking` 可开启实时预览的 `event_start`/`event_delta` 事件——详见 `shared/managed-agents-events.md` 中的“实时预览”章节。 |

## 会话线程

多代理会话中的每个子代理事件流。详见 `shared/managed-agents-multiagent.md`。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/threads` | ListThreads | 列出线程（分页） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}` | GetThread | 获取单个线程信息（包含 `agent` 快照、`status`、`parent_thread_id`、`stats`、`usage`） |
| `POST` | `/v1/sessions/{session_id}/threads/{thread_id}/archive` | ArchiveThread | 归档线程 |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/events` | ListThreadEvents | 列出单一线程的历史事件（分页） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/stream` | StreamThreadEvents | 通过 SSE 流式传输单一线程的事件（SDK：`threads.events.stream`）。 |

## 会话资源

| 方法   | 路径                                                    | 操作        | 描述                              |
| -------- | ------------------------------------------------------- | ---------------- | ---------------------------------------- |
| `GET` | `/v1/sessions/{session_id}/resources` | ListResources | 列出会话所附加的资源 |
| `POST` | `/v1/sessions/{session_id}/resources` | AddResource | 附加 `file` 或 `github_repository` 类型的资源（SDK 方法为 `add`，而非 `create`）。`memory_store` 类型的资源仅可在会话创建时附加。自托管环境**仅**支持 `memory_store` 类型资源（创建时有效）；`file` 和 `github_repository` 类型资源在该环境下将被拒绝。 |
| `GET` | `/v1/sessions/{session_id}/resources/{resource_id}` | GetResource | 获取单个资源 |
| `POST` | `/v1/sessions/{session_id}/resources/{resource_id}` | UpdateResource | 更新资源 |
| `DELETE` | `/v1/sessions/{session_id}/resources/{resource_id}` | DeleteResource | 从会话中移除资源 |

## 环境| 方法   | 路径                                                             | 操作            | 描述                         |
| -------- | ---------------------------------------------------------------- | -------------------- | ----------------------------------- |
| `POST`   | `/v1/environments`                                     | CreateEnvironment    | 创建环境                  |
| `GET`    | `/v1/environments`                                     | ListEnvironments     | 列出所有环境                   |
| `GET`    | `/v1/environments/{environment_id}`                    | GetEnvironment       | 获取环境详情             |
| `POST`   | `/v1/environments/{environment_id}`                    | UpdateEnvironment    | 更新环境                  |
| `DELETE` | `/v1/environments/{environment_id}`                    | DeleteEnvironment    | 删除环境。返回 204 状态码。 |
| `POST`   | `/v1/environments/{environment_id}/archive`            | ArchiveEnvironment   | 归档环境。使其变为**只读**；现有会话继续运行，新会话无法引用该环境。不可解档——此为最终状态。 |
| `GET`    | `/v1/environments/{environment_id}/work/stats`         | WorkQueueStats       | 自托管工作队列的深度、待处理任务数及当前工作者数。需使用 `x-api-key` 认证。参见 `shared/managed-agents-self-hosted-sandboxes.md`。 |
| `POST`   | `/v1/environments/{environment_id}/work/{work_id}/stop` | StopWork            | 自托管：停止已领取的工作项。需使用 `x-api-key` 认证。 |

对于 `type: "self_hosted"`，`config` 只是简化的 `{"type": "self_hosted"}`——`networking` 和 `packages` 不适用。（无论哪种类型，`networking` 始终不用于控制 `web_search` 或 `web_fetch`——这些功能由代理工具集中的 `allowed_domains` 和 `blocked_domains` 进行逐个工具的限制；参见 `shared/managed-agents-tools.md`。）

## 部署

计划部署（ID 以 `depl_` 开头）会按设定的 Cron 定时调度运行代理——每次触发都会创建一个会话。有关概念性说明（Cron 和夏令时语义、失败处理、生命周期），请参阅 `shared/managed-agents-scheduled-deployments.md`。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `POST`   | `/v1/deployments`                                | CreateDeployment | 创建一个计划部署            |
| `POST`   | `/v1/deployments/{deployment_id}`                | UpdateDeployment | 更新部署配置（参见 `shared/managed-agents-scheduled-deployments.md`） |
| `POST`   | `/v1/deployments/{deployment_id}/pause`          | PauseDeployment  | 暂停计划触发（可恢复；仍允许手动运行） |
| `POST`   | `/v1/deployments/{deployment_id}/unpause`        | UnpauseDeployment | 从下一次触发开始恢复（不会补发） |
| `POST`   | `/v1/deployments/{deployment_id}/archive`        | ArchiveDeployment | **最终状态**——调度停止，部署变为不可修改 |
| `POST`   | `/v1/deployments/{deployment_id}/run`            | RunDeployment    | 立即触发一次手动运行（`trigger_context.type: "manual"`）；在暂停状态下也可执行 |

## 部署运行

每次触发尝试（无论是计划触发还是手动触发）都会生成一条 `deployment_run` 记录（ID 以 `drun_` 开头），其中包含所创建的 `session_id`，或记录一种错误类型（`environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited`、`service_unavailable`）。| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `GET`    | `/v1/deployment_runs?deployment_id=...`          | ListDeploymentRuns | 列出某个部署的运行记录（分页；可通过 `has_error=true` 过滤失败的运行） |
| `GET`    | `/v1/deployment_runs/{deployment_run_id}`        | GetDeploymentRun   | 根据 ID 获取单个运行记录（`deployment_run.*` Webhook 事件会将此 ID 作为 `data.id` 传递） |

## 保险库

保险库用于存储由 Anthropic 代表您管理的凭据——包括具有自动刷新功能的 OAuth 凭据或静态 Bearer Token，以及在出站请求时被替换到请求中的 `environment_variable` 类型凭据。可通过 `vault_ids` 将其附加到会话中。有关概念性说明及凭据格式，请参阅 `managed-agents-tools.md` 中的“保险库”部分。

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ---------------- | ---------------------------------------- |
| `POST`   | `/v1/vaults`                                     | CreateVault      | 创建一个保险库                           |
| `GET`    | `/v1/vaults`                                     | ListVaults       | 列出所有保险库                           |
| `GET`    | `/v1/vaults/{vault_id}`                          | GetVault         | 获取保险库详情                           |
| `POST`   | `/v1/vaults/{vault_id}`                          | UpdateVault      | 更新保险库                               |
| `DELETE` | `/v1/vaults/{vault_id}`                          | DeleteVault      | 删除保险库                               |
| `POST`   | `/v1/vaults/{vault_id}/archive`                  | ArchiveVault     | 归档保险库                               |

## 凭据

凭据是存储在保险库中的单个密钥。

| 方法   | 路径                                                              | 操作          | 描述                  |
| -------- | ----------------------------------------------------------------- | ------------------ | ---------------------------- |
| `POST`   | `/v1/vaults/{vault_id}/credentials`                               | CreateCredential   | 创建一条凭据          |
| `GET`    | `/v1/vaults/{vault_id}/credentials`                               | ListCredentials    | 列出保险库中的所有凭据    |
| `GET`    | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | GetCredential      | 获取凭据的元数据      |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | UpdateCredential   | 更新凭据              |
| `DELETE` | `/v1/vaults/{vault_id}/credentials/{credential_id}`               | DeleteCredential   | 删除凭据              |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}/archive`       | ArchiveCredential  | 归档凭据              |
| `POST`   | `/v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate` | McpOauthValidate | 验证 MCP OAuth 凭据 |

## 内存存储

工作区范围内的持久化内存，可在多个会话之间保持数据。可通过在创建会话时于 `resources[]` 中添加 `{"type": "memory_store", "memory_store_id": ...}` 来将其附加到会话中。有关概念性说明、FUSE 挂载代理接口、前置条件及版本控制，请参阅 `shared/managed-agents-memory.md`。| 方法   | 路径                                             | 操作          | 描述                              |
| -------- | ------------------------------------------------ | ------------------ | ---------------------------------------- |
| `POST`   | `/v1/memory_stores`                              | CreateMemoryStore  | 创建存储（`name`、`description`、`metadata`） |
| `GET`    | `/v1/memory_stores`                              | ListMemoryStores   | 列出存储（`include_archived`、`created_at_{gte,lte}`） |
| `GET`    | `/v1/memory_stores/{memory_store_id}`            | GetMemoryStore     | 获取存储详情                        |
| `POST`   | `/v1/memory_stores/{memory_store_id}`            | UpdateMemoryStore  | 更新存储                             |
| `DELETE` | `/v1/memory_stores/{memory_store_id}`            | DeleteMemoryStore  | 删除存储                             |
| `POST`   | `/v1/memory_stores/{memory_store_id}/archive`    | ArchiveMemoryStore | 归档存储。使其变为**只读**；现有会话可继续使用，新会话无法引用。不可解档。

## 记忆

存储中的单个文本文档（每篇不超过100KB）。`create` 按指定路径创建，若该路径已被占用则返回 `409` 错误（`memory_path_conflict_error`，并附带冲突的 `memory_id`）；`update` 按 `mem_...` ID 进行变更（可修改名称和/或内容）。只有 `update` 支持设置前置条件（`{"type": "content_sha256", "content_sha256": ...}`），若不匹配则返回 `409` 错误（`memory_precondition_failed_error`）。列表接口支持 `view: "basic"|"full"` 参数（控制是否填充 `content`；`retrieve` 默认为 `full`）。

| 方法   | 路径                                                              | 操作      | 描述                              |
| -------- | ----------------------------------------------------------------- | -------------- | ---------------------------------------- |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memories`                    | ListMemories   | 返回 `Memory \| MemoryPrefix`；可按 `path_prefix` 和 `depth` 过滤 |
| `POST`   | `/v1/memory_stores/{memory_store_id}/memories`                    | CreateMemory   | 按指定路径创建（SDK：`memories.create`）；若路径已占用则返回 `409 memory_path_conflict_error` |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | GetMemory      | 读取某条记忆（默认视图为 `full`） |
| `PATCH`  | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | UpdateMemory   | 按 ID 更改 `content`、`path` 或两者；可选前置条件 |
| `DELETE` | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}`        | DeleteMemory   | 删除记忆（可选 `expected_content_sha256`） |

## 记忆版本

每次变更生成的不可变快照（`memver_...`），用于审计与回滚。记录操作类型为 `created` / `modified` / `deleted`。

| 方法   | 路径                                                                          | 操作             | 描述                              |
| -------- | ----------------------------------------------------------------------------- | --------------------- | ---------------------------------------- |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memory_versions`                         | ListMemoryVersions    | 按时间倒序排列；可按 `memory_id`、`operation`、`session_id`、`api_key_id`、`created_at_{gte,lte}` 过滤 |
| `GET`    | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}`            | GetMemoryVersion      | 显示字段及完整 `content`             |
| `POST`   | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact`     | RedactMemoryVersion   | 清除 `content`、`content_sha256`、`content_size_bytes` 和 `path`；保留操作者信息及时间戳 |

## 文件

| 方法   | 路径                                             | 操作        | 描述                              |
| -------- | ------------------------------------------------ | ----------- | ---------------------------------------- |
| `POST`   | `/v1/files`                            | 上传文件       | 上传文件                            |
| `GET`    | `/v1/files`                            | 列出文件       | 列出文件                               |
| `GET`    | `/v1/files/{file_id}`                  | 获取文件       | 获取文件元数据（SDK方法：`retrieve_metadata`） |
| `GET`    | `/v1/files/{file_id}/content`          | 下载文件     | 下载文件内容                    |
| `DELETE` | `/v1/files/{file_id}`                  | 删除文件       | 删除文件                            |

## 技能

| 方法   | 路径                                                            | 操作          | 描述                  |
| -------- | --------------------------------------------------------------- | -------------- | ---------------------------- |
| `POST`   | `/v1/skills`                                          | 创建技能        | 创建技能               |
| `GET`    | `/v1/skills`                                          | 列出技能         | 列出技能                  |
| `GET`    | `/v1/skills/{skill_id}`                               | 获取技能           | 获取技能详情            |
| `DELETE` | `/v1/skills/{skill_id}`                               | 删除技能        | 删除技能               |
| `POST`   | `/v1/skills/{skill_id}/versions`                      | 创建版本      | 创建技能版本         |
| `GET`    | `/v1/skills/{skill_id}/versions`                      | 列出版本       | 列出技能版本          |
| `GET`    | `/v1/skills/{skill_id}/versions/{version}`            | 获取版本         | 获取技能版本            |
| `DELETE` | `/v1/skills/{skill_id}/versions/{version}`            | 删除版本      | 删除技能版本         |

---

## 请求/响应模式速查

### CreateAgent 请求体

**请始终从这里开始。** `model`、`system`、`tools`、`mcp_servers`、`skills` 是该对象的顶级字段，它们**不属于会话对象**。

```json
{
  "name": "字符串（必填，1-256个字符）",
  "model": "claude-opus-5-5（必填 - 纯字符串，或 {id, speed?, effort?, inference_geo?} 对象）",
  "description": "字符串（可选，最多2048个字符）",
  "system": "字符串（可选，最多100,000个字符）",
  "tools": [
    { "type": "agent_toolset_20260401" }
  ],
  "skills": [
    { "type": "anthropic", "skill_id": "xlsx" },
    { "type": "custom", "skill_id": "skill_abc123", "version": "1" }
  ],
  "mcp_servers": [
    {
      "type": "url",
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  ],
  "multiagent": {
    "type": "coordinator",
    "agents": [
      "agent_abc123",
      { "type": "agent", "id": "agent_def456", "version": 4 },
      { "type": "self" }
    ]
  },
  "metadata": {
    "key": "value（最多16对，键<=64个字符，值<=512个字符）"
  }
}
```

> 限制：`tools` 最多128个，`skills` 最多20个，`mcp_servers` 最多20个（名称需唯一）。`multiagent.agents` 可包含1至20项（字符串ID | `{type:"agent",id,version?}` | `{type:"self"}` | `{type:"advisor",model}`，且最多允许一个顾问）——详情参见 `shared/managed-agents-multiagent.md`。

### CreateSession 请求体

```json
{
  "agent": "agent_abc123（必填 - 字符串，表示最新版本，或使用 {type: \"agent\", id, version} 对象）",
  "environment_id": "env_abc123（必填）",
  "title": "字符串（可选）",
  "resources": [
    {
      "type": "github_repository",
      "url": "https://github.com/owner/repo（必填）",
      "authorization_token": "ghp_...（必填）",
      "mount_path": "/workspace/repo（可选 - 默认为 /workspace/<repo-name>）",
      "checkout": { "type": "branch", "name": "main" }
    }
  ],
  "initial_events": [
    { "type": "user.message", "content": [{ "type": "text", "text": "审查认证模块。" }] }
  ],
  "vault_ids": ["vlt_abc123（可选 - 用于存储凭据：MCP 认证及环境变量）"],
  "budget": {
    "type": "limit",
    "max_list_cost": { "amount": "2500", "currency": "USD" }
  },
  "metadata": {
    "key": "value"
  }
}
```

> `agent` 字段可以接受字符串形式的 ID，也可以接受 `{type: "agent", id, version}` 或 `{type: "agent_with_overrides", id, version?, ...}` 的对象格式，用于在会话中局部覆盖 `model`、`system`、`tools`、`mcp_servers` 和 `skills` 等配置。在非覆盖形式下，这些字段属于代理本身，而非此处定义。如果在 `model` 覆盖中指定了 `effort`，则会应用该值（代理自身的 `effort` 不会被继承；若 `model` 覆盖中未指定 `effort`，则按该模型的默认力度运行）。而在 `model` 覆盖中指定的 `inference_geo` 则会被应用（若省略，则会清除代理在本次会话中的地域限制）。  
>  
> **`budget`**（可选，仅限创建时设置）是会话按列表价计算的支出上限；`amount` 是以最小单位（美分）表示的整数字符串（例如 `"2500"` 表示 25.00 美元），仅支持 USD 币种。后续可通过更新会话来修改或移除预算，但无法新增。详情请参阅 `shared/managed-agents-core.md` 中的“会话预算”部分。  
>  
> **`initial_events`**（可选，最多 50 条）会在创建时发送事件，并在同一调用中启动代理循环。仅支持 `user.message` 和 `user.define_outcome` 类型的事件，不支持 `system.message` 及任何工具执行结果类事件。验证采用全有或全无机制。详情请参阅 `shared/managed-agents-core.md` 中的“通过 initial_events 种子化会话”部分。  
>  
> **`checkout`** 支持 `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}` 格式。若省略，则使用仓库的默认分支。

### CreateEnvironment 请求体

```json
{
  "name": "字符串（必填）",
  "description": "字符串（可选）",
  "config": {
    "type": "cloud | self_hosted",
    "networking": {
      "type": "unrestricted | limited（联合类型，请参考 SDK 类型定义）"
    },
    "packages": { }
  },
  "metadata": { "key": "value" }
}
```

### CreateDeployment 请求体

```json
{
  "name": "每周合规扫描",
  "agent": "agent_abc123（必填 - 与 CreateSession 相同的结构）",
  "environment_id": "env_abc123（必填）",
  "initial_events": [
    { "type": "user.message", "content": [{ "type": "text", "text": "执行每周合规扫描。" }] }
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
```

> 可选的会话配置（如 `resources`、`vault_ids` 等）与 CreateSession 的处理方式相同，包括 `budget`——它会被复制到每次触发的会话中；与会话级别的预算不同，部署级别的预算可以在不存在时添加，也可在清空后重新添加（详见 `shared/managed-agents-scheduled-deployments.md` 中的“部署预算”章节）。响应中包含 `status`、`paused_reason` 和 `schedule.upcoming_runs_at`（下次触发时间）。详情请参阅 `shared/managed-agents-scheduled-deployments.md`。

### SendEvents 请求体

```json
{
  "events": [
    {
      "type": "user.message",
      "content": [
        {
          "type": "text",
          "text": "你好"
        }
      ]
    }
  ]
}
```

> `system.message` 事件（为本轮及后续轮次追加系统级上下文）使用具有 `type: "system.message"` 的相同信封——支持的模型包括 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5.5、Claude Opus 5、Claude Opus 4.8 和 Claude Sonnet 5.5（不包括 Claude Sonnet 5），仅针对代理的*主*模型进行检查；详情请参阅 `shared/managed-agents-events.md` 中的“在会话中途添加系统上下文”一节。

### 定义结果事件

```json
{
  "type": "user.define_outcome",
  "description": "用 .xlsx 格式构建 Costco 的现金流折现模型",
  "rubric": { "type": "file", "file_id": "file_01..." },
  "max_iterations": 5
}
```

> `rubric` 为必填项：可为 `{type: "text", content}` 或 `{type: "file", file_id}`。`max_iterations` 默认值为 3，最大值为 20。响应中会返回 `outcome_id` 和 `processed_at`。详情请参阅 `shared/managed-agents-outcomes.md`。

### 工具结果事件

```json
{
  "type": "user.custom_tool_result",
  "custom_tool_use_id": "sevt_abc123",
  "content": [{ "type": "text", "text": "结果数据" }],
  "is_error": false
}
```

---

## 错误处理

托管代理端点采用标准 Anthropic API 错误格式。错误以 HTTP 状态码和包含 `type`、`error` 和 `request_id` 的 JSON 响应体形式返回：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "描述出错原因"
  },
  "request_id": "req_011CRv1W3XQ8XpFikNYG7RnE"
}
```

向 Anthropic 报告问题时，请附上 `request_id`——这有助于我们对请求进行端到端追踪。内部 `error.type` 包括以下几种：

| 状态 | 错误类型 | 描述 |
|---|---|---|
| 400 | `invalid_request_error` | 请求格式错误或缺少必要参数 |
| 401 | `authentication_error` | API 密钥无效或缺失 |
| 403 | `permission_error` | 当前 API 密钥无权执行此操作 |
| 404 | `not_found_error` | 请求的资源不存在 |
| 409 | `invalid_request_error` | 请求与资源当前状态冲突（例如，发送至已归档的会话） |
| 413 | `request_too_large` | 请求体超出允许的最大大小 |
| 429 | `rate_limit_error` | 请求次数过多——请查看速率限制头信息以获取重试时间 |
| 500 | `api_error` | 发生了服务器内部错误 |
| 529 | `overloaded_error` | 服务暂时过载——请退避后重试 |

请注意，“409 Conflict”对应的 `error.type` 为 `invalid_request_error`（没有单独的 `conflict_error` 类型）；需同时检查 HTTP 状态码和 `message` 字段，以区分冲突与其他无效请求。

---

## 分页

大多数托管代理的列表端点均采用 `page` / `next_page` 游标机制：

| 字段 | 出现位置 | 备注 |
|---|---|---|
| `limit` | 查询参数 | 每页最多返回的项目数 |
| `page` | 查询参数 | 来自上一次响应的不透明游标——在此处传入 `next_page` 或 `prev_page` 值 |
| `order` | 查询参数 | 支持排序的端点上的 `asc` / `desc` 排序方式。游标编码了生成它的请求的排序顺序——若使用不同的排序方式重复使用该游标，将返回 400 错误。其他参数（如过滤条件、`limit`）可在分页请求间更改。 |
| `next_page` | 响应 | 下一页的游标；当无更多结果时为 `null` |
| `prev_page` | 响应 | 支持反向分页的端点上的上一页游标——目前**仅限于 `GET /v1/sessions`**。在第一页时为 `null`。对于不支持反向分页的端点，该字段**不存在**（而非 `null`）。 |

每个 SDK 都提供一个遵循 `next_page` 的自动分页迭代器。在 Python 和 TypeScript 中，可直接迭代列表结果；其他 SDK 则通过单独的方法暴露迭代器（直接迭代原始列表结果只会返回一页）。SDK 自动分页**仅支持向前**——若要返回上一页，需从响应中读取 `prev_page`，并将其作为 `page` 参数手动传回。> 警告：部分端点使用**不同的**游标机制：消息批次、文件、模型以及若干管理 API 端点采用 `after_id`/`before_id` 参数，并返回 `has_more`、`first_id` 和 `last_id`，而非 `page` 和 `next_page`。某些采用 `page` 机制的端点（例如 `GET /v1/skills`）也会在返回 `next_page` 的同时返回一个 `has_more` 布尔值。请参阅相应端点的参考文档，以了解其具体的分页字段。

---

## 速率限制

Managed Agents 相关端点具有按组织划分的每分钟请求数（RPM）限制，该限制与您的 [Messages API 令牌限制]（https://platform.claude.com/docs/en/api/rate-limits）相互独立。会话内的模型推理仍会占用您所在组织的标准 ITPM/OTPM 限额。

| 端点组 | 作用范围 | RPM | 最大并发数 |
|---|---|---|---|
| 创建操作（Agent、会话、Vault） | 组织级 | 300 | - |
| 其他所有操作（Agent、会话、Vault） | 组织级 | 600 | - |
| 所有操作（环境） | 组织级 | 60 | 5 |

文件和技能相关端点遵循标准的分级[速率限制]（https://platform.claude.com/docs/en/api/rate-limits）。

当超出限制时，API 将返回状态码 `429`，并附带 `rate_limit_error` 错误（响应结构详见[错误处理](#error-handling)），同时在 `retry-after` 头中指明应等待的秒数后再进行重试。Anthropic SDK 会自动读取该头信息并执行重试。
