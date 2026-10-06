# 原生 Muse 插件契约

本参考文档总结了当前 Muse 验证器所接受的清单格式。关于预留、变更、验证顺序、修正限制及报告，插件创建者的说明仍具有权威性。

## 包布局

原生插件是当前工作区中的一个新目录，其清单文件为：

```text
.muse-plugin/plugin.json
```

所有以能力名称命名的文件均相对于插件根目录。一个最小化的清单示例如下：

```json
{
  "schemaVersion": 1,
  "name": "example-plugin",
  "displayName": "示例插件",
  "version": "0.1.0",
  "description": "一句简明的说明。",
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

仅使用请求中必需的字段。对于可选字段，是否允许省略由已安装的验证器决定。

## 标识符

插件和能力的 ID 使用以下可移植的语法：

```text
^[a-z0-9][a-z0-9._-]{0,79}$
```

第一个点之前的基名不得在大小写折叠后等于 `CON`、`PRN`、`AUX`、`NUL`、`COM1` 至 `COM9`，或 `LPT1` 至 `LPT9`。插件 ID `loop` 和 `muse-core` 由产品包保留。

## 路径

- 清单中应使用相对 UTF-8 路径，并以 `/` 作为分隔符。
- 拒绝绝对路径、父目录遍历、反斜杠以及空路径。
- 在验证之前，必须创建所有被引用的文件。
- 对包含关系进行规范化；符号链接不得逃出插件或工作区的根目录。
- 生成的组件名称应在 macOS、Linux 和 Windows 之间保持可移植性。

## 能力字段

### 技能

```json
{"id":"review","path":"skills/review/SKILL.md","enabledDefault":false}
```

目标文件为带有有效 Frontmatter 的 UTF-8 格式 `SKILL.md`。`enabledDefault` 为可选项；仅当请求的激活行为明确时才可省略。

### 命令

```json
{"id":"summarize","path":"commands/summarize.md","enabledDefault":true}
```

目标文件为 UTF-8 格式的 Markdown 命令模板。

### 钩子

```json
{
  "id":"pre-check",
  "event":"PreToolUse",
  "command":["sh","hooks/pre-check.sh"],
  "timeoutMs":1000,
  "statusMessage":"正在检查插件策略"
}
```

命令采用结构化 argv 格式，而非 shell 字符串。若 argv 中的某个元素指定了相对源路径，则该常规文件必须存在于插件根目录之下。两个钩子 ID 不得共享同一源路径。

### MCP 服务器

对于本地 stdio 服务器：

```json
{
  "id":"workspace-index",
  "transport":"stdio",
  "command":["python3","mcp/server.py"]
}
```

`transport` 的默认值为 `stdio`。若使用 HTTP 传输，则必须指定一个非空的 `url`；请勿自行构造端点。自定义模型工具应通过 MCP 服务器暴露，而非直接通过 `tools` 能力提供。

### 提醒

```json
{
  "id":"review-policy",
  "path":"reminders/review-policy.md",
  "tools":["read_file"],
  "blocking":false,
  "decision":{
    "version":1,
    "envelope":{"version":1,"template":"<system-reminder>\n{text}\n</system-reminder>"},
    "deliveryRole":"developer",
    "...":"从 capability-examples.json 中复制其余可执行字段"
  }
}
```

职责文件必须存在且为 UTF-8 编码。`decision` 为必填项，包括其 V1 版本的 `envelope` 和 `deliveryRole` 权限。可选的策略字段包括 `defaultPriority`、`maxPriority`、`maxChildSteps`、`maxInstallsPerRun`、`reasoningEffort` 和 `context`。仅在收到请求且核对过可执行示例后方可添加这些字段。

## 不支持的类别

验证器会拒绝直接使用 `tools`、`agents`、`outputStyles`、`settings` 和 `apps` 这些能力键。请勿将其转换为推测的字段。如果用户意图符合，可要求其将自定义工具重新表述为 MCP 服务器。

## 验证流程

首先运行每个生成的技能：

```json
["muse","skills","validate","<skill-directory>","--json"]
```

然后运行插件验证器：

```json
["muse","plugins","validate","<plugin-directory>","--json"]
```

只有当进程启动、以零退出、返回可解析的 JSON、将顶级字段 `valid` 设置为 `true`，并且返回一个空的 `diagnostics` 数组时，结果才算有效。对于创建操作而言，警告即视为失败。每次仅修复一层验证，且在首次验证失败后，最多进行三轮修正。

## 创建边界

在写入工件之前，先使用原生的原子性“不可替换”目录创建操作预留一个不存在的目标路径。预留完成后，立即重新检查工作区的规范包含关系。若发现目标已被占用、权限被拒绝、操作被取消，或任何包含关系检查失败，则应立即停止。切勿安装、启用、信任、执行、获取、更新或发布该草稿。