# 令牌计数

请使用 `count_tokens` 端点（`POST /v1/messages/count_tokens`）来对 Claude 模型进行准确的令牌计数。令牌计数是**特定于模型的**——请传入与推理时相同的模型 ID。

**请勿使用 `tiktoken`。** 它是 OpenAI 的分词器，在处理普通文本时会低估 Claude 的令牌数量约 15%-20%，而在处理代码或非英语输入时，低估幅度会更大。任何来自 `tiktoken`、`gpt-tokenizer` 或类似工具的估算值对 Claude 都是不准确的。

## 对文件或字符串进行计数

```python
from anthropic import Anthropic

client = Anthropic()
resp = client.messages.count_tokens(
    model="claude-opus-5-5",
    messages=[{"role": "user", "content": open("CLAUDE.md").read()}],
)
print(resp.input_tokens)
```

TypeScript：`await client.messages.countTokens({model, messages})` -> `.input_tokens`。其他 SDK 请参阅 `{lang}/claude-api/README.md`。

## 命令行

```sh
ant messages count-tokens --model claude-opus-5-5 \
  --message '{role: user, content: "@./CLAUDE.md"}' \
  --transform input_tokens -r
```

## 对比两个版本的文件差异

该端点是无状态的——请分别对每个版本进行计数，然后相减：

```python
from anthropic import Anthropic
import subprocess

client = Anthropic()
def count(text: str) -> int:
    return client.messages.count_tokens(
        model="claude-opus-5-5",
        messages=[{"role": "user", "content": text}],
    ).input_tokens

before = subprocess.check_output(["git", "show", "HEAD:CLAUDE.md"], text=True)
after = open("CLAUDE.md").read()
print(count(after) - count(before))
```

完整文档：请参阅 `shared/live-sources.md` 中的“令牌计数”条目。