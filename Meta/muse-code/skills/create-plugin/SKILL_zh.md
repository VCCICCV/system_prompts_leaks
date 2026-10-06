---
name: create-plugin
description: 在当前工作区中创建并验证一个新的原生 Muse 插件包。仅当用户明确请求创建 Muse 插件或调用“create-plugin”技能时使用。请勿用于应用程序/库的插件类、第三方插件系统或普通的代码变更。
---
# 创建插件

在当前工作区中创建一个全新的原生 Muse 插件。此技能仅用于创建 Muse 原生插件包，不得用于在其他项目中实现插件类或插件功能。生成满足需求的最小化完整插件，并使用已安装的 muse CLI 进行验证，安装步骤由用户自行完成。

## 边界条件

- 仅在当前工作区内的一个新目标路径下操作。
- 绝不修改、合并、删除或替换现有路径。
- 不得安装、启用、信任、执行或获取所生成插件及其依赖项。
- 不得更新现有插件。应说明更新行为需采用单独的工作流程。
- 不得发布或写入全局配置。
- 当插件 ID、目标路径、请求的能力类型或所需命令不明确时，须先询问确认。
- 遇到不支持的能力类别，应直接拒绝，而非自行构建 schema。

目前仅支持以下能力类别：`skills`、`commands`、`hooks`、`mcpServers` 和 `reminders`。对于 `tools`、`agents`、`outputStyles`、`settings` 和 `apps` 等类别，应予以拒绝。自定义模型工具应通过 `mcpServers` 条目接入，而非直接作为 `tools` 能力提供。

## 可选参考

本技能无需参考即可独立完成。若 `read_skill` 返回了物理存在的 `SKILL.md` 文件，且更多信息有助于理解，可使用 `read_file` 查看以下相关文档：

- `references/native-plugin-contract.md`
- `references/capability-examples.json`

这些文件仅作为只读参考，即使未读取也不会导致任务失败。

## 先澄清问题

在着手处理之前，请先明确以下事项：

1. 插件的可移植 ID 及其人类友好的显示名称；
2. 相对于当前工作区的目标路径；
3. 请求的能力类别及其 ID；
4. 所有必需的构件及命令；
5. 请求是否涉及不支持的能力类别、依赖项的获取，或对现有路径的修改。

如遇信息缺失，请提出具体问题。若请求超出本技能的边界，则应在任何改动前予以拒绝。

## 可移植 ID 与路径

插件及能力的 ID 必须满足以下要求：

- 长度为 1 至 80 个 ASCII 字节；
- 必须以小写 ASCII 字母或数字开头；
- 后续字符仅允许使用小写 ASCII 字母、数字、`.`、`_` 和 `-`；
- 第一个 `.` 之前的不区分大小写的基名不能等于 `CON`、`PRN`、`AUX`、`NUL`、`COM1` 至 `COM9`，或 `LPT1` 至 `LPT9`；
- 不得使用产品保留的插件 ID：`loop` 或 `muse-core`。

构件路径必须是相对于插件根目录的 UTF-8 路径。清单中的路径使用 `/` 分隔。拒绝绝对路径、`..`、清单值中的反斜杠、符号链接逃逸，以及任何规范路径的父目录会超出当前工作区的路径。

## 必需的清单基础

从以下原生清单模板开始，仅填充请求的能力条目：

```json
{
  "schemaVersion": 1,
  "name": "<portable-id>",
  "displayName": "<human name>",
  "version": "0.1.0",
  "description": "<plain description>",
  "compat": {
    "source": "native",
    "manifestDir": ".muse-plugin"
  },
  "capabilities": {
    "skills": [],
    "commands": [],
    "hooks": [],
    "mcpServers": [],
    "reminders": []
  }
}
```

清单文件位于 `.muse-plugin/plugin.json`。除非已安装的验证器允许省略空的能力数组且不产生任何诊断，否则请保留空的能力数组。

针对每项请求的能力，创建清单中引用的所有路径：

- `skills`: `{id, path, enabledDefault?}`，以及包含有效 frontmatter 的 UTF-8 格式的 `SKILL.md` 文件；
- `commands`: `{id, path, enabledDefault?}`，以及一个 UTF-8 编码的 Markdown 模板；
- `hooks`: `{id, event, command, timeoutMs?, statusMessage?}`，以及任意以其 argv 命名的相对命令源；
- `mcpServers`: 一个通过 stdio 传输的 `{id, transport?, command}` 条目，或一个通过 HTTP 传输的 `{id, transport:"http", url}` 条目，以及任意以其 argv 命名的相对命令源；
- `reminders`: `{id, path, tools?, blocking?, decision, defaultPriority?, maxPriority?, maxChildSteps?, maxInstallsPerRun?, reasoningEffort?, context?}`，以及一个 UTF-8 编码的职责文件。使用来自 `references/capability-examples.json` 的可执行决策对象，包括其 V1 版本的 `envelope` 和 `deliveryRole`。

## 预留目标位置

在所有预留步骤成功之前，不得进行任何工件写入操作：

1. 将当前工作区和请求的父目录解析为规范路径。
2. 确保父目录和拟议的目标位置均位于工作区内。
3. 拒绝符号链接形式的目标位置，或其规范身份已超出工作区范围的父目录。
4. 确认目标位置不存在。
5. 通过本会话所公布的托管 Shell 工具，调用一次平台原生的、原子性的“不可覆盖”目录创建操作。不得使用先检查后覆盖的命令。
6. 如果创建操作报告目标位置已被占用或失败，则立即停止，绝不删除或重用该路径。
7. 再次对新目录及其父目录进行规范化处理，确保其身份与预期一致，且仍在工作区内，方可进行首次 `write_file` 调用。

对于 UTF-8 编码的工件，请使用 `write_file` 和 `edit_file` 进行操作。切勿使用 Shell 重定向来绕过文件工具的约束。预留完成后，仅允许对新目录进行修改。

## 验证与修正

首先对每个生成的技能目录进行验证。使用等效的结构化 argv 调用已安装的程序。此即 `muse skills validate` 操作；请保持其参数的结构化形式：

```json
["muse", "skills", "validate", "<skill-directory>", "--json"]
```

随后，以结构化参数运行 `muse plugins validate` 操作：

```json
["muse", "plugins", "validate", "<plugin-directory>", "--json"]
```

只有当进程正常启动并以零退出码结束、标准输出为可解析的 JSON、顶层字段 `valid` 为 `true`、且 `diagnostics` 存在但为空时，验证结果才被视为“干净”。存在警告则视为不干净。如果缺少命令、JSON 格式错误、超时、被拒绝、被取消、退出码非零、`valid:false`，或有任何诊断信息，则该草稿仍为不完整状态。

对于每一层验证，在首次失败后最多进行三轮修正。仅编辑新的草稿，并重新运行同一层验证。第四次失败将导致该层验证终止。只要嵌套的技能未通过验证，就绝不继续进行整个插件的验证。

## 报告

成功时，应报告以下内容：
- 工件的规范路径；
- 能力清单及文件清单；
- 嵌套技能验证的次数及结果；
- 整个插件验证的结果；
- 一份尚未执行的安装提案，以结构化的 `program` 和 `argv` 形式呈现：

```json
{
  "program": "muse",
  "argv": ["plugins", "install", "<canonical-plugin-path>"],
  "executed": false
}
```

明确说明安装尚未执行。

预留后若发生失败或被拒绝，应报告草稿路径、具体失败的操作、剩余的诊断信息，以及尚未完成的检查。不得声称该草稿已创建、已通过验证、已完成或已准备好安装。如果因取消而终止运行，则无需后续报告；应保留所有原有字节，并以运行时的取消决定为准。