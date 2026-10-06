# 工具使用 - Java

有关概念性概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用（Beta版）

Java SDK 通过注解类支持 Beta 版的工具使用。工具类实现 `Supplier<String>`，可通过 `BetaToolRunner` 自动执行。

### 工具运行器（自动循环）

```java
import com.anthropic.models.beta.messages.MessageCreateParams;
import com.anthropic.models.beta.messages.BetaMessage;
import com.anthropic.helpers.BetaToolRunner;
import com.fasterxml.jackson.annotation.JsonClassDescription;
import com.fasterxml.jackson.annotation.JsonPropertyDescription;
import java.util.function.Supplier;

@JsonClassDescription("获取给定地点的天气")
static class GetWeather implements Supplier<String> {
    @JsonPropertyDescription("城市和州，例如旧金山，加利福尼亚州")
    public String location;

    @Override
    public String get() {
        return "在 " + location + " 的天气是晴朗，气温为 72°F";
    }
}

BetaToolRunner toolRunner = client.beta().messages().toolRunner(
    MessageCreateParams.builder()
        .model("claude-opus-5-5")
        .maxTokens(16000L)
        .putAdditionalHeader("anthropic-beta", "structured-outputs-2025-11-13")
        .addTool(GetWeather.class)
        .addUserMessage("旧金山的天气如何？")
        .build());

for (BetaMessage message : toolRunner) {
    System.out.println(message);
}
```

### 记忆工具

Java SDK 提供了 `BetaMemoryToolHandler`，用于实现记忆工具的后端。您只需提供一个管理文件存储的处理器，`BetaToolRunner` 就会自动处理记忆工具的调用。

```java
import com.anthropic.helpers.BetaMemoryToolHandler;
import com.anthropic.helpers.BetaToolRunner;
import com.anthropic.models.beta.messages.BetaMemoryTool20250818;
import com.anthropic.models.beta.messages.BetaMessage;
import com.anthropic.models.beta.messages.MessageCreateParams;
import com.anthropic.models.beta.messages.ToolRunnerCreateParams;

// 使用您的存储后端（例如文件系统）实现 BetaMemoryToolHandler
BetaMemoryToolHandler memoryHandler = new FileSystemMemoryToolHandler(sandboxRoot);

MessageCreateParams createParams = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(4096L)
    .addTool(BetaMemoryTool20250818.builder().build())
    .addUserMessage("记住我最喜欢的颜色是蓝色")
    .build();

BetaToolRunner toolRunner = client.beta().messages().toolRunner(
    ToolRunnerCreateParams.builder()
        .betaMemoryToolHandler(memoryHandler)
        .initialMessageParams(createParams)
        .build());

for (BetaMessage message : toolRunner) {
    System.out.println(message);
}
```

有关记忆工具的更多详细信息，请参阅 [共享的记忆工具概念](../../shared/tool-use-concepts.md)。

### 非 Beta 版工具声明（手动 JSON 模式）

`Tool.InputSchema.Properties` 是一个自由格式的 `Map<String, JsonValue>` 包装器——可以通过 `putAdditionalProperty` 构建属性模式。默认类型为 `"object"`。构建器还提供了一个直接的 `.addTool(Tool)` 重载，会自动将其包装为 `ToolUnion`。

```java
import com.anthropic.core.JsonValue;
import com.anthropic.models.messages.Tool;

Tool tool = Tool.builder()
    .name("get_weather")
    .description("获取给定地点的当前天气")
    .inputSchema(Tool.InputSchema.builder()
        .properties(Tool.InputSchema.Properties.builder()
            .putAdditionalProperty("location", JsonValue.from(Map.of("type", "string")))
            .build())
        .required(List.of("location"))
        .build())
    .build();

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(16000L)
    .addTool(tool)
    .addUserMessage("巴黎的天气怎么样？")
    .build();
```

对于手动工具循环，处理响应中的 `tool_use` 块，发送 `tool_result` 回来，循环直到 `stop_reason` 为 `"end_turn"`。请参阅[共享工具使用概念](../../shared/tool-use-concepts.md)。

### 使用内容块构建 `MessageParam`（工具结果往返）

`MessageParam.Content` 是一个内部联合类（字符串或列表）。请使用构建器的 `.contentOfBlockParams(List<ContentBlockParam>)` 别名——没有单独的带有静态 `ofBlockParams` 方法的 `MessageParamContent` 类：

```java
import com.anthropic.models.messages.MessageParam;
import com.anthropic.models.messages.ContentBlockParam;
import com.anthropic.models.messages.ToolResultBlockParam;

List<ContentBlockParam> results = List.of(
    ContentBlockParam.ofToolResult(ToolResultBlockParam.builder()
        .toolUseId(toolUseBlock.id())
        .content(yourResultString)
        .build())
);

MessageParam toolResultMsg = MessageParam.builder()
    .role(MessageParam.Role.USER)
    .contentOfBlockParams(results)   // 构建器别名，等同于 Content.ofBlockParams(...)
    .build();
```

---

## 结构化输出

基于类的重载会自动从您的 POJO 中推导出 JSON 模式，并为您提供一个类型化的 `.text()` 返回值——无需手动编写模式，也无需手动解析。

```java
import com.anthropic.models.messages.StructuredMessageCreateParams;

record Book(String title, String author) {}
record BookList(List<Book> books) {}

StructuredMessageCreateParams<BookList> params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(16000L)
    .outputConfig(BookList.class)  // 返回一个类型化的构建器
    .addUserMessage("列出3部经典小说")
    .build();

client.messages().create(params).content().stream()
    .flatMap(cb -> cb.text().stream())
    .forEach(typed -> {
        // typed.text() 返回的是 BookList，而不是 String
        for (Book b : typed.text().books()) System.out.println(b.title());
    });
```

支持 Jackson 注解：`@JsonPropertyDescription`、`@JsonIgnore`、`@ArraySchema(minItems=...)`。手动指定模式的路径为：`OutputConfig.builder().format(JsonOutputFormat.builder().schema(...).build())`。

---

## Anthropic 定义的工具

版本后缀类型的工具；构建器会自动设置 `name` 和 `type`。大多数工具类型都有直接的 `.addTool()` 重载；如果缺少某个重载（较新的或较少使用的工具——请参阅下方的建议说明），则可通过联合类型的静态工厂进行包装：`.addTool(BetaToolUnion.of<ToolName>(builder...build()))`。网络搜索和代码执行由服务器端执行；Bash 和文本编辑器由客户端执行（您需要在本地处理 `tool_use`——请参阅 `shared/tool-use-concepts.md`）。

```java
import com.anthropic.models.messages.WebSearchTool20260209;
import com.anthropic.models.messages.ToolBash20250124;
import com.anthropic.models.messages.ToolTextEditor20250728;
import com.anthropic.models.messages.CodeExecutionTool20260120;

.addTool(WebSearchTool20260209.builder()
    .maxUses(5L)                              // 可选
    .allowedDomains(List.of("example.com"))   // 可选
    .build())
.addTool(ToolBash20250124.builder().build())
.addTool(ToolTextEditor20250728.builder().build())
.addTool(CodeExecutionTool20260120.builder().build())
```

此外还提供：`WebFetchTool20260209`、`MemoryTool20250818`、`ToolSearchToolBm25_20251119`。对于顾问工具，请在 beta 命名空间中使用 `BetaAdvisorTool20260301`，并通过 `.addBeta("advisor-tool-2026-03-01")` 添加（服务器端；顾问模型版本需高于执行模型版本）。beta 构建器上没有直接的 `.addTool(BetaAdvisorTool20260301)` 重载——请通过 `BetaToolUnion` 静态工厂对顾问类型进行包装；如果 `javac` 拒绝特定的工厂方法名称，可运行 `javap com.anthropic.models.beta.messages.BetaToolUnion | grep -i advisor` 查看确切的方法名。

### Beta 命名空间（MCP，压缩）对于仅限 Beta 版的功能，请使用 `com.anthropic.models.beta.messages.*`——类名带有 `Beta` 前缀，并且位于 beta 包中。Beta 版的 `MessageCreateParams.Builder` 同时提供了直接的 `.addTool(BetaToolBash20250124)` 重载方法，以及 `.addMcpServer()` 方法：

```java
import com.anthropic.models.beta.messages.MessageCreateParams;
import com.anthropic.models.beta.messages.BetaToolBash20250124;
import com.anthropic.models.beta.messages.BetaCodeExecutionTool20260120;
import com.anthropic.models.beta.messages.BetaRequestMcpServerUrlDefinition;

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(16000L)
    .addBeta("mcp-client-2025-11-20")
    .addTool(BetaToolBash20250124.builder().build())
    .addTool(BetaCodeExecutionTool20260120.builder().build())
    .addMcpServer(BetaRequestMcpServerUrlDefinition.builder()
        .name("my-server")
        .url("https://example.com/mcp")
        .build())
    .addUserMessage("...")
    .build();

client.beta().messages().create(params);
```

`BetaTool*` 类型与非 Beta 版的 `Tool*` 类型不可互换——每次请求只能选择一个命名空间。

**在响应中读取服务器工具块：** `ServerToolUseBlock` 提供了 `.id()`、`.name()`（枚举类型）以及返回原始 `JsonValue` 的 `._input()` 方法——没有强类型的 `.input()` 方法。对于代码执行结果，需要展开两层：

```java
for (ContentBlock block : response.content()) {
    block.serverToolUse().ifPresent(stu -> {
        System.out.println("工具: " + stu.name() + " 输入: " + stu._input());
    });
    block.codeExecutionToolResult().ifPresent(r -> {
        r.content().resultBlock().ifPresent(result -> {
            System.out.println("标准输出: " + result.stdout());
            System.out.println("标准错误: " + result.stderr());
            System.out.println("退出码: " + result.returnCode());
        });
    });
}
```

---