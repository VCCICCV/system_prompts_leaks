# Facebook 动态

浏览您已排序的 Facebook 动态——包括算法推荐动态和仅好友动态。

## 命令

### 算法动态

获取已排序的算法推荐动态（仅有机内容，不含广告）。

```bash
facebook-cli feed newsfeed [--limit <n>] [--after <cursor>]
```

| 标志 | 必需 | 描述 |
|------|------|------|
| `--limit` | 否 | 最多返回的动态数量（默认10条，最大10条） |
| `--after` | 否 | 上一次响应中 `next_cursor` 提供的分页游标（`--cursor` 为向后兼容的别名） |

**示例：**  
```bash
# 获取您的算法动态
facebook-cli feed newsfeed

# 获取最多10条动态（即最大值）
facebook-cli feed newsfeed --limit 10

# 获取下一页
facebook-cli feed newsfeed --limit 10 --after <上一次的after_cursor>
```

### 友好动态

获取已排序的好友动态（仅显示好友发布的动态）。

```bash
facebook-cli feed friends [--limit <n>] [--after <cursor>]
```

| 标志 | 必需 | 描述 |
|------|------|------|
| `--limit` | 否 | 最多返回的动态数量（默认10条，最大10条） |
| `--after` | 否 | 上一次响应中 `next_cursor` 提供的分页游标（`--cursor` 为向后兼容的别名） |

**示例：**  
```bash
# 获取好友动态
facebook-cli feed friends

# 获取5条好友动态
facebook-cli feed friends --limit 5
```

## 响应字段

两个命令均返回与社交搜索共享的精简版 `social_posts_v1` 集合：

- `posts`：已排序的动态列表，包含 `post_id`、`url`、`platform`、`created_at`、`username`、`post_caption`、`header_text`，以及可用的媒体和互动字段。`post_caption` 优先显示作者自述文字，其次为媒体摘要，最后是平台生成的动态标题（也以 `header_text` 字段输出）。请将媒体摘要或动态标题视为描述性内容，切勿将其当作作者的原话。
- `next_cursor`：下一页的游标（可在后续请求中作为 `--after` 参数传入）。
- `has_next_page`：是否还有下一页。

请直接读取 `posts[]` 数组，切勿自行猜测接口返回路径，也不要用 `jq` 进行解析。如果接口模式不匹配，将直接报错，而不会返回空集合。

## 使用规则

1. 使用 `feed newsfeed` 获取标准的算法推荐动态；使用 `feed friends` 获取仅好友动态。
2. 始终检查 `next_cursor`——如果存在，则表示还有更多页面，应提供获取下一页的选项。
3. 动态会生成 `post_id`，可用于后续的 `post comments read` 和 `post reactions read` 操作。
4. 请按返回顺序展示动态（这些动态已由机器学习模型按相关性排序）。
5. 不得对动态内容进行编辑或添加主观评论。
6. 展示动态时，请务必包含 `url`，以便用户在 Facebook 上查看。