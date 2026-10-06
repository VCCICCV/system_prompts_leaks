# 托管代理 - 环境与资源

## 环境

创建会话需要提供 `environment_id`。环境是用于在 Anthropic 基础设施中启动容器的**可复用配置模板**——您可以为不同的使用场景（例如数据可视化与 Web 开发，配备不同的软件包集合）创建不同的环境。Anthropic 负责处理扩缩容、容器生命周期及任务编排。

**环境名称必须唯一。** 使用已存在的名称创建环境将返回 409 错误。

### 网络配置

| 网络策略   | 描述                                                   |
| ---------------- | ------------------------------------------------------------- |
| `unrestricted`   | 完全出站访问（除法律禁止列表外）                          |
| `limited`        | 默认拒绝；可通过 `allowed_hosts` / `allow_package_managers` / `allow_mcp_servers` 进行白名单设置 |

```json
{
  "networking": {
    "type": "limited",
    "allow_package_managers": true,
    "allow_mcp_servers": true,
    "allowed_hosts": ["api.example.com"]
  }
}
```

`limited` 网络下的三个字段均为可选。`allow_package_managers`（默认为 `false`）允许访问 PyPI、npm 等包管理服务；`allow_mcp_servers`（默认为 `false`）允许代理配置的 MCP 服务器端点，无需将其列入 `allowed_hosts`。

**MCP 注意事项：** 在 `limited` 网络模式下，需将 `allow_mcp_servers: true` 或将每个 MCP 服务器域名添加至 `allowed_hosts`。否则容器无法访问这些服务器，工具将静默失败。

**软件包注意事项：** 在 `limited` 网络模式下，若使用 `packages` 字段，则必须启用 `allow_package_managers: true`；否则请求将返回 400 错误。仅将注册表列入 `allowed_hosts` 并不足以解决问题。

**`networking` 不影响 `web_search` 和 `web_fetch` 的行为。** 这些工具运行在 Anthropic 的服务器上（包括云端和自托管环境），因此 `limited` 出站策略和 `allowed_hosts` 无法对其施加限制。如需限制其可访问的站点，请在代理工具集中该工具的 `configs` 配置项中设置 `allowed_domains` 和 `blocked_domains`——详情请参阅 `shared/managed-agents-tools.md` 中的“网络搜索与网页抓取设置”部分。

### 创建环境

SDK 会自动添加 `managed-agents-2026-04-01`。TypeScript 示例：

```ts
const env = await client.beta.environments.create({
  name: "my_env",
  config: {
    type: "cloud",
    networking: { type: "unrestricted" },
  },
});
```

### 自托管沙盒

如需将工具执行置于**您自己的基础设施**而非 Anthropic 的基础设施中，请设置 `config: {type: "self_hosted"}`——代理循环仍保留在 Anthropic 一侧，但 `bash`、文件操作及代码将在由您通过轮询工作进程控制的容器中执行。此时 `networking` 配置块不适用（出站策略由您掌控）。资源挂载（`file`、`github_repository`）及内存存储的行为有所不同——有关工作进程、凭据以及云环境与自托管环境的对比，请参阅 `shared/managed-agents-self-hosted-sandboxes.md`。

### 环境的 CRUD 操作

| 操作        | 方法   | 路径                                       | 备注 |
| ---------------- | -------- | ------------------------------------------ | ----- |
| 创建           | `POST`   | `/v1/environments`                         | |
| 列表             | `GET`    | `/v1/environments`                         | 支持分页（`limit`、`after_id`、`before_id`） |
| 获取              | `GET`    | `/v1/environments/{id}`                    | |
| 更新           | `POST`   | `/v1/environments/{id}`                    | 更改仅适用于**新**容器；现有会话保持原有配置 |
| 删除           | `DELETE` | `/v1/environments/{id}`                    | 返回 204。 |
| 归档          | `POST`   | `/v1/environments/{id}/archive`            | 使环境变为**只读**状态；现有会话继续运行，新会话无法引用该环境。不可解档——最终状态。 |

--- 

## 资源

可以将文件、GitHub 仓库和内存存储附加到会话中。资源会在会话创建时进行解析，因此无效的 `file_id` 或无法访问的仓库会在创建调用时立即报错，而不会在运行过程中才暴露问题。创建会话本身**不会**启动工作或预置沙箱——如果没有 `initial_events`，会话仅会被注册，沙箱会在会话首次需要时启动（参见 `shared/managed-agents-core.md` 中的“通过 `initial_events` 种子化会话”部分）。每个会话最多可关联**999个文件资源**。每个会话支持多个 GitHub 仓库。对于 `type: "memory_store"` 类型的资源（跨会话持久化内存，每个会话最多 8 个），请参阅 `shared/managed-agents-memory.md`。

### 文件上传（输入：宿主机 → 代理）

首先通过 Files API 上传文件，然后通过 `file_id` 和 `mount_path` 进行引用：

```ts
// 1. 上传文件
const file = await client.beta.files.upload({
  file: fs.createReadStream("data.csv"),
  purpose: "agent",
});

// 2. 作为会话资源附加
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  resources: [
    { type: "file", file_id: file.id, mount_path: "/workspace/data.csv" }
  ],
});
```

**`mount_path` 是必填项**，且必须为绝对路径。父目录会自动创建。代理的工作目录默认为 `/workspace`。文件以只读方式挂载——代理会将修改后的版本写入新的路径。

### 会话输出（输出：代理 → 宿主机）

代理可以在会话期间将文件写入 `/mnt/session/outputs/` 目录。这些文件会自动被 Files API 捕获，后续可列出并下载：

```ts
// 在本轮对话结束后，列出与该会话相关的输出文件：
for await (const f of client.beta.files.list({
  scope_id: session.id,
  betas: ["managed-agents-2026-04-01"],
})) {
  console.log(f.filename, f.size_bytes);
  const resp = await client.beta.files.download(f.id);
  const text = await resp.text();
}
```

**要求：**
- 必须启用 `write` 工具（或 `bash`）才能让代理创建输出文件。
- 使用会话范围的 `files.list` / `files.download` 可捕获写入 `/mnt/session/outputs/` 的输出文件。
- 筛选参数为 **`scope_id`**（REST 查询参数为 `?scope_id=<session_id>`）。按 `scope_id` 进行筛选需要 `managed-agents-2026-04-01` 头部，而 `client.beta.files` 不会自动添加该头部，因此需显式传入 `betas: ["managed-agents-2026-04-01"]`（在原始 HTTP 请求中需发送 `anthropic-beta: managed-agents-2026-04-01`）；列表调用仅使用 `beta` files 命名空间来传递该头部，而上传和下载操作在 `client.files` 上同样适用。需要 `@anthropic-ai/sdk` ≥ 0.88.0 或 `anthropic`（Python）≥ 0.92.0——较旧版本不支持 `scope_id` 类型。在 `ant` CLI 中，请使用 `ant beta:files list --scope-id <session_id> --beta managed-agents-2026-04-01`。
- 请原样传递 `sessions.create()` 返回的会话 ID（例如 `sesn_011CZx...`）——API 会验证其前缀。
- 从 `session.status_idle` 到输出文件出现在 `files.list` 中之间存在短暂的索引延迟（约 1–3 秒）。如果结果为空，可重试一两次。

> **当无法使用 `scope_id` 筛选时的备用方案**（较旧的 SDK 或端点返回错误）：发送一条后续的 `user.message`，要求代理读取 `/mnt/session/outputs/` 下的每份文件并返回其内容。代理会将文件内容以 `agent.message` 文本的形式流式返回。此方法仅适用于文本文件，并会产生输出 token 费用——请将其作为应急手段，而非主要路径。

由此构建了一个双向文件通道：向内上传参考数据，向外下载代理生成的成果。

### GitHub 仓库

在初始化阶段，会将 GitHub 仓库克隆到会话容器中，在代理开始执行之前完成。代理可以通过 `bash`（即 `git`）进行读取、编辑、提交和推送操作。每个会话支持多个仓库——每个仓库对应一个 `resources` 条目。仓库会被缓存，因此后续使用相同仓库的会话启动速度更快。
挂载一个代码库还会加载其根目录下 `.claude/skills` 目录中存储的所有技能——每个会话仅发现一次，基于会话开始时检出的代码库状态（仅限云端沙盒）。详见 `shared/managed-agents-tools.md` 中的“来自 GitHub 代码库的技能”部分。

代码库会在整个会话期间保持挂载；如需更换挂载的代码库，请创建一个新的会话。您可以通过 `client.beta.sessions.resources.update(resource_id, {session_id, authorization_token})` 在运行中的会话中更新某个代码库的 `authorization_token`；资源 ID 会在会话创建时返回，也可通过 `resources.list()` 获取。

**字段说明：**

| 字段 | 必填 | 备注 |
|---|---|---|
| `type` | 是 | `"github_repository"` |
| `url` | 是 | GitHub 代码库的 URL |
| `authorization_token` | 是 | 具有代码库访问权限的 GitHub 个人访问令牌。**绝不会在 API 响应中显示。** |
| `mount_path` | 否 | 代码库将被克隆到的路径。默认为 `/workspace/<repo-name>`。 |
| `checkout` | 否 | `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}`。默认为代码库的默认分支。 |

**令牌权限级别**（细粒度 PAT）：
- `Contents: Read` - 仅支持克隆
- `Contents: Read and write` - 支持推送更改并创建拉取请求

**认证机制：** `authorization_token` 永远不会进入容器内部。针对已挂载代码库的 `git pull`、`git push` 以及 GitHub REST API 调用，都会通过 Anthropic 侧的 Git 代理进行路由，并在请求离开沙盒后注入该令牌。容器内运行的代码——包括代理写入的任何内容——都无法读取或泄露该令牌。

> 重要提示：**要生成拉取请求**，您还需要具备 GitHub **MCP 服务器**的访问权限——`github_repository` 资源仅提供文件系统和 Git 访问权限。详情请参阅 `shared/managed-agents-tools.md` 中的“MCP 服务器”部分。PR 工作流为：在挂载的代码库中编辑文件 -> 通过 `bash` 推送分支（经由 Git 代理使用 `authorization_token` 进行认证）-> 使用 MCP 的 `create_pull_request` 工具创建 PR（经由 Vault 进行认证）。

**TypeScript 示例：**

```ts
// 1. 创建代理 - 声明 GitHub MCP（此处无需认证）
const agent = await client.beta.agents.create(
  {
    name: 'GitHub Agent',
    model: 'claude-opus-5-5',
    mcp_servers: [
      { type: 'url', name: 'github', url: 'https://api.githubcopilot.com/mcp/' },
    ],
    tools: [
      { type: 'agent_toolset_20260401', default_config: { enabled: true } },
      { type: 'mcp_toolset', mcp_server_name: 'github' },
    ],
  },
);

// 2. 启动会话 - 挂载用于 MCP 认证的 Vault 并挂载代码库
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  vault_ids: [vaultId],  // Vault 中包含 GitHub MCP 的 OAuth 凭证
  resources: [
    {
      type: 'github_repository',
      url: 'https://github.com/owner/repo',
      authorization_token: process.env.GITHUB_TOKEN,  // 代码库克隆用令牌（≠ MCP 认证）
      checkout: { type: 'branch', name: 'main' },
    },
  ],
});
```

**Python 示例：**

```python
import os

agent = client.beta.agents.create(
    name="GitHub Agent",
    model="claude-opus-5-5",
    mcp_servers=[{
        "type": "url",
        "name": "github",
        "url": "https://api.githubcopilot.com/mcp/",
    }],
    tools=[
        {"type": "agent_toolset_20260401", "default_config": {"enabled": True}},
        {"type": "mcp_toolset", "mcp_server_name": "github"},
    ],
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=env_id,
    vault_ids=[vault_id],  # Vault 中包含 GitHub MCP 的 OAuth 凭证
    resources=[{
        "type": "github_repository",
        "url": "https://github.com/owner/repo",
        "authorization_token": os.environ["GITHUB_TOKEN"],  // 代码库克隆用令牌（≠ MCP 认证）
        "checkout": {"type": "branch", "name": "main"},
    }],
)
```

---

## Files API

上传并管理用作会话资源的文件，以及下载代理写入 `/mnt/session/outputs/` 的文件。

| 操作           | 方法   | 路径                                  | SDK |
| -------------- | ------ | ------------------------------------- | --- |
| 上传           | `POST` | `/v1/files`                           | `client.beta.files.upload({ file })` |
| 列出           | `GET`  | `/v1/files?scope_id=...`              | `client.beta.files.list({ scope_id, betas: ["managed-agents-2026-04-01"] })` |
| 获取元数据     | `GET`  | `/v1/files/{id}`                      | `client.beta.files.retrieveMetadata(id)` |
| 下载           | `GET`  | `/v1/files/{id}/content`              | `client.beta.files.download(id)` -> `Response` |
| 删除           | `DELETE` | `/v1/files/{id}`                      | `client.beta.files.delete(id)` |

在“列出”操作中，`scope_id` 过滤器可将结果限定为该会话写入 `/mnt/session/outputs/` 的文件。若不使用该过滤器，则会返回上传到您账户的所有文件。
