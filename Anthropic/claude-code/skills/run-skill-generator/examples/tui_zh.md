# 示例：TUI / 交互式终端应用

交互式终端应用（文本编辑器、REPL、基于 curses 的 UI）无法直接由代理的 bash 工具驱动——它们会接管整个终端。技能必须展示如何将这些应用封装在 `tmux` 中，以便代理能够发送输入、捕获输出并截取屏幕截图。

## tmux 模式

这是标准做法：

1. 在一个分离的 tmux 会话中启动 TUI。
2. 使用 `tmux send-keys` 发送按键。
3. 使用 `tmux capture-pane` 读取屏幕内容。
4. 最后用 `tmux kill-session` 进行清理。

技能的 `SKILL.md` 应将此方法作为驱动应用的主要方式。可以在技能目录中放置一个简短的 `driver.sh` 脚本，用于封装启动和附加的流程；但对于大多数 TUI 来说，在技能主体中直接使用原生的 tmux 命令就已足够。

## 示例片段

> ## 运行（交互式，适用于代理）
>
> 在 tmux 中启动 TUI：
>
> ```bash
> tmux new-session -d -s app -x 120 -y 40 './myapp'
> ```
>
> 轮询直到出现“就绪”标记（比固定睡眠更快且更可靠——应用一启动即返回，若未启动则立即报错）：
>
> ```bash
> timeout 10 bash -c 'until tmux capture-pane -t app -p | grep -q "Ready"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 发送输入（本例导航到设置界面并切换某个选项）：
>
> ```bash
> tmux send-keys -t app 's'
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -q "Settings"; do sleep 0.2; done'
> tmux send-keys -t app 'Down' 'Down' 'Space'  # 导航并切换
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -qF "[x]"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 如果发现需要编写多条这样的轮询语句，可将其提取到技能旁边的 `driver.sh` 脚本中，定义为一个 `wait_for()` 辅助函数。
>
> 退出：
>
> ```bash
> tmux send-keys -t app 'q'
> tmux kill-session -t app 2>/dev/null || true
> ```
>
> ### 键位参考
>
> | 键 | 动作 |
> |---|---|
> | `j` / `k` 或 `Down` / `Up` | 在列表中导航 |
> | `Enter` | 选择 |
> | `s` | 设置 |
> | `q` | 退出 |

## 值得记录的细节

- **终端尺寸。** 一些 TUI 在宽度较小时会出现显示异常或部分内容被遮挡。请在 `tmux new-session -x -y` 参数中指定一个确定可用的尺寸。
- **启动时间。** 使用轮询“就绪”标记（`until tmux capture-pane | grep -q X`）代替固定的 `sleep N`——这样应用一启动就会立即返回，并且在应用始终未启动时也能给出明确的失败提示。请说明哪个字符串表示“就绪”状态。
- **键位参考。** 列出主要的按键及其功能。这是 TUI 的“API”，代理需要它来操作应用。
- **干净退出。** 同时提供退出快捷键以及作为后备的 `tmux kill-session` 命令。
- **颜色与 Unicode 的特殊处理。** 如果 `capture-pane` 的输出难以阅读，请注明有助于改善显示的选项（如 `-e` 处理转义序列，`-J` 将换行合并为单行）。

## 也需记录直接调用方式

对于以交互方式运行应用的人类用户来说，使用 tmux 显得过于复杂。因此也应提供一条简洁的直接运行命令：

> ## 运行（直接，适用于人类）
>
> ```bash
> ./myapp
> ```
>
> 按下 `q` 键退出。