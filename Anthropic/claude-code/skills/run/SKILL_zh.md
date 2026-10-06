---
name: run
description: 启动并运行该项目的应用程序，以验证更改已生效。当需要运行、启动或截屏应用程序，或确认更改在实际应用中有效（而不仅仅是通过测试）时，请使用此功能。首先会查找项目中是否已有相应的技能来负责启动应用程序；如果不存在，则会根据项目类型（CLI、服务器端、TUI、Electron、浏览器驱动型、库）回退到内置的处理模式。
---
**运行意味着启动实际的应用程序并与之交互**——而不是运行测试套件，也不是仅仅导入一个内部函数并打印一条日志。这里的“应用”是指用户（无论是人类还是程序）会与之交互的方式：命令行工具通过命令行调用，服务器通过套接字连接，图形界面则通过窗口操作。

## 首先：项目技能是否已覆盖此场景？

如果某个项目技能能够启动该应用，那就是最可靠的路径——其作者已经从 Linux 容器中完成了冷启动，并提交了有效的配置：精确的 `apt-get` 命令、环境变量、补丁以及驱动程序。请直接使用它，无需重新摸索。

```bash
d=$PWD; while :; do
  grep -Hm1 '^description:' "$d"/.claude/skills/*/SKILL.md 2>/dev/null
  [ -e "$d/.git" ] || [ "$d" = / ] && break
  d=$(dirname "$d")
done
```

- **有一项描述了如何启动/驱动该应用** → 请仔细阅读对应的 `SKILL.md` 文件，并完全按照其中的说明执行。不要自行转述，也不要跳过任何补丁。
- **大型仓库，有多个可能的匹配，但没有明确对应项** → 请询问用户应运行哪个单元。
- **过时（因与您的任务无关的机制问题而失败）** → 通知用户，并建议通过 `/run-skill-generator` 进行更新。
- **完全没有关于运行的说明** → 回退到下文的模式。

## 否则：匹配形态，采用相应模式

选择与您的项目最接近的方案。每个示例都详细展示了启动及首次交互的过程；请忽略任何后续的“编写技能”部分——您是在使用现成的配方，而非自己创作。

| 项目类型 | 处理方式 | 示例 |
|---|---|---|
| 命令行工具 | 直接调用、检查退出码、处理标准输入/输出 | [examples/cli.md](examples/cli.md) |
| Web 服务器 / API | 后台启动 + 使用 `curl` 进行简单测试 | [examples/server.md](examples/server.md) |
| TUI / 交互式终端 | 使用 `tmux send-keys` 和 `capture-pane` | [examples/tui.md](examples/tui.md) |
| Electron / 桌面 GUI | 在 xvfb 下使用 Playwright 的 `_electron` REPL | [examples/electron.md](examples/electron.md) |
| 浏览器驱动 | 开发服务器 + `chromium-cli` 脚本 | [examples/playwright.md](examples/playwright.md) |
| 库 / SDK | 在包边界处编写导入并调用的简单脚本 | [examples/library.md](examples/library.md) |

如果没有完全匹配的方案，请从最接近的模板入手并加以调整。例如，对于 Web 应用，可以参考 [examples/playwright.md](examples/playwright.md)，使用 `chromium-cli` 来驱动，无需自定义驱动程序。对于桌面应用，则可参考 [examples/electron.md](examples/electron.md)，其中提供了 `_electron` REPL 驱动的框架和 `tmux` 包装。

## 不仅要启动，还要驱动它

仅启动而不进行任何交互只能证明入口点能够解析。这并不等同于真正运行应用——更像是多了一步的类型检查。必须驱动应用，直到用户能看到某些结果：

- 命令行工具 → 输入一条具有代表性的命令，检查退出码和输出。
- 服务器 → 使用 `curl` 访问本次改动涉及的接口，查看响应体。
- TUI → 发送导航指令，截取显示结果。
- GUI → 点击按钮，对窗口截图。**务必查看截图。** 如果截图是空白，即为启动失败。

如果回退模式无法开箱即用——您不得不安装软件包、设置环境变量、修改配置或编写驱动程序——请在报告中推荐使用 `/run-skill-generator`，以便将这些工作记录为项目技能。如果一切顺利，则无需这样做。