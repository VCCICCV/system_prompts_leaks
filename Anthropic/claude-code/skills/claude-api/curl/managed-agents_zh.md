# 可管理代理 - cURL / 原始 HTTP

当用户需要原始 HTTP 请求或在没有 SDK 的情况下工作时，请使用这些示例。

## 设置

```bash
export ANTHROPIC_API_KEY="您的 API 密钥"

# 公共请求头
HEADERS=(
  -H "Content-Type: application/json"
  -H "x-api-key: $ANTHROPIC_API_KEY"
  -H "anthropic-version: 2023-06-01"
  -H "anthropic-beta: managed-agents-2026-04-01"
)
```

---

## 创建环境

```bash
curl -X POST https://api.anthropic.com/v1/environments \
  "${HEADERS[@]}" \
  -d '{
    "name": "my-dev-env",
    "config": {
      "type": "cloud",
      "networking": { "type": "unrestricted" }
    }
  }'
```

### 使用受限网络

```bash
curl -X POST https://api.anthropic.com/v1/environments \
  "${HEADERS[@]}" \
  -d '{
    "name": "restricted-env",
    "config": {
      "type": "cloud",
      "networking": {
        "type": "limited",
        "allow_package_managers": true,
        "allow_mcp_servers": true,
        "allowed_hosts": ["api.example.com"]
      }
    }
  }'
```

---

## 创建代理（必需的第一步）

> 警告：**不存在内联代理配置。** 在 `managed-agents-2026-04-01` 下，`model`/`system`/`tools` 是 `POST /v1/agents` 的顶级字段，而不是会话的字段。请务必先创建代理——会话仅接受 `"agent": {"type": "agent", "id": "..."}`。

### 最小化配置

```bash
# 1. 创建代理
curl -X POST https://api.anthropic.com/v1/agents \
  "${HEADERS[@]}" \
  -d '{
    "name": "编程助手",
    "model": "claude-opus-5-5",
    "tools": [{ "type": "agent_toolset_20260401" }]
  }'
# -> { "id": "agent_abc123", ... }

# 2. 启动会话
curl -X POST https://api.anthropic.com/v1/sessions \
  "${HEADERS[@]}" \
  -d '{
    "agent": { "type": "agent", "id": "agent_abc123", "version": 1 },
    "environment_id": "env_abc123"
  }'
# -> { "id": "sesn_abc123", ... }
# 跟踪链接：https://platform.claude.com/workspaces/default/sessions/sesn_abc123 （如果 API 密钥不在默认工作区，请将 'default' 替换为您的工作区 ID）
```

### 包含系统提示、自定义工具和 GitHub 仓库```bash
# 1. 创建代理
curl -X POST https://api.anthropic.com/v1/agents \
  "${HEADERS[@]}" \
  -d '{
    "name": "代码评审员",
    "model": "claude-opus-5-5",
    "system": "您是一位资深的代码评审员。请做到全面且富有建设性。",
    "tools": [
      { "type": "agent_toolset_20260401" },
      {
        "type": "custom",
        "name": "run_linter",
        "description": "对文件运行项目代码检查工具",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": { "type": "string", "description": "待检查的文件路径" }
          },
          "required": ["file_path"]
        }
      }
    ]
  }'

# 2. 启动一个挂载了代码库的会话
curl -X POST https://api.anthropic.com/v1/sessions \
  "${HEADERS[@]}" \
  -d '{
    "agent": { "type": "agent", "id": "agent_abc123", "version": 1 },
    "environment_id": "env_abc123",
    "title": "代码评审会话",
    "resources": [
      {
        "type": "github_repository",
        "url": "https://github.com/owner/repo",
        "mount_path": "/workspace/repo",
        "authorization_token": "ghp_...",
        "branch": "feature-branch"
      }
    ]
  }'
```### 使用会话预算

```bash
# 创建一个硬性支出上限为25.00美元的会话（按标价计算；仅支持美元；仅限创建）。
# 金额以最小单位（美分）表示，采用整数字符串格式：“2500”即25.00美元
curl -X POST https://api.anthropic.com/v1/sessions \
  "${HEADERS[@]}" \
  -d '{
    "agent": { "type": "agent", "id": "agent_abc123" },
    "environment_id": "env_abc123",
    "budget": {
      "type": "limit",
      "max_list_cost": { "amount": "2500", "currency": "USD" }
    }
  }'

# 更改上限——可以调高或调低，但必须高于已消耗的标价成本。
# 成功更新后，因达到预算而暂停的工作将恢复执行。
curl -X POST https://api.anthropic.com/v1/sessions/$SESSION_ID \
  "${HEADERS[@]}" \
  -d '{ "budget": { "type": "limit", "max_list_cost": { "amount": "4000", "currency": "USD" } } }'
```# 完全移除预算上限——单向操作；已移除的预算无法再次添加
curl -X POST https://api.anthropic.com/v1/sessions/$SESSION_ID \
  "${HEADERS[@]}" \
  -d '{ "budget": null }'
```

有关列表成本构成、预算上限处的结算事件白名单以及多智能体语义，请参阅 `shared/managed-agents-core.md` 中的“会话预算”部分。

---

## 发送用户消息

```bash
curl -X POST https://api.anthropic.com/v1/sessions/$SESSION_ID/events \
  "${HEADERS[@]}" \
  -d '{
    "events": [
      {
        "type": "user.message",
        "content": [{ "type": "text", "text": "请审查认证模块是否存在安全问题" }]
      }
    ]
  }'
```

---

## 流式传输事件（SSE）

```bash
curl -N https://api.anthropic.com/v1/sessions/$SESSION_ID/events/stream \
  "${HEADERS[@]}"
```

响应格式：

```
event: session.status_running
data: {"type":"session.status_running","id":"sevt_...","processed_at":"..."}

event: agent.message
data: {"type":"agent.message","id":"sevt_...","content":[{"type":"text","text":"我会进行审查..."}],"processed_at":"..."}

event: session.status_idle
data: {"type":"session.status_idle","id":"sevt_...","processed_at":"..."}
```

---

## 轮询事件

```bash
# 获取所有事件
curl https://api.anthropic.com/v1/sessions/$SESSION_ID/events \
  "${HEADERS[@]}"

# 分页获取下一页事件
curl "https://api.anthropic.com/v1/sessions/$SESSION_ID/events?page=page_abc123" \
  "${HEADERS[@]}"
```

---

## 提供自定义工具结果

当智能体调用自定义工具时，将结果返回：

```bash
curl -X POST https://api.anthropic.com/v1/sessions/$SESSION_ID/events \
  "${HEADERS[@]}" \
  -d '{
    "events": [
      {
        "type": "user.custom_tool_result",
        "custom_tool_use_id": "sevt_abc123",
        "content": [{ "type": "text", "text": "未发现任何代码质量问题。" }]
      }
    ]
  }'
```

---

## 中断正在运行的会话

```bash
curl -X POST https://api.anthropic.com/v1/sessions/$SESSION_ID/events \
  "${HEADERS[@]}" \
  -d '{
    "events": [
      {
        "type": "user.interrupt"
      }
    ]
  }'
```

---

## 获取会话详情

```bash
curl https://api.anthropic.com/v1/sessions/$SESSION_ID \
  "${HEADERS[@]}"
```

---

## 列出所有会话

```bash
curl https://api.anthropic.com/v1/sessions \
  "${HEADERS[@]}"
```

---

## 删除会话

```bash
curl -X DELETE https://api.anthropic.com/v1/sessions/$SESSION_ID \
  "${HEADERS[@]}"
```

---

## 上传文件

```bash
curl -X POST https://api.anthropic.com/v1/files \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -F "file=@path/to/file.txt" \
  -F "purpose=agent"
```

---

## 列出并下载会话文件

列出智能体在会话期间写入 `/mnt/session/outputs/` 的文件，并将其下载。

```bash
# 列出与某会话关联的文件
curl "https://api.anthropic.com/v1/files?scope_id=$SESSION_ID" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01"

# 下载特定文件
curl "https://api.anthropic.com/v1/files/$FILE_ID/content" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -o downloaded_file.txt
```

---

## 列出所有智能体

```bash
curl https://api.anthropic.com/v1/agents \
  "${HEADERS[@]}"
```

---

## MCP 服务器集成

```bash
# 1. 智能体声明 MCP 服务器（此处无需认证——认证信息存放在保险库中）
curl -X POST https://api.anthropic.com/v1/agents \
  "${HEADERS[@]}" \
  -d '{
    "name": "MCP 智能体",
    "model": "claude-opus-5-5",
    "mcp_servers": [
      { "type": "url", "name": "my-tools", "url": "https://my-mcp-server.example.com/sse" }
    ],
    "tools": [
      { "type": "agent_toolset_20260401" },
      { "type": "mcp_toolset", "mcp_server_name": "my-tools" }
    ]
  }'

# 2. 会话附加包含该 MCP 服务器 URL 凭证的保险库
curl -X POST https://api.anthropic.com/v1/sessions \
  "${HEADERS[@]}" \
  -d '{
    "agent": "agent_abc123",
    "environment_id": "env_abc123",
    "vault_ids": ["vlt_abc123"]
  }'
```

有关创建保险库及添加凭证的信息，请参阅 `shared/managed-agents-tools.md` 中的“保险库”章节。

---

## 工具配置

```bash
curl -X POST https://api.anthropic.com/v1/agents \
  "${HEADERS[@]}" \
  -d '{
    "name": "受限智能体",
    "model": "claude-opus-5-5",
    "tools": [
      {
        "type": "agent_toolset_20260401",
        "default_config": { "enabled": true },
        "configs": [
          { "name": "bash", "enabled": false }
        ]
      }
    ]
  }'
```