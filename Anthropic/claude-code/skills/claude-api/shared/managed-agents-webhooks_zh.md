# 托管代理 - Webhook

当托管代理资源的状态发生变化时，Anthropic 可以向您的 HTTPS 端点发送 POST 请求——这是保持 SSE 流或轮询的一种替代方案。有效载荷是**精简的**（仅包含事件类型和资源 ID）；收到后，请获取该资源以了解其当前状态。每次交付都经过 HMAC 签名。

> **方向很重要。** 本页面介绍的是 *Anthropic -> 您* 关于会话/保险库状态的通知。它**不**涉及由第三方触发会话的 webhook（例如，调用 `sessions.create()` 的 GitHub 推送处理器）——那属于您端的普通应用代码，没有 Anthropic 特有的通信格式。

---

## 注册端点（仅限控制台）

在控制台中，前往 **管理 -> Webhook**。目前尚无用于管理端点的程序化 API。在同一页面上还支持密钥轮换。

| 字段 | 约束 |
|---|---|
| URL | 使用 HTTPS 协议，端口为 443，主机名需可公开解析 |
| 事件类型 | 按 `data.type` 订阅——端点仅接收其所订阅的事件类型 |
| 签名密钥 | 以 `whsec_` 为前缀，长度 32 字节，**创建时仅显示一次**——请妥善保管 |

---

## 验证签名

每次交付都会携带 `webhook-id`、`webhook-timestamp` 和 `webhook-signature` 这三个请求头。**请使用 SDK 中的 `client.beta.webhooks.unwrap()` 方法**——它会验证签名、拒绝超过约 5 分钟的旧有效载荷，并返回解析后的事件。该方法会从 `ANTHROPIC_WEBHOOK_SIGNING_KEY` 环境变量中读取 `whsec_` 密钥。请原封不动地传递这些请求头，不要自行实现针对单个 `X-Webhook-Signature` 头的校验逻辑，因为那并非标准的通信格式。

```python
import anthropic
from flask import Flask, request

client = anthropic.Anthropic()  # 从环境变量中读取 ANTHROPIC_WEBHOOK_SIGNING_KEY
app = Flask(__name__)


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        event = client.beta.webhooks.unwrap(
            request.get_data(as_text=True),
            headers=dict(request.headers),
        )
    except Exception:
        return "签名无效", 400

    if event.id in seen_event_ids:  # 去重重试——ID 是针对每个事件的，而非每次交付的
        return "", 204
    seen_event_ids.add(event.id)

    match event.data.type:
        case "session.status_idled":
            session = client.beta.sessions.retrieve(event.data.id)
            notify_user(session)
        case "vault_credential.refresh_failed":
            alert_oncall(event.data.id)

    return "", 204
```

请将**原始请求体**传递给 `unwrap()` 方法——那些会对 JSON 再序列化的框架（如 Express 的 `.json()` 或 Flask 的 `.get_json()`）会改变字节内容，从而导致 MAC 校验失败。对于其他语言，请在 SDK 仓库的 `shared/live-sources.md` 文件中查找 `beta.webhooks.unwrap` 的绑定；切勿自行实现校验逻辑。

---

## 有效载荷封装结构

```json
{
  "type": "event",
  "id": "whe_9d5c1f7e...",
  "created_at": "2026-03-18T14:05:22Z",
  "data": {
    "type": "session.status_idled",
    "id": "session_01XYZ...",
    "organization_id": "8a3d2f1e-...",
    "workspace_id": "c7b0e4d9-..."
  }
}
```

根据 `data.type` 进行分支处理，通过 `data.id` 获取相应资源，并返回任意 **2xx** 状态码以确认已接收。`created_at` 表示*事件发生的时间*，而非尝试交付的时间——尝试交付的时间则由 `webhook-timestamp` 请求头记录（参见“交付行为”部分）。顶层的 `id` 与 `webhook-id` 请求头中的值相同，且它是针对每个事件的，而不是每次交付的——每次重试都会携带相同的 ID，请据此进行去重。

---

## 支持的 `data.type` 值

| `data.type` | 触发时机 |
|---|---|
| `session.status_scheduled` | 会话已创建并可接收事件 |
| `session.status_run_started` | 代理执行已启动（每次状态变为“running”时触发） |
| `session.status_idled` | 代理处于等待输入状态（工具审批、自定义工具结果或下一条消息），或因达到会话预算而暂停。Webhook 负载较轻——仅列出会话的事件，并检查最新的 `session.status_idle` 事件中的 `stop_reason`（会话对象本身没有 `stop_reason` 字段）：若其值为 `budget_reached`，则后续的 `user.message` 事件将返回 400 错误，只有通过修改或移除预算才能恢复会话（参见 `shared/managed-agents-core.md` § 会话预算）。 |
| `session.status_rescheduled` | 发生了临时性错误；会话正在自动重试 |
| `session.status_terminated` | 会话结束——**无论正常完成还是发生错误**，并非仅在出错时触发 |
| `session.thread_created` | 多代理场景：协调器开启了新的子代理线程，或正在调用会话的顾问（参见 `shared/managed-agents-multiagent.md` → 顾问） |
| `session.thread_idled` | 仅适用于子线程：子代理线程正在等待输入，或因会话达到预算上限而暂停。当整个会话因预算上限而暂停时，也会触发 `session.status_idled` Webhook，且流中的 `session.status_idle` 事件会携带 `stop_reason: budget_reached`——除非另有线程正在等待工具调用，此时该线程的优先级高于会话级别的预算上限（参见 `shared/managed-agents-core.md` § 会话预算）。 |
| `session.thread_terminated` | 线程结束——子线程已完成任务，或该线程已被归档。**仅适用于子线程**；主线程的结束会以 `session.status_terminated` 的形式体现。 |
| `session.outcome_evaluation_ended` | 结果评分器完成了一轮评估 |
| `session.updated` | 会话属性发生变化（名称、配置等） |
| `session.deleted` | 会话被永久删除——不再有可获取的对象；应将该事件本身视为最终状态。 |
| `vault.archived` | 秘钥库被归档 |
| `vault.created` | 秘钥库被创建 |
| `vault.deleted` | 秘钥库被删除——同时还会触发与之关联凭据的 `vault_credential.deleted` 事件。不再有可获取的对象；应将该事件本身视为最终状态。 |
| `vault_credential.archived` | 凭据被归档，无论是直接归档还是因秘钥库归档而间接归档 |
| `vault_credential.created` | 秘钥库凭据被创建 |
| `vault_credential.deleted` | 凭据被删除，无论是直接删除还是因秘钥库删除而间接删除。不再有可获取的对象；应将该事件本身视为最终状态。 |
| `vault_credential.refresh_failed` | MCP OAuth 秘钥库凭据刷新失败 |
| `agent.created` | 代理被创建 |
| `agent.updated` | 新版本代理发布。未生成新版本的更新不会触发此事件。 |
| `agent.archived` | 代理被归档 |
| `agent.deleted` | 代理被永久删除——不再有可获取的对象；应将该事件本身视为最终状态。 |
| `deployment.created` | 计划部署被创建 |
| `deployment.updated` | 部署属性发生变化（例如调度被编辑） |
| `deployment.paused` | 部署被暂停——可由用户请求暂停，也可在计划运行因**不可恢复**的错误（代理已归档、环境缺失）而失败时自动暂停。可恢复的错误，包括速率限制，**不会**导致自动暂停。 |
| `deployment.unpaused` | 部署解除暂停，调度恢复正常 |
| `deployment.archived` | 部署被归档——可直接归档，或因代理归档/删除而间接归档 |
| `deployment.deleted` | 部署被永久删除——不再有可获取的对象；应将该事件本身视为最终状态。 |
| `deployment_run.started` | 一次**计划**运行已开始。手动运行不会触发 `deployment_run.*` 事件。 |
| `deployment_run.succeeded` | 计划运行已创建会话。其 `data.id`（运行 ID）与 `.started` 事件相同——可通过部署运行的 `session_id` 获取会话信息，然后订阅会话事件以跟踪工作进展。 |
| `deployment_run.failed` | 计划运行未能创建会话。其 `data.id` 与 `.started` 事件相同——可通过部署运行获取 `error.type` 和 `error.message`。 |
| `environment.created` | 环境被创建 |
| `environment.updated` | 环境至少有一个字段发生了变化。无实际变更的更新不会触发任何事件。 |
| `environment.archived` | 环境被归档。对已归档环境再次归档不会触发任何事件。 |
| `environment.deleted` | 环境被删除，包括对已归档环境的删除。不再有可获取的对象；应将该事件本身视为最终状态。 |
| `memory_store.created` | 内存存储被创建——由您创建，或由 Anthropic 运营的流程克隆了您的某个存储 |
| `memory_store.archived` | 内存存储被归档。对已归档存储再次归档不会触发任何事件。 |
| `memory_store.deleted` | 内存存储被删除，包括对已归档存储的删除。删除操作会级联至其下的所有记忆和版本，**不**会针对每条记忆单独触发事件——仅此单一事件即为信号。不再有可获取的对象；应将其视为最终状态。> **特意未提供 `memory_store.updated` 事件。** 单个记忆及其版本均不会发出任何 Webhook 事件，环境的自托管工作项亦是如此。如需跟踪每条记忆的变化，请轮询记忆版本相关端点（参见 `shared/managed-agents-memory.md`）。

> 这些是 **Webhook** 的 `data.type` 值——与 SSE 事件类型（如 `session.status_idle`、`span.outcome_evaluation_end` 等，详见 `shared/managed-agents-events.md`）属于不同的命名空间。请勿在 Webhook 处理程序中复用 SSE 常量。

---

## 投递行为与注意事项

- **重复投递。** 某个端点可能会多次收到同一事件；每次尝试都会携带相同的顶级 `event.id`（即 `webhook-id` 头）。请据此进行去重。
- **订阅范围。** 事件仅会送达那些在**事件发出时**已订阅其类型的端点。若在事件发出时没有任何端点订阅，则该事件将永远不会被投递；后续再订阅也不会补发——请务必在需要某类事件之前就完成订阅。
- **无顺序保证。** 事件的投递顺序不等于其发生顺序：`session.status_idled` 可能先于 `session.outcome_evaluation_ended` 到达，且针对同一资源的 `.deleted` 事件也可能先于 `.archived` 到达。**应根据获取到的资源状态来驱动应用逻辑，而非依赖事件到达的先后顺序。**
- **重试机制：每个事件对每个端点最多重试三次**，每次重试间隔采用抖动的指数退避策略，范围为 5 至 120 秒。触发自动禁用的响应将不再重试。**最后一次重试失败后，事件将被丢弃**——不会进入队列，且不会有任何丢失的提示信息。Webhook 并非持久化日志；若必须记录每一次状态变化，请通过列出或获取资源来实现状态同步。
- **每次重试时都会重新打上 `webhook-timestamp` 时间戳**，因此重试不会导致 SDK 的五分钟新鲜度检查失败。该时间戳记录的是*投递尝试*的时间；事件的实际发生时间请使用负载中的 `created_at` 字段。
- **自动禁用：三种触发条件**，每种都会设置 `disabled_reason`，且均可在控制台中解除：
  - 返回 `3xx` 状态码的响应。系统不会跟随重定向；首次尝试即触发禁用，原因为：“自动禁用：端点 URL 返回了重定向（3xx）”。
  - 在连接时，URL 解析出非公网 IP 地址。立即禁用，原因为：“自动禁用：端点 URL 解析出了无效地址”。
  - 长时间持续性投递失败。原因为：“因持续投递失败而自动禁用”。**触发条件是持续时间，而非投递次数**——只要有一次 `2xx` 响应，计时窗口就会重置，因此单次异常不会导致端点被禁用。
- **负载精简属设计使然。** 不要在 Webhook 有效载荷中期望获得 `stop_reason`（可查询会话的事件列表以获取该信息——会话对象本身并无 `stop_reason` 字段）、`outcome_evaluations`、凭据密钥等信息；请直接获取相关资源。