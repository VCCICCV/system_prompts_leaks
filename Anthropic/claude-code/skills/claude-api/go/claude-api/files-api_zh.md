# Files API - Go

## Files API

> **已退出 Beta 阶段。** 在当前 SDK 中，`client.Beta.Files` 的接口设计与之前版本发生了重大变化，现已与稳定版的 `client.Files` 保持一致——请按照 `shared/live-sources.md` 中 Files API 对应的迁移说明进行更新。以下示例均基于此变更之前的版本。

在 `client.Beta.Files` 下，方法为 **`Upload`**（而非 `New`/`Create`），参数结构体为 `BetaFileUploadParams`。其中的 `File` 字段接受一个 `io.Reader`；可使用 `anthropic.File()` 为文件附加文件名和内容类型，以便进行 multipart 编码。

```go
f, _ := os.Open("./upload_me.txt")
defer f.Close()

meta, err := client.Beta.Files.Upload(ctx, anthropic.BetaFileUploadParams{
    File:  anthropic.File(f, "upload_me.txt", "text/plain"),
    Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaFilesAPI2025_04_14},
})
// meta.ID 即为后续消息请求中引用的文件 ID
```

`Beta.Files` 的其他方法包括：`List`、`Delete`、`Download` 和 `GetMetadata`。

---