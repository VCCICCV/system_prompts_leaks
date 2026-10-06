# Facebook 动态

获取您的动态信息（动态托盘），查看您关注的好友和主页的最新动态。

## 命令

### 获取动态信息

```bash
facebook-cli story feed [--limit <n>]
```

**选项：**
- `--limit`（可选）：返回的最大动态分桶数（默认20，最大50）

**示例：**
```bash
# 获取您的动态信息
facebook-cli story feed

# 获取较少数量的动态
facebook-cli story feed --limit 5
```

**响应字段：**

每个动态分桶代表来自一位发布者的动态：
- `owner_name`：动态发布者姓名
- `owner_id`：动态发布者的个人主页ID
- `seen`：该分桶是否已被查看
- `cards`：包含多个单条动态卡片的数组，每张卡片包含：
  - `story_id`：动态的唯一ID
  - `creation_time`：动态发布时间（Unix时间戳）
  - `expiry_time`：动态失效时间（Unix时间戳）
  - `message`：动态文本内容（如有）
  - `story_url`：动态的直接链接
