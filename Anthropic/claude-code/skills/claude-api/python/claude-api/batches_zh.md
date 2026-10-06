# 消息批次 API - Python

批次 API（`POST /v1/messages/batches`）以标准价格的 50% 异步处理消息 API 请求。

## 关键信息

- 每个批次最多支持 10 万次请求或 256 MB 数据
- 大多数批次在 1 小时内完成，最长不超过 24 小时
- 结果在创建后 29 天内可用
- 所有令牌使用费用减半
- 支持所有消息 API 功能（视觉、工具、缓存等）

---

## 创建一个批次

```python
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

message_batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id="request-1",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5-5",
                max_tokens=16000,
                messages=[{"role": "user", "content": "总结气候变化的影响"}]
            )
        ),
        Request(
            custom_id="request-2",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5-5",
                max_tokens=16000,
                messages=[{"role": "user", "content": "解释量子计算的基本原理"}]
            )
        ),
    ]
)

print(f"批次 ID: {message_batch.id}")
print(f"状态: {message_batch.processing_status}")
```

---

## 轮询任务完成情况

```python
import time

while True:
    batch = client.messages.batches.retrieve(message_batch.id)
    if batch.processing_status == "ended":
        break
    print(f"状态: {batch.processing_status}, 正在处理: {batch.request_counts.processing}")
    time.sleep(60)

print("批次已完成！")
print(f"成功: {batch.request_counts.succeeded}")
print(f"出错: {batch.request_counts.errored}")
```

---

## 获取结果

> **注意：** 下面的示例使用了 `match/case` 语法，需要 Python 3.10+。如果使用较早版本，请改用 `if/elif` 链。

```python
for result in client.messages.batches.results(message_batch.id):
    match result.result.type:
        case "succeeded":
            msg = result.result.message
            text = next((b.text for b in msg.content if b.type == "text"), "")
            print(f"[{result.custom_id}] {text[:100]}")
        case "errored":
            if result.result.error.type == "invalid_request":
                print(f"[{result.custom_id}] 验证错误 - 请修正请求并重试")
            else:
                print(f"[{result.custom_id}] 服务器错误 - 可安全重试")
        case "canceled":
            print(f"[{result.custom_id}] 已取消")
        case "expired":
            print(f"[{result.custom_id}] 已过期 - 请重新提交")
```

---

## 取消一个批次

```python
cancelled = client.messages.batches.cancel(message_batch.id)
print(f"状态: {cancelled.processing_status}")  # "canceling"
```

---

## 列出批次（自动分页）

对任何 `list()` 调用的返回值进行迭代会自动遍历所有页面——如果您想要获取全部结果，请不要直接索引 `.data`：

```python
for batch in client.messages.batches.list(limit=20):
    print(batch.id, batch.processing_status)
```

如需手动控制分页，请使用 `first_page.has_next_page()`、`first_page.get_next_page()` 和 `first_page.next_page_info()`；`first_page.data` 包含当前页的项目，`first_page.last_id` 是游标。

---

## 使用提示缓存的批次

```python
shared_system = [
    {"type": "text", "text": "您是一位文学分析师。"},
    {
        "type": "text",
        "text": large_document_text,  # 所有请求共享的内容
        "cache_control": {"type": "ephemeral"}
    }
]

message_batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id=f"analysis-{i}",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5-5",
                max_tokens=16000,
                system=shared_system,
                messages=[{"role": "user", "content": question}]
            )
        )
        for i, question in enumerate(questions)
    ]
)
```

---

## 完整端到端示例

```python
import anthropic
import time
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

# 1. 准备请求
items_to_classify = [
    "产品质量非常好！",
    "客户服务太差了，再也不会来了。",
    "还行吧，没什么特别的。",
]

requests = [
    Request(
        custom_id=f"classify-{i}",
        params=MessageCreateParamsNonStreaming(
            model="claude-haiku-4-5",
            max_tokens=50,
            messages=[{
                "role": "user",
                "content": f"请用一个词分类为正面/负面/中性：{text}"
            }]
        )
    )
    for i, text in enumerate(items_to_classify)
]

# 2. 创建批次
batch = client.messages.batches.create(requests=requests)
print(f"已创建批次：{batch.id}")

# 3. 等待完成
while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(10)

# 4. 收集结果
results = {}
for result in client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        msg = result.result.message
        results[result.custom_id] = next((b.text for b in msg.content if b.type == "text"), "")

for custom_id, classification in sorted(results.items()):
    print(f"{custom_id}: {classification}")
```
