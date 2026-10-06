---
name: session-start-hook
description: 为 Claude Code 的 Web 端创建并开发启动钩子。当用户希望在 Web 端为 Claude Code 设置代码仓库时使用，通过创建 SessionStart 钩子，确保其项目在 Web 会话期间能够运行测试和代码检查工具。
---
# Claude Code Web 端的启动钩子技能

创建 SessionStart 钩子，用于在 Claude Code Web 会话中安装依赖，使测试和 linter 正常工作。

## 钩子基础

### 输入（通过标准输入）
```json
{
  "session_id": "abc123",
  "source": "startup|resume|clear|compact",
  "transcript_path": "/path/to/transcript.jsonl",
  "permission_mode": "default",
  "hook_event_name": "SessionStart",
  "cwd": "/workspace/repo"
}
```

### 异步模式
```bash
#!/bin/bash
set -euo pipefail

echo '{"async": true, "asyncTimeout": 300000}'

npm install
```

钩子会在会话启动时在后台运行。使用异步模式可以减少延迟，但也会引入竞态条件：代理循环可能会在启动钩子尚未完成时就依赖于其中正在执行的操作。

### 环境变量

可用的环境变量：
- `$CLAUDE_PROJECT_DIR` - 仓库根目录路径
- `$CLAUDE_ENV_FILE` - 用于写入环境变量的文件路径
- `$CLAUDE_CODE_REMOTE` - 是否在远程环境中运行（即 Claude Code Web）

使用 `$CLAUDE_ENV_FILE` 可以为会话持久化变量：
```bash
echo 'export PYTHONPATH="."' >> "$CLAUDE_ENV_FILE"
```

使用 `$CLAUDE_CODE_REMOTE` 可以仅在远程环境中运行脚本：
```bash
if [ "${CLAUDE_CODE_REMOTE:-}" != "true" ]; then
  exit 0
fi
```

## 工作流程

列出此工作流程中的所有任务，并逐一完成它们。

### 1. 分析依赖

找到依赖清单并进行分析。示例：
- `package.json` / `package-lock.json` → npm
- `pyproject.toml` / `requirements.txt` → pip/Poetry
- `Cargo.toml` → cargo
- `go.mod` → go
- `Gemfile` → bundler

此外，还应阅读相关文档（如 README.md 等），以获取有关环境配置的更多背景信息。

### 2. 设计钩子

编写一个用于安装依赖的脚本。

**关键原则：**
- 第一次迭代时不使用异步模式，只有在用户要求时才切换到异步模式。
- 默认只为 Web 端编写钩子，除非用户另有要求（参见 `$CLAUDE_CODE_REMOTE`）。
- 容器状态会在钩子完成后被缓存，因此优先选择能利用这一特性的依赖安装方式（例如，优先使用 `npm install` 而不是 `npm ci`）。
- 确保脚本是幂等的（可多次安全运行）。
- 不需要交互式操作（无需用户输入）。

### 3. 创建钩子文件

```bash
mkdir -p .claude/hooks
cat > .claude/hooks/session-start.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo '{"async": true, "asyncTimeout": 300000}'
# 在此处安装依赖
EOF

chmod +x .claude/hooks/session-start.sh
```

### 4. 在设置中注册

将以下内容添加到 `.claude/settings.json` 文件中（如果不存在则创建）：
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

如果 `.claude/settings.json` 已存在，则合并钩子配置。

### 5. 验证钩子

直接运行钩子脚本：
```bash
CLAUDE_CODE_REMOTE=true ./.claude/hooks/session-start.sh
```

重要提示：请确认依赖已成功安装，且脚本能够顺利执行。

### 6. 验证 linter

重要提示：确定运行 linter 的正确命令，并对一个示例文件进行测试。无需对整个项目进行 lint 检查。如有问题，请相应更新启动脚本并重新测试。

### 7. 验证测试

重要提示：确定运行测试的正确命令，并针对一个测试用例进行执行。无需运行整个测试套件。如有问题，请相应更新启动脚本并重新测试。

### 8. 提交并推送

提交更改并推送到远程分支。

## 总结

我们已经完成了所有步骤。在给用户的最后一封邮件中，请按照以下格式提供一份详细的总结：

* 已执行更改的摘要
* 验证结果
  1. ✅/‼️ 会话钩子执行（若失败，请附详细信息）
  2. ✅/‼️ 代码检查工具执行（若失败，请附详细信息）
  3. ✅/‼️ 测试执行（若失败，请附详细信息）
* 钩子执行模式：同步
  * 告知用户当前钩子以同步方式运行，并说明其权衡。同时告知他们，如果希望会话启动速度更快，可以将其改为异步模式。
    * 优点：确保在会话启动前所有依赖项均已安装，避免出现 Claude 在测试或代码检查工具尚未就绪时就尝试运行它们的竞态条件。
    * 缺点：远程会话只有在会话启动钩子执行完毕后才会启动。
* 告知用户，一旦将会话启动钩子合并到其仓库的默认分支，所有未来的会话都将使用该钩子。
