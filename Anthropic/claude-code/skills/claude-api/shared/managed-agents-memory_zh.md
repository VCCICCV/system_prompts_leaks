# 托管代理 - 内存存储

> **公开测试版。** 内存存储在 `agent-memory-2026-07-22` 测试版标头下发布；SDK 会在所有 `client.beta.memory_stores.*` 调用中自动设置该标头。请勿在这些调用中添加 `managed-agents-2026-04-01`——同时发送两个标头的内存存储请求会返回 400 错误。将存储附加到会话属于会话级操作，仍需使用 `managed-agents-2026-04-01` 标头。如果缺少 `client.beta.memory_stores`，请升级至最新 SDK 版本。

默认情况下，会话是短暂的——会话结束时，代理所学内容将全部丢失。**内存存储**是一个工作空间范围内的小型文本文档集合，可在多个会话间持久保存。当存储通过 `resources[]` 附加到会话时，它会被挂载为容器中的一个文件系统目录；代理可使用常规文件工具对其进行读写，且系统提示中会告知其该挂载点的存在。

对内存的每次变更都会生成一个不可变的 **内存版本**（`memver_...`），从而提供审计追踪和时间点回滚/删除功能。

> 警告：**切勿在内存存储中存放凭据、API 密钥或令牌。** 内存会在会话间持久保存，并原样传递到后续上下文中——一旦密钥被写入，它将在所有后续挂载该存储的会话中被重复使用。请改用 Vault 的 `environment_variable` 凭据（参见 `shared/managed-agents-tools.md` -> Vaults）。如果已有秘密被写入，请删除相关内存并删除受影响的版本（详见下方“删除某个版本”）。

## 对象模型

| 对象 | ID 前缀 | 作用域 | 备注 |
| --- | --- | --- | --- |
| 内存存储 | `memstore_...` | 工作空间 | 通过 `resources[]` 附加到会话 |
| 内存 | `mem_...` | 存储 | 单个文本文件，按 `path` 地址访问（每个不超过 100KB——建议使用大量小文件） |
| 内存版本 | `memver_...` | 内存 | 每次变更生成一个不可变快照；`operation` 取值为 `created` / `modified` / `deleted` |

## 创建存储

`description` 会传递给代理，以便其了解存储的内容——请以模型理解的方式撰写，而非面向人类。

```python
store = client.beta.memory_stores.create(
    name="用户偏好",
    description="用户的个性化偏好及项目上下文。",
)
print(store.id)  # memstore_01Hx...
```

其他 SDK：TypeScript 使用 `client.beta.memoryStores.create({...})`；Go 使用 `client.Beta.MemoryStores.New(ctx, ...)`. 完整的语言对应表请参见 `shared/managed-agents-api-reference.md` -> SDK 方法参考。

存储支持 `retrieve` / `update` / `list`（带 `include_archived` 和 `created_at_{gte,lte}` 过滤器）/ `delete` / **`archive`**。归档后，存储变为只读——现有会话的挂载继续有效，但新会话无法引用；不支持取消归档。

### （可选）预置内容

在任何会话运行之前预先加载参考资料。`memories.create` 会在指定的 `path` 下创建一条内存；若该路径已存在内存，则调用会返回 409 错误（`memory_path_conflict_error`，并附带 `conflicting_memory_id`）。存储 ID 是第一个位置参数。

```python
client.beta.memory_stores.memories.create(
    store.id,
    path="/formatting_standards.md",
    content="所有报告均采用 GAAP 格式。日期格式为 ISO-8601...",
)
```

## 附加到会话

内存存储与 `file` 和 `github_repository` 资源一同放入会话的 `resources[]` 数组中（参见 `shared/managed-agents-environments.md` -> 资源）。内存存储只能在**会话创建时**附加——`sessions.resources.add()` 不接受 `memory_store`。在**自托管**环境中，会话的附加方式相同（且 `memory_store` 是这些环境唯一支持的资源类型）——详情请参阅下方的自托管说明。
```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[
        {
            "type": "memory_store",
            "memory_store_id": store.id,
            "access": "read_write",  # 或 "read_only"；默认为 "read_write"
            "instructions": "用户偏好和项目上下文。在开始任何任务前请先查看。",
        }
    ],
)
```

| 字段 | 必填 | 备注 |
| --- | --- | --- |
| `type` | 是 | `"memory_store"` |
| `memory_store_id` | 是 | `memstore_...` |
| `access` | - | `"read_write"`（默认）或 `"read_only"`——在云端挂载时由文件系统层面强制执行；在自托管沙盒中，则由工作进程的 `write`/`edit` 工具以及上传路径（见下文）来强制实施 |
| `instructions` | - | 针对该存储的会话级指导说明，与存储的 `name`/`description` 相辅相成。不超过 4,096 个字符。 |

**每个会话最多可附加 8 个内存存储。** 当不同片段的内存具有不同的所有者或生命周期时，可附加多个存储——例如，一个只读的共享参考存储加上一个每位用户的读写存储，或者为每个终端用户、团队或项目分配一个存储，并共用同一个代理配置。

### 代理如何看到它（FUSE 挂载）

每个附加的存储都会在会话容器中挂载到 `/mnt/memory/<store-name>/`。代理通过标准的文件操作工具（`bash`、`read`、`write`、`edit`、`glob`、`grep`）与其交互——没有专门的内存操作工具。在云沙盒中，`access: "read_only"` 会使该挂载点在文件系统层面变为只读（在自托管沙盒中，则由工作进程的 `write`/`edit` 工具及上传路径来强制实施——见下文）；而 `"read_write"` 则允许代理在其下创建、编辑和删除文件。每个挂载点的简要说明（名称、路径、`instructions`、访问权限）会自动注入到系统提示中，因此代理无需您特别提及便能知晓该存储的存在。

代理在该挂载点下的写入操作会被持久化回存储，并生成与主机端 `memories.update` 调用相同的记忆版本。

**自托管沙盒：同步的本地副本，而非实时挂载。** 在 `self_hosted` 环境中，SDK 工作进程（`EnvironmentWorker`——Python、TypeScript、Go；`ant` CLI 工作进程不挂载存储）会将每个附加的存储下载到同一路径 `/mnt/memory/<store-name>/`，并按一定间隔与存储进行同步，因此写入内容仅在同步后才会对其他会话可见，冲突时以存储中的内容为准，且 `read_only` 由工作进程的工具而非文件系统来强制实施（`bash` 仍可修改本地副本）。其余内容——同步间隔、会话级 `secret`、主机准备、故障排查等——详见 `shared/managed-agents-self-hosted-sandboxes.md` 中的“内存存储”部分。在 AWS 上的 Claude Platform 自托管环境中不可用。

## 主机端直接管理记忆

可用于审核流程、修正错误记忆，或在带外方式下预置存储内容。

### 列表

返回 `Memory | MemoryPrefix` 条目——`MemoryPrefix`（`type: "memory_prefix"`，仅为 `path`）在分层列出时表现为目录类节点。使用 `path_prefix` 可限定范围（需包含尾部斜杠：`"/notes/"` 匹配 `/notes/a.md`，但不匹配 `/notes_backup/old.md`），使用 `depth` 可限制遍历深度。传入 `view="full"` 可在每个条目中包含 `content`；默认的 `"basic"` 仅返回元数据。

```python
for m in client.beta.memory_stores.memories.list(store.id, path_prefix="/"):
    if m.type == "memory":
        print(f"{m.path}  ({m.content_size_bytes} 字节, sha={m.content_sha256[:8]})")
    else:  # "memory_prefix"
        print(f"{m.path}/")
```

### 读取

```python
mem = client.beta.memory_stores.memories.retrieve(memory_id, memory_store_id=store.id)
print(mem.content)
```

`retrieve` 默认为 `view="full"`（包含内容）；`view` 参数主要影响列表接口。

### 创建与更新的区别| 操作 | 受影响对象 | 语义 |
| --- | --- | --- |
| `memories.create(store_id, path=..., content=...)` | **路径** | 在指定路径下创建。如果该路径已被占用，则返回 `409` 错误（`memory_path_conflict_error`，包含 `conflicting_memory_id`）。 |
| `memories.update(mem_id, memory_store_id=..., path=..., content=...)` | **`mem_...` ID** | 修改现有记忆。可更改 `content`、`path`（重命名）或两者。若重命名为已占用的路径，仍会返回相同的 `409 memory_path_conflict_error`。 |

```python
mem = client.beta.memory_stores.memories.create(
    store.id,
    path="/preferences/formatting.md",
    content="始终使用制表符，而非空格。",
)

client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    path="/archive/2026_q1_formatting.md",  # 重命名
)
```

### 乐观并发控制（`update` 的前提条件）

`memories.update` 支持 `precondition` 参数，允许您在读取 -> 修改 -> 写回的过程中避免覆盖其他并发写入。目前仅支持 `content_sha256` 类型。若校验失败，API 将返回 `409` 错误（`memory_precondition_failed_error`），此时需重新读取并基于最新状态重试。

```python
client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    content="已修正：始终使用 2 空格缩进。",
    precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
)
```

### 删除

```python
client.beta.memory_stores.memories.delete(mem.id, memory_store_id=store.id)
```

可传入 `expected_content_sha256` 实现条件性删除。

## 审计与回滚——记忆版本

每次变更都会生成一个不可变的 `memver_...` 快照。版本会在父记忆的生命周期内持续累积；`memories.retrieve` 始终返回当前的最新版本，而版本相关接口则提供历史记录。

| 触发操作 | 版本上的 `operation` 字段 |
| --- | --- |
| 在新路径上执行 `memories.create` | `"created"` |
| 执行 `memories.update` 并修改了 `content`、`path` 或两者（或代理对挂载点的写入） | `"modified"` |
| 执行 `memories.delete` | `"deleted"` |

每个版本还会记录 `created_by`——一个包含 `type` 属性的主体对象，其值为 `session_actor`、`api_actor` 或 `user_actor`——以及在内容被遮蔽后的时间戳和操作者信息（`redacted_at` 和 `redacted_by`）。

### 列出版本

按时间倒序分页显示。可按 `memory_id`、`operation`、`session_id`、`api_key_id` 或 `created_at_gte`/`created_at_lte` 进行过滤。传入 `view="full"` 可获取完整内容，默认仅返回元数据。

```python
for v in client.beta.memory_stores.memory_versions.list(store.id, memory_id=mem.id):
    print(f"{v.id}: {v.operation}")
```

### 获取某个版本

```python
version = client.beta.memory_stores.memory_versions.retrieve(
    version_id, memory_store_id=store.id
)
print(version.content)
```

### 遮蔽某个版本

在保留审计轨迹（操作者及时间戳）的同时，清除历史版本中的具体内容。将清空 `content`、`content_sha256`、`content_size_bytes` 和 `path`，其余字段保持不变。适用于敏感信息泄露、个人隐私数据或用户删除请求场景。

```python
client.beta.memory_stores.memory_versions.redact(version_id, memory_store_id=store.id)
```

## 接口参考

完整的 HTTP 方法与路径表格请参见 `shared/managed-agents-api-reference.md` 中的“Memory Stores / Memories / Memory Versions”部分。原始 HTTP 基础路径如下：

```
POST   /v1/memory_stores
POST   /v1/memory_stores/{memory_store_id}/archive
GET    /v1/memory_stores/{memory_store_id}/memories
PATCH  /v1/memory_stores/{memory_store_id}/memories/{memory_id}
GET    /v1/memory_stores/{memory_store_id}/memory_versions
POST   /v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact
```

有关 cURL 示例及 CLI 命令（`ant beta:memory-stores ...`），请在 `shared/live-sources.md` 的“Managed Agents”部分中查找 Memory URL 并通过 WebFetch 获取。