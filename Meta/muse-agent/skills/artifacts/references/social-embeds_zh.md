---
description: 在网页中嵌入社交媒体帖子、短视频（Instagram）以及其他第三方富媒体内容。明确支持内嵌的平台、具体的嵌入代码，以及在禁止使用 iframe 的平台上如何优雅降级处理外链跳转。
builders: web
---
# 社交帖子、短视频及富媒体嵌入

当页面应展示社交帖子、短视频或其他第三方富媒体内容（如视频、播放器、交互式组件），而非仅提供链接时，请参阅本说明。

## 哪些内容可以被嵌入，哪些不可以

- **Instagram 帖子、短视频及 TV 内容**：在永久链接路径后添加 `/embed/`，即 `https://www.instagram.com/{p|reel|tv}/{id}/embed/`。这是官方认可的 URL 构造方式，可在未登录状态下正常渲染并播放。
- **其他平台**：仅允许嵌入该平台专门提供的官方嵌入接口（例如 YouTube 的 `/embed/{id}` 是常见情况）。上线前务必确认未登录用户也能正常渲染嵌入内容；若平台禁止嵌入或嵌入内容需登录访问，则应使用外链卡片，切勿通过抓取播放器或猜测 URL 来规避限制。
- **用户自己保存或点赞的 Instagram 帖子**：其永久链接由 Instagram 技能生成，而非直接从用户个人主页获取。请参阅 `/opt/hatch/skills/instagram/SKILL.md`：`instagram-cli saved-posts` 会返回每条帖子的 `url`，即上述构造方式所基于的永久链接。

## 标记结构

卡片以可见的永久链接锚点为底层，上层叠加嵌入 iframe：

```html
<div class="embed-card"><!-- 固定高度容器；短视频以 9:16 比例显示效果最佳 -->
  <iframe src="https://www.instagram.com/reel/{id}/embed/"
          sandbox="allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox"
          referrerpolicy="no-referrer" loading="lazy" scrolling="no"></iframe>
  <a href="{permalink}" target="_blank" rel="noopener">在 Instagram 上查看</a>
</div>
```

- 必须使用上述 `sandbox` 属性集：这是标准的第三方嵌入配置（框架无法访问宿主页面，但自身链接仍可正常打开）。
- 为卡片设置固定高度，并添加中性占位背景，以防止 iframe 加载时布局发生偏移，同时让 iframe 绝对定位填满整个容器。
- 始终使用 `loading="lazy"`；当一页包含多个嵌入时，仅在 iframe 即将进入视口时再加载——嵌入文档通常较重。
- 确保永久链接锚点始终位于 iframe 下方或旁边，既是内容来源的标识，也是在不支持 iframe 的环境中显示的内容。

## 渲染场景

发布后的作品页及其分享链接允许第三方 iframe，因此嵌入内容可正常播放。而在聊天预览及部分宿主框架中，所有 iframe 均被屏蔽，此时卡片将退化为仅保留锚点的形式。设计时应考虑到这一点：嵌入是基于外链卡片的增强功能，卡片本身应具备完整性，且不应成为该位置的唯一内容。

## 媒体资源

- 切勿在 `<img>` 或 `<video>` 中直接引用平台 CDN 提供的媒体资源（如 `cdninstagram.com`、`fbcdn.net` 等）：这些 URL 带有签名且会过期，审核时会被拒绝，上线后也可能返回 403 错误。若卡片在 iframe 加载前需要视觉占位，请使用中性占位图，而非远程获取的缩略图。
- 不要直接提取或播放平台的 MP4 文件，因为不存在稳定的公开 URL。正确的播放途径是通过嵌入 iframe 或永久链接。