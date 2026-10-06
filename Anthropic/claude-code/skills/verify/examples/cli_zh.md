# 验证 CLI 变更

验证方式是直接调用命令，证据来自标准输出、标准错误和退出码。

## 模式

1. 构建（如果 CLI 需要构建）
2. 使用能触发变更代码的参数运行
3. 捕获输出和退出码
4. 与预期结果进行比对

CLI 通常是最容易验证的——没有生命周期，也不需要端口。

## 示例

**差异：** 在 `status` 子命令中新增了 `--json` 标志。在 `cmd/status.go` 中增加了新的标志解析逻辑，并新增了 JSON 输出分支。

**承诺（提交信息）：** “提供机器可读的状态输出。”

**推断：** 现在可以运行 `tool status --json`，它会输出包含与人类可读输出相同字段的合法 JSON；不带该标志的 `tool status` 则保持不变。

**计划：**
1. 构建
2. 运行 `tool status`，确保输出与之前一致（防止回归）
3. 运行 `tool status --json`，确保输出为合法且可解析的 JSON
4. JSON 字段应与人类可读输出的字段匹配

**执行：**
```bash
go build -o /tmp/tool ./cmd/tool

/tmp/tool status
# -> Status: healthy
# -> Uptime: 3h12m
# -> Connections: 47

/tmp/tool status --json
# -> {"status":"healthy","uptime_seconds":11520,"connections":47}

/tmp/tool status --json | jq -e .status
# -> "healthy"
# (jq -e 在路径为 null 或 false 时会返回非零退出码，用于快速验证合法性)

echo $?
# -> 0
```

**结论：** 通过——标志有效，JSON 合法，字段一致。

## 失败的情况

- 出现 `unknown flag: --json` 错误——说明未正确接入新功能，或使用了旧的构建版本。
- 输出不是合法的 JSON（`jq` 报错）——说明序列化存在缺陷。
- 不带标志的 `tool status` 输出发生变化——说明出现了回归，差异触及了不应修改的部分。
- JSON 的字段名与预期不符——说明承诺与代码实现不一致，可能问题不大，但需记录下来。

## 从标准输入读取数据及破坏性命令

如果 CLI 会读取标准输入，可在测试中通过管道注入测试数据。
如果 CLI 会写文件、访问网络或删除内容，可将其指向一个临时目录、使用模拟服务，或启用试运行模式。如果不存在安全模式，且差异触及了破坏性逻辑，请明确指出，并尽可能验证其周边行为。