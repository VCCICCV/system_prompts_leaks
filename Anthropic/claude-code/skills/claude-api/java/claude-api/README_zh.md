# Claude API - Java

> **注意：** Java SDK 支持使用注解类调用 Claude API 和 Beta 工具。Agent SDK 尚未提供 Java 版本。

## 包引用

类型按包组织。如果某个您需要的类未在下方示例中列出，请先通过下表定位，不要因从网络获取 SDK 源码而阻塞。

| `import` 前缀 | 包含内容 |
|---|---|
| `com.anthropic.client` / `com.anthropic.client.okhttp` | `AnthropicClient`、`AnthropicOkHttpClient` |
| `com.anthropic.models.messages` | 非 Beta 请求/响应类型——`MessageCreateParams`、`Model`、`Message`、`TextBlockParam`、`ContentBlockParam`、`ToolUseBlockParam`、`ToolResultBlockParam`、`CacheControlEphemeral`、`Tool*`（如 `ToolBash20250124`、`ToolTextEditor20250728`）、`StopReason`、`StructuredMessage*` |
| `com.anthropic.models.messages.batches` | 批量 API——`BatchResultsParams`、`MessageBatchIndividualResponse` |
| `com.anthropic.models.beta` | `AnthropicBeta`（Beta 标志常量） |
| `com.anthropic.models.beta.messages` | Beta 端点类型——`MessageCreateParams`、`BetaMessage`、`BetaStopReason`、`BetaContextManagementConfig`、`BetaMcpToolset`、`BetaRequestMcpServerUrlDefinition`、`BetaTool*` |
| `com.anthropic.core` | `JsonValue`、`JsonField`、`JsonSchemaLocalValidation`、`com.anthropic.core.http.StreamResponse` |
| `com.anthropic.errors` | 类型化异常——`AnthropicServiceException`、`RateLimitException`、`NotFoundException` 等（参见 `shared/error-codes.md`） |

`client.messages()` 使用 `com.anthropic.models.messages.*`；`client.beta().messages()` 使用 `com.anthropic.models.beta.messages.*`。两个包都定义了 `MessageCreateParams`——请根据调用的客户端路径导入相应版本。

### 各功能的关键类型

请参考此表，而非直接使用 `javap` 或查看 JAR 文件。端点列指示应使用 `client.messages()` 还是 `client.beta().messages()`。

| 功能 | 端点 | 关键 Java 类型 / 构建器调用 |
|---|---|---|
| 用户档案 | Beta | `client.beta().userProfiles().create(...)` / `.retrieve(id)` / `.list()`。将返回的用户档案 ID 传递给 Beta 的 `MessageCreateParams`。需要 Beta 头部——请查阅 SDK 的 Beta 头部参考以获取当前标志。 |
| Agent 技能 | Beta | `BetaContainerParams`、`BetaSkillParams`、`BetaCodeExecutionTool20250825`。`.addBeta("code-execution-2025-08-25")`（技能已退出 Beta——无 `skills-2025-10-02`）。可通过 `client.beta().files().download(fileId)` 下载输出结果。 |
| 缓存诊断 | Beta | `BetaDiagnosticsParam`、`BetaCacheControlEphemeral` |
| 上下文编辑 | Beta | `.contextManagement(BetaContextManagementConfig.builder()...)`。编辑策略为 `BetaClearToolUses20250919Edit`（或 `BetaClearThinking20251015Edit`）；其触发条件为单独构建的 `BetaInputTokensTrigger`，并传入编辑器的构建器——编辑器构建器上没有直接的 `.inputTokensTrigger(N)` 快捷方法。请使用 `javap` 查看编辑和触发器类的具体 setter 名称。 |
| 记忆工具 | 非 Beta | 从 `com.anthropic.models.messages` 中使用 `.addTool(MemoryTool20250818.builder().build())` |
| 程序化工具调用 | 非 Beta | `CodeExecutionTool20260120`、`Tool`、`ContentBlockParam` |
| 严格工具使用 | 非 Beta | `Tool`、`Tool.InputSchema` |
| 任务预算 | Beta | `.outputConfig(BetaOutputConfig.builder().taskBudget(BetaTokenTaskBudget.builder()...))` |
| 工具搜索 | 非 Beta | 从 `com.anthropic.models.messages` 中使用 `.addTool(ToolSearchToolRegex20251119.builder()...)` |
| 网络搜索 | 非 Beta | `WebSearchTool20260209` 来自 `com.anthropic.models.messages`——这是最新版本，支持动态过滤（Claude Fable 5.1 + Claude Opus 5.5 + Claude Opus 5 + Opus 4.8/4.7/4.6 + Claude Sonnet 5.5 + Claude Sonnet 5 + Sonnet 4.6）。对于较旧模型或 Vertex，请使用 `WebSearchTool20250305` |

### 发现类型与成员名称如果所需的类或构建方法未在上述表格中列出，可以使用 `jar tf <anthropic-java-core jar> | grep -i <term>` 或 `javap -classpath <jar> com.anthropic.models....` 快速查找名称。**请勿编译并运行单独的反射程序**来枚举成员——首次构建在许多环境中已经足够耗时，可能会使您陷入轮询循环。请根据查找到的名称编写脚本，并让编译器报错（“无法找到符号”）指出任何错误的成员。

## 安装

Maven：

```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>2.34.0</version>
</dependency>
```

Gradle：

```groovy
implementation("com.anthropic:anthropic-java:2.34.0")
```

## 客户端初始化

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;

// 默认方式（从环境变量中读取 ANTHROPIC_API_KEY）
AnthropicClient client = AnthropicOkHttpClient.fromEnv();

// 显式指定 API 密钥
AnthropicClient client = AnthropicOkHttpClient.builder()
    .apiKey("your-api-key")
    .build();
```

---

## 基本消息请求

```java
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.Message;

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")  // .model(String) 重载适用于所有模型 ID；使用 Model.* 类型常量会滞后于新模型发布
    .maxTokens(16000L)
    .addUserMessage("法国的首都是哪里？")
    .build();

Message response = client.messages().create(params);
response.content().stream()
    .flatMap(block -> block.text().stream())
    .forEach(textBlock -> System.out.println(textBlock.text()));
```

---

## 思考模式

**对于 Claude 4.6 及以上版本的模型，推荐使用自适应思考模式。** Claude 会动态决定何时以及以何种程度进行思考。构建器提供了直接的 `.thinking(ThinkingConfigAdaptive)` 重载，无需手动包装联合类型。

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7 和 Sonnet 4.6：** 使用自适应思考模式（见下文）。`ThinkingConfigEnabled.builder().budgetTokens(N)` 在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 上已被移除（若传入则为 400 错误）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。  
> **Claude Opus 5.5：** 思考始终开启——可省略 `.thinking(...)`（或传入等价的 `ThinkingConfigAdaptive`）；使用 `ThinkingConfigDisabled` 会在任何努力级别上返回 400 错误，设置思考预算也同样如此。请改用 `.outputConfig(OutputConfig.builder().effort(...))` 来控制深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认启用思考——省略 `.thinking(...)` 即为自适应模式（等同于 `ThinkingConfigAdaptive`），这与 Opus 4.8/4.7 不同，在后者中省略则表示不进行思考。`ThinkingConfigDisabled` 仅在努力级别为 `HIGH` 或更低时有效；若与 `XHIGH` 或 `MAX` 搭配，则会返回 400 错误。  
> **较旧的模型：** 请使用 `.thinking(ThinkingConfigEnabled.builder().budgetTokens(N).build())`（预算必须小于 `maxTokens`，且最小为 1024）。

```java
import com.anthropic.models.messages.ContentBlock;
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.ThinkingConfigAdaptive;

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(16000L)
    // 显示选项：默认情况下会被省略（思考文本为空），适用于 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5。
    .thinking(ThinkingConfigAdaptive.builder().display(ThinkingConfigAdaptive.Display.SUMMARIZED).build())
    .addUserMessage("请逐步解答：27 * 453")
    .build();

for (ContentBlock block : client.messages().create(params).content()) {
    block.thinking().ifPresent(t -> System.out.println("[思考] " + t.thinking()));
    block.text().ifPresent(t -> System.out.println(t.text()));
}
```

`ContentBlock` 类型收窄：`.thinking()` / `.text()` 返回 `Optional<T>`——请使用 `.ifPresent(...)` 或 `.stream().flatMap(...)`。另一种方式是使用 `isThinking()` / `asThinking()` 这样的布尔判断加解包组合（若类型不匹配则会抛出异常）。

---

## 努力参数

努力程度嵌套在 `OutputConfig` 中——`MessageCreateParams.Builder` 上没有直接的 `.effort()` 方法。

```java
import com.anthropic.models.messages.OutputConfig;

.outputConfig(OutputConfig.builder()
    .effort(OutputConfig.Effort.HIGH)  // 或 LOW、MEDIUM、XHIGH、MAX
    .build())
```

结合 `Thinking = ThinkingConfigAdaptive` 可实现成本与质量的平衡控制。

---

## 提示缓存

系统消息应以 `TextBlockParam` 列表的形式提供，并附带 `CacheControlEphemeral` 缓存控制。请使用 `.systemOfTextBlockParams(...)` 方法——普通的 `.system(String)` 重载无法携带缓存控制信息。关于放置模式及静默失效审计清单，请参阅 `shared/prompt-caching.md`。

```java
import com.anthropic.models.messages.TextBlockParam;
import com.anthropic.models.messages.CacheControlEphemeral;

.systemOfTextBlockParams(List.of(
    TextBlockParam.builder()
        .text(longSystemPrompt)
        .cacheControl(CacheControlEphemeral.builder()
            .ttl(CacheControlEphemeral.Ttl.TTL_1H)  // 可选；也支持 TTL_5M
            .build())
        .build()))
```

此外，`MessageCreateParams.Builder` 和 `Tool.builder()` 上还提供了顶层的 `.cacheControl(CacheControlEphemeral)` 方法。

可通过 `response.usage().cacheCreationInputTokens()` 和 `response.usage().cacheReadInputTokens()` 来验证缓存命中情况。

---

## 令牌计数

```java
import com.anthropic.models.messages.MessageCountTokensParams;

long tokens = client.messages().countTokens(
    MessageCountTokensParams.builder()
        .model("claude-opus-5-5")
        .addUserMessage("Hello")
        .build()
).inputTokens();
```

---

## PDF/文档输入

`DocumentBlockParam` 构建器提供了便捷的来源设置方法。将其包装为 `ContentBlockParam.ofDocument()` 后，通过 `.addUserMessageOfBlockParams()` 方法传入。

```java
import com.anthropic.models.messages.DocumentBlockParam;
import com.anthropic.models.messages.ContentBlockParam;
import com.anthropic.models.messages.TextBlockParam;

DocumentBlockParam doc = DocumentBlockParam.builder()
    .source(Base64PdfSource.builder().data(base64String).build())
    // 或 .source(UrlPdfSource.builder().url("https://...").build())
    .title("My Document")        // 可选
    .build();
```

对于 **Files API** 的文档引用，请使用测试版路径和测试版类型——详见 `files-api.md`：`BetaRequestDocumentBlock.builder().source(BetaFileDocumentSource.builder().fileId(id).build())`。

```java
.addUserMessageOfBlockParams(List.of(
    ContentBlockParam.ofDocument(doc),
    ContentBlockParam.ofText(TextBlockParam.builder().text("Summarize this").build())))
```

---

## 停止详情

当 `stopReason()` 为 `"refusal"` 时，响应中会包含结构化的 `stopDetails()`：

```java
response.stopDetails().ifPresent(details -> {
    System.out.println("类别: " + details.category());
    System.out.println("说明: " + details.explanation());
});
```

**拒绝回退机制（Claude Fable 5.1）——默认启用。** 回退机制需显式启用；若未启用，被拒绝的请求将直接停止。新的 `claude-fable-5-1` 代码应默认包含服务端的 `fallbacks` 参数（测试版头文件 `server-side-fallback-2026-06-01`，回退模型为 `claude-opus-4-8`，在测试版的消息调用中启用）。具体的 Java 构建器方法（以及针对不支持服务端回退的提供商的客户端中间件）未在此处详述——请从 `shared/live-sources.md` 中获取 Java SDK 仓库的 `examples/` 目录；完整语义请参见 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”章节下的“`refusal` 停止原因”部分。

---

## 错误类型

`AnthropicServiceException` 提供了 `.errorType()` 方法，返回 `Optional<ErrorType>`，便于程序化地对错误进行分类：

```java
try {
    client.messages().create(params);
} catch (AnthropicServiceException e) {
    e.errorType().ifPresent(type ->
        System.out.println("错误类型：" + type)  // RATE_LIMIT_ERROR、OVERLOADED_ERROR 等
    );
}
```

---