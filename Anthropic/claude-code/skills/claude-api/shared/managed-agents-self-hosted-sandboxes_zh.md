# 受管代理 - 自托管沙箱

当 `config.type: "self_hosted"` 时，**代理循环仍保留在 Anthropic 的编排层**，但 **工具执行会转移到您控制的基础设施中**——bash 脚本、文件操作和代码都在您的容器内运行，因此文件系统内容以及沙箱的网络出站流量都不会离开您的环境。（`web_search` 和 `web_fetch` 是例外：在这两种环境中它们都运行在 Anthropic 的服务器上——可通过代理工具集中的 `allowed_domains` 和 `blocked_domains` 来限制它们，详见 `shared/managed-agents-tools.md` § 网络搜索与网页抓取设置。）工具的输入和输出仍然会流向 Anthropic 的控制平面，以便模型能够看到结果；代理的技能以及任何附加的记忆存储的内容都由 Anthropic 存储，并在会话期间复制到您的沙箱中（记忆变化会同步回传——参见 § 记忆存储）。相比之下，在 `config.type: "cloud"` 模式下，容器由 Anthropic 运行。连接方式为 **仅出站**：您的工作进程会轮询 Anthropic 的任务队列，而 Anthropic 绝不会主动接入您的网络。

## 流程

```
1. 创建环境：      config: {type: "self_hosted"}        -> env_...
2. 生成环境密钥（在控制台的环境页面）   -> sk-ant-oat01-...  作为 ANTHROPIC_ENVIRONMENT_KEY
3. 运行工作进程：            EnvironmentWorker.run()  或  ant beta:worker poll
4. 会话引用       environment_id=env_...  与云端环境完全一致
```

## 创建环境

```python
client = anthropic.Anthropic()

environment = client.beta.environments.create(
    name="自托管", config={"type": "self_hosted"}
)
```

`{"type": "self_hosted"}` 就是全部配置——没有池、容量或网络相关的子字段；这些都由您在本地进行管理。

## 运行工作进程——SDK（主路径）

`EnvironmentWorker` 包装了轮询 → 分配 → 执行工具的循环。`.run()` 是常驻运行的循环（持续运行直到被取消）。`.handle_item()` / `.handleItem()` / `.HandleItem()` 用于处理 **已领取的单个** 工作项，无需轮询——ID 会回退到 `ANTHROPIC_WORK_ID` / `ANTHROPIC_ENVIRONMENT_ID` / `ANTHROPIC_SESSION_ID`，这是工作进程自身 `environment_key` 的密钥，也是 `ANTHROPIC_ENVIRONMENT_KEY` 以及每个会话的 `ANTHROPIC_WORK_SECRET` 的密钥，因此在 `ant beta:worker poll --on-work` 容器中无需任何参数。它会自行忽略（并强制停止）非会话类的工作项。没有 `run_one()` 方法；领取工作项的操作由 `.run()` 或中间层的轮询器完成（见下文）。

**Python - 常驻运行：**

```python
import asyncio
import contextlib
import os
import signal
from anthropic import AsyncAnthropic
from anthropic.lib.environments import EnvironmentWorker


async def main() -> None:
    environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
    environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
    async with AsyncAnthropic(auth_token=environment_key) as client:
        worker = EnvironmentWorker(
            client,
            environment_id=environment_id,
            environment_key=environment_key,
            workdir="/workspace",
        )
        task = asyncio.create_task(worker.run())
        # 取消任务（不终止进程）：工作进程会停止当前正在处理的工作项，并在退出前上传修改后的记忆文件。
        loop = asyncio.get_running_loop()
        for signum in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(signum, task.cancel)
        with contextlib.suppress(asyncio.CancelledError):
            await task


asyncio.run(main())
```

**TypeScript - 常驻运行：**

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";

const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
const client = new Anthropic({ authToken: environmentKey });
const ctrl = new AbortController();
process.once("SIGTERM", () => ctrl.abort());
process.once("SIGINT", () => ctrl.abort());

await new EnvironmentWorker({
  client,
  environmentId,
  environmentKey,
  workdir: "/workspace",
  signal: ctrl.signal
}).run();
```

**自定义工具。** `EnvironmentWorker` 默认运行内置工具集。要添加或替换工具，请使用 `AgentToolContext(workdir=, client=, session_id=)` 配合 `beta_agent_toolset(env)` / `betaAgentToolset(env)`，并将得到的工具传递给底层的 `tool_runner()`。附加到代理的技能会在工具调用开始前下载到 `{workdir}/skills/<name>/` 目录中（当提供 `client` 和 `session_id` 时，`AgentToolContext` 会自动处理此操作）。通过 CLI 和 SDK 下载的技能文件会被自动设置为可执行权限；如果您自行实现技能下载，则需要手动设置文件权限。

> **运行时依赖：** SDK 辅助工具要求 `/bin/bash` 必须位于该确切路径（不通过 `PATH` 查找）。TypeScript SDK 还要求 `PATH` 中存在 `unzip` 和 `tar`，并且 Node.js 版本需为 22 或更高；Python 和 Go 则使用各自的标准库进行归档解压。内存存储还需要一个 POSIX 兼容的主机（Linux 或 macOS，不支持 Windows，因为工作进程会以 `O_NOFOLLOW` 打开内存文件），且 `/mnt/memory` 目录可写——参见 § 内存存储。

**文件工具的沙箱限制。** `AgentToolContext` 将 `read`/`write`/`edit`/`glob`/`grep` 的操作范围限制在工作目录及 `allowed_roots`（`allowedRoots` / `AllowedRoots`）内；`write` 和 `edit` 还会拒绝访问 `read_only_roots`（`readOnlyRoots` / `ReadOnlyRoots`）下的路径。`EnvironmentWorker` 会自动将会话的内存存储目录加入这些列表。这仅是对文件工具的约束，并不会限制 `bash` 的行为。旧选项 `unrestricted_paths` 已不再接受（传递该选项会引发错误），请改用 `allowed_roots` 添加目录。

## 运行工作进程——`ant` CLI（固定工具集）

`ant` CLI 提供了一个带有固定内置工具集（`bash`、`read`、`write`、`edit`、`glob`、`grep`）的工作进程。按照 `shared/anthropic-cli.md` 的说明进行安装，然后：

```sh
export ANTHROPIC_ENVIRONMENT_KEY=sk-ant-oat01-...
ant beta:worker poll --environment-id env_... --workdir /workspace
```

- `--workdir` 是工具操作的目录（默认为 `.`），工具调用会被限制在此目录内。
- `--environment-key` 会覆盖环境变量。
- `--on-work <script>` 会在每个工作项执行时运行您的脚本（例如为每个会话启动一个新的容器——参见下文的容器编排）。
- `--unrestricted-paths`、`--max-idle`（默认 `60s`）、`--log-format` 等选项——详见 `ant beta:worker poll --help`。
- 命令行参数会回 fall back 到环境变量（`ANTHROPIC_ENVIRONMENT_ID`、`ANTHROPIC_ENVIRONMENT_KEY`）。
- 在接收到 SIGTERM/SIGINT 信号后，会先清空正在处理的任务再正常退出。
- **固定工具集**——如需自定义工具，请使用上文的 SDK 工作进程。
- **不挂载内存存储。** 即使会话绑定了内存存储，代理仍会正常运行，但无法在存储的 `/mnt/memory/<store-name>/` 目录中找到任何数据，也不会有数据同步回存储。若需将 CLI 轮询器与内存存储结合使用，请在宿主机上保持 `ant beta:worker poll --on-work` 运行，并在每个会话的沙箱内运行 **SDK** 工作进程（`EnvironmentWorker.handle_item()`）——参见 § 内存存储 -> 每会话沙箱。

在 `--on-work` 容器内，以 `ant beta:worker run --workdir <dir>` 作为入口点运行（或者如果会话需要内存存储，则运行 SDK 工作进程）。

## 基于 Webhook 的唤醒机制（替代常驻模式）

注册 `session.status_run_started` 的 Webhook（参见 `shared/managed-agents-webhooks.md`），确认消息已成功送达后，使用轮询器 **清空** 队列（`drain=True` 会在队列为空时停止；`block_ms=None` 表示非阻塞；`auto_stop=False` 因为 `handle_item` 会自行强制结束当前任务），并将每个已领取的任务交给 `handle_item()` 处理。**不要在 HTTP 处理函数中 `await` 清空操作**——会话的运行时间可能超过 Webhook 的交付超时时间，因此应在确认消息送达后，将清空操作作为后台任务执行（使用 `asyncio.create_task`、分离的 Promise 或从 `context.Background()` 启动的 goroutine），并保持进程运行直到清空完成：

```python
import asyncio
import os
import anthropic

environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
client = anthropic.AsyncAnthropic(
    auth_token=environment_key,
)  # 从环境变量中读取 ANTHROPIC_WEBHOOK_SIGNING_KEY，用于解包 Webhook

async def handle(raw: bytes, headers: dict[str, str]) -> dict:
    event = client.beta.webhooks.unwrap(raw.decode(), headers=headers)
    if event.data.type != "session.status_run_started":
        return {"status": "ignored"}
    asyncio.create_task(drain())  # 如果你的框架可能会将其垃圾回收，请保留一个引用
    return {"status": "accepted"}


async def drain() -> None:
    async for work in client.beta.environments.work.poller(
        environment_id=environment_id,
        environment_key=environment_key,
        block_ms=None,
        reclaim_older_than_ms=2000,
        drain=True,
        auto_stop=False,
    ):
        await client.beta.environments.work.worker(workdir="/workspace").handle_item(
            work_id=work.id,
            environment_id=environment_id,
            session_id=work.data.id,
            environment_key=environment_key,
            work_secret=work.secret,  # 允许工作进程挂载会话的内存存储
        )
```

TypeScript：与 `client.beta.webhooks.unwrap(body, {headers})`、`client.beta.environments.work.poller({environmentId, environmentKey, blockMs: null, reclaimOlderThanMs: 2000, drain: true, autoStop: false})` 以及 `client.beta.environments.work.worker({workdir}).handleItem({workId, environmentId, sessionId, environmentKey, workSecret: work.secret})` 的接口形状相同。Go：也没有 `RunOne` 这一便捷方法——使用 `environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{EnvironmentID, EnvironmentKey, BlockMs, ReclaimOlderThanMs, Drain, AutoStop: param.NewOpt(false)})`，然后对 `poller.Next()` 返回的每个任务项调用 `worker.HandleItem(ctx, environments.HandleItemOptions{WorkID: item.ID, EnvironmentID: item.EnvironmentID, SessionID: item.Data.ID, EnvironmentKey, WorkSecret: item.Secret})`，并在 `context.Background()` 的协程中执行。务必始终传递任务项的 `secret`，否则带有内存存储的会话在领取时会失败。`handle_item` 本身会跳过非会话类型的任务，因此排水循环无需检查 `work.data.type`。

## 容器编排（中级）

`EnvironmentWorker.run()` 在同一进程中轮询并执行工具。若希望每个会话运行在**独立**的容器中，请在轻量级编排器中使用中级轮询器——Python 中为 `client.beta.environments.work.poller(environment_id=, environment_key=, drain=, block_ms=, reclaim_older_than_ms=, auto_stop=)`；TypeScript 中为 `@anthropic-ai/sdk/helpers/beta/environments` 提供的 `new WorkPoller({client, environmentId, environmentKey, autoStop})`——并对每个返回的任务项启动一个新容器，注入以下环境变量，其入口点运行 `ant beta:worker run` 或 `EnvironmentWorker(...).handle_item()`（如果会话附加了内存存储，则必须如此）。`block_ms` 取值范围为 1–999（或设置为 `None` 表示非阻塞）；`reclaim_older_than_ms` 用于回收被已死亡工作进程占用的任务；`drain` 在队列为空时停止；`auto_stop` 在迭代器退出后发送停止信号（当启动的容器负责处理停止调用时应设为 `False`）。Go：`environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{EnvironmentID, EnvironmentKey, BlockMs, ReclaimOlderThanMs, Drain, AutoStop: param.NewOpt(false)})`，配合 `poller.Next()` / `poller.Current()` / `poller.Err()` 使用。

| 环境变量 | 值 |
|---|---|
| `ANTHROPIC_SESSION_ID` | `work.data.id` |
| `ANTHROPIC_WORK_ID` | `work.id` |
| `ANTHROPIC_ENVIRONMENT_ID` | `work.environment_id` |
| `ANTHROPIC_ENVIRONMENT_KEY` | 直接传递 |
| `ANTHROPIC_BASE_URL` | 直接传递 |
| `ANTHROPIC_WORK_SECRET` | `work.secret`——这是工作进程内部挂载内存存储所需的会话级凭据。`ant beta:worker poll --on-work` 不会为子进程设置该变量；请从标准输入的任务 JSON 中读取（`jq -r '.secret // empty'`），并将其传递进去。仅注入到服务于该会话的沙盒中，切勿记录日志。

当你自行分发容器时，请跳过 `work.data.type != "session"` 的任务项（`handle_item` 会为你完成这一检查）。

## 内存存储

自托管环境中的会话附加内存存储的方式与云端会话完全相同——在创建会话时指定 `resources=[{"type": "memory_store", "memory_store_id": ..., "access": ...}]`，每会话最多可附加 8 个（参见 `shared/managed-agents-memory.md`）。区别在于*由谁来实现存储的挂载*：在云端，Anthropic 会挂载一个实时的 FUSE 文件系统；而在自托管环境中，**SDK 工作进程**（`EnvironmentWorker` 或其 `handle_item()` / `handleItem()` / `HandleItem()`）会下载一份工作副本并进行同步。这需要使用 Python、TypeScript 或 Go SDK；`ant` CLI 工作进程以及 C#/Java/PHP/Ruby SDK 均不支持挂载存储。此功能在 AWS 上的 Claude Platform 中不可用。

**工作进程在领取附有存储的会话任务时会执行以下操作：**

1. 将每个存储下载到其挂载路径 `/mnt/memory/` 下——路径基于存储名称生成，而非可配置字段（例如，名为“User Preferences”的存储会挂载到 `/mnt/memory/user-preferences/`）；该路径与云端会话使用的路径相同，且会话的系统提示会向代理描述该路径。使用任务项的会话级 `secret` 进行认证。
2. 将这些目录添加到文件工具的 `allowed_roots` 中，并将访问权限为 `read_only` 的存储加入 `read_only_roots`，以便代理可以使用常规的 `read`/`write`/`edit`/`glob`/`grep` 工具操作记忆内容。
3. 在工具调用后进行协调，同步间隔内最多执行一次（默认 15 秒）：远程更改会被写入磁盘，代理修改的文件会被上传。
4. 会话结束时：执行最终同步，等待最多 30 秒以完成未上传的更改，然后移除这些目录。如果工作进程在会话中途被**取消**，则会跳过最终同步，但仍会上传已修改的文件并移除目录；如果工作进程被**杀死**，则不会执行任何清理操作。

Anthropic 一侧的存储仍然是事实来源——版本管理、编辑和控制台查看/编辑等功能与云端会话一致，代理的内存读写也会作为普通工具事件出现在事件流中。由于同步是基于间隔的，一个自托管会话写入的更改只有在另一个正在运行的会话也完成同步后才会可见（通常不到一分钟）；而云端会话几乎可以立即看到彼此的更改。每个存储目录下都会有一个标记文件 `.anthropic-memory-store`——请勿动它；如果标记文件缺失或被修改，工作进程将拒绝同步该目录。

**准备宿主机。** 仅支持 POSIX 系统（Linux/macOS）；建议使用区分大小写的文件系统。在启动工作进程之前：

```bash
sudo mkdir -p /mnt/memory && sudo chown "$USER" /mnt/memory
```

请勿手动创建各存储目录——工作进程会在会话启动时创建每个存储的目录，**如果该路径下已有内容，则会拒绝该任务**，并在会话结束时将其删除。由此产生两条规则：(a) 同一台主机上不能同时挂载两个会话的同一存储（它们需要相同的路径）——请为每个会话提供独立的沙盒；(b) 请优雅地停止工作进程。`EnvironmentWorker` 不会安装信号处理器：请自行将 SIGTERM/SIGINT 绑定到取消操作（在 TypeScript 中调用 `signal.abort()`，在 Go 中取消上下文，在 Python 中取消执行 `run()` 或 `handle_item()` 的任务），发送 SIGTERM，并至少等待 30 秒后再进行强制终止。如果工作进程在清理前被杀死，请在下一个附加该存储的会话开始前手动移除 `/mnt/memory/` 下残留的目录——其中未同步的更改将会丢失。**每个会话一个沙箱**（参见§ 容器编排中的模式）自动满足规则 (a)。将 `ant beta:worker poll --on-work`（或 SDK 轮询器）保留在宿主机上；围绕 SDK 工作进程而非 `ant beta:worker run` 构建每个会话的镜像——其入口点会构造 `EnvironmentWorker` 并调用 `handle_item()`，该方法从 `ANTHROPIC_*` 环境变量中读取会话/工作/环境 ID，并从 `ANTHROPIC_WORK_SECRET` 中读取该会话的密钥（或者显式传递 `work_secret=` / `workSecret` / `WorkSecret`）。`--on-work` 不会为子进程脚本设置 `ANTHROPIC_WORK_SECRET`，因此需从标准输入的工作项 JSON 中读取：

```bash
#!/bin/bash
# spawn.sh - 每个已领取的工作项调用一次；工作项以 JSON 格式通过标准输入传入
ANTHROPIC_WORK_SECRET="$(jq -r '.secret // empty')"
export ANTHROPIC_WORK_SECRET
exec docker run --rm \
  -e ANTHROPIC_SESSION_ID -e ANTHROPIC_WORK_ID -e ANTHROPIC_ENVIRONMENT_ID \
  -e ANTHROPIC_ENVIRONMENT_KEY -e ANTHROPIC_BASE_URL -e ANTHROPIC_WORK_SECRET \
  my-sdk-worker-image
```

每个会话的入口点代码只有几行——无需额外参数，`handle_item()` 会读取传递过来的 `ANTHROPIC_*` 环境变量，包括 `ANTHROPIC_WORK_SECRET`；并绑定信号处理以支持取消操作，确保容器被停止时仍能上传数据：

```python
import asyncio, contextlib, os, signal
from anthropic import AsyncAnthropic
from anthropic.lib.environments import EnvironmentWorker


async def main() -> None:
    async with AsyncAnthropic(auth_token=os.environ["ANTHROPIC_ENVIRONMENT_KEY"]) as client:
        task = asyncio.create_task(EnvironmentWorker(client, workdir="/workspace").handle_item())
        loop = asyncio.get_running_loop()
        for signum in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(signum, task.cancel)
        with contextlib.suppress(asyncio.CancelledError):
            await task


asyncio.run(main())
```

TypeScript：`new EnvironmentWorker({ client, workdir: "/workspace", signal: controller.signal }).handleItem()`，并配合 `process.once("SIGTERM"/"SIGINT", () => controller.abort())`。Go：`signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`，然后调用 `environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{Workdir: "/workspace"}).HandleItem(ctx, environments.HandleItemOptions{})`。

镜像需要一个可写的 `/mnt/memory`；这些内存目录**无需**挂载到宿主机——工作进程会在沙箱退出前完成上传，而废弃的沙箱也不会留下任何需要清理的残留物。通过信号提前停止容器，入口点会将其转换为取消操作，而不是直接终止进程，从而保证上传能够顺利完成。

**同步配置**——`EnvironmentWorker` 提供两个选项（Python 中可通过构造函数或 `client.beta.environments.work.worker()` 工厂方法设置；TypeScript 中为选项对象；Go 中为 `environments.EnvironmentWorkerOptions`）：

| 选项 | Python / TypeScript / Go | 行为 |
|---|---|---|
| 同步间隔 | `memory_sync_interval`（秒）/ `memorySyncIntervalMs`（毫秒）/ `MemorySyncInterval`（持续时间） | 默认 15 秒，最小 5 秒。缩短间隔会缩小数据陈旧窗口，但会增加对内存存储的请求次数。设置为 `None` / `null` / 负值将持续时间**完全禁用内存支持**——既不下载也不同步存储，且即使系统提示中仍提及存储，附加了存储的会话也会在没有存储的情况下运行。仅在那些会话从不附加存储的工作进程中禁用。启用时，如果某个会话已附加存储但工作项中缺少 `secret`，该工作项将**失败**，而不会以无存储状态运行。 |
| 删除传播 | `memory_sync_deletes` / `memorySyncDeletes` / `MemorySyncDeletes` | `"enabled"`（默认——当后续同步确认文件确实已被删除时才从存储中移除）、`"log_only"`（执行相同检查，仅记录原本会删除的内容——用于在信任 `"enabled"` 前进行审计）、`"disabled"`（从不从存储中删除）。Go：`environments.MemorySyncDeletesEnabled`（零值）/ `LogOnly` / `Disabled`。上传和下载不受影响。 |

例如，每10秒同步一次，并且仅对可能的删除操作进行日志记录：Python `EnvironmentWorker(client, environment_id=..., environment_key=..., workdir="/workspace", memory_sync_interval=10, memory_sync_deletes="log_only")`；TypeScript `new EnvironmentWorker({ client, environmentId, environmentKey, workdir: "/workspace", memorySyncIntervalMs: 10_000, memorySyncDeletes: "log_only" })`；Go `environments.EnvironmentWorkerOptions{..., MemorySyncInterval: 10 * time.Second, MemorySyncDeletes: environments.MemorySyncDeletesLogOnly}`。

**只读存储与冲突。** 对于 `access: "read_only"`，`write`/`edit` 操作会拒绝对该目录下的任何更改（这些内存错误只会作为工具错误传递给代理），并且不会有任何数据上传；内存存储端点也会拒绝使用会话密钥进行的写入操作。本地的 `bash` 编辑不会被阻止——它们永远不会同步，下一次远程变更会覆盖这些本地修改。冲突时**以存储内容为准**：如果代理修改了某个文件，而该文件自上次同步以来也在远程发生了变化，那么在下一次同步时，工作进程会保留存储中的版本，覆盖本地文件，并记录一条警告信息——`write`/`edit` 操作仍然成功，且不会向代理返回任何错误；代理可以重新读取并再次应用。

**故障排除。** 挂载和后台同步失败会被**记录到日志中**，但不会上报给会话。如果在申领时无法挂载存储，工作进程会使该工作项失败——会话不会触发任何错误事件，并处于 `idle` 状态（停止原因为 `requires_action`）。

| 日志行 / 症状 | 原因 | 解决方法 |
|---|---|---|
| `the work item carried no sessions token`（Go：`ErrSessionMemoryNoToken`），工作项失败 | 每个会话的 `secret` 没有传递给工作进程——自托管环境中未为您的组织启用内存功能，或者您的启动脚本未将其传递过来 | 将 `ANTHROPIC_WORK_SECRET` 传递到沙箱中。如果进程内工作进程（poll 和 run 在同一进程中）仍然记录此错误，请联系支持团队 |
| `something already exists at the memory store's path` | 杀死工作进程后残留的目录 | 删除指定目录（未同步的编辑将丢失） |
| `cannot create the memory store's folder` + `the worker host must make this mount path writable` | 工作进程用户无法在 `/mnt/memory` 下创建目录 | `mkdir -p /mnt/memory && chown <worker-user> /mnt/memory` |
| 会话在申领后不久进入 `idle` 状态，显示 `requires_action`，无错误事件 | 工作进程因上述挂载错误使工作项失败 | 修复宿主机后发送 `user.interrupt`——工作将被重新加入队列，下次申领时会重试挂载 |

## 监控与控制

这些是**控制平面**调用——使用 `x-api-key` 进行认证（而非环境密钥）；需携带 `managed-agents-2026-04-01` 测试版头。请**从工作进程宿主机之外发起这些调用**——在工作进程宿主机上设置 `ANTHROPIC_API_KEY` 会将一个组织范围的凭据暴露给代理的工具调用。

| SDK (`client.beta.environments.work.*`) | REST | CLI | 返回值 |
|---|---|---|---|
| `stats(environment_id)` | `GET /v1/environments/{id}/work/stats` | `ant beta:environments:work stats` | `{type:"work_queue_stats", depth, pending, oldest_queued_at, workers_polling}` |
| `stop(work_id, environment_id=)` | `POST /v1/environments/{id}/work/{work_id}/stop` | `ant beta:environments:work stop` | `work.state` |

## 与 `cloud` 的区别| 关注点 | `cloud` | `self_hosted` |
|---|---|---|
| 容器生命周期、加固与网络 | Anthropic | **您** - 以非 root 用户运行，使用只读根文件系统，禁用不必要的权限；出站流量受限于您的 VPC/防火墙策略 - 唯独 `web_search` 和 `web_fetch` 无论如何都在 Anthropic 的服务器上执行（可通过 `allowed_domains` / `blocked_domains` 对每个工具进行限制）|
| `file` / `github_repository` 资源挂载 | Anthropic 在容器内自动挂载 | **您** - 通过 `sessions.create(metadata={...})` 传递资源指针，并由您的编排系统在调度前完成获取或克隆 |
| `memory_store` 资源 | Anthropic 挂载至 `/mnt/memory/<name>/`（实时 FUSE 挂载） | **由 SDK worker 支持**（Python / TypeScript / Go 的 `EnvironmentWorker`），该 worker 会将每个存储下载至 `/mnt/memory/<store-name>/` 并按设定的间隔同步 - 参见 § 内存存储。`ant` CLI worker 不支持挂载；C#、Java、PHP 和 Ruby SDK 中也不可用。`memory_store` 是自托管环境唯一接受的资源类型 - `file` 和 `github_repository` 仍会被拒绝，返回 400 错误：“环境 env_... 是自托管环境，不支持 `resources`。”（针对自托管环境的部署同样适用此规则；控制台部署表单不会为这些环境提供内存存储选项，请使用 API/SDK）。|
| Vault `environment_variable` 凭证 | 支持（在 Anthropic 管理的出站流量中替换） | **暂不支持** - 出站流量由您负责，因此无处替换密钥。请使用 MCP 凭证，或采用主机侧的自定义工具（参见 `shared/managed-agents-client-patterns.md` 模式 9）。|
| 内置工具 | 通过 `agent_toolset_20260401` 提供 | 由您的 worker 提供（`EnvironmentWorker` 默认配置 / `beta_agent_toolset(env)` / `ant` CLI 固定集合）。|
| 技能下载 | 自动下载 | `EnvironmentWorker` / `AgentToolContext` 会将技能下载至 `{workdir}/skills/`（需提供 `client` 和 `session_id`）。|
| AWS 上的 Claude Platform | 支持 | 支持 - worker 使用 AWS IAM（SigV4）或 AWS 控制台生成的 API 密钥进行身份验证（控制台生成的环境密钥无法用于 AWS 端点）；请将 `AnthropicSelfHostedEnvironmentAccess` 托管策略附加到 worker 的主体。**在自托管环境中，内存存储无法附加到会话**（创建会话时会被拒绝）；云端环境则可照常附加。|
| SDK worker 辅助工具 | 所有 SDK | **仅 Python、TypeScript 和 Go 支持**（Java、Ruby、PHP 和 C# 不提供 `EnvironmentWorker` 或轮询器）- 请使用这三种语言之一，或使用 `ant` CLI。|

## 凭证

| 凭证 | 格式 | 作用范围 |
|---|---|---|
| `ANTHROPIC_ENVIRONMENT_KEY` | `sk-ant-oat01-...` | 单个环境的工作队列。可在控制台中生成（“生成环境密钥”）。在客户端作为 `auth_token=` / `authToken` 传入，同时在 `EnvironmentWorker` 中作为 `environment_key=` / `environmentKey` 传入。应存储于密钥管理服务中，一旦泄露即需轮换。|
| `ANTHROPIC_WEBHOOK_SIGNING_KEY` | `whsec_...` | 用于 Webhook 签名验证（若使用 Webhook 触发机制）。SDK 会自动读取此环境变量，用于 `client.beta.webhooks.unwrap()`。|
| 工作项密钥（`ANTHROPIC_WORK_SECRET`） | 每会话专用，由 Anthropic 在工作项被领取时颁发 | 用于发布该会话的事件，并读写与其关联的内存存储。您无需生成它；进程内 worker 会从工作项中获取，而在“每个会话一个沙盒”的模式下，您需要自行将其传递至沙盒（或显式传入 `work_secret=` / `workSecret` / `WorkSecret`）。其使用方式与环境密钥相同：仅限于服务于该会话的沙盒内，切勿放入镜像、共享卷或日志中。|

## 安全责任 - 您需承担的部分容器加固；沙箱出站流量限制（无默认设置；服务器端的 `web_search` 和 `web_fetch` 仅受其 `allowed_domains` 和 `blocked_domains` 的约束）；`ANTHROPIC_ENVIRONMENT_KEY` 的保管与轮换；运行不可信代码时，每个信任边界对应一个工作空间和环境；工具进程采用最小权限原则；日志保留与脱敏处理。**Anthropic 无法**：快速撤销已泄露的环境密钥、验证您的镜像或供应链、在您的容器内对工具执行进行沙箱隔离，或在工具输出到达您的基础设施后强制实施日志保留。**内存存储**仍由 Anthropic 托管（并保留版本历史），但 `/mnt/memory/` 下的工作副本在会话期间归您所有：工作进程会在销毁时将其删除，被终止的工作进程则会将其遗留，而共享同一文件系统的不同会话之间的权限与隔离由您自行负责。只读存储可防止上传，但无法防止本地修改——`bash` 仍可更改本地副本（该会话中后续的工具调用将读取已更改的副本，直到该存储下次发生变更）；如果希望确保代理连其本地视图都不被篡改，请禁用 `bash` 或将该路径以只读方式挂载。完整检查清单请参阅 `shared/live-sources.md` 中的“自托管沙箱安全”页面。