# 管理 API（组织管理）

当用户希望通过编程方式管理其 Anthropic 组织时，请阅读本文件：包括成员与角色、邀请、工作空间及工作空间成员、API 密钥、速率限制报告、服务账户、工作负载身份联合（WIF）以及客户管理的加密密钥（CMEK）。

管理 API 的地址为 `https://api.anthropic.com/v1/organizations/*`。它用于管理组织本身，而不用于发送消息。自 **2026 年 8月26日** 起，该 API 已在所有七种 SDK 中提供（Python、TypeScript、C#、Go、Java、PHP、Ruby），可通过 `client.beta.organization` 访问；在 `ant` CLI 中则通过 `ant beta:organization` 命令使用。用量报告、费用报告以及 Claude Enterprise 的用户管理和分析相关端点 **未** 包含在 SDK 中——这些端点需通过原生 HTTP 请求调用。

## 身份验证

有两种凭据类型，均会由默认 SDK 客户端和 CLI 自动读取：

| 凭据 | 环境变量 | HTTP 头 | 涉及范围 |
| --- | --- | --- | --- |
| 管理 API 密钥 (`sk-ant-admin...`) | `ANTHROPIC_API_KEY` | `x-api-key` | 大多数端点 |
| `org:admin` OAuth 令牌 | `ANTHROPIC_AUTH_TOKEN` | `authorization: Bearer` | 所有端点，包括仅限 OAuth 的端点 |

- **仅限 OAuth 的端点：** 服务账户、联合颁发者和联合规则均不接受 API 密钥，必须使用 `org:admin` OAuth 令牌。
- **优先级陷阱：** 当两个环境变量同时设置时，某些客户端会优先使用 API 密钥。使用 Bearer 令牌时，请确保该 Shell 中未设置 `ANTHROPIC_API_KEY`。
- 管理 API 密钥由组织管理员在 Claude 控制台中创建。
- 普通（非管理员）API 密钥无法用于任何上述端点，而管理员凭据也无法用于消息 API。
- `org:admin` 令牌可授予对整个组织的访问权限，不受任何工作空间绑定的限制。

**交互式 OAuth 令牌：** 使用 `ant` CLI 登录并指定专用配置文件（以避免常规命令以高权限运行），然后导出令牌。令牌有效期较短；遇到 401 错误时，请重新执行导出操作。关于配置文件和作用域机制（为何 `org:admin` 需显式指定 `--scope`，以及如何切换配置文件）：请参阅 `shared/anthropic-cli.md`。

```bash
ant auth login --profile admin --scope "org:admin"
export ANTHROPIC_AUTH_TOKEN=$(ant auth print-credentials --profile admin --access-token)
# 使用完毕后：unset ANTHROPIC_AUTH_TOKEN && ant profile activate default
```

**自动化工作负载（CI）：** 不应进行交互式登录。创建一条具有 `oauth_scope: org:admin` 的联合规则，目标服务账户的 `organization_role` 设置为 `admin`（此规则必须由人工在 Claude 控制台中创建），然后通过联合相关的环境变量指向该规则，并在初始化客户端时不传入任何参数——SDK 或 CLI 会自动完成令牌交换并在过期前刷新：

```bash
export ANTHROPIC_FEDERATION_RULE_ID=fdrl_...       # `org:admin` 规则的 ID
export ANTHROPIC_ORGANIZATION_ID=<org-uuid>
export ANTHROPIC_SERVICE_ACCOUNT_ID=svac_...       # 规则的目标服务账户
export ANTHROPIC_IDENTITY_TOKEN_FILE=/path/to/jwt  # 或 ANTHROPIC_IDENTITY_TOKEN
```

**curl** 在每次请求时也需添加 `anthropic-version: 2023-06-01`。

## 端点覆盖范围

SDK 访问器采用 Python 语法表示；各语言的具体命名约定请参见下表。| 资源 | REST 路径 | SDK 访问器（`client.beta.organization` +） | CLI（`ant beta:organization` +） |
| --- | --- | --- | --- |
| 组织信息 | `GET /v1/organizations/me` | `.retrieve()` | `retrieve` |
| 成员 | `/v1/organizations/users` | `.users` - `list`, `update`, `remove` | `:users list\|update\|remove` |
| 邀请 | `/v1/organizations/invites` | `.invites` - `create`, `list`, `delete` | `:invites create\|list\|delete` |
| 工作空间 | `/v1/organizations/workspaces` | `.workspaces` - `create`, `retrieve`, `list`, `update`, `archive` | `:workspaces create\|list\|update\|archive` |
| 工作空间成员 | `/v1/organizations/workspaces/{id}/members` | `.workspaces.members` - `add`, `list`, `update`, `remove` | `:workspaces:members add\|list\|update\|remove` |
| API 密钥 | `/v1/organizations/api_keys` | `.api_keys` - `list`, `update` | `:api-keys list\|update` |
| 组织级速率限制 | `GET /v1/organizations/rate_limits` | `.rate_limits.list(model=..., group_type=...)` | `:rate-limits list` |
| 工作空间级速率限制 | `GET /v1/organizations/workspaces/{id}/rate_limits` | `.workspaces.rate_limits.list(workspace_id)` | `:workspaces:rate-limits list` |
| 服务账户 (*) | `/v1/organizations/service_accounts` | `.service_accounts` - `create`, `list`, `archive` | `:service-accounts create\|list\|archive` |
| 联邦身份提供商 | `/v1/organizations/federation_issuers` | `.federation.issuers` - `create`, `list`, `archive` | `:federation:issuers create\|list\|archive` |
| 联邦规则 (*) | `/v1/organizations/federation_rules` | `.federation.rules` - `create`, `list`, `archive` | `:federation:rules create\|list\|archive` |
| CMEK 外部密钥 | `/v1/organizations/external_keys` | `.external_keys` - `create`, `validate` | - |

(*) 仅限 OAuth：需要 `org:admin` 的 Bearer Token，而非 API Key。

将 CMEK 外部密钥关联到工作空间属于工作空间更新操作：`client.beta.organization.workspaces.update("<workspace-id>", external_key_id="ekey_...")`。

## 各语言的命名与分页
| 语言 | 访问器风格（以列出成员为例） | 列表行为 |
| --- | --- | --- |
| Python | `client.beta.organization.users.list(limit=10)` | 迭代器会自动加载更多页面；`limit` 表示每页大小，而非总条数 |
| TypeScript | `client.beta.organization.users.list({ limit: 10 })` - 子资源采用驼峰命名：`apiKeys`, `rateLimits`, `serviceAccounts`, `externalKeys` | 使用 `for await` 自动分页 |
| C# | `client.Beta.Organization.Users.List(new() { Limit = 10 })` | 使用 `await foreach (var u in page.Paginate())` 自动分页 |
| Go | `client.Beta.Organization.Users.ListAutoPaging(ctx, params)`；组织信息使用 `Organization.Get(ctx)` | 通过 `.Next()` 和 `.Current()` 自动分页 |
| Java | `client.beta().organization().users().list(params)`，参数采用构建器模式（`UserListParams.builder().limit(10).build()`） | 使用 `.autoPager()` 自动分页 |
| PHP | `$client->beta->organization->users->list(limit: 10)` | 直接获取单页数据——遍历 `->getItems()`；SDK 的自动分页辅助功能尚未针对这些端点实现 |
| Ruby | `client.beta.organization.users.list(limit: 10)` | 直接获取单页数据——遍历 `.data`；SDK 的自动分页辅助功能尚未针对这些端点实现 |
| CLI | `ant beta:organization:users list --limit 10` | 在成员、邀请、工作空间、工作空间成员和 API 密钥列表中，`--limit` 用于限制返回结果的数量（不同于大多数 `ant` 列表命令，后者用 `--limit` 设置每页大小而用 `--max-items` 来限制总条数——参见 `shared/anthropic-cli.md`） |
| curl | `GET /v1/organizations/users?limit=10` | 每次请求仅返回一页；按 Admin API 参考文档进行游标分页 |

速率限制列表（`rate_limits` 和 `workspaces.rate_limits`）在发布时也支持分页——应像其他列表端点一样进行分页，而不是假定只返回单个响应。Go 参数类型遵循 `anthropic.BetaOrganizationUserListParams` 的模式（其中 `Limit` 使用 `anthropic.Int(10)`）；Java 参数则使用 `com.anthropic.models.beta.organization.*` 中的构建器（例如 `UserListParams.builder().limit(10).build()`）。Go 和 Java 的分页循环如下：

```go
users := client.Beta.Organization.Users.ListAutoPaging(ctx, anthropic.BetaOrganizationUserListParams{Limit: anthropic.Int(10)})
for users.Next() {
	user := users.Current() // ...
}
if err := users.Err(); err != nil { /* 处理错误 */ }
```

```java
for (var user : client.beta().organization().users().list(params).autoPager()) { /* ... */ }
```

## 示例

常见操作（Python 语法；可参照上表映射到其他语言——每种语言的操作形式均相同）：

```python
# 获取组织信息
org = client.beta.organization.retrieve()

# 列出成员（迭代器会自动获取更多页面；limit 即为每页大小）
for user in client.beta.organization.users.list(limit=10):
    print(f"{user.id}: {user.email} ({user.role})")

# 更改成员角色或移除成员
client.beta.organization.users.update("user_...", role="developer")
client.beta.organization.users.remove("user_...")

# 邀请用户
client.beta.organization.invites.create(email="user@example.com", role="developer")

# 创建工作空间并添加成员
ws = client.beta.organization.workspaces.create(name="Production")
client.beta.organization.workspaces.members.add(
    ws.id, user_id="user_...", workspace_role="workspace_developer"
)

# 停用或重命名 API 密钥
client.beta.organization.api_keys.update("apikey_...", status="inactive", name="New Key Name")

# 速率限制报告（可选过滤条件：model=..., group_type=...）
client.beta.organization.rate_limits.list(model="claude-opus-5-5")
client.beta.organization.workspaces.rate_limits.list("wrkspc_...")

# 服务账户与 WIF（需具备 org:admin OAuth 令牌）
sa = client.beta.organization.service_accounts.create(name="inference-worker", organization_role="developer")
issuer = client.beta.organization.federation.issuers.create(
    name="github-actions",
    issuer_url="https://token.actions.githubusercontent.com",
    jwks={"type": "discovery"},
)
client.beta.organization.federation.rules.create(
    name="gha-deploy",
    issuer_id=issuer.id,
    match={"subject_prefix": "repo:my-org/my-repo:ref:refs/heads/main",
           "claims": {"repository_owner": "my-org"}},
    target={"type": "service_account", "service_account_id": sa.id},
    workspace_id="wrkspc_...",
    oauth_scope="workspace:developer",
    token_lifetime_seconds=600,
)

# CMEK：注册、验证后将外部密钥绑定至工作空间
key = client.beta.organization.external_keys.create(
    display_name="prod-key", geo="us",
    provider_config={"type": "aws", "kms_arn": "arn:aws:kms:..."},
)
client.beta.organization.external_keys.validate(key.id)
client.beta.organization.workspaces.update("wrkspc_...", external_key_id=key.id)
```

## 组织角色

| 角色 | 权限 |
| --- | --- |
| `user` | Playground |
| `claude_code_user` | Playground + Claude Code |
| `developer` | Playground + 管理 API 密钥 |
| `billing` | Playground + 管理计费 |
| `admin` | 上述所有权限 + 管理用户 |

所有者和主要所有者拥有全部管理员权限，还可管理其他管理员。工作空间角色包括 `workspace_user`、`workspace_developer`、`workspace_admin` 和 `workspace_billing`。

## 平台限制

- **Claude Platform on AWS：**仅支持工作空间相关端点。成员、工作空间成员、邀请、API 密钥以及用量/费用/速率限制报告等功能不可用。CMEK 外部密钥相关端点目前也暂不可用——请在 Claude 控制台中注册并绑定密钥。
- **Claude Enterprise（claude.ai 组织）：**仅支持通过该界面进行的成员和邀请操作，以及企业版专用端点（组和自定义角色的读取、支出限额等），这些功能不在 SDK 中提供。

## 实时文档| 主题 | URL |
| --- | --- |
| 管理 API 指南 | `https://platform.claude.com/docs/en/manage-claude/admin-api.md` |
| 管理 API 参考 | `https://platform.claude.com/docs/en/api/admin.md` |
| 工作空间 | `https://platform.claude.com/docs/en/manage-claude/workspaces.md` |
| 速率限制 API | `https://platform.claude.com/docs/en/manage-claude/rate-limits-api.md` |
| WIF 管理 | `https://platform.claude.com/docs/en/manage-claude/wif-admin-api.md` |
| 使用量与费用报表（仅支持 curl） | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md` |
