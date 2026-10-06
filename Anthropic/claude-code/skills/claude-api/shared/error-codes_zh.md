# HTTP 错误码参考

本文件记录了 Claude API 返回的 HTTP 错误码、常见原因及处理方法。有关特定语言的错误处理示例，请参阅 `python/` 或 `typescript/` 文件夹。

## 错误码概览

| 状态码 | 错误类型              | 是否可重试 | 常见原因                         |
| ------ | --------------------- | ---------- | -------------------------------- |
| 400    | `invalid_request_error` | 否         | 请求格式或参数无效               |
| 401    | `authentication_error`  | 否         | API 密钥无效或缺失               |
| 402    | `billing_error`         | 否         | 计费或支付问题                   |
| 403    | `permission_error`      | 否         | 当前凭据无权执行此操作           |
| 404    | `not_found_error`       | 否         | 未知端点，或模型未找到或对贵组织不可用 |
| 413    | `request_too_large`     | 否         | 请求超出大小限制                 |
| 429    | `rate_limit_error`      | 是         | 请求次数过多                     |
| 500    | `api_error`             | 是         | Anthropic 服务出现故障           |
| 529    | `overloaded_error`      | 是         | API 暂时过载                     |

## 错误详情

### 400 Bad Request（请求无效）

**原因：**

- 请求体中的 JSON 格式错误
- 缺少必填参数（如 `model`、`max_tokens`、`messages`）
- 参数类型不正确（如应为整数却传入字符串）
- `messages` 数组为空
- `messages` 中用户和助手角色未交替出现
- 提供的 `anthropic-beta` 值不存在，或您的组织未启用该功能。这两种情况都会返回相同的消息：“`anthropic-beta` 头中包含意外值 `<value>`。”

**错误示例：**

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "messages: 角色必须在 'user' 和 'assistant' 之间交替"
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

**解决方法：** 发送请求前请验证请求结构，确保：

- `model` 是有效的模型 ID
- `max_tokens` 是正整数
- `messages` 数组非空且角色交替正确

---

### 401 Unauthorized（未授权）

**原因：**

- 缺少 `x-api-key` 头或 `Authorization` 头
- API 密钥格式无效
- API 密钥已被撤销或删除
- 使用 `x-api-key` 而非 `Authorization: Bearer` 发送 OAuth Bearer 令牌
- 同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`——SDK 会同时发送这两个头，导致 API 拒绝请求

**解决方法：** 设置 `ANTHROPIC_API_KEY`，或运行 `ant auth login` 并将客户端构造函数留空。对于使用 OAuth 令牌的原始 HTTP 请求，请使用 `Authorization: Bearer <token>`（而非 `x-api-key:`）。

---

### 403 Forbidden（禁止访问）

**原因：**

- 当前凭据所属的组织或工作空间无权执行此操作。
- 请求因访问要求被阻止，例如区域限制或身份验证要求，即使该模型通常对贵组织可用。错误信息会提示您如何操作。
- 极少数情况下，模型服务器会拒绝通过 API 访问检查的请求，此时错误信息为：“访问此模型需要访问许可，而您的请求未获得该许可。”

通常，贵组织无法使用的模型会返回 404 错误，而非 403（详见下文）。对于贵组织未启用的 Beta 功能，则会返回 400 错误。

**解决方法：** 请在控制台中检查贵组织的访问权限和工作空间设置。

---

### 404 Not Found（未找到）

**原因：**

- 模型 ID 输入错误（如将 `claude-sonnet-4.6` 错写为 `claude-sonnet-4-6`）
- 使用已弃用的模型 ID
- 模型存在但对贵组织不可用
- API 端点无效

对于不存在的模型以及贵组织无法使用的模型，API 会返回相同的响应：`not_found_error`，且消息以“model: <id>”开头。API 不会向无法使用某模型的调用方透露该模型是否存在。**修复方法：** 使用模型文档中提供的精确模型 ID。您也可以使用别名（例如 `claude-opus-5-5`）。要查看您的组织可以使用哪些模型，请调用 `GET /v1/models`。

---

### 413 请求过大

**原因：**

- 请求体超出最大尺寸
- 输入中的 token 数量过多
- 图像数据过大

**修复方法：** 缩小输入规模——截断对话历史、压缩或调整图像大小，或将大文档拆分成多个小块。

---

### 400 验证错误

部分 400 错误专门与参数验证有关：

- `max_tokens` 超出模型限制
- `temperature` 值无效（必须在 0.0 到 1.0 之间）
- 在扩展思考模式下，`budget_tokens` ≥ `max_tokens`
- 工具定义的 Schema 无效

**Claude Opus 5.5 / Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 的特定于模型的 400 错误：**- `temperature`、`top_p`、`top_k` 参数已被移除——发送这些参数均会返回 400 错误。请删除相关参数；详情参见 `shared/model-migration.md` 中的“各 SDK 语法参考”部分。
- `thinking: {type: "enabled", budget_tokens: N}` 已被移除——发送该参数会返回 400 错误。请改用 `thinking: {type: "adaptive"}`。
- **Claude Opus 5：** 当 `effort` 设置为 `xhigh` 或 `max` 时，`thinking: {type: "disabled"}` 会返回 400 错误；在 `high` 及其以下级别则可接受。默认情况下思考功能已开启，因此省略该参数将启用自适应模式，而非禁用思考。
- **仅限 Fable 5/5.1：** 显式指定 `thinking: {type: "disabled"}` 在任何努力级别下都会返回 400 错误（在 Opus 4.8/4.7 上则可接受）。请直接省略 `thinking` 参数。
- **Fable 5/5.1、Mythos 5/5.1：** 如果组织或工作空间的数据保留策略设置为零数据保留（ZDR），或低于规定的 30 天，则针对这些模型的所有请求都会返回 `400 invalid_request_error` 错误（“要访问此模型，您的组织或工作空间必须启用数据保留”），即使请求体完全有效；ZDR 仅在 Anthropic 明确授权的情况下才允许。请在调试请求体之前检查数据保留配置。
- **Claude Opus 5.5：** `thinking: {type: "disabled"}` 或 `{type: "enabled", budget_tokens: N}` 在所有努力级别下都会返回 400 错误：“此模型不支持 `thinking.type.disabled`。请使用 `thinking.type.adaptive` 和 `output_config.effort` 来控制思考行为。”（对于预算形式则为 `"thinking.type.enabled"`）——请省略 `thinking` 并降低 `output_config.effort`。如果工具条目类型为 `computer_20251124`，则会返回 400 错误：“'claude-opus-5-5' 不支持工具类型：computer_20251124。”随后会提示“您是否指的是以下类型之一”，并列出可接受的类型——请改为声明 `{type: "computer_toolset_20260801"}`（无需 beta 标头，也无需 `name` 或显示名称）。详情参见 `shared/model-migration.md` 中的“迁移到 Claude Opus 5.5”部分。
- **Claude Fable 5.1 / Claude Mythos 5.1 / Claude Opus 5.5 / Claude Sonnet 5.5：** `tool_choice: {type: "any"}` 或 `{type: "tool", name: ...}` 会返回 400 错误：“此模型不支持 `tool_choice: type "tool"` 和 `"any"`。”——在 `count_tokens` 和批处理中亦同。请使用 `{type: "auto"}` 并辅以明确的指令提示（如要求严格遵循 schema 的参数时可设置 `strict: true`），或采用结构化输出。
- **Claude Sonnet 5.5：** `thinking: {type: "disabled"}` 会返回 400 错误：“此模型不支持 `thinking.type.disabled`。请使用 `thinking.type.between_tools` 达到最低的思考强度，或使用 `thinking.type.adaptive` 和 `output_config.effort` 来控制思考行为。”——发送 `{type: "between_tools"}` 可关闭思考，或保持思考开启但调低努力级别。`between_tools` 也有其特定的 400 错误场景：在 `xhigh` 或 `max` 努力级别下（“当此模型的思考被禁用时，不支持 `output_config.effort 'xhigh'`。请使用 `high` 或更低的努力级别，或启用思考。”），以及当同时指定了 `display`、`budget_tokens` 或 `block_binding` 时，在逐条消息调整努力级别时（“messages.N: output_config.effort 'low' 与前一条消息的 'high' 不一致；...”），或在其他任何模型上（“此模型不支持 `thinking.type.between_tools`。”）。在 Claude API 和 Google Cloud 上，如果工具类型为 `computer_20251124`，也会返回 400 错误：“'claude-sonnet-5-5' 不支持工具类型：computer_20251124。”——请声明为 `{type: "computer_toolset_20260801"}`（Amazon Bedrock 仍接受较早的工具类型）。如果顾问工具的 `model` 是 Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5 或 Sonnet 4.6，则在使用 Claude Sonnet 5.5 执行器时会返回 400 错误。下一条所述的历史编辑检查同样适用于 Claude Sonnet 5.5 的思考块——新账户在 Claude API 和 Amazon Bedrock 上默认强制执行——且只有在启用思考时才可使用 `block_binding`。详情参见 `shared/model-migration.md` 中的“迁移到 Claude Sonnet 5.5”部分。
- **Claude Fable 5.1 / Claude Opus 5.5——保留的思考/历史编辑检查（2026 年 8 月 31 日及之后在所有平台上创建的新账户，或任何设置了 `prefix_mismatch_behavior` 的请求；Claude Mythos 5.1 不执行此检查）：** “messages.N.content.M：思考块中的 `signature` 无效。该块绑定到了不同的对话。请移除该块，或设置 `thinking.block_binding.prefix_mismatch_behavior` 为 'drop_block'。”（若未发送 beta 标头，还会额外提示标头名称；若存在变更消息，也可选择注明首条变更消息）——这意味着自该思考块生成以来，系统提示、工具列表或之前的某条消息发生了变化。重复发送相同请求也无法解决此问题；`count_tokens` 也会返回相同的 400 错误。（在 Message Batches API 中，默认行为是丢弃失败的块而非使整个批次出错——只有在显式设置 `prefix_mismatch_behavior: "error"` 时，批次才会标记为 errored。）请移除该命名块及其后的所有思考块后重试一次，或在 beta 标头 `thinking-binding-controls-2026-08-01` 下重新发送，并设置 `thinking.block_binding.prefix_mismatch_behavior: "drop_block"”（该 beta 在 Claude API、AWS 上的 Claude Platform、Bedrock 和 Vertex 上可用；Foundry 尚未确认——参见 `shared/platform-availability.md`；若无该标头，则会因“block_binding: 不允许额外输入”而返回 400 错误）。随后应修复代码逻辑，避免继续修改历史记录（参见 `shared/model-migration.md` 中的“从 Claude Fable 5 迁移到 Claude Fable 5.1”部分）。若上述错误信息中缺少“绑定到不同对话”的说明，则表示签名已被篡改——无论设置如何，均会返回 400 错误。**在旧版本模型（Opus 4.6 及更早版本）中常见的思维扩展错误：**

```
# 错误：budget_tokens 必须小于 max_tokens
thinking: budget_tokens=10000, max_tokens=1000  -> 报错！

# 正确
thinking: budget_tokens=10000, max_tokens=16000
```

---

### 429 请求受限

**原因：**

- 每分钟请求数（RPM）超出限制
- 每分钟令牌数（TPM）超出限制
- 每日令牌数（TPD）超出限制

**需检查的响应头：**

- `retry-after`：重试前需等待的秒数
- `x-ratelimit-limit-*`：您的各项限额
- `x-ratelimit-remaining-*`：剩余配额

**解决方法：** Anthropic SDK 会自动对 429 和 5xx 错误进行指数退避重试（默认最大重试次数为 2 次）。如需自定义重试策略，请参阅各语言的错误处理示例。

---

### 500 内部服务器错误

**原因：**

- Anthropic 服务临时出现问题
- API 处理过程中出现 bug

**解决方法：** 使用指数退避策略重试。若问题持续存在，请访问 [status.anthropic.com](https://status.anthropic.com) 查看服务状态。

---

### 529 系统过载

**原因：**

- API 请求量过高
- 服务容量已满

**解决方法：** 使用指数退避策略重试。建议尝试使用其他模型（Haiku 的负载通常较低），或分散请求时间，亦可实现请求队列机制。

---

## 常见错误及解决方法

| 错误类型                         | 错误码            | 修复方法                                                     |
| ------------------------------- | ---------------- | ------------------------------------------------------- |
| 在 Claude Opus 5.5 / Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 上使用 `temperature`/`top_p`/`top_k` | 400 | 移除该参数（参见 `shared/model-migration.md`）  |
| 在 Claude Opus 5.5 / Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 上使用 `budget_tokens` | 400  | 使用 `thinking: {type: "adaptive"}`                      |
| 在 Fable 5/5.1 上使用 `thinking: {type: "disabled"}` | 400    | 完全省略 `thinking` 参数（在 Opus 4.8/4.7 上允许） |
| 组织设置为 ZDR 或数据保留时间低于 30 天（Fable 5/5.1、Mythos 5/5.1） | 每次请求返回 400 | 修正组织的数据保留配置——问题不在请求体 |
| 在 Claude Opus 5.5 上使用 `thinking: {type: "disabled"}` 或 `budget_tokens` | 400 `"thinking.type.disabled" 不适用于此模型` | 省略 `thinking`；通过 `output_config.effort` 控制生成深度（默认为 `medium`） |
| 在 Claude Opus 5.5 上使用 `computer_20251124` 工具 | 400 `不支持工具类型：computer_20251124` | 使用 `{type: "computer_toolset_20260801"}`——无需测试版头信息，无需 `name` 和显示尺寸；更新代理循环以处理成员工具调用 |
| 在 Claude Sonnet 5.5 上使用 `thinking: {type: "disabled"}` | 400 `"thinking.type.disabled" 不适用于此模型` | 使用 `{type: "between_tools"}`，努力程度设为 `high` 或更低（不得使用其他 `thinking` 字段，也不得逐条消息调整努力程度），或降低整体思考力度 |
| 在其他任何模型上使用 `thinking: {type: "between_tools"}`，或将其努力程度设为 `xhigh` / `max` | 400 | 只能在 Claude Sonnet 5.5 上以 `high` 或更低的努力程度使用；否则请省略 `thinking` |
| 在 Claude Fable 5.1 / Claude Mythos 5.1 / Claude Opus 5.5 / Claude Sonnet 5.5 上使用 `tool_choice` 的值为 `any` 或 `tool` | 400 | 使用 `{type: "auto"}`，并在提示中明确指定工具名称（对于符合 schema 的参数，需设置 `strict: true`），或采用结构化输出 |
| 编辑过的对话历史重新发送，并包含思考块（Claude Fable 5.1 / Claude Opus 5.5 / Claude Sonnet 5.5 支持保留思考；Claude Mythos 5.1 不进行此项检查） | 400 `思考块签名无效……与不同对话绑定` | 停止编辑历史——保持对话记录仅追加模式，使用会话中途的 `role: "system"` 消息、工具变更消息、仅用于提醒且永不删除的轮次范围 `clear_at` 计时器、服务端上下文编辑，以及仅摘要压缩而非直接编辑；若已发生错误，可通过剥离指定块及其后的所有思考块来恢复（文本和工具调用保留），或启用 `prefix_mismatch_behavior: "drop_block"`（仅适用于开启思考的情况，Claude Sonnet 5.5 的 `between_tools` 不适用） |
| 使用 `thinking.block_binding` 但未提供 `thinking-binding-controls-2026-08-01` | 400 `block_binding: 不允许额外输入` | 在提供相关控制功能的测试版区域发送测试版头信息（参见 `shared/platform-availability.md`）；其他情况下请移除 `block_binding` 并尝试重新发送 |
| `budget_tokens` ≥ `max_tokens`（旧模型） | 400 | 确保 `budget_tokens` 小于 `max_tokens`                  |
| 模型 ID 拼写错误                | 404              | 使用正确的模型 ID，例如 `claude-opus-5-5`               |
| 第一条消息为 `assistant`    | 400              | 第一条消息必须是 `user`                            |
| 连续两条相同角色的消息  | 400              | 应交替发送 `user` 和 `assistant` 消息                        |
| API 密钥硬编码在代码中                 | 401（密钥泄露） | 使用环境变量                                |
| 需要自定义重试策略              | 429/5xx          | SDK 会自动重试；可通过 `max_retries` 进行自定义 |

## SDK 中的类型化异常

**始终使用 SDK 提供的类型化异常类**，而不是通过字符串匹配来检查错误信息。每种 HTTP 状态码都对应着各 SDK 中特定的异常类。

### 各语言的异常类命名| HTTP 状态码 | Python (`anthropic.*`) / TypeScript (`Anthropic.*`) | Ruby (`Anthropic::Errors::*`) | Java (`com.anthropic.errors.*`) | C# | PHP (`Anthropic\Core\Exceptions\*`) |
|---|---|---|---|---|---|
| 400 | `BadRequestError` | `BadRequestError` | `BadRequestException` | `AnthropicBadRequestException` | `BadRequestException` |
| 401 | `AuthenticationError` | `AuthenticationError` | `UnauthorizedException` | `AnthropicUnauthorizedException` | `AuthenticationException` |
| 403 | `PermissionDeniedError` | `PermissionDeniedError` | `PermissionDeniedException` | `AnthropicForbiddenException` | `PermissionDeniedException` |
| 404 | `NotFoundError` | `NotFoundError` | `NotFoundException` | `AnthropicNotFoundException` | `NotFoundException` |
| 422 | `UnprocessableEntityError` | `UnprocessableEntityError` | `UnprocessableEntityException` | `AnthropicUnprocessableEntityException` | `UnprocessableEntityException` |
| 429 | `RateLimitError` | `RateLimitError` | `RateLimitException` | `AnthropicRateLimitException` | `RateLimitException` |
| ≥500 | `InternalServerError` | `InternalServerError` | `InternalServerException` | `Anthropic5xxException` | `InternalServerException` |
| 网络错误 | `APIConnectionError` | `APIConnectionError` | `AnthropicIoException` | `AnthropicIOException` | `APIConnectionException` |
| 基类 | `APIError`（两者均适用）；`APIStatusError`（仅 Python） | `APIStatusError` / `APIError` | `AnthropicServiceException` | `AnthropicApiException` | `APIStatusException` / `APIException` |

Ruby 和 PHP 的异常类都位于专门的 errors 命名空间中——应使用 `Anthropic::Errors::RateLimitError` 和 `Anthropic\Core\Exceptions\RateLimitException`，而非直接使用 `Anthropic::RateLimitError`。所有 C# 的 4xx 异常也都继承自 `Anthropic4xxException`。

### 按最具体到最抽象的顺序捕获异常

在 `catch`/`except`/`rescue` 子句中，应按从最具体的子类到基类的顺序排列，并为每种需要不同处理方式的类别分别设置一个子句：可重试的（429、≥500、网络错误）与不可重试的（4xx）。SDK 之所以为每个状态码定义单独的异常类，就是为了便于区分；如果只用一个宽泛的捕获语句，就会丢失这些信息。

```python
try:
    msg = client.messages.create(...)
except anthropic.NotFoundError as e:          # 404 - 例如模型 ID 无效
    ...
except anthropic.RateLimitError as e:         # 429 - 等待后重试
    ...
except anthropic.APIStatusError as e:         # 其他非 2xx 的 HTTP 响应
    print(e.status_code, e.message)
except anthropic.APIConnectionError as e:     # 在收到响应之前的网络故障
    ...
```

这种捕获顺序在各个 SDK 中都适用：TypeScript 中依次为 `instanceof Anthropic.NotFoundError` -> `RateLimitError` -> `APIConnectionError` -> `APIError`（需先检查 `APIConnectionError`，因为它在 TypeScript SDK 中是 `APIError` 的子类，而在 Python 中则是其兄弟类）；Ruby 中为 `rescue Anthropic::Errors::NotFoundError` -> `...::RateLimitError` -> `...::APIStatusError`；Java 中为 `catch (NotFoundException) ... catch (RateLimitException) ... catch (AnthropicServiceException)`；C# 中为 `catch (AnthropicNotFoundException) ... catch (AnthropicRateLimitException) ... catch (AnthropicApiException)`；PHP 中为 `catch (NotFoundException) ... catch (RateLimitException) ... catch (APIStatusException)`。

### Go 语言 — 使用 `errors.As` 后按状态码分支

Go SDK 对于所有非 2xx 的响应都返回一个统一的 `*anthropic.Error` 类型。可通过 `errors.As` 来解包该错误，然后根据 `StatusCode` 进行分支处理：

```go
_, err := client.Messages.New(ctx, params)
if err != nil {
    var apierr *anthropic.Error
    if errors.As(err, &apierr) {
        switch apierr.StatusCode {
        case 404:
            // 模型 ID 或资源不存在
        case 429:
            // 等待后重试
        default:
            // 其他 API 错误 - 可通过 apierr.StatusCode 和 apierr.RequestID 获取更多信息
        }
    } else {
        // 传输层错误（如 *url.Error 包装了 *net.OpError 等）
    }
}
```

### 错误的 `.type` 字段所有 `APIStatusError` 的子类现在都公开了一个 `.type` 属性（Python：`.type`，TypeScript：`.type`，Java：`.errorType()`，Go：`.Type()`，Ruby：`.type`，PHP：`.type`），该属性返回 API 错误类型字符串（例如 `"invalid_request_error"`、`"authentication_error"`、`"rate_limit_error"`、`"overloaded_error"`）。请使用此属性按错误类型名称而非状态码对错误进行分类。`"billing_error"` 对应 402 状态码，`"permission_error"` 对应 403 状态码。

```python
except anthropic.APIStatusError as e:
    if e.type == "rate_limit_error":
        # 处理限流
    elif e.type == "overloaded_error":
        # 处理系统过载
```
