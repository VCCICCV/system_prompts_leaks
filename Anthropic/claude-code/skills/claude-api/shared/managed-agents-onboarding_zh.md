# 托管代理——上线流程

> **通过 `/claude-api managed-agents-onboard` 调用？** 您来对地方了。请执行下方的访谈流程——不要将内容总结后反馈给用户，而是直接提问。

Claude 托管代理是一种托管型代理：Anthropic 负责运行代理循环，并为每个会话提供一个沙箱容器，代理的工具在其中执行（或者由您自己的工作进程运行，使用 `self_hosted` 环境——参见 `shared/managed-agents-self-hosted-sandboxes.md`）。您只需提供一个**代理配置**（工具、技能、模型、系统提示——可复用且版本化）和一个**环境配置**（沙箱——可在多个代理间复用），每次运行即为一个**会话**。

整个流程分为四个步骤——**描述 -> 代理 -> 环境 -> 会话**——与控制台快速入门的逻辑一致，也秉持同样的理念：**先价值，后凭据**。用户无需任何身份验证即可从想法过渡到可运行的会话；每项凭据仅在设计中明确其必要性时（§2）予以标注，并在会话设置阶段（§4）一次性收集，此时完成绑定（`sessions.create()`）并进行初步测试。请同时参阅 `shared/managed-agents-core.md`，其中对各项参数有详尽说明；本文档则作为访谈脚本。

---

## 1. 描述任务

**以一句简明扼要的引导语开场，提出一个开放式问题——切勿猜测或采用问卷式提问。** 请用您自己的话：

> 托管代理由 Anthropic 运营：代理循环、沙箱及基础设施均由我们负责，您只需定义代理本身。我们将分三步完成：首先是代理的定义，其次是其运行的环境，最后是实际的测试会话。那么，请描述您希望创建的代理——它应该实现什么功能？以及由什么触发（人、事件还是定时计划）？

请让用户完整作答，再开始任何配置操作。

## 2. 配置代理——提出建议，而非逐一询问

用户的描述已完成了访谈的核心工作。请据此起草代理配置，并**以提案形式呈现，同时在文中插入您的建议**——让用户针对一份具体的配置作出回应，而非回答一连串问题。如有确实遗漏之处，最多追加一次补充提问。当描述中出现相关线索时，请适时提出建议：

- **工具**——默认启用完整的预置工具集（`agent_toolset_20260401`：`bash`、`read`、`write`、`edit`、`glob`、`grep`、`web_fetch`、`web_search`）。对于任务中提及的任何第三方服务（如 GitHub、Linear、Slack 等），**建议接入 MCP 服务器**，并在提出建议时同步标注所需凭据（例如：“接入 Linear 的 MCP 后，启动时需提供 Linear API Token”），以便 §4 的身份验证环节只是例行手续，而非意外。凭据的具体收集则延至 §4 进行。自定义工具仅在用户自有应用必须响应调用时才添加——包括名称、描述及输入 Schema；处理代码由用户自行提供，我们不代为生成。
- **技能**——当任务涉及特定格式文件（如 XLSX、DOCX、PPTX 或 PDF）时，**建议启用**相应的预置技能；自定义技能则按 `skill_id` 添加（单个代理的预置与自定义技能总数上限为 20 项）。
- **成果——适用于所有有交付物的任务的默认触发方式。** 若任务会产生可检验的成果（如文档、报告、拉取请求或数据集），请根据描述拟定一份初始评估标准——明确且可独立评分的指标：例如，不应写“一份好的报告”，而应写成“一份包含 SKU 对应数值型 `price` 列的 CSV 文件”，并将该标准内嵌于配置中；框架会依据此标准进行评分与迭代（参见 `shared/managed-agents-outcomes.md`）。即使用户未提供评估标准，也不应因此跳过这一步——拟定标准正是您的职责；请将其标记为待调整的初始版本。只有当任务具有高度交互性（如聊天界面或需要人工介入的场景）时，才退而求其次，采用对话式触发。
- **可用资源**——本地磁盘上的代码库（`github_repository`：URL，可选 `mount_path` 和 `checkout`；凭据在 §4 提供），以及用于初始化的文件（通过 Files API 上传，格式为 `{type: "file", file_id, mount_path}`；只读），若任务中引用了这些资源。
- **模型**——默认使用 `claude-opus-5-5`；对于最复杂的长周期任务，则推荐 `claude-fable-5-1`（参见 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”部分）。

> 重要提示：**创建 PR 也需要 GitHub MCP 服务器**——`github_repository` 挂载仅限于文件系统。请在挂载目录中编辑文件，通过 `bash` 推送分支，然后使用 MCP 的 `create_pull_request` 工具打开 PR。

每个配置项的详细说明参见：`shared/managed-agents-tools.md`（工具集、MCP、自定义工具、技能），以及 `shared/managed-agents-environments.md`（仓库、文件）。

## 3. 环境

通常只需回答一个问题：

- **复用还是新建？** 环境是跨代理共享的——请先检查是否存在合适的现有环境。
- **网络设置**——默认允许无限制出站访问。只有当用户需要控制出站流量时才切换为 `limited` 模式；此时需设置 `allow_mcp_servers: true`，或在 `allowed_hosts` 中列出所有 MCP 服务器域名，否则相关工具会静默失败。
- **建议使用 `self_hosted`**：当出现以下情况时——工具必须运行在自有基础设施上、机密信息不能流出、或者需要云端容器中不存在的二进制文件/数据时（参见 `shared/managed-agents-self-hosted-sandboxes.md`；在 AWS 上的 Claude Platform 中，工作节点通过 IAM 而非环境密钥进行身份验证，且该平台上的会话无法附加内存存储）。否则应选择 `cloud`——对于简单任务，不要主动推荐自托管模式。

## 4. 会话——认证与测试运行

**认证在此阶段完成——在配置确定后，收集 §2 中标记的凭据：** 一个 Vault（已有或通过 `vaults.create()` 创建）+ 为 §2 中声明的每个 MCP 服务器调用一次 `vaults.credentials.create()`；为作业使用的 API 密钥创建 `environment_variable` 类型的凭据（在出站时被替换，沙箱中显示占位符）；以及为每个仓库挂载提供 `authorization_token`。凭据均为只写权限；MCP 凭据按 URL 匹配服务器并自动刷新。详情参见 `shared/managed-agents-tools.md` → Vaults。

**无声的可行性检查——在输出任何内容之前，请自行执行此步骤，并仅报告缺失的部分。** 按照作业条款逐条检查：每个动词都对应一个已启用的工具或 MCP 服务器（“打开 PR”指向 GitHub MCP，而不仅仅是挂载）；每个 MCP 服务器和仓库挂载都有来自认证步骤的凭据；根据网络设置，所有外部主机均可访问；作业引用的每个文件/仓库/数据集均已挂载；“完成”状态可被验证。如有缺失，请明确指出并解决——切勿输出一份已知资源不足的配置。

**启动方式——只能选择其一，不可同时使用。默认结果如下：**
- `user.define_outcome` + 评分标准——只要作业有交付物，就采用此默认方式（§2 中已起草评分标准）；框架会迭代并评分，直到满足评分标准为止。
- `user.message`——仅适用于真正的对话式会话。
- **是否为定时任务？** 如果是，则完全跳过每次会话的启动步骤——直接创建一个**部署**（通过 `deployments.create()` 并指定 `schedule` 和 `initial_events`）；每次触发都会自动创建会话。详情参见 `shared/managed-agents-scheduled-deployments.md`。

应在运行时代码中内置以下机制：会话创建会解析资源（挂载问题会在这一阶段暴露，而非等到令牌时），但不会自行 provision 沙箱；发送启动指令前务必先打开事件流；遇到 `session.status_terminated` 或 `session.status_idle` 且 `stop_reason` 不为 `requires_action` 时应中断——前者表示会话终止，后者则表示达到预算上限，但并不终止（只有更改或移除预算才能恢复会话）（参见 `shared/managed-agents-client-patterns.md` 模式 5）；用量信息记录在 `span.model_request_end` 中；产出物保存在 `/mnt/session/outputs/` 目录下（可通过 `files.list({scope_id: session.id, ...})` 获取）。

## 5. 集成——输出代码

从最后一步的答案直接进入代码编写，无需铺垫，也不必讲解设置与运行的区别；两段式结构清晰地体现了这一点。生成**两个明确分隔的代码块**：

**第1块——设置（文件 + `ant apply`；ID 将写入 `claude-lock.json`）。** 代理和环境都是版本化的定义——以文件形式编写，并通过 `ant apply` 同步（参见 `shared/anthropic-cli.md` → 版本化管理的代理资源）：

1. `agents/<name>.md`——YAML 前置元数据（包括 `name`、`model`、`tools`、`mcp_servers`、`skills`），Markdown 正文为系统提示；同时创建 `environments/<name>.yaml`。如果复用现有环境（§3），则无需编写环境文件（以免创建重复），并在后续命令中省略它，在需要指定环境的地方直接使用现有的 `env_...` ID（第2块中，部署文件的 `environment_id`）。
2. ```sh
   ant apply --dry-run -v agents/<name>.md environments/<name>.yaml   # 打印完整计划，包含所有字段，但不作任何变更
   ant apply agents/<name>.md environments/<name>.yaml                # 提示确认后执行创建操作；每次修改后重新运行以更新
   ```
   为刚刚编写的文件命名——切勿使用 `.` 或目录名，因为这会导致遍历整个目录，并可能将其他看起来像资源的文件也当作资源处理（例如 Claude Code 插件的 `agents/*.md` 和 `skills/*/SKILL.md`，这些文件用户可能从未阅读过）。如果没有终端（如编码代理的 Shell），第二条命令只会打印计划并退出；只有加上 `--yes` 参数才会真正执行，而这代表用户的批准，而非您的判断：请先向用户展示干运行计划，待其同意后再执行。如果计划会创建或更改您未编写的对象，或某个路径上存在您未编写的文件，应立即停止并询问；切勿擅自添加 `--force` 或 `--prune` 参数。
3. 将 `claude-lock.json` 与文件放在一起（如果是仓库，则两者一起提交）——其中保存了各资源的 ID，缺少该文件会导致下次 `ant apply` 创建重复资源。将第2块所需的 ID（代理、环境；定时任务场景中的部署）一次性复制到应用自身的配置或环境变量中——例如 `resources["./agents/<name>.md"].id` 等，键名为计划中打印的路径——这样运行时的应用就不依赖于锁文件。

如果 `ant` 缺失或版本低于 1.30.0（可通过 `ant --version` 查看），并且用户不在 AWS 上的 Claude Platform（见下文），请告知并提供安装或升级服务（参见 `shared/anthropic-cli.md` → 安装与认证）。在运行安装程序前务必征得用户同意，切勿静默回退至 SDK；只有在用户拒绝或无法安装时，才使用下方的 SDK 回退方案。

**SDK 回退方案——仅在用户要求时使用，且在 AWS 上的 Claude Platform 上为必需**：在该平台上，认证采用 SigV4 方式，而 `ant` CLI 无 SigV4 模式（请使用 `shared/claude-platform-on-aws.md` 中的平台客户端）。将这部分标注为 `# 一次性设置——运行一次，保存 ID`，并调用 `environments.create()` → `agents.create()`。

> 注意：**部署功能比 MA 其他功能更新。** 在输出 `ant beta:deployments ...` 或 `client.beta.deployments` / `client.beta.deployment_runs` 相关调用之前，请先确认用户安装的 CLI/SDK 是否支持这些功能（运行 `ant beta:deployments --help`；检查 `hasattr(client.beta, "deployments")`）。如果不支持，则需直接发起 HTTP 请求，调用 `POST /v1/deployments`，并附带 `managed-agents-2026-04-01` Beta 标头（使用 Bearer Token 认证时还需添加 `oauth-2025-04-20`），同时留下一条升级提示，说明哪些操作将来可以简化为 SDK 调用。

**如果是定时任务，部署即为设置，而非运行时操作。** 请在第1块中创建部署。使用 `ant apply` 时，编写 `deployments/<name>.md` 并将其加入同一 `ant apply` 命令中。该文件通过路径指定代理和环境（复用环境则使用 `env_...` ID）；前置元数据包括 `schedule` 及其余创建参数，Markdown 正文则作为 `user.message` 类型的启动消息（若采用 Outcome 启动，则将 `initial_events` 放在前置元数据中，正文留空，二者不可同时使用）。使用 SDK 时，在代理和环境 ID 确定后，调用 `deployments.create()` 并传入 `schedule` 和 `initial_events`。此时第2块**不是会话循环**——无需发送每轮的启动指令。而是输出一个手动触发接口（`POST /v1/deployments/{id}/run`），以便用户立即测试，而不必等待首次触发；该手动运行也可兼作冒烟测试，此外还提供一个获取辅助函数（获取最新的 `deployment_runs` 条目 -> `session_id` -> 控制台 URL + `files.list(scope_id=session_id)` 用于获取产出物）。**第2块 - 运行时（每次调用；对话和结果形态）。** 使用检测到的语言编写的 SDK 代码（Python/TS/cURL - SKILL.md -> 语言检测）；此处不要输出 shell 循环：

1. 从配置或环境变量中加载 `agent_id` 和 `env_id`（由第1块放置的位置）
2. 调用 `sessions.create(agent=AGENT_ID, environment_id=ENV_ID, resources=[...], vault_ids=[...])`，然后打印控制台 URL，以便用户实时查看：`https://platform.claude.com/workspaces/default/sessions/{session.id}`（将 `default` 替换为用户的工作空间 slug）
3. **当任务依赖于 MCP 服务器、凭据或受限制的主机时进行冒烟测试**——这些失败不会在 `sessions.create()` 时暴露，而是在首次使用时才会出现。执行一次低成本的探测请求（“确认可以访问 `<service>` 并列出 1-2 项内容；不要启动任务”），验证无误后再发送真正的启动请求。如果没有外部依赖，则跳过此步骤。
4. 打开流式连接 → 发送第4节中的启动指令 → 按照第4节中的终端判断逻辑循环处理。

> 警告：**切勿在同一未加保护的代码块中同时发出 `agents.create()` 和 `sessions.create()` 请求**——这会导致每次运行都创建一个新代理，这是最常见的反模式。对于单脚本请求，请将创建操作包裹在 `if not os.getenv("AGENT_ID"):` 中。

请从 `{lang}/managed-agents/README.md` 中获取所检测语言的确切语法（对于 cURL 和 C#，以 `curl/managed-agents.md` 作为接口级参考）。请勿自创字段名。