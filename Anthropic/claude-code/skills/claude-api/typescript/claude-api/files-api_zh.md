# Files API - TypeScript

Files API 用于上传文件，以便在 Messages API 请求中使用。在内容块中通过 `file_id` 引用文件，从而避免在多次 API 调用中重复上传。

Files API 已经退出 Beta 阶段。在当前的 SDK 中，`client.beta.files` 的接口结构与之前版本发生了重大变化，现已与正式版的 `client.files` 保持一致——请按照 `shared/live-sources.md` 中 Files API 对应的迁移说明进行更新。以下示例均基于变更前的版本。

## 核心信息

- 最大文件大小：500 MB
- 总存储空间：每个组织 100 GB
- 文件将一直保留，直到被删除
- 文件操作（上传、列出、删除）免费；在消息中使用的文件内容按输入 token 计费
- 不适用于 Amazon Bedrock 或 Google Vertex AI

---

## 上传文件

```typescript
import Anthropic, { toFile } from "@anthropic-ai/sdk";
import fs from "fs";

const client = new Anthropic();

const uploaded = await client.beta.files.upload({
  file: await toFile(fs.createReadStream("report.pdf"), undefined, {
    type: "application/pdf",
  }),
  betas: ["files-api-2025-04-14"],
});

console.log(`文件 ID: ${uploaded.id}`);
console.log(`大小: ${uploaded.size_bytes} 字节`);
```

---

## 在消息中使用文件

### PDF / 文本文档

```typescript
const response = await client.beta.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "请总结这份报告中的主要发现。" },
        {
          type: "document",
          source: { type: "file", file_id: uploaded.id },
          title: "Q4 报告",
          citations: { enabled: true },
        },
      ],
    },
  ],
  betas: ["files-api-2025-04-14"],
});

console.log(response.content[0].text);
```

---

## 管理文件

### 列出文件

```typescript
const files = await client.beta.files.list({
  betas: ["files-api-2025-04-14"],
});
for (const f of files.data) {
  console.log(`${f.id}: ${f.filename} (${f.size_bytes} 字节)`);
}
```

### 删除文件

```typescript
await client.beta.files.delete("file_011CNha8iCJcU1wXNR6q4V8w", {
  betas: ["files-api-2025-04-14"],
});
```

### 下载文件

```typescript
const response = await client.beta.files.download(
  "file_011CNha8iCJcU1wXNR6q4V8w",
  { betas: ["files-api-2025-04-14"] },
);
const content = Buffer.from(await response.arrayBuffer());
await fs.promises.writeFile("output.txt", content);
```
