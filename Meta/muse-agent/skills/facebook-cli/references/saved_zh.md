# 已保存

阅读并管理 Facebook 的已保存内容和收藏集。

## 命令

### `saved list`

```
facebook-cli saved list [--type <类别>] [--collection-id <ID>] [--limit N] [--after <不透明值>]
```

列出已保存的内容，按最新到最旧的顺序排列。

- `--type`（可选）：按类别筛选。可选值为 `post`、`video`、`link`、`product`、`reel`、`event`、`page`。
- `--collection-id`（可选）：仅列出特定收藏集中的内容。
- `--limit`（可选）：每页最多显示的条目数（最大 20 条）。
- `--after`（可选）：从前一次响应的 `paging.cursors.after` 中获取的下一页不透明游标（`--cursor` 作为向后兼容的别名也可接受）。

**响应格式**：

```json
{
  "data": [
    {
      "id": "<保存关系ID>",
      "savable_id": "<内容ID>",
      "type": "post",
      "saved_time": 1719500000,
      "title": "...",
      "description": "...",
      "permalink": "https://www.facebook.com/..."
    }
  ],
  "paging": { "cursors": { "before": "...", "after": "..." } }
}
```

要获取下一页，请将 `paging.cursors.after` 作为 `--after` 参数传入。

### `saved add`

```
facebook-cli saved add --savable-id <FBID> [--type <类别>] [--collection-id <ID>]
```

保存一条内容。在用户明确请求的情况下，无需额外确认即可执行。

- `--savable-id`（必填）：内容的 Facebook ID（帖子、页面、商品等），以 `id` 字段传递。
- `--type`（可选）：服务端默认为 `post`。可选值为 `post`、`video`、`link`、`product`、`reel`、`event`、`page`。
- `--collection-id`（可选）：将内容添加到指定收藏集。省略则保存到默认的“全部已保存”分类中。

**响应**：`{ "id": "<savable-id>", "saved": true }`

### `saved remove`

```
facebook-cli saved remove --savable-id <FBID> [--type <类别>] [--collection-id <ID>]
```

移除之前保存的内容。在用户明确请求的情况下，无需额外确认即可执行。

- `--savable-id`（必填）：已保存内容的 Facebook ID（帖子、页面、商品等），即 `saved list` 中的 `savable_id`，以 `id` 字段传递。
- `--type`（可选）：服务端默认为 `post`。请指定内容的类别以避免歧义。
- `--collection-id`（可选）：仅从该收藏集中移除内容。省略则完全取消保存。

此命令仅移除保存关系，并不会删除底层的帖子、视频或页面。

**响应**：`{ "id": "<savable-id>", "unsaved": true }`

### `saved collections list`

```
facebook-cli saved collections list
```

列出所有收藏集。

**响应格式**：

```json
{
  "data": [
    { "id": "<收藏集ID>", "name": "...", "item_count": 5 }
  ]
}
```

### `saved collections create`

```
facebook-cli saved collections create --name "My Collection"
```

创建一个空的收藏集。在用户明确请求的情况下，无需额外确认即可执行。

**响应**：`{ "collection_id": "<ID>", "name": "<名称>" }`

## 操作说明

- 分页：`saved list` 返回 `paging.cursors.after`；将其作为 `--after` 参数传入即可获取下一页。请勿在未经用户指示的情况下自动分页。
- 已保存内容的 ID——使用 `savable_id` 进行取消保存及链式调用：`saved list` 中的每条内容都有两个 ID。“id”是保存关系的记录；“savable_id”是对应的内容本身。`saved remove` 是通过内容来识别目标，因此在调用 `saved remove --savable-id <savable_id>` 时应传入 `savable_id`（可选加 `--type <类型>` 以避免歧义，默认为 POST）。省略 `--collection-id` 则会完全取消保存；指定 `--collection-id` 则仅从该收藏集中移除。`saved remove` 不会删除帖子、视频或页面本身。在与其他命令（如 `post comments read`、`timeline fetch`）进行链式调用时，也请使用 `savable_id`。
- 默认情况下，对已保存内容的写操作是自动允许的。在修改之前，请先明确具体的内容或收藏集。