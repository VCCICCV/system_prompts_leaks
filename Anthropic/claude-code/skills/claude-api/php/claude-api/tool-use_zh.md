# 工具使用 - PHP

有关概念性概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 工具运行器（Beta）

**Beta 版：** PHP SDK 通过 `$client->beta->messages->toolRunner()` 提供了一个工具运行器。使用 `BetaRunnableTool` 定义工具——一个定义数组加上一个 `run` 闭包：

```php
use Anthropic\Lib\Tools\BetaRunnableTool;

$weatherTool = new BetaRunnableTool(
    definition: [
        'name' => 'get_weather',
        'description' => '获取某个地点的当前天气。',
        'inputSchema' => [
            'type' => 'object',
            'properties' => [
                'location' => ['type' => 'string', 'description' => '城市和州'],
            ],
            'required' => ['location'],
        ],
    ],
    run: function (array $input): string {
        return "在{$input['location']}，天气晴朗，气温72华氏度。";
    },
);

$runner = $client->beta->messages->toolRunner(
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => '巴黎的天气如何？']],
    model: 'claude-opus-5-5',
    tools: [$weatherTool],
);

foreach ($runner as $message) {
    foreach ($message->content as $block) {
        if ($block->type === 'text') {
            echo $block->text;
        }
    }
}
```

### 手动循环

工具以数组形式传递。**SDK 使用驼峰命名法的键**（`inputSchema`、`toolUseID`、`stopReason`），并在 v0.5.0 及以上版本中自动映射到 API 的蛇形命名法。有关循环模式，请参阅 [共享工具使用概念](../../shared/tool-use-concepts.md)。

```php
use Anthropic\Messages\ToolUseBlock;

$tools = [
    [
        'name' => 'get_weather',
        'description' => '获取给定地点的当前天气',
        'inputSchema' => [  // 驼峰命名法，而非 input_schema
            'type' => 'object',
            'properties' => [
                'location' => ['type' => 'string', 'description' => '城市和州'],
            ],
            'required' => ['location'],
        ],
    ],
];

$messages = [['role' => 'user', 'content' => '旧金山的天气如何？']];

$response = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    tools: $tools,
    messages: $messages,
);

while ($response->stopReason === 'tool_use') {  // 驼峰命名法属性
    $toolResults = [];
    foreach ($response->content as $block) {
        if ($block instanceof ToolUseBlock) {
            // $block->name  : string               - 要调度的工具名称
            // $block->input : array<string,mixed>  - 解析后的 JSON 输入
            // $block->id    : string               - 作为 toolUseID 返回
            $result = executeYourTool($block->name, $block->input);
            $toolResults[] = [
                'type' => 'tool_result',
                'toolUseID' => $block->id,  // 驼峰命名法，而非 tool_use_id
                'content' => $result,
            ];
        }
    }

    // 追加助手回复 + 带有工具结果的用户回复
    $messages[] = ['role' => 'assistant', 'content' => $response->content];
    $messages[] = ['role' => 'user', 'content' => $toolResults];

    $response = $client->messages->create(
        model: 'claude-opus-5-5',
        maxTokens: 16000,
        tools: $tools,
        messages: $messages,
    );
}

// 最终文本回复
foreach ($response->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
    }
}
```

`$block->type === 'tool_use'` 也可以工作；`instanceof ToolUseBlock` 可以为 PHPStan 缩小类型范围。

---

## 结构化输出

### 使用 StructuredOutputModel（推荐）

定义一个实现 `StructuredOutputModel` 接口的 PHP 类，并将其作为 `outputConfig` 传递：

```php
use Anthropic\Lib\Contracts\StructuredOutputModel;
use Anthropic\Lib\Concerns\StructuredOutputModelTrait;
use Anthropic\Lib\Attributes\Constrained;

class Person implements StructuredOutputModel
{
    use StructuredOutputModelTrait;

    #[Constrained(description: '全名')]
    public string $name;

    public int $age;

    public ?string $email = null;  // 可为空 = 可选字段
}

$message = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => '生成一位 30 岁的 Alice 的个人资料']],
    outputConfig: ['format' => Person::class],
);
$person = $message->parsedOutput();  // Person 实例
echo $person->name;
```

类型会根据 PHP 类型提示自动推断。使用 `#[Constrained(description: '...')]` 可以添加描述。可空属性（`?string`）会变为可选字段。

### 原始 Schema

```php
$message = $client->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => '提取：John (john@co.com)，企业版']],
    outputConfig: [
        'format' => [
            'type' => 'json_schema',
            'schema' => [
                'type' => 'object',
                'properties' => [
                    'name' => ['type' => 'string'],
                    'email' => ['type' => 'string'],
                    'plan' => ['type' => 'string'],
                ],
                'required' => ['name', 'email', 'plan'],
                'additionalProperties' => false,
            ],
        ],
    ],
);

// 第一个文本块包含有效的 JSON
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        $data = json_decode($block->text, true);
        break;
    }
}
```

---

## Beta 功能与 Anthropic 定义的工具

**`betas:` 并不是 `$client->messages->create()` 的参数**——它仅存在于 beta 命名空间中。对于需要显式启用标头的功能，请使用它：

```php
use Anthropic\Beta\Messages\BetaRequestMCPServerURLDefinition;

$response = $client->beta->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    mcpServers: [
        BetaRequestMCPServerURLDefinition::with(
            name: 'my-server',
            url: 'https://example.com/mcp',
        ),
    ],
    betas: ['mcp-client-2025-11-20'],  // 仅在 ->beta->messages 中有效
    messages: [['role' => 'user', 'content' => '使用 MCP 工具']],
);
```

### 任务预算

```php
$response = $client->beta->messages->create(
    model: 'claude-opus-5-5',
    maxTokens: 16000,
    outputConfig: ['taskBudget' => ['type' => 'tokens', 'total' => 64000]],
    tools: [...],
    messages: [...],
    betas: ['task-budgets-2026-03-13'],
);
```

### 缓存诊断

在下一次请求中传入上一次响应的 `id`；在响应中打印 `diagnostics` 对象：

```php
$r2 = $client->beta->messages->create(
    model: 'claude-opus-5-5', maxTokens: 1024,
    diagnostics: ['previousMessageId' => $r1->id],
    betas: ['cache-diagnosis-2026-04-07'],
    messages: [...],
);
```

**Anthropic 定义的工具**（bash、网络搜索、文本编辑器、代码执行）已正式上线，两种调用方式均适用。其中，网络搜索和代码执行由服务器端执行；bash 和文本编辑器则由客户端执行（您需在本地处理 `tool_use`）——非 Beta 版本为 `Anthropic\Messages\ToolBash20250124` / `WebSearchTool20260209` / `ToolTextEditor20250728` / `CodeExecutionTool20260120`，Beta 版本为 `Anthropic\Beta\Messages\BetaToolBash20250124` / `BetaWebSearchTool20260209` / `BetaToolTextEditor20250728` / `BetaCodeExecutionTool20260120`。这些工具无需 `betas:` 标头。

### 工具搜索（非 Beta，服务器端）

```php
tools: [
    ['type' => 'tool_search_tool_regex_20251119', 'name' => 'tool_search_tool_regex'],
    ['name' => 'get_weather', 'description' => '...', 'inputSchema' => [...], 'deferLoading' => true],
    // ... 其他用户工具，设置 'deferLoading' => true
],
```

### 内存工具（非 Beta，客户端执行）

声明 `['type' => 'memory_20250818', 'name' => 'memory']`。通过读写固定 `/memories` 目录下的文件来处理 `tool_use`。**验证模型提供的每一个路径**：将其解析为规范形式，并确保其仍位于内存目录内；拒绝任何越界访问（如 `..` 或符号链接）——详情请参阅 `shared/tool-use-concepts.md` 第“客户端工具”章节。

---

