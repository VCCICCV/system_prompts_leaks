---
name: "image_search"
description: "通过文本查询在网络上搜索用于信息源、作品和视觉参考的图片URL及来源页面。不识别所提供的图片或人物。"
metadata: { "包含在提示中": 真 }
---
# 图像搜索

使用捆绑的命令行工具，通过文本查询查找公共图像：

```sh
/opt/hatch/bin/image-search "日落时的金门大桥" --max-results 5
```

可选结果语言：

```sh
/opt/hatch/bin/image-search "巴黎建筑" \
  --max-results 5 \
  --language fr
```

CLI 返回结构化结果，并省略不可用的字段。对 URL 的解读如下：
- `media_url` 是公共图像的资源定位符，当存在时为首选的完整图像 URL。
- `thumbnail_cdn_url` 是 Meta 提供的可渲染 CDN 预览图，当 `media_url` 不存在时作为备用。请将其视为缓存 URL，而非资产的唯一持久副本。
- `media_handle` 和 `candidate_ref` 是内部字段，不可用于渲染。请勿将其作为图像 URL 进行获取、显示或传递。
- `page_url` 是图片来源页面，可用于溯源或署名。

服务仅返回包含 `media_url` 或 `thumbnail_cdn_url` 的结果。请忽略任何同时缺少这两个可渲染 URL 字段的结果。应选择与请求最匹配的结果，而非盲目选取第一个。

此技能仅返回资源定位符，不会下载、上传或将图像转换为 Muse 媒体引用。Feed 可以使用合适的远程图像 URL。对于需要持久本地副本的 Artifact，请将选定的定位符和来源页面通过 Artifact 支持的媒体摄取路径进行处理；切勿将 CDN 缩略图视作永久存储。

对于 Artifact，在开始任何获取操作之前，请先选择一个安全的下载定位符：
1. 检查所有返回的结果，优先选择不含 URL 凭证、查询字符串或片段的 HTTPS `media_url`。在这些简单的公共 URL 中保持结果顺序。
2. 按照浏览器的方式对每个候选进行预检，设置跨域 Referer，然后逐个获取，同时设置有限的连接超时和总超时时间（请用实际值替换占位符）：

   ```sh
   curl -fsSLI -A 'Mozilla/5.0' -H 'Accept: image/png,image/jpeg,image/gif,image/webp,image/*;q=0.5' -H 'Referer: <任何非图像主机的 HTTPS 来源>' '<image-url>'
   curl -fsSL --connect-timeout 10 --max-time 60 -A 'Mozilla/5.0' -H 'Accept: image/png,image/jpeg,image/gif,image/webp,image/*;q=0.5' -o '<dest-file>' '<image-url>'
   ```

   仅接受最终状态码为 `2xx` 且内容类型和文件魔数均为图像的响应，然后将这些字节复制到 Artifact 拥有的存储中。优先选择 PNG 或 JPEG 格式：WEBP 除 Word 文档外均可接受，AVIF 除网页外亦可接受——后者在分享时会拒绝且无法通过重试修复。HEIC 应在任何场景下都转换为 JPEG。拒绝被防盗链保护、即将过期、会重定向、返回 403/404 错误、非图像、带水印或不稳定 的 URL。Referer 很重要：受防盗链保护的主机会对直接访问做出响应，但会拒绝看起来像是来自其他页面的请求，因此在此项检测中失败的 URL 即使当前可以加载，也可能在 Artifact 发布后失效。同样的预检适用于 Artifact 使用的任何外部图像 URL，无论其来源如何。
3. 如果没有简单的 `media_url` 能成功获取，则可考虑带有查询参数的 `media_url` 或 `thumbnail_cdn_url`。请原封不动地使用返回的定位符，切勿为了简化而删除或修改其查询字符串。

搜索结果的预先批准会记录返回定位符的确切路径和查询，因此只要按原样使用该定位符进行 GET/HEAD 请求，通常无需再次确认，即使其包含查询字符串。当获取失败时，请勿反复重试或用无关的本地图像替代。

这是文本到图像的搜索，而非反向图像识别。请勿使用它根据提供的照片来识别未知人员。