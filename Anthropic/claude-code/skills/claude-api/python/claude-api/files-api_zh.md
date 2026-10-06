# Files API - Python

Files API 用于上传文件，以便在 Messages API 请求中使用。在内容块中通过 `file_id` 引用文件，从而避免在多次 API 调用中重复上传。

Files API 已经退出测试阶段。在当前的 SDK 中，`client.beta.files` 的接口结构与之前版本发生了重大变化，现已与稳定的 `client.files` 保持一致——请根据 `shared/live-sources.md` 中的 Files API 相关说明进行迁移。以下示例均为此变更之前的版本。

## 主要信息

- 最大文件大小：500 MB
- 总存储空间：每个组织 100 GB
- 文件会一直保留，直到被删除
- 文件操作（上传、列出、删除）免费；在消息中使用的文件内容按输入 token 计费
- 不适用于 Amazon Bedrock 或 Google Vertex AI

---

## 上传文件

`file` 参数可接受一个 `(filename, content, content_type)` 元组，也可以传入 `pathlib.Path` 对象（或任何 `PathLike` 类型的对象——SDK 会自动读取，且与 `AsyncAnthropic` 兼容，支持异步操作），或者直接传入已打开的二进制文件对象。

```python
import anthropic
from pathlib import Path

client = anthropic.Anthropic()

uploaded = client.beta.files.upload(
    file=("report.pdf", open("report.pdf", "rb"), "application/pdf"),
)
# 或者：client.beta.files.upload(file=Path("report.pdf"))
print(f"文件 ID: {uploaded.id}")
print(f"大小: {uploaded.size_bytes} 字节")
```

---

## 在消息中使用文件

### PDF / 文本文档

```python
response = client.beta.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "请总结这份报告中的主要发现。"},
            {
                "type": "document",
                "source": {"type": "file", "file_id": uploaded.id},
                "title": "Q4 报告",           # 可选
                "citations": {"enabled": True}   # 可选，启用引用功能
            }
        ]
    }],
    betas=["files-api-2025-04-14"],
)
for block in response.content:
    if block.type == "text":
        print(block.text)
```

### 图片

```python
image_file = client.beta.files.upload(
    file=("photo.png", open("photo.png", "rb"), "image/png"),
)

response = client.beta.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "这张图片里有什么？"},
            {
                "type": "image",
                "source": {"type": "file", "file_id": image_file.id}
            }
        ]
    }],
    betas=["files-api-2025-04-14"],
)
```

---

## 管理文件

### 列出文件

直接遍历列表结果——SDK 会自动分页处理所有页面。如果只需要第一页，请使用 `.data`。

```python
for f in client.beta.files.list():
    print(f"{f.id}: {f.filename} ({f.size_bytes} 字节)")
```

### 获取文件元数据

```python
file_info = client.beta.files.retrieve_metadata("file_011CNha8iCJcU1wXNR6q4V8w")
print(f"文件名: {file_info.filename}")
print(f"MIME 类型: {file_info.mime_type}")
```

### 删除文件

```python
client.beta.files.delete("file_011CNha8iCJcU1wXNR6q4V8w")
```

### 下载文件

只有由代码执行工具或技能创建的文件才能下载（用户上传的文件不可下载）。

```python
file_content = client.beta.files.download("file_011CNha8iCJcU1wXNR6q4V8w")
file_content.write_to_file("output.txt")
```

---

## 完整端到端示例

只需上传一次文档，即可针对该文档提出多个问题：

```python
import anthropic

client = anthropic.Anthropic()

# 1. 上传一次
uploaded = client.beta.files.upload(
    file=("contract.pdf", open("contract.pdf", "rb"), "application/pdf"),
)
print(f"已上传: {uploaded.id}")

# 2. 使用相同的file_id提出多个问题
questions = [
    "关键条款和条件是什么？",
    "终止条款是什么？",
    "请总结付款时间表。",
]

for question in questions:
    response = client.beta.messages.create(
        model="claude-opus-5-5",
        max_tokens=16000,
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": question},
                {
                    "type": "document",
                    "source": {"type": "file", "file_id": uploaded.id}
                }
            ]
        }],
        betas=["files-api-2025-04-14"],
    )
    print(f"\n问: {question}")
    text = next((b.text for b in response.content if b.type == "text"), "")
    print(f"答: {text[:200]}")

# 3. 完成后进行清理
client.beta.files.delete(uploaded.id)
```
