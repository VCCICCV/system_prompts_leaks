# 受管代理 - Java

> **此处未展示的绑定：** 本自述文件涵盖了 Java 中最常见的受管代理流程。如果您需要未在此处展示的类、方法、命名空间、字段或行为，请通过 WebFetch 从 `shared/live-sources.md` 获取 Java SDK 仓库**或相关文档页面**，而不要自行猜测。请勿根据 cURL 的形式或其他语言的 SDK 进行推断。

> **代理是持久化的——只需创建一次，之后通过 ID 引用。** 请保存 `client.beta().agents().create` 返回的代理 ID，并在后续每次调用 `client.beta().sessions().create` 时传入该 ID；切勿在请求路径中重复调用 `agents().create`。**建议：** 将代理和环境定义为使用版本控制的文件，并通过 `ant apply` 同步——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 负责控制平面（创建/更新）；您的代码负责数据平面（使用已存储 ID 的会话）。以下示例展示了在必须以编程方式预置时的代码内创建；在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
</dependency>
```

## 客户端初始化

```java
import com.anthropic.client.okhttp.AnthropicOkHttpClient;

// 默认配置（使用 ANTHROPIC_API_KEY 环境变量）
var client = AnthropicOkHttpClient.fromEnv();
```

---

## 创建环境

```java
import com.anthropic.models.beta.environments.BetaCloudConfigParams;
import com.anthropic.models.beta.environments.BetaUnrestrictedNetwork;
import com.anthropic.models.beta.environments.EnvironmentCreateParams;

var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
    .name("my-dev-env")
    .config(BetaCloudConfigParams.builder()
        .networking(BetaUnrestrictedNetwork.builder().build())
        .build())
    .build());
System.out.println("环境 ID: " + environment.id()); // env_...
```

---

## 创建代理（必需的第一步）

> 警告：**不存在内联代理配置。** 模型、系统和工具都存在于代理对象上，而不是会话中。始终先调用 `client.beta().agents().create()`——会话要么使用 `.agent(agent.id())`，要么使用类型化的 `BetaManagedAgentsAgentParams.builder()...build()`。

### 最简配置

```java
import com.anthropic.models.beta.agents.AgentCreateParams;
import com.anthropic.models.beta.agents.BetaManagedAgentsAgentToolset20260401Params;
import com.anthropic.models.beta.sessions.BetaManagedAgentsAgentParams;
import com.anthropic.models.beta.sessions.SessionCreateParams;

// 1. 创建代理（可复用、支持版本管理）
var agent = client.beta().agents().create(AgentCreateParams.builder()
    .name("编码助手")
    .model("claude-opus-5-5")
    .system("你是一位乐于助人的编码助手。")
    .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
        .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
        .build())
    .build());

// 2. 开始会话
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(BetaManagedAgentsAgentParams.builder()
        .type(BetaManagedAgentsAgentParams.Type.AGENT)
        .id(agent.id())
        .version(agent.version())
        .build())
    .environmentId(environment.id())
    .title("快速入门会话")
    .build());
System.out.println("会话 ID: " + session.id());
System.out.println("追踪链接: https://platform.claude.com/workspaces/default/sessions/" + session.id()); // 如果 API 密钥不在默认工作区，请将 'default' 替换为您的工作区 ID
```

### 更新代理

更新会创建新版本；每个版本的代理对象都是不可变的。

```java
import com.anthropic.models.beta.agents.AgentUpdateParams;

var updatedAgent = client.beta().agents().update(agent.id(), AgentUpdateParams.builder()
    .version(agent.version())
    .system("你是一位乐于助人的编码代理。务必编写测试。")
    .build());
System.out.println("新版本: " + updatedAgent.version());

// 列出所有版本
for (var version : client.beta().agents().versions().list(agent.id()).autoPager()) {
    System.out.println("版本 " + version.version() + ": " + version.updatedAt());
}
// 归档该代理
var archived = client.beta().agents().archive(agent.id());
System.out.println("归档时间：" + archived.archivedAt().orElseThrow());
```

---

## 发送用户消息

```java
import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserMessageEventParams;
import com.anthropic.models.beta.sessions.events.EventSendParams;

client.beta().sessions().events().send(session.id(), EventSendParams.builder()
    .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
        .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
        .addTextContent("审查 auth 模块")
        .build())
    .build());
```

> 提示：**流优先**：在发送消息之前（或同时）打开流。流仅传递在其打开之后发生的事件——先发送后打开流会导致早期事件以批处理形式缓冲到达。参见[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式接收事件（SSE）

```java
import com.anthropic.models.beta.sessions.events.StreamEvents;

// 先打开流，再发送用户消息
try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
    client.beta().sessions().events().send(session.id(), EventSendParams.builder()
        .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
            .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
            .addTextContent("总结仓库的 README")
            .build())
        .build());

    for (var event : (Iterable<StreamEvents>) stream.stream()::iterator) {
        if (event.isAgentMessage()) {
            event.asAgentMessage().content().forEach(block -> System.out.print(block.text()));
        } else if (event.isAgentToolUse()) {
            System.out.println("\n[正在使用工具：" + event.asAgentToolUse().name() + "]");
        } else if (event.isSessionStatusIdle()) {
            break;
        } else if (event.isSessionError()) {
            System.out.println("\n[错误]");
            break;
        }
    }
}
```

### 重新连接与尾随

在会话中途重新连接时，应先列出过去事件以去重，然后再尾随实时事件。跨变体的 `id` 字段可从原始 `_json()` 值中读取：

```java
import com.anthropic.core.JsonValue;
import java.util.HashSet;
import java.util.Map;
import java.util.Optional;

try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
    // 流已打开并开始缓冲。在尾随实时事件之前先列出历史记录。
    var seenEventIds = new HashSet<String>();
    for (var past : client.beta().sessions().events().list(session.id()).autoPager()) {
        Optional<Map<String, JsonValue>> obj = past._json().orElseThrow().asObject();
        seenEventIds.add(obj.orElseThrow().get("id").asStringOrThrow());
    }

    // 尾随实时事件，跳过所有已见过的事件
    for (var event : (Iterable<StreamEvents>) stream.stream()::iterator) {
        Optional<Map<String, JsonValue>> obj = event._json().orElseThrow().asObject();
        if (!seenEventIds.add(obj.orElseThrow().get("id").asStringOrThrow())) continue;
        if (event.isAgentMessage()) {
            event.asAgentMessage().content().forEach(block -> System.out.print(block.text()));
        } else if (event.isSessionStatusIdle()) {
            break;
        }
    }
}
```

---

## 提供自定义工具结果

> 注意：Java 的 managed-agents 绑定中关于 `user.custom_tool_result` 的部分尚未在本技能或应用源码示例中记录。请参考 `shared/managed-agents-events.md` 了解数据格式，并参阅 `anthropic-java` 仓库获取对应的参数类型。

---

## 轮询事件

```java
for (var event : client.beta().sessions().events().list(session.id()).autoPager()) {
    System.out.println(event.type() + ": " + event);
}
```

---

## 上传文件

```java
import com.anthropic.models.beta.files.FileUploadParams;
import com.anthropic.models.beta.sessions.BetaManagedAgentsFileResourceParams;
import java.nio.file.Path;

var dataCsv = Path.of("data.csv");

var file = client.beta().files().upload(FileUploadParams.builder()
    .file(dataCsv)
    .build());
System.out.println("文件 ID: " + file.id());

// 挂载到会话中
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(agent.id())
    .environmentId(environment.id())
    .addResource(BetaManagedAgentsFileResourceParams.builder()
        .type(BetaManagedAgentsFileResourceParams.Type.FILE)
        .fileId(file.id())
        .mountPath("/workspace/data.csv")
        .build())
    .build());
```

### 在现有会话上添加和管理资源

```java
import com.anthropic.models.beta.sessions.resources.ResourceAddParams;
import com.anthropic.models.beta.sessions.resources.ResourceDeleteParams;

// 将额外的文件附加到一个已打开的会话
var resource = client.beta().sessions().resources().add(session.id(), ResourceAddParams.builder()
    .betaManagedAgentsFileResourceParams(BetaManagedAgentsFileResourceParams.builder()
        .type(BetaManagedAgentsFileResourceParams.Type.FILE)
        .fileId(file.id())
        .build())
    .build());
System.out.println(resource.id()); // "sesrsc_01ABC..."

// 列出会话中的资源——条目是联合类型
var listed = client.beta().sessions().resources().list(session.id());
for (var entry : listed.data()) {
    if (entry.isFile()) {
        var fileResource = entry.asFile();
        System.out.println(fileResource.id() + " " + fileResource.type());
    } else if (entry.isGitHubRepository()) {
        var repoResource = entry.asGitHubRepository();
        System.out.println(repoResource.id() + " " + repoResource.type());
    }
}

// 分离某个资源
client.beta().sessions().resources().delete(resource.id(), ResourceDeleteParams.builder()
    .sessionId(session.id())
    .build());
```

---

## 列出并下载会话文件

> 注意：在本技能或应用源代码示例中，尚未为 Java 记录列出和下载代理在会话期间所写文件的功能。请参阅 `shared/managed-agents-events.md` 以及 `anthropic-java` 仓库，以获取文件列表/下载的绑定信息。

---

## 会话管理

```java
// 列出环境
var environments = client.beta().environments().list();

// 获取特定环境
var env = client.beta().environments().retrieve(environment.id());

// 归档环境（只读，现有会话继续运行）
client.beta().environments().archive(environment.id());

// 删除环境（仅当没有会话引用该环境时）
client.beta().environments().delete(environment.id());

// 删除会话
client.beta().sessions().delete(session.id());
```

---

## MCP 服务器集成

```java
import com.anthropic.models.beta.agents.BetaManagedAgentsMcpToolsetParams;
import com.anthropic.models.beta.agents.BetaManagedAgentsUrlMcpServerParams;

// 代理声明 MCP 服务器（此处无认证——认证信息存于保险库中）
var agent = client.beta().agents().create(AgentCreateParams.builder()
    .name("GitHub 助手")
    .model("Claude Opus 5.5")
    .addMcpServer(BetaManagedAgentsUrlMcpServerParams.builder()
        .type(BetaManagedAgentsUrlMcpServerParams.Type.URL)
        .name("github")
        .url("https://api.githubcopilot.com/mcp/")
        .build())
    .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
        .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
        .build())
    .addTool(BetaManagedAgentsMcpToolsetParams.builder()
        .type(BetaManagedAgentsMcpToolsetParams.Type.MCP_TOOLSET)
        .mcpServerName("github")
        .build())
    .build());

// 会话附加包含这些MCP服务器URL凭据的保险库
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(BetaManagedAgentsAgentParams.builder()
        .type(BetaManagedAgentsAgentParams.Type.AGENT)
        .id(agent.id())
        .version(agent.version())
        .build())
    .environmentId(environment.id())
    .addVaultId(vault.id())
    .build());
```

有关创建保险库和添加凭据，请参阅`shared/managed-agents-tools.md`中的“保险库”部分。

---

## 保险库

```java
import com.anthropic.core.JsonValue;
import com.anthropic.models.beta.vaults.VaultCreateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthCreateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthRefreshParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthRefreshUpdateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthUpdateParams;
import com.anthropic.models.beta.vaults.credentials.CredentialCreateParams;
import com.anthropic.models.beta.vaults.credentials.CredentialUpdateParams;
import java.time.OffsetDateTime;

// 创建一个保险库
var vault = client.beta().vaults().create(VaultCreateParams.builder()
    .displayName("Alice")
    .metadata(VaultCreateParams.Metadata.builder()
        .putAdditionalProperty("external_user_id", JsonValue.from("usr_abc123"))
        .build())
    .build());
System.out.println(vault.id()); // "vlt_01ABC..."

// 添加一个OAuth凭据
var credential = client.beta().vaults().credentials().create(vault.id(),
    CredentialCreateParams.builder()
        .displayName("Alice的Slack")
        .auth(BetaManagedAgentsMcpOAuthCreateParams.builder()
            .type(BetaManagedAgentsMcpOAuthCreateParams.Type.MCP_OAUTH)
            .mcpServerUrl("https://mcp.slack.com/mcp")
            .accessToken("xoxp-...")
            .expiresAt(OffsetDateTime.parse("2026-04-15T00:00:00Z"))
            .refresh(BetaManagedAgentsMcpOAuthRefreshParams.builder()
                .tokenEndpoint("https://slack.com/api/oauth.v2.access")
                .clientId("1234567890.0987654321")
                .scope("channels:read chat:write")
                .refreshToken("xoxe-1-...")
                .clientSecretPostTokenEndpointAuth("abc123...")
                .build())
            .build())
        .build());

// 轮换凭据（例如，在令牌刷新后）
client.beta().vaults().credentials().update(credential.id(),
    CredentialUpdateParams.builder()
        .vaultId(vault.id())
        .auth(BetaManagedAgentsMcpOAuthUpdateParams.builder()
            .type(BetaManagedAgentsMcpOAuthUpdateParams.Type.MCP_OAUTH)
            .accessToken("xoxp-new-...")
            .expiresAt(OffsetDateTime.parse("2026-05-15T00:00:00Z"))
            .refresh(BetaManagedAgentsMcpOAuthRefreshUpdateParams.builder()
                .refreshToken("xoxe-1-new-...")
                .build())
            .build())
        .build());

// 归档一个保险库
client.beta().vaults().archive(vault.id());
```

---

## GitHub仓库集成

将GitHub仓库作为会话资源挂载（保险库中保存GitHub MCP凭据）：

```java
import com.anthropic.models.beta.sessions.BetaManagedAgentsGitHubRepositoryResourceParams;

var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(agent.id())
    .environmentId(environment.id())
    .addVaultId(vault.id())
    .addResource(BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/repo")
        .mountPath("/workspace/repo")
        .authorizationToken("ghp_your_github_token")
        .build())
    .build());
```

在同一会话中挂载多个仓库：

```java
import java.util.List;

var resources = List.of(
    BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/frontend")
        .mountPath("/workspace/frontend")
        .authorizationToken("ghp_your_github_token")
        .build(),
    BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/backend")
        .mountPath("/workspace/backend")
        .authorizationToken("ghp_your_github_token")
        .build());
```

轮换仓库的授权令牌：

```java
import com.anthropic.models.beta.sessions.resources.ResourceUpdateParams;

var listed = client.beta().sessions().resources().list(session.id());
var repoResourceId = listed.data().get(0).asGitHubRepository().id();

client.beta().sessions().resources().update(repoResourceId, ResourceUpdateParams.builder()
    .sessionId(session.id())
    .authorizationToken("ghp_your_new_github_token")
    .build());
```
