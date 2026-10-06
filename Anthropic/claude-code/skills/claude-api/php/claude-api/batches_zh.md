# 消息批处理 - PHP

## 消息批处理 API

```php
$batch = $client->messages->batches->create(requests: [
    ['customId' => 'req-1', 'params' => ['model' => 'claude-opus-5-5', 'maxTokens' => 1024, 'messages' => [...]]],
    ['customId' => 'req-2', 'params' => [...]],
]);
// 轮询 $client->messages->batches->retrieve($batch->id)，直到 processingStatus === 'ended'，
// 然后遍历 $client->messages->batches->results($batch->id)。
```

---