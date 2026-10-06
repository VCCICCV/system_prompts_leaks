# 流式传输 - PHP

## 流式传输

> **需要 SDK v0.5.0 及以上版本。** v0.4.0 及更早版本使用单一的 `$params` 数组；使用命名参数调用会抛出 `Unknown named parameter $model` 异常。升级方法：`composer require "anthropic-ai/sdk:^0.7"`

```php
use Anthropic\Messages\RawContentBlockDeltaEvent;
use Anthropic\Messages\TextDelta;

$stream = $client->messages->createStream(
    model: 'claude-opus-5-5',
    maxTokens: 64000,
    messages: [
        ['role' => 'user', 'content' => '写一首俳句'],
    ],
);

foreach ($stream as $event) {
    if ($event instanceof RawContentBlockDeltaEvent && $event->delta instanceof TextDelta) {
        echo $event->delta->text;
    }
}
```

---