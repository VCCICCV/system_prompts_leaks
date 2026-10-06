---
name: claude-api-in-prototypes
description: "通过 window.claude.complete 从你的 HTML 文档中调用 Claude。"
user-invocable: true
---
# 原型中的 Claude API

您的 HTML 文档可以通过内置辅助函数调用 Claude，无需 SDK 或 API 密钥。

```html
<script>
(async () => {
  const text = await window.claude.complete("请概括以下内容：...");
  // 或者使用 messages 数组：
  const text2 = await window.claude.complete({
    messages: [{ role: 'user', content: '...' }],
  });
})();
</script>
```

默认调用 `claude-haiku-4-5` 模型，输出 token 数上限为 1024。请求体还可设置 `model`（仅限 haiku 和 sonnet 系列）、`max_tokens`（最高 32000）、`system`、`tool_choice` 以及客户端 `tools` — 这些均为标准 Messages API 的参数形式，但每个工具还需包含 `run: async (input) => string` 方法；该辅助函数会在页面内执行工具调用并进行循环（最多 8 次模型调用），最终返回处理后的文本。如果处理过程中抛出异常，则会以 `is_error` 形式的工具结果返回。服务器端工具（如网络搜索等）将被拒绝；不支持流式传输；每用户每分钟有 15 次调用的速率限制，循环迭代也计入其中。共享资源将在查看者的配额下运行。
