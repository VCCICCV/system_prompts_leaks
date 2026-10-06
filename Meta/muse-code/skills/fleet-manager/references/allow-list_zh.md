# 允许列表模板

按子命令拆分辅助工具，绝不允许直接调用辅助工具：`read` 和 `steer` 动词无需提示即可运行；而所有涉及启动、中断、结束、采纳、附加、连接或遗忘的操作都必须经过权限提示。防护机制较为宽松，因此这种拆分是安全措施。请将 `<fleet>` 替换为该技能提供的绝对路径。

## 无需提示的允许（读取）

```text
python3 <fleet> doctor *
python3 <fleet> detect *
python3 <fleet> context *
python3 <fleet> list *
python3 <fleet> machines
python3 <fleet> status *
python3 <fleet> read *
python3 <fleet> dialog *
python3 <fleet> resources *
python3 <fleet> fetch *
python3 <fleet> wait *
python3 <fleet> events *
```

## 无需提示的允许（操控）——可选，视需求而定

```text
python3 <fleet> approve *
python3 <fleet> deny *
python3 <fleet> send * --keys *
```

禁止使用文本形式的 `send *` 通配符：`send * …` 无法区分通知形式与 `--type` 参数（标志可能跟在文本之后），且允许匹配会优先生效，因此任何携带文本的 `send` 操作都必须经过提示。`--automated` 标记的是输入内容，但并不授予任何权限。

## 始终提示（受保护）

```text
python3 <fleet> open *
python3 <fleet> stop *
python3 <fleet> close *
python3 <fleet> adopt *
python3 <fleet> attach *
python3 <fleet> connect *
python3 <fleet> forget *
python3 <fleet> send * --type
```

## Claude Code 设置模板

Claude Code 通过命令前缀匹配 `Bash(...)` 规则（`:*` 表示“及其后的任意内容”），因此每条规则都需指定辅助工具的路径及一个动词。请将 `<fleet>` 替换为绝对路径，并将以下内容放入 `.claude/settings.json`（项目级）或 `~/.claude/settings.json`（用户级）：

```json
{
  "permissions": {
    "allow": [
      "Bash(python3 <fleet> doctor:*)",
      "Bash(python3 <fleet> detect:*)",
      "Bash(python3 <fleet> context:*)",
      "Bash(python3 <fleet> list:*)",
      "Bash(python3 <fleet> machines)",
      "Bash(python3 <fleet> status:*)",
      "Bash(python3 <fleet> read:*)",
      "Bash(python3 <fleet> dialog:*)",
      "Bash(python3 <fleet> resources:*)",
      "Bash(python3 <fleet> fetch:*)",
      "Bash(python3 <fleet> wait:*)",
      "Bash(python3 <fleet> events:*)"
    ],
    "ask": [
      "Bash(python3 <fleet> open:*)",
      "Bash(python3 <fleet> send:*)",
      "Bash(python3 <fleet> stop:*)",
      "Bash(python3 <fleet> close:*)",
      "Bash(python3 <fleet> adopt:*)",
      "Bash(python3 <fleet> attach:*)",
      "Bash(python3 <fleet> connect:*)",
      "Bash(python3 <fleet> forget:*)"
    ]
  }
}
```

Muse 并未提供设置级别的 `Bash(...)` 允许列表：其设置文件中的 `permissions` 成员是一个权限配置文件，而非命令规则列表，仅包含上述内容的设置文件将无法加载。在 Muse 中，上述拆分是在权限提示时应用的。