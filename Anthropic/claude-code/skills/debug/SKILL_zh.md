---
name: debug
description: 为当前会话启用调试日志记录，以帮助诊断问题。
disable-model-invocation: true
---
# 调试技能

帮助用户调试他们在当前 Claude Code 会话中遇到的问题。

## 调试日志已启用

在本次会话中，调试日志此前处于关闭状态。在此 /debug 命令执行之前的所有内容均未被记录。

告知用户，调试日志现已启用，路径为 `~/.claude/debug/{{SESSION_ID}}.txt`；请用户重现问题，然后重新查看日志。如果无法重现问题，也可通过 `claude --debug` 重启程序，以捕获从启动时开始的日志。

## 会话调试日志

当前会话的调试日志位于：`~/.claude/debug/{{SESSION_ID}}.txt`

目前尚不存在该日志文件。

如需更多上下文信息，请在整个日志文件中搜索包含 [ERROR] 和 [WARN] 的行。

## 守护进程

未找到守护进程锁文件或状态文件——后台守护进程似乎未在运行。若问题与后台会话或 `claude agents` 相关，则守护进程日志（如有）位于 `~/.claude/daemon.log`。

## 问题描述

用户未提供具体问题描述。请阅读调试日志，并汇总其中的错误、警告或其他值得注意的问题。

## 设置

请注意，设置文件的位置如下：
* 用户级：`~/.claude/settings.json`
* 项目级：`/private/tmp/skillcap/.claude/settings.json`
* 本地级：`/private/tmp/skillcap/.claude/settings.local.json`

## 操作指南

1. 审阅用户的問題描述。
2. 日志文件格式的最后 20 行已展示，请在整份日志中查找 [ERROR] 和 [WARN] 条目、堆栈跟踪以及失败模式。
3. 可考虑启动 claude-code-guide 子代理，以更好地理解相关的 Claude Code 功能。
4. 用通俗易懂的语言说明您的发现。
5. 提出具体的修复建议或后续步骤。