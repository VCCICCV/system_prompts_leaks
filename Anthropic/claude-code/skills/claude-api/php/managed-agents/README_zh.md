# 托管代理 - PHP

> **此处未列出的绑定：** 本 README 涵盖了 PHP 中最常见的托管代理工作流。如果您需要未在此处展示的类、方法、命名空间、字段或行为，请通过 WebFetch 从 `shared/live-sources.md` 获取 PHP SDK 仓库**或相关文档页面**，而不要自行猜测。请勿根据 cURL 的结构或其他语言的 SDK 进行推断。

> **代理是持久化的——创建一次，通过 ID 引用。** 请保存 `$client->beta->agents->create` 返回的代理 ID，并在后续每次调用 `->sessions->create` 时传入该 ID；切勿在请求路径中重复调用 `agents->create`。**建议：** 将代理和环境定义为受版本控制的文件，并通过 `ant apply` 同步——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 负责控制平面（创建/更新）；您的代码负责数据平面（使用已保存的 ID 创建会话）。以下示例展示了在必须以编程方式预置时的代码内创建；但在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```bash
composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7"
```

## 客户端初始化

```php
use Anthropic\Client;

// 默认方式（使用 ANTHROPIC_API_KEY 环境变量）
$client = new Client();

// 显式指定 API 密钥
$client = new Client(apiKey: 'your-api-key');
```

---

## 创建一个环境

```php
$environment = $client->beta->environments->create(
    name: 'my-dev-env',
    config: ['type' => 'cloud', 'networking' => ['type' => 'unrestricted']],
);
echo "环境 ID: {$environment->id}\n"; // env_...
```

---

## 创建一个代理（必需的第一步）

> 注意：**不存在内联的代理配置。** `model`/`system`/`tools` 属于代理对象，而非会话。始终从 `$client->beta->agents->create()` 开始——会话要么使用 `agent: $agent->id`，要么使用类型化的 `BetaManagedAgentsAgentParams::with(type: 'agent', id: $agent->id, version: $agent->version)`。

### 最简配置

```php
use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;

// 1. 创建代理（可复用、带版本）
$agent = $client->beta->agents->create(
    name: 'Coding Assistant',
    model: 'claude-opus-5-5',
    system: 'You are a helpful coding assistant.',
    tools: [
        BetaManagedAgentsAgentToolset20260401Params::with(
            type: 'agent_toolset_20260401',
        ),
    ],
);

// 2. 开始一个会话
$session = $client->beta->sessions->create(
    agent: ['type' => 'agent', 'id' => $agent->id, 'version' => $agent->version],
    environmentID: $environment->id,
    title: 'Quickstart session',
);
echo "会话 ID: {$session->id}\n";
echo "追踪链接: https://platform.claude.com/workspaces/default/sessions/{$session->id}\n"; // 如果 API 密钥不在 Default 工作空间，请将 'default' 替换为您的工作空间 ID
```

### 更新代理

更新会创建新版本；每个版本的代理对象都是不可变的。

```php
$updatedAgent = $client->beta->agents->update(
    $agent->id,
    version: $agent->version,
    system: 'You are a helpful coding agent. Always write tests.',
);
echo "新版本: {$updatedAgent->version}\n";

// 列出所有版本
foreach ($client->beta->agents->versions->list($agent->id)->pagingEachItem() as $version) {
    echo "版本 {$version->version}: {$version->updatedAt->format(DateTimeInterface::ATOM)}\n";
}

// 归档代理
$archived = $client->beta->agents->archive($agent->id);
echo "归档时间: {$archived->archivedAt->format(DateTimeInterface::ATOM)}\n";
```

---

## 发送一条用户消息

```php
$client->beta->sessions->events->send(
    $session->id,
    events: [
        [
            'type' => 'user.message',
            'content' => [['type' => 'text', 'text' => 'Review the auth module']],
        ],
    ],
);
```

> 提示：**先开启流**：在发送消息*之前*（或同时）打开流。流仅传递在其打开之后发生的事件——如果在发送消息后才开启流，早期的事件会以批处理的形式缓冲到达。请参阅[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

> 注意：**流式传输器**：PHP 默认的缓冲 PSR-18 客户端无法正确处理持续开放的会话事件流。对于 `streamStream()` 调用，请使用支持流式的 Guzzle 传输器；其他调用则继续使用默认客户端。

```php
$streamingClient = new GuzzleHttp\Client(['stream' => true]);

// 先打开流，再发送用户消息
$stream = $client->beta->sessions->events->streamStream(
    $session->id,
    requestOptions: ['transporter' => $streamingClient],
);
$client->beta->sessions->events->send(
    $session->id,
    events: [
        [
            'type' => 'user.message',
            'content' => [['type' => 'text', 'text' => '总结一下仓库的 README']],
        ],
    ],
);

foreach ($stream as $event) {
    match ($event->type) {
        'agent.message' => array_walk(
            $event->content,
            static fn($block) => $block->type === 'text' ? print($block->text) : null,
        ),
        'agent.tool_use' => print("\n[正在使用工具：{$event->name}]\n"),
        'session.error' => printf("\n[错误：%s]", $event->error?->message ?? '未知'),
        default => null,
    };
    if ($event->type === 'session.status_idle' || $event->type === 'session.error') {
        break;
    }
}
$stream->close();
```

### 重连与尾随

在会话中途重新连接时，应先列出历史事件以去重，然后再尾随实时事件：

```php
$stream = $client->beta->sessions->events->streamStream(
    $session->id,
    requestOptions: ['transporter' => $streamingClient],
);

// 流已打开并处于缓冲状态。在尾随实时事件之前，先列出历史记录。
$seenEventIds = [];
foreach ($client->beta->sessions->events->list($session->id)->pagingEachItem() as $event) {
    $seenEventIds[$event->id] = true;
}

// 尾随实时事件，跳过所有已见过的事件
foreach ($stream as $event) {
    if (isset($seenEventIds[$event->id])) {
        continue;
    }
    $seenEventIds[$event->id] = true;
    match ($event->type) {
        'agent.message' => array_walk(
            $event->content,
            static fn($block) => $block->type === 'text' ? print($block->text) : null,
        ),
        default => null,
    };
    if ($event->type === 'session.status_idle') {
        break;
    }
}
$stream->close();
```

---

## 提供自定义工具结果

> 注意：`user.custom_tool_result` 的 PHP 托管代理绑定尚未在本技能或应用源码示例中记录。有关数据格式，请参阅 `shared/managed-agents-events.md`；相关参数请参考 `anthropic-ai/sdk` PHP 仓库。

---

## 轮询事件

```php
foreach ($client->beta->sessions->events->list($session->id)->pagingEachItem() as $event) {
    echo "{$event->type}: {$event->id}\n";
}
```

---

## 上传文件

> 注意：**PHP 文件上传：** 应用源码示例中未展示 PHP SDK 的 Beta 版托管代理文件上传绑定；标准的 PHP 示例是通过原始 cURL 发送 `POST /v1/files` 请求。如果您的代码库偏好使用 SDK，请在编写代码前从 `anthropic-ai/sdk` PHP 仓库获取最新绑定。
```php
use Anthropic\Beta\Sessions\BetaManagedAgentsFileResourceParams;

// 原始 cURL 上传（来自应用程序源代码的规范示例）
$csvPath = 'data.csv';
$ch = curl_init('https://api.anthropic.com/v1/files');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => [
        'x-api-key: ' . getenv('ANTHROPIC_API_KEY'),
        'anthropic-version: 2023-06-01',
        'anthropic-beta: files-api-2025-04-14',
    ],
    CURLOPT_POSTFIELDS => ['file' => new CURLFile($csvPath, 'text/csv', 'data.csv')],
]);
$file = json_decode(curl_exec($ch));
echo "文件 ID: {$file->id}\n";

// 在会话中挂载
$session = $client->beta->sessions->create(
    agent: $agent->id,
    environmentID: $environment->id,
    resources: [
        BetaManagedAgentsFileResourceParams::with(
            type: 'file',
            fileID: $file->id,
            mountPath: '/workspace/data.csv',
        ),
    ],
);
```

### 在现有会话中添加和管理资源

```php
// 将额外的文件附加到一个已打开的会话
$resource = $client->beta->sessions->resources->add(
    $session->id,
    type: 'file',
    fileID: $file->id,
);
echo "{$resource->id}\n"; // "sesrsc_01ABC..."

// 列出会话中的资源
$listed = $client->beta->sessions->resources->list($session->id);
foreach ($listed->data as $entry) {
    echo "{$entry->id} {$entry->type}\n";
}

// 拆离资源
$client->beta->sessions->resources->delete($resource->id, sessionID: $session->id);
```

---

## 列出并下载会话文件

```php
$files = $client->beta->files->list(
    scopeID: 'sesn_abc123',
    betas: ['managed-agents-2026-04-01'],
);
$content = $client->beta->files->download($files->data[0]->id);
file_put_contents('output.txt', $content);
```

---

## 会话管理

```php
// 列出环境
$environments = $client->beta->environments->list();

// 获取特定环境
$env = $client->beta->environments->retrieve($environment->id);

// 归档环境（只读，已有会话继续运行）
$client->beta->environments->archive($environment->id);

// 删除环境（仅当没有会话引用时）
$client->beta->environments->delete($environment->id);

// 删除会话
$client->beta->sessions->delete($session->id);
```

---

## MCP 服务器集成

```php
use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetParams;
use Anthropic\Beta\Agents\BetaManagedAgentsURLMCPServerParams;
use Anthropic\Beta\Sessions\BetaManagedAgentsAgentParams;

// 代理声明 MCP 服务器（此处无需认证——认证信息存于保险库中）
$agent = $client->beta->agents->create(
    name: 'GitHub Assistant',
    model: 'claude-opus-5-5',
    mcpServers: [
        BetaManagedAgentsURLMCPServerParams::with(
            type: 'url',
            name: 'github',
            url: 'https://api.githubcopilot.com/mcp/',
        ),
    ],
    tools: [
        BetaManagedAgentsAgentToolset20260401Params::with(type: 'agent_toolset_20260401'),
        BetaManagedAgentsMCPToolsetParams::with(
            type: 'mcp_toolset',
            mcpServerName: 'github',
        ),
    ],
);

// 会话关联包含这些 MCP 服务器 URL 凭证的保险库
$session = $client->beta->sessions->create(
    agent: BetaManagedAgentsAgentParams::with(
        type: 'agent',
        id: $agent->id,
        version: $agent->version,
    ),
    environmentID: $environment->id,
    vaultIDs: [$vault->id],
);
```

有关创建保险库及添加凭证，请参阅 `shared/managed-agents-tools.md` 中的“保险库”部分。

---

## 保险库

```php
// 创建保险库
$vault = $client->beta->vaults->create(
    displayName: 'Alice',
    metadata: ['external_user_id' => 'usr_abc123'],
);
echo $vault->id . "\n"; // "vlt_01ABC..."

// 添加 OAuth 凭证
$credential = $client->beta->vaults->credentials->create(
    vaultID: $vault->id,
    displayName: "Alice 的 Slack",
    auth: [
        'type' => 'mcp_oauth',
        'mcp_server_url' => 'https://mcp.slack.com/mcp',
        'access_token' => 'xoxp-...',
        'expires_at' => '2026-04-15T00:00:00Z',
        'refresh' => [
            'token_endpoint' => 'https://slack.com/api/oauth.v2.access',
            'client_id' => '1234567890.0987654321',
            'scope' => 'channels:read chat:write',
            'refresh_token' => 'xoxe-1-...',
            'token_endpoint_auth' => [
                'type' => 'client_secret_post',
                'client_secret' => 'abc123...',
            ],
        ],
    ],
);
// 轮换凭据（例如，在令牌刷新后）
$client->beta->vaults->credentials->update(
    $credential->id,
    vaultID: $vault->id,
    auth: [
        'type' => 'mcp_oauth',
        'access_token' => 'xoxp-new-...',
        'expires_at' => '2026-05-15T00:00:00Z',
        'refresh' => ['refresh_token' => 'xoxe-1-new-...'],
    ],
);

// 归档一个保险库
$client->beta->vaults->archive($vault->id);
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为会话资源（保险库中保存 GitHub 的 MCP 凭据）：

```php
$session = $client->beta->sessions->create(
    agent: $agent->id,
    environmentID: $environment->id,
    vaultIDs: [$vault->id],
    resources: [
        [
            'type' => 'github_repository',
            'url' => 'https://github.com/org/repo',
            'mount_path' => '/workspace/repo',
            'authorization_token' => 'ghp_your_github_token',
        ],
    ],
);
```

在同一会话中挂载多个仓库：

```php
$resources = [
    [
        'type' => 'github_repository',
        'url' => 'https://github.com/org/frontend',
        'mount_path' => '/workspace/frontend',
        'authorization_token' => 'ghp_your_github_token',
    ],
    [
        'type' => 'github_repository',
        'url' => 'https://github.com/org/backend',
        'mount_path' => '/workspace/backend',
        'authorization_token' => 'ghp_your_github_token',
    ],
];
```

轮换某个仓库的授权令牌：

```php
$listed = $client->beta->sessions->resources->list($session->id);
$repoResourceId = $listed->data[0]->id;

$client->beta->sessions->resources->update(
    $repoResourceId,
    sessionID: $session->id,
    authorizationToken: 'ghp_your_new_github_token',
);
```
