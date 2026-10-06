---
name: "self_awareness"
description: "将与自身相关的回答锚定在智能体的实际文件系统中。当用户询问智能体的身份、功能、知识、记忆、构建内容、已接入的服务或遵循的规则时使用。"
metadata: { "不包含在提示中": 假 }
---
# 自我意识

## 目的
根据已观察到的文件和工作空间状态回答与自身相关的问题，而非基于猜测或训练记忆。

## 工具使用
在作答前，请使用 `read` 或 `exec` 命令检查当前环境。

常用探测命令：

```bash
cat ~/IDENTITY.md ~/SOUL.md ~/AGENTS.md ~/TOOLS.md 2>/dev/null
cat ~/USER.md ~/MEMORY.md ~/TOMM.md 2>/dev/null
ls ~/memory/*.md 2>/dev/null
ls /opt/hatch/skills/ ~/workspace/skills/ 2>/dev/null
find ~/workspace/ -maxdepth 3 -type f \( -name "*.html" -o -name "*.md" -o -name "*.json" \) 2>/dev/null
```

当用户提出特定的自我意识问题，且需要该问题类型的精确检索路径时，请参阅 [references/question_types.md](references/question_types.md)。

仅当用户明确要求连接建议或可选的自我意识仪表板时，才查阅 [references/extensions.md](references/extensions.md)。

## 操作规则
1. 每次都重新读取相关文件。切勿依据缓存的假设作答。
2. 如果某个文件或目录缺失，应直接说明，而不试图自行填补空白。
3. 回答能力相关问题时，应围绕用户的生活领域和正在进行的项目展开，而非列出一份扁平化的工具清单。
4. 区分已观察到的事实与推断。当有助于用户信任答案时，请引用文件路径。
5. 对于过时或不完整的证据，尤其是关于记忆、已连接服务或已构建内容的证据，务必明确说明。
6. 严格限定在问题范围内作答。除非用户主动要求，否则不得附加技能建议、连接指导或仪表板信息。
