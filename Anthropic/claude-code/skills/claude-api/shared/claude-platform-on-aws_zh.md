# AWS 上的 Claude 平台

通过 AWS 基础设施提供由 Anthropic 运营的 Claude 开发者平台访问权限——采用 SigV4 身份验证、AWS IAM 访问控制以及 AWS Marketplace 计费。由于由 Anthropic 运营，**API 表面与官方版本保持同日同步**——如遇特定功能的例外情况，请参阅 `shared/platform-availability.md`（这是唯一权威来源；请勿依赖此处的内联例外列表）。模型 ID 为原生的官方字符串（如 `claude-opus-5-5`、`claude-sonnet-5-5`），**无提供商前缀**。

> **与 Amazon Bedrock 不同。** Bedrock 是由合作伙伴运营的（AWS 负责运行服务；发布节奏各异，功能集有所差异，模型 ID 带有 `anthropic.` 前缀）。AWS 上的 Claude 平台与 Bedrock 共存；请根据需求选择：若需要原生 AWS 的 IAM 和计费，并享有与 Anthropic 完全一致的 API，则使用本页面所述方案；若希望融入 Bedrock 自有的生态系统，则选择 Bedrock。

---

## 客户端与安装

| 语言 | 安装 | 客户端 |
|---|---|---|
| Python | `pip install -U "anthropic[aws]"` | `from anthropic import AnthropicAWS` -> `AnthropicAWS()` |
| TypeScript | `npm install @anthropic-ai/aws-sdk` | `import AnthropicAws from "@anthropic-ai/aws-sdk"` -> `new AnthropicAws()` |
| Go | `go get github.com/anthropics/anthropic-sdk-go` | `import anthropicaws "github.com/anthropics/anthropic-sdk-go/aws"` -> `anthropicaws.NewClient(ctx, anthropicaws.ClientConfig{})` |
| C# | `dotnet add package Anthropic.Aws` | `new AnthropicAwsClient()` |
| Java | 参见 `shared/live-sources.md` 中的 SDK 仓库 | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| Ruby | `gem install anthropic aws-sdk-core` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| PHP | `composer require anthropic-ai/sdk aws/aws-sdk-php` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |

实例化后，**可按与 `Anthropic()` 相同的方式使用客户端**——例如 `client.messages.create(...)`、`client.beta.sessions.*` 等，并直接使用原生模型 ID。

```python
from anthropic import AnthropicAWS

client = AnthropicAWS()  # 区域和工作空间 ID 从环境变量中读取；详见下文
client.messages.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "你好"}],
)
```

---

## 必需配置

必须提供以下两项值（可通过构造函数参数或环境变量设置）——两者均**无默认值**：

| 参数 | 环境变量 | 备注 |
|---|---|---|
| AWS 区域 | `AWS_REGION` | 必填。与 `AnthropicBedrock` 不同，本服务不支持 `us-east-1` 作为默认区域。 |
| 工作空间 ID | `ANTHROPIC_AWS_WORKSPACE_ID` | 必填。用于将请求路由至您的 Claude 工作空间。 |

端点格式：`https://aws-external-anthropic.{region}.api.aws/v1/...`。请求采用 SigV4 签名，服务名称为 `aws-external-anthropic`。

## 身份验证

客户端会按标准优先级顺序解析 AWS 凭证：显式构造函数参数 -> 环境变量（`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`）-> 共享配置文件 -> 角色扮演或实例元数据。

对于无法使用 SigV4 的场景（如浏览器环境或简单脚本），也支持使用**短期 API 密钥**。可通过各语言对应的令牌生成工具包获取密钥，并在客户端初始化时以 `api_key` 参数传入。其有效期为所申请时长、底层凭证到期时间以及**12 小时**中的**最短值**。有关工具包名称及 IAM 权限详情，请在 `shared/live-sources.md` 中查阅“AWS 上的 Claude 平台”页面。

---

## 应告知用户的内容- 视其为第一方：本技能的各部分均按原样适用，**不得**应用 Bedrock 的功能可用性掩码。仅在以下三个方面存在差异：(1) 会话可在无需用户事件的情况下最多自主运行 **6 小时**，之后需重新认证——请发送任何用户角色事件以继续；(2) 在 **自托管** 环境中，会话 **无法附加内存存储**（在创建会话时即被拒绝）——云环境则可照常附加；(3) 自托管工作节点使用 IAM/SigV4 或 AWS 控制台 API 密钥，并附加 `AnthropicSelfHostedEnvironmentAccess` 托管策略进行认证——由控制台生成的环境密钥无法用于 AWS 终端节点。
- 模型 ID 应为裸 ID（如 `claude-opus-5-5`），**不得**添加 `anthropic.` 前缀。
- 如果缺少区域或 `workspace_id`，将在客户端构造时抛出异常（不会发出请求）。返回 **403** 错误表示请求已到达服务器——请检查是否 `workspace_id` **错误**，或主体缺少必要的 IAM 操作权限。有关 IAM 操作的参考，请参阅 `shared/live-sources.md` 中的相关内容。