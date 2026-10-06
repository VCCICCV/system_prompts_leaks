---
name: "github"
title: "GitHub"
description: "通过 GitHub 的官方 MCP 服务器搜索并操作用户的 GitHub 仓库。"
icon: "github"
metadata: { "不包含在提示中": 假 }
---
# GitHub

使用已安装的 `github` CLI。除非这是成功的 OAuth 后续流程——该流程已建立授权——否则请从 `github status` 开始。成功的状态检查仅验证 OAuth 和 MCP 的连通性，而不涉及安装或仓库访问权限。

如果 OAuth 已断开连接，请运行 `github authorize-url` 并提供其返回的 `connect_url`。每位用户都需通过共享的 Connect 流程，以自己的 GitHub 身份对 Muse 进行授权。只需执行授权步骤，无需进行连接；成功完成 OAuth 后续流程后，系统会自动处理仓库的安装。

授权完成后，在回复前运行 `github install-url`，但不要添加 `--state` 参数。切勿自行构造 URL 或复用 OAuth 状态。如果该命令失败或未返回 `install_url`，请说明无法生成安装链接；切勿创建小部件、虚构链接或声称设置已完成。

简要确认授权，并说明由账户所有者或组织管理员选择 Muse 可访问的仓库。使用 `widget.create` 呈现完全相同的返回 `install_url`，并指定 `kind: "list"`。在 `data.items` 中，仅包含一行，其内容为：`title: "Select repositories"`、`subtitle: "Choose which repositories Muse can access"`、`type: "link"`，且 `data.url` 应直接取自 `install_url`。将该行的 `image_url` 设置为 GitHub 官方图标：`https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png`。省略列表标题。在回复中单独列出返回的 `embed_token`，不要同时输出 URL、图标或 Markdown 链接。该行点击后将打开 GitHub，用于安装或配置仓库访问权限。

如果 `widget.create` 不可用或创建失败，请在句子中使用一个标注为 **Select repositories** 的 Markdown 链接，链接后的文字与之在同一行（例如：“打开 `[Select repositories](URL)` 以选择 Muse 可访问的仓库。”，其中 URL 为实际返回的链接）。切勿将链接单独成行或置于行尾，以免显示为大型预览卡片。

如果 OAuth 已连接且需要配置仓库，请直接进入此选择步骤，无需重新启动 OAuth。对于已存在的安装，GitHub 会打开其设置页面，可能不会重定向回 Muse。请用户在完成仓库访问配置后返回聊天界面。

用户完成后，应通过经审核的只读工具验证其任务所需的仓库访问权限。切勿仅凭 OAuth 状态或点击安装链接就声称拥有私有仓库的访问权限。用户的访问令牌受限于其自身的权限以及应用安装时所选的仓库范围。安装过程无需重复 OAuth。

切勿仅因缺少某个仓库而断开或重复 OAuth。应先检查当前登录的身份及安装所选的仓库；只有在身份验证需要修复或已配置的 GitHub App 身份发生变更时，才重新连接。

GitHub 对所有用户使用同一个固定的、归 Muse 拥有的 GitHub App 身份，类似于 Slack 和 QuickBooks 的固定应用身份。该应用必须允许在目标账户上安装，但这不需要在 GitHub Marketplace 上架。在验证期间，仅将其安装在临时测试仓库上。更广泛的生产部署仍需经过审核，并由组织明确批准后方可实施。

运行 `github list-tools` 以查看实时的提供商目录及其架构。每个工具都包含一个本地的 `hatch_command` 分类：

- `call-read-tool` 仅限于 Muse 经审核的只读白名单，使用连接器的读权限。
- `call-tool` 是用于变更操作以及所有新工具或未知工具的可写备用选项，每次使用均需获得批准。

使用返回的命令调用工具：

```text
github call-read-tool --name <reviewed-read-tool> --arguments-json '<json-object>'
github call-tool --name <write-or-unknown-tool> --arguments-json '<json-object>'
```

GitHub 应用的用户令牌不使用传统的 OAuth 作用域，因此不存在可用于公开的 OAuth 作用域升级流程。渐进式访问权限的实现依赖于应用安装时的仓库选择，以及 Muse 的独立读写权限控制。切勿仅凭 GitHub 的实时目录来推断读取操作的安全性：该二进制程序会在 `call-read-tool` 调用时拒绝未经审核的名称，而新发布的工具在通过审核之前仍具备写入能力。切勿在聊天中请求 OAuth 令牌或应用凭据。