---
name: "figma"
description: >-
  检查 Figma 设计，并为界面生成实现上下文。
  界面。用于读取组件变体、间距和设计令牌，并进行检查。
  布局和设计系统库，并提取用于实现的图像资源
  通过 Figma 的官方 MCP 服务器进行设计。
icon: "figma"
metadata: { "不包含在提示中": 假 }
---
# Figma

使用已安装的 `figma` CLI。首先运行 `figma status`。如果显示 `not_connected`，请运行 `figma authorize-url`，并将返回的 `connect_url` 仅分享给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Figma 为每次注册颁发一个机密客户端；authd 将其生成的密钥存储在运行时单元之外，因此绝不能在聊天中请求凭据。

运行 `figma list-tools` 以查看实时提供商目录和模式，然后调用已公布的工具，命令如下：

```text
figma call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出已审核的 Figma 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新发布的提供商工具在未审核之前仍不可用。读取权限遵循用户的连接器设置；设计、文件、资源、Code Connect、插件和着色器的更改需要逐项审批。切勿自动重试失败或超时的写操作，因为其副作用可能已经完成。

## 加载 Figma 的提供者技能

在调用需要 Figma 技能描述的工具之前，请先使用 `get_figma_skill` 获取该技能，并遵循其前提条件。这些 `skill://` URI 指向 Figma 的 MCP 服务器上的资源。例如，在调用 `create_new_file` 之前，请先读取：

```text
figma call-tool --name get_figma_skill --arguments-json '{"uri":"skill://figma/figma-create-new-file/SKILL.md"}'
```

请使用工具所公布的技能 URI。如果相关 URI 不明，可使用同一工具获取 `skill://index.json`，并从中选择匹配的技能。仅在必要时获取支持性参考，并根据父技能 URI 解析相对路径。对于当前任务已加载的指导内容，请直接复用，无需将这些提供者技能安装或复制到本地技能目录。技能读取需使用 `designs.read` 权限。

仅当 `list-tools` 中有相应提示时才调用 `get_figma_skill`。如果工具不可用，或 Figma 确认缺少某项技能，则仅在该技能为可选（例如“如果存在”）的情况下，方可继续使用实时工具描述、模式及本地指南。若该技能为必选项，则应停止依赖该技能的操作，并报告缺失的指导内容。认证失败、权限被拒、超时及其他读取错误均不能作为技能缺失的依据。避免重复尝试查找失败的操作；对于不依赖于该缺失指导的工作，可继续进行。

## 选择工作流程

在调用提供者工具前，仅阅读与任务相符的指南：

- 将 Figma 设计转化为代码：[design-to-code](references/design-to-code.md)
- 使用 `use_figma` 创建或编辑画布内容：[canvas editing](references/canvas-editing.md)
- 生成 FigJam 流程图：[diagrams](references/diagrams.md)
- 创建 Code Connect 映射：[Code Connect](references/code-connect.md)

对于 Figma URL，请提取其中的 `fileKey` 和 `node-id`；并将节点 ID 从 URL 格式（`123-456`）转换为 API 格式（`123:456`）。在构造参数之前，请先检查实时模式，因为 Figma 可能在不改变该技能的情况下新增可选字段。

避免重复调用目录、上下文、截图及验证接口。Figma 对账户和席位施加使用限制，重复读取可能会消耗用户有限的月度配额。请复用任务前期已收集的结果。