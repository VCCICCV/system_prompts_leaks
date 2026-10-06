# Claude API - PHP

> **注意：** PHP SDK 是 Anthropic 官方提供的 PHP SDK。可通过 `$client->beta->messages->toolRunner()` 使用 Beta 版工具运行器。结构化输出助手通过 `StructuredOutputModel` 类提供支持。代理 SDK 尚不可用。支持 Bedrock、Vertex AI 和 Foundry 客户端。

## 安装

```bash
composer require "anthropic-ai/sdk"
```

## 客户端初始化

```php
use Anthropic\Client;

// 从环境变量中读取 API 密钥
$client = new Client(apiKey: getenv("ANTHROPIC_API_KEY"));
```

### Amazon Bedrock

```php
use Anthropic\Bedrock\MantleClient;

// 使用 Messages-API 的 Bedrock 端点。从环境变量中读取 AWS 凭证。
$client = new MantleClient(awsRegion: 'us-east-1');
```

Bedrock 上的模型 ID 需要加上 `anthropic.` 前缀，例如：`model: 'anthropic.claude-opus-5-5'`。

### Google Vertex AI

```php
use Anthropic\Vertex;

// 构造函数为私有。参数是 `location`，而非 `region`。
$client = Vertex\Client::fromEnvironment(
    location: 'us-east5',
    projectId: 'my-project-id',
);
```

### Anthropic Foundry

```php
use Anthropic\Foundry;

// 构造函数为私有。需要指定 `baseUrl` 或 `resource`。
$client = Foundry\Client::withCredentials(
    apiKey: getenv('ANTHROPIC_FOUNDRY_API_KEY'),
    baseUrl: 'https://<resource>.services.ai.azure.com/anthropic/v1',
);
```

---

## 基本消息请求

```php
$message = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    messages: [
        ['role' => 'user', 'content' => '法国的首都是哪里？'],
    ],
);

// content 是一个由多态块组成的数组（TextBlock、ToolUseBlock、ThinkingBlock）。如果未检查块类型而直接访问 content[0]->text，当第一个块不是 TextBlock 时（例如启用了扩展思考且首个块为 ThinkingBlock 时）会抛出异常。请始终进行类型检查：
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
    }
}
```

如果您只想要第一个文本块：

```php
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
        break;
    }
}
```

---

## 扩展思考

**对于 Claude 4.6 及以上版本的模型，建议使用自适应思考模式。** Claude 会动态决定何时以及如何进行思考。

```php
use Anthropic\Messages\ThinkingBlock;

$message = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    thinking: ['type' => 'adaptive', 'display' => 'summarized'], // 显示选项：默认情况下，在 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 中会省略思考内容（即显示为空）
    messages: [
        ['role' => 'user', 'content' => '计算：27 * 453'],
    ],
);

// 内容中，ThinkingBlock 会排在 TextBlock 之前
foreach ($message->content as $block) {
    if ($block instanceof ThinkingBlock) {
        echo "思考：\n{$block->thinking}\n\n";
        // $block->signature 是一个不透明的字符串——在多轮对话中若需将思考块原样传递，请务必完整保留
    } elseif ($block->type === 'text') {
        echo "答案：{$block->text}\n";
    }
}
```

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7 和 Sonnet 4.6：** 使用自适应思维（见上文）。`['type' => 'enabled', 'budgetTokens' => N]` 在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 中已被移除（若发送则返回 400 错误）；在 Opus 4.6 和 Sonnet 4.6 中已弃用。  
> **Claude Opus 5.5：** 思维始终开启——可省略 `thinking:`（或发送 `['type' => 'adaptive']`，两者等效）；`['type' => 'disabled']` 在任何调用级别都会返回 400 错误，设置思维预算亦然。请改用 `outputConfig:` 参数来控制输出深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认启用思维——省略 `thinking:` 即为自适应模式（等同于 `['type' => 'adaptive']`），这与 Opus 4.8/4.7 不同，在后者中省略则表示不启用思维。`['type' => 'disabled']` 仅在调用级别为 `high` 或更低时有效；若与 `xhigh` 或 `max` 搭配，则会返回 400 错误。  
> **较旧模型：** 使用 `thinking: ['type' => 'enabled', 'budgetTokens' => N]`（预算必须小于 `maxTokens`，且不低于 1024）。

检查时也可使用 `$block->type === 'thinking'`；对于 PHPStan，使用 `instanceof` 可以缩小类型范围。

---

## 提示缓存

`system:` 接受一个文本块数组，并可在最后一个块上设置 `cacheControl`。采用数组形式语法（驼峰命名键）是惯用做法。有关放置模式和静默失效器审计清单，请参阅 `shared/prompt-caching.md`。

```php
$message = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    system: [
        ['type' => 'text', 'text' => $longSystemPrompt, 'cacheControl' => ['type' => 'ephemeral']],
    ],
    messages: [['role' => 'user', 'content' => '总结要点']],
);
```

如需 1 小时 TTL：`'cacheControl' => ['type' => 'ephemeral', 'ttl' => '1h']`。此外，在 `messages->create(...)` 的顶层也提供了一个 `cacheControl:` 参数，它会自动应用于最后一个可缓存的块。

可通过 `$message->usage->cacheCreationInputTokens` 和 `$message->usage->cacheReadInputTokens` 来验证缓存命中情况。

---

## 停止详情

当 `stopReason` 为 `'refusal'` 时，响应中会包含结构化的 `stopDetails`：

```php
if ($message->stopReason === 'refusal' && $message->stopDetails !== null) {
    echo "类别: " . $message->stopDetails->category . "\n";     // 例如："cyber"、"bio"、"reasoning_extraction"、"frontier_llm"，或 null——完整列表请参阅文档
    echo "说明: " . $message->stopDetails->explanation . "\n";
}
```

**拒绝回退（Claude Fable 5.1）——默认启用。** 回退功能需手动启用；若未启用，被拒绝的请求将直接停止。新的 `claude-fable-5-1` 代码应默认包含服务器端的 `fallbacks` 参数（测试版头部 `server-side-fallback-2026-06-01`，回退模型为 `claude-opus-4-8`，在测试版消息调用中启用）。具体的 PHP 绑定（以及针对无服务器端支持的提供商的客户端中间件）未在此处说明——请通过 WebFetch 从 `shared/live-sources.md` 获取 PHP SDK 仓库的 `examples/` 目录；完整语义请参阅 `shared/model-migration.md` -> 迁移到 Claude Fable 5.1 -> `refusal` 停止原因。

---

## 错误类型

`APIStatusException` 提供了一个 `->type` 属性，用于程序化地对错误进行分类：

```php
try {
    $client->messages->create(...);
} catch (\Anthropic\Core\Exceptions\APIStatusException $e) {
    echo $e->type?->value;  // "rate_limit_error"、"overloaded_error" 等
}
```
