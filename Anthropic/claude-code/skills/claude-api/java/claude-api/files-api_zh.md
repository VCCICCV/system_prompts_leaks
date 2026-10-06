# Files API - Java

## Files API

> **已退出测试阶段。** 在当前 SDK 中，`client.beta().files()` 的接口设计与之前版本发生了重大变化，现已与稳定版的 `client.files()` 保持一致——请按照 `shared/live-sources.md` 中 Files API 一栏的说明进行迁移。以下示例均基于此变更之前的内容。

在 `client.beta().files()` 下。消息中的文件引用需要使用测试版的消息类型（非测试版的 `DocumentBlockParam.Source` 没有包含文件 ID 的变体）。

```java
import com.anthropic.models.beta.files.FileUploadParams;
import com.anthropic.models.beta.files.FileMetadata;
import com.anthropic.models.beta.messages.BetaRequestDocumentBlock;
import com.anthropic.models.beta.messages.BetaFileDocumentSource;
import java.nio.file.Paths;

FileMetadata meta = client.beta().files().upload(
    FileUploadParams.builder()
        .file(Paths.get("/path/to/doc.pdf"))  // 或者 .file(InputStream) 或 .file(byte[])
        .build());

// 在测试版消息中引用：
BetaRequestDocumentBlock doc = BetaRequestDocumentBlock.builder()
    .source(BetaFileDocumentSource.builder().fileId(meta.id()).build())
    .build();
```

其他方法：`.list()`、`.delete(String fileId)`、`.download(String fileId)`、`.retrieveMetadata(String fileId)`。
