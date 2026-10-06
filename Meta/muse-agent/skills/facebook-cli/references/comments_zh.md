# Facebook 评论

读取 Facebook 帖子下的评论。

## 命令

```bash
facebook-cli post comments read --post-id <帖子ID> [--limit <n>] [--after <游标>]
```

**选项：**
- `--post-id`（必填）：要读取评论的帖子 ID
- `--limit`（可选）：每页最多显示的评论数（最大 20 条；超过此值时，服务器端会自动截断）
- `--after`（可选）：分页游标——将上一次响应中的 `paging.cursors.after` 值传入，以获取下一页内容

## 响应字段

响应采用游标分页：
- `data`：评论对象数组，每个对象包含：
  - `id`：评论 ID
  - `author_name`：评论作者姓名
  - `text`：评论文本内容
  - `created_time`：评论创建时间的 Unix 时间戳
  - `reply_count`：该评论的回复数
  - `comment_url`：指向该特定评论的 Facebook 直接链接。当用户希望查看或分享某条评论时，请使用此链接。
- `summary.post_url`：原始帖子在 Facebook 上的永久链接。始终存在——可用于引导用户返回原帖。（在该接口改为分页后，原先位于顶层的 `post_url` 已移至 `summary` 中。）
- `paging.cursors.after`：用于下一页的不透明游标。**仅在还有更多评论时才会出现**——若不存在，则表示已到达末尾。通过 `--after` 参数将其传回，即可继续获取后续内容。

向用户展示评论时，务必包含 `summary.post_url`，以便用户能跳转到原始帖子，并说明每条评论都有一个直接的 `comment_url` 链接。如需获取多页评论，请根据 `paging.cursors.after` 的值不断使用 `--after` 参数，直到该字段不再出现为止。

## 操作规范

1. 展示评论时，务必同时提供父帖的 `summary.post_url` 和每条评论的 `comment_url`。切勿直接显示评论 ID 或帖子 ID，一律使用对应的 URL。
2. 当用户询问“最近”或“最新”的评论时，务必在回复中说明所使用的日期范围（例如：“以下是过去 7 天内的评论”）。
3. 在跨多个帖子整理评论时，应按帖子分组展示，并做好清晰的区分。
4. 展示评论时，应一并提供时间戳（`created_time`）。
5. 在不同帖子间交叉比对评论者时，仅准确识别同时出现在多个讨论串中的用户，切勿虚构评论者姓名。