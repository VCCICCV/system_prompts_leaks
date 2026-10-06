# Files API - PHP

## Files API

> **已退出 Beta 阶段。** 在当前 SDK 中，`$client->beta->files` 的接口结构与之前版本发生了重大变化，现已与稳定版的 `$client->files` 保持一致——请按照 `shared/live-sources.md` 中 Files API 一节的说明进行迁移。以下示例为变更前的代码。

```php
$file = $client->beta->files->upload(
    file: fopen('upload_me.txt', 'r'),
    betas: ['files-api-2025-04-14'],
);
// 可在 ->beta->messages->create() 中将 $file->id 作为文件内容块引用。
```
