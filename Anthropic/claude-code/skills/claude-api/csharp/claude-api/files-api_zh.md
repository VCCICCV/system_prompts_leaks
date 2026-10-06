# Files API - C#

## Files API

> **已退出 Beta 阶段。** 在当前 SDK 中，`client.Beta.Files` 的接口结构与之前的版本发生了重大变化，现已与稳定的 `client.Files` 保持一致——请按照 `shared/live-sources.md` 中的 Files API 相关说明进行迁移。以下示例均基于此变更之前的内容。

Files 相关功能现位于 `client.Beta.Files`（命名空间为 `Anthropic.Models.Beta.Files`）。`BinaryContent` 可从 `Stream` 和 `byte[]` 隐式转换。

```csharp
using Anthropic.Models.Beta.Files;
using Anthropic.Models.Beta.Messages;

FileMetadata meta = await client.Beta.Files.Upload(
    new FileUploadParams { File = File.OpenRead("doc.pdf") });

// 引用已上传的文件需要使用 Beta 版的消息类型：
new BetaRequestDocumentBlock {
    Source = new BetaFileDocumentSource { FileID = meta.ID },
}
```

非 Beta 版的 `DocumentBlockParamSource` 联合类型中没有包含文件 ID 的变体——引用文件时需使用 `client.Beta.Messages.Create()`。

---