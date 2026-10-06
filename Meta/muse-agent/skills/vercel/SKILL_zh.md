---
name: "vercel"
description: >-
  检查并管理用户的 Vercel 网站、Web 应用和项目，以及
  部署。用于读取构建和运行时日志，排查生产环境中的错误。
  以及部署失败的情况，检查部署状态，并通过……部署项目
  Vercel 的官方 MCP 服务器。
icon: "vercel"
metadata: { "不包含在提示中": 假 }
---
# Vercel

使用已安装的 `vercel` CLI。首先运行 `vercel status`。如果显示 `not_connected`，请运行 `vercel authorize-url`，并将返回的 `connect_url` 仅提供给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Vercel 将客户端声明为公开型且不颁发客户端密钥，因此不会有任何共享凭据进入 Muse 系统，也绝不能在聊天中请求凭据。对于本 MCP 服务器，Vercel 仅开放身份验证和会话相关的 OAuth 范围，因此默认的只读行为由下方的连接器权限而非更窄的提供商范围来强制执行。

运行 `vercel list-tools` 可以查看当前可用的提供商目录及其 Schema，然后通过以下命令调用已公布的工具：

```text
vercel call-tool --name <tool> --arguments-json '<json-object>'
```

经审核的目录遵循 Vercel 当前公开的 MCP 工具文档，其中包括 `create_deployment`。旧名称 `deploy_to_vercel` 以及其他近期已被替换的名称，仅在上线过程中实时服务器仍将其作为公开接口时才被接受。务必使用 `list-tools` 返回的实时 Schema；例如，当前 `create_deployment` 的参数将部署定义置于 `requestBody` 下。

`list-tools` 仅列出已审核的 Vercel 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新出现的提供商工具在未完成审核前均不可用。普通读取操作遵循用户的连接器设置。解密后的密钥、部署或沙盒文件内容、部署记录、购买记录、凭据创建、安全配置变更、沙盒执行等涉及写入的操作，则需单独申请细粒度权限，默认情况下会进行权限提示。Vercel 仅开放身份验证/会话 OAuth 范围，因此不存在可针对单个工具向提供商申请的增量范围。

购买类工具可能会产生即时且不可退款的费用。当实时工具的合约中提供报价时，请获取该报价并展示给用户，切勿自行推断用户已确认。对于任何失败或超时的写入操作，切勿自动重试，因为其副作用可能已经执行完毕。