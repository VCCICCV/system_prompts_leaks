# Web 工件操作

此设备上每次调用的 Web 工件操作的 PostgreSQL 日志，无论该调用是由用户界面点击、代理工具调用还是 Cron 定时任务触发。每次被接受的调用都会生成一行记录，并在工作进程成功、失败或被取消时更新该行。

## 适用场景

Web 工件的数据库保存了当前状态和操作调用的时间线。当用户询问近期的 Web 工件活动或操作失败情况时，可使用此日志来解答历史查询、审计查询和诊断问题。

## 查看方法

可通过守护进程沙盒 API 以只读方式查看此日志（无需直接访问数据库）。可按 `space_slug`、`invocation_id` 或 `source_kind` 进行过滤：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/spaces-actions?space_slug=<space_slug>&limit=20"
```

结果按调用时间从近到远排序。响应格式为 `{"ok":true,"result":{"invocations":[...]}}`。每个调用包含以下字段：

- `invocation_id`：稳定的 ID，用于关联 HTTP 请求、运行时遥测数据、工作进程日志以及调试界面中的相关条目。
- `space_slug`：该调用所属的 Web 工件标识符（可用于跨工件查询，通过 `invocation_id` 或 `source_kind`）。
- `action`：操作名称。
- `status`：调用状态，可能为 `in_flight`（进行中）、`succeeded`（成功）、`failed`（失败）或 `cancelled`（已取消）。
- `invoked_at_ms`：调用时间，以 Unix 毫秒表示。
- `settled_at_ms`：结算时间，以 Unix 毫秒表示（调用进行中时该字段为空）。
- `source_kind`、`source_ref`、`trigger_ref`：调用来源信息。
- `args_preview`：请求参数的受限 JSON 快照。
- `result_preview`：单值成功结果的受限 JSON 快照。
- `error`：失败操作的工作进程错误信息截断版本。

## 查询示例

按 ID 查询单个调用（跨所有 Web 工件）：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/spaces-actions?invocation_id=<invocation_id>"
```