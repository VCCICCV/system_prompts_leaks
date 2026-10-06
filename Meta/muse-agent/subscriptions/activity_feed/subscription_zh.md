# 活动动态

活动动态是侧边栏中显示的已完成与进行中的工作日志。
每一条记录都对应用户在其活动侧边栏上可见的一张卡片：
包括已发送的邮件、已创建的文件、网络搜索、提醒、目标以及其他值得关注的操作。
这并非“动态”标签页（即已发布动态单元的个人资讯流）；那些动态单元仅由 `feed` 和 `feed.units` 工具提供，而不会通过此日志呈现。

## 适用场景

当您需要了解用户侧边栏上已显示的内容、用户提及了近期活动，或希望避免重复叙述侧边栏上已可见的工作时，请使用此日志。

## 查看方式

可通过守护进程沙盒 API 以只读方式查看活动动态（无需访问数据库）：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/activity/recent?limit=10"
```

查询参数：

- `is_goal` — 设置为 `true` 时表示目标/活动线程条目，设置为 `false` 时表示普通活动条目；省略则返回全部。
- `activity_type` — 按类型筛选：`email_sent`（已发送邮件）、`message_sent`（已发送消息）、`file_created`（已创建文件）、`file_updated`（已更新文件）、`reminder_set`（已设置提醒）、`web_search`（网络搜索）、`task_running`（正在运行的任务）、`goal`（目标）。
- `limit` — 最大返回条数，按完成或创建时间从近到远排序（默认 10 条，最大 100 条）。

响应格式为 `{"ok":true,"result":{"entries":[...]}}`。每条记录包含：

- `timestamp` — RFC3339 格式的完成或创建时间。
- `activity_type` — 活动类型（详见上述取值）。
- `status` — 状态，可为 `success`（成功）、`error`（失败）、`pending`（待处理）、`blocked`（被阻塞）或 `stopped`（已停止）。
- `title` — 短小的用户端标题，显示在侧边栏卡片上。
- `status_title` — 卡片上的简洁状态标签。
- `subtitle` — 较长的上下文说明，显示在标题下方。
- `task_label` — 可选的任务标签，用于分组显示的活动。
- `message_id` — 当活动源自聊天轮次时，关联的消息 ID。

## 查询示例

最近的非目标侧边栏活动：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/activity/recent?is_goal=false&limit=10"
```

最近的目标：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/activity/recent?is_goal=true&limit=10"
```

按类型筛选的活动：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/activity/recent?activity_type=email_sent&limit=10"
```