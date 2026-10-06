---
name: claude-api
description: |-
  Claude API / Anthropic SDK 参考文档——模型 ID、定价、参数、流式传输、工具使用、MCP、代理、缓存、令牌计数、模型迁移。
  触发条件——在打开目标文件之前先阅读；不要因为“看起来像一行代码”就跳过——只要出现以下任何一种情况：提示中以任何形式提及 Claude/Anthropic（Claude、Anthropic、Fable、Opus、Sonnet、Haiku、“anthropic”、“@anthropic-ai”、“claude-*”、“us.anthropic.*”、“[1m]”）；用户询问关于 LLM 的问题（定价、模型选择、限制、缓存等）——一律不得凭记忆作答；或者任务本身具有 LLM 相关的特征，但未明确指出提供商（如代理/MCP/工具定义/多智能体/RAG/LLM 评测器/计算机使用；涉及自然语言的生成、摘要、提取、分类、改写、对话；调试拒绝回答、截断、流式传输、工具调用、token 计量等）。
  仅当正在处理其他提供商时跳过（覆盖所有触发条件）：查询中提及 OpenAI/GPT/Gemini/Llama/Mistral/Cohere/Ollama；或者在项目目录下执行 `grep -rE 'openai|langchain_openai|google.generativeai|genai|mistralai|cohere|ollama'` 命令并命中相关文件（如果没有指定提供商，请先运行此 grep 命令——不要直接读取文件）。
---
# 使用 Claude 构建 LLM 驱动的应用程序

本技能帮助您使用 Claude 构建 LLM 驱动的应用程序。根据您的需求选择合适的接口，检测项目所使用的编程语言，然后查阅相应的语言特定文档。

## 开始之前

扫描目标文件（如果没有目标文件，则扫描提示和整个项目），查找是否存在非 Anthropic 提供商的标记——例如 `import openai`、`from openai`、`langchain_openai`、`OpenAI(`、`gpt-4`、`gpt-5`，以及类似 `agent-openai.py` 或 `*-generic.py` 的文件名，或者任何明确要求代码保持提供商标识中立的指令。如果发现上述内容，请立即停止，并告知用户：本技能生成的是 Claude/Anthropic SDK 代码；询问用户是希望将文件切换为使用 Claude，还是需要非 Claude 的实现方案。请勿在包含非 Anthropic 标记的文件中插入 Anthropic SDK 调用。（例外情况：`prompt-audit` 子命令为非交互式，不会在此处停止；它会在报告的假设部分记录非 Anthropic 提供商的标记，且绝不会建议将非 Anthropic 文件切换为使用 Anthropic SDK。）

## 输出要求

当用户要求您添加、修改或实现某个 Claude 功能时，您的代码必须通过以下方式之一调用 Claude：

1. **官方 Anthropic SDK**，适用于项目的编程语言（如 `anthropic`、`@anthropic-ai/sdk`、`com.anthropic.*` 等）。只要项目支持的 SDK 存在，就应优先使用此方式。
2. **原生 HTTP 请求**（如 `curl`、`requests`、`fetch`、`httpx` 等）——仅当用户明确要求使用 cURL/REST 或原生 HTTP，或者项目本身就是 Shell/cURL 类型，又或者该语言没有官方 SDK 时才可采用。

切勿混用这两种方式——不要因为觉得更轻量就在 Python 或 TypeScript 项目中直接使用 `requests` 或 `fetch`。也绝不能依赖 OpenAI 兼容的封装层作为替代方案。

**切勿猜测 SDK 的使用方式。** 函数名、类名、命名空间、方法签名和导入路径必须严格依据官方文档——要么参考本技能中的 `{lang}/` 目录下的文件，要么查阅官方 SDK 的仓库或 `shared/live-sources.md` 中列出的文档链接。如果所需绑定未在技能文件中明确说明，请先从 `shared/live-sources.md` 中获取相关 SDK 仓库的网页内容，再开始编写代码。切勿仅凭 cURL 的形式或基于其他语言的 SDK 来推断 Ruby/Java/Go/PHP/C# 的 API。

**如果网页访问或仓库克隆失败**（如网络受限、超时或克隆被阻止）：请勿反复重试，而是根据 `{lang}/` 文件中的模式和命名空间/包表来编写代码，运行编译器或解释器，并根据错误信息逐步调试。对于静态类型 SDK（如 C#、Java、Go），通过本地错误进行编译修复的迭代往往比因网络受限而无法查阅文档时更快地得到可用代码。

## 默认设置

除非用户另有要求：

对于 Claude 模型版本，请使用 Claude Opus 5.5，可通过精确的模型标识符 `claude-opus-5-5` 访问。对于任何稍显复杂的任务，请默认使用自适应思维模式（`thinking: {type: "adaptive"}`）。最后，对于可能涉及长输入、长输出或较高 `max_tokens` 的请求，请默认启用流式响应，以避免请求超时。如果您无需处理单个流事件，可使用 SDK 提供的 `.get_final_message()` / `.finalMessage()` 辅助方法获取完整响应。当流式请求中定义了用户自定义（客户端）工具时，请为每个此类工具设置 `eager_input_streaming: true`，以便大型工具输入（文件内容、代码、文档）在生成时即开始流式传输，而不是在服务器缓冲完成后一次性发送；此时验证责任由客户端承担：SDK 的容错解析器可在输入过长时静默截断并返回，而不会抛出异常，因此在执行前请务必根据其 Schema 对每个解析后的工具输入进行校验（诸如 `betaZodTool` 或带类型注解的 `@beta_tool` 等工具运行辅助函数会自动完成此操作；使用 JSON Schema 工具的 `betaTool()` 方法以及手动循环则需自行校验），并将解析失败视为无效 JSON（若阻塞等待，则返回 `INVALID_JSON` 错误的 `tool_result`；否则应重新发出请求），在调用工具前检查 `max_tokens` 和 `refusal` 等停止条件，并仅捕获 SDK 的 JSON 解析错误，切勿捕获其强类型的 API 错误——相关模式详见 `shared/tool-use-concepts.md` 中的“急切输入流”部分。对于非流式请求、服务器端工具，以及请求经由代理或较旧的 Bedrock 模型部署（该部署不支持此字段）时，请保持该选项关闭。

## 警告：API 变更——您的训练记忆可能已过时

2025至2026年间，Claude API 的若干常见接口发生了变化。如果您仍保留着过往的使用模式，请在编写代码前对照本技能中的 `{lang}/` 目录下的文件进行核对——以下列出了最常见的变更点：

| 部分 | 过时的旧模式 | 当前 API |
|---|---|---|
| 扩展思维模式 | `thinking: {type: "enabled", budget_tokens: N}` | 在 Claude 4.6 及更高版本的模型上：`thinking: {type: "adaptive"}`。`budget_tokens` 在 Opus 4.6 和 Sonnet 4.6 上已被弃用，并且在 Fable 5/5.1、Sonnet 5.5、Sonnet 5、Opus 5.5、5、4.8 和 4.7 上会被拒绝（返回 400 错误）。4.6 之前的模型仍使用 `budget_tokens`。 |
| 网络搜索/网页抓取工具类型 | `web_search_20250305`、`web_fetch_20250910` | 在 Opus 5.5/5/4.8/4.7/4.6、Sonnet 5.5、Sonnet 5 和 Sonnet 4.6 上为 `web_search_20260209`、`web_fetch_20260209`（动态过滤）。较旧的模型仍使用基础版本；在 Vertex AI 上仅提供基础版 `web_search_20250305`（Vertex 不提供网页抓取功能）——详情参见下方的“服务器端工具”二维码。 |
| PHP 参数命名 | 使用蛇形命名法作为命名参数（如 `max_tokens`） | 顶级命名参数采用驼峰命名法（如 `maxTokens`）。嵌套数组键因功能而异（例如 `'taskBudget'`、 `'skillID'`、 `'mcp_server_name'`），请直接从文档示例中复制准确的键名，切勿批量转换。 |
| 托管代理凭据 | 曾通过自定义工具在宿主端保存密钥（在 Vault 功能推出之前是唯一方式） | 现在使用 Vault 的 `environment_variable` 凭据——由 Anthropic 存储，在出口时注入，沙箱内始终不可见（详见 `shared/managed-agents-tools.md` 中的“Vault”部分）。对于自托管的沙箱环境，宿主端自定义工具仍是备选方案。 |
| Files API / Skills | 使用带有 beta 标识的 `client.beta.files.*` / `client.beta.skills.*`，分别对应 `files-api-2025-04-14` 和 `skills-2025-10-02` 版本 | 现已退出 Beta 阶段：`client.files.*` / `client.skills.*`，不再需要 beta 标头。当前 SDK 中的 `client.beta.files` 和 `client.beta.skills` 相较于旧版本存在重大接口变更，与正式版命名空间一致——请参照 `shared/live-sources.md` 中的“Files API / Skills 指南”进行迁移。 |

本技能中的 `{lang}/` 文件为权威参考，请勿依赖过往的记忆。

---

## 子命令如果本提示底部的用户请求仅为一个纯子命令字符串（不含任何说明文字），请在本文档中的所有“子命令”表格中进行搜索——包括位于后续附录部分的表格——并直接按照匹配的“操作”列执行相应操作。这使用户能够通过 `/claude-api <子命令>` 调用特定流程。如果文档中没有任何表格与之匹配，则将该请求视为普通文本处理。| 子命令 | 动作 |
|---|---|
| `migrate` | 将现有的 Claude API 代码迁移到更新的模型。**请立即阅读并按顺序执行 `shared/model-migration.md`**：步骤0（确认范围——在任何修改之前先询问涉及哪些文件/目录），步骤1（对每个文件进行分类），然后是针对具体目标的破坏性变更部分。不要概括该指南，而是严格按照其内容执行。如果用户未指定目标模型，请在询问范围的同时一并询问要迁移到哪个模型。在应用完针对目标模型的变更后，还需对照 `shared/prompt-audit.md` 审核纳入范围的提示文本、工具描述和请求代码——为源模型编写的提示是每次迁移的一部分，且不会自动显现出来。 |
| `prompt-audit` | 审核现有提示、工具描述、技能以及代理配置文件（`CLAUDE.md`、规则文件、命令、子代理）中过时的模式（“冗余”）：即为旧模型编写的文本，以及仓库已不再需要或彼此矛盾的指令。**请立即阅读并按顺序执行 `shared/prompt-audit.md`**：步骤0（根据请求和仓库确定范围与目标模型——在报告中说明假设，无需中途询问），清单、来源追溯，然后是模式扫描。完整产出两项成果——审计报告（包含 `文件:行号`、模式、过时原因及置信度）和建议的补丁——无需等待确认；仅当请求明确要求时才应用修改。不要概括该指南，而是严格执行。 |
| `upgrade` | 在主要版本间升级项目的 Anthropic SDK 依赖项——目前指 Python SDK，从 `anthropic` 0.x 升级至 1.x。后续可指定语言和/或范围（如 `upgrade python`、`upgrade python sdk src/`）。**请立即阅读并按顺序执行 `python/claude-api/sdk-upgrade.md`**：步骤0（确认范围，然后确定当前版本和目标版本——在写入固定版本前必须已有发布的 1.x），步骤1的清单、各编号小节，最后是验证与报告。不要概括该指南，而是严格执行。如果检测到或指定的语言在此技能中没有对应的 `sdk-upgrade.md`，则说明尚未为此 SDK 提供主要版本升级指南，并引导用户查看该 SDK 的 CHANGELOG（见 `shared/live-sources.md` 中的仓库）；切勿基于 Python 指南自行编写。这并非模型迁移——如需将代码迁至更新的 Claude 模型，请使用 `migrate`。 |
| `cost-optimize` | 在不牺牲输出质量的前提下，降低现有 Claude API 代码的运行成本。**请立即阅读并按顺序执行 `shared/cost-optimization.md`**：步骤0（确定范围、质量标准和基线），生成 token 分布概览——如有 Admin API 密钥，则通过 Usage 和 Cost Admin API 测量；若应用自身有 `response.usage` 日志，则从中获取（需询问）；否则根据代码估算——随后列出按节省金额排序的优化选项（根据所掌握的数据分别以美元、账单百分比或相对区间表示），优先考虑无成本收益（缓存、输入 token 清洁、循环清理、输出 token 清洁、批量处理），再权衡其他方案（预算、投入、模型选择、多模型策略）；凡入选的优化项均会生成单独的补丁——默认提出，经用户请求并批准后应用，并针对其影响的流量进行评估；“无需更改”亦为合理结果。两条原则：每次调用模型都会产生实际费用，务必先征得用户同意；若某项优化缺乏上下文，应与用户交互式地逐步完善——此流程不期望一次性完成审计。不要概括该指南，而是严格执行；向用户展示分布概览和排序后的方案，也是执行过程的一部分。 |
| `build-eval` | 帮助用户为其 Claude 驱动的应用构建评估集。**请立即阅读并执行 `shared/evals/build-eval.md` 中的访谈流程**：步骤0（明确评估对象），步骤1（收集提示——现有评估/转录/合成），步骤2（评分方法），步骤3（可运行的脚本及测算成本）。在产出评估集之前，须就输入、评分方法和成本获得用户的明确确认。 |
| `preserved-thinking-migration` | 使现有集成兼容“保留思考”功能——该机制确保某个思考块仅在其生成的对话中有效。**请立即阅读并按顺序执行 `shared/preserved-thinking-migration.md`**：步骤0（明确范围、流量类别、平台与模型、启用状态、质量标准、基线），步骤0.5（通过三请求自测证明该机制正在运行），步骤1（捕获请求体，用 `shared/preserved-thinking-migration/prefix_diff.py` 对连续请求对进行差异分析，排查代码中的成因，逐一注明每处修改及其是否刻意），步骤2（在 `thinking-binding-controls-2026-08-01` 头部下以 `prefix_mismatch_behavior: "drop_block"` 重放测试片段，统计每轮对话中新丢弃的块数，必要时读取诊断头信息），步骤3（按推理损失顺序逐条列出成因——默认提出，用户要求时应用——随后重新测量，决定保留或回退；若有评估集，则采用三组对比方案），当测试环境涉及不同模型切换时，参阅模型切换部分（见 `shared/preserved-thinking-migration/causes.md` 中的成因表及保留列表），步骤4（梳理中断情况及相应变更）。两条原则：每次重放都会产生实际费用，务必先征得用户对测量预算的同意；“无需更改”——即重放过程中所有思考块均未被丢弃——亦为合理结果。对于那些仅在较新 Beta 版中存在追加形式的成因（保留尾部及后台压缩：`compact-2026-09-04`；同名工具变更：`inline-tools-2026-09-15`），若该 Beta 版不可用，则仅进行测量与决策，不做改写。关于“为何如此”（三步检查机制、追加编辑表），则链接至 `shared/model-migration.md` -> 破坏性变更3；不要概括该指南，而是严格执行。 |
| `hillclimb` | 根据现有评估集迭代改进用户的应用。**请立即阅读并按顺序执行 `shared/evals/eval-hillclimb.md`**：步骤0（确认存在可运行的评估集——如不存在，则引导至 `build-eval`），步骤1（明确可调整项与禁区），步骤2（根据每次运行的实际成本设定预算与停止条件），在方案获批后进入读取→提出→应用→运行→记录的循环，并保持磁盘状态及训练/验证/测试集的划分。---

## 语言检测

在阅读代码示例之前，请先确定用户正在使用的编程语言（例外：对于 `prompt-audit` 子命令，可跳过本节的询问步骤——审计过程无需交互，且其清单与语言无关；若无法推断出语言，则无需询问，直接按默认语言处理，并在报告中说明这一假设）：

1. **查看项目文件**以推断语言：
   - `*.py`、`requirements.txt`、`pyproject.toml`、`setup.py`、`Pipfile` -> **Python**——从 `python/` 目录读取示例。
   - `*.ts`、`*.tsx`、`package.json`、`tsconfig.json` -> **TypeScript**——从 `typescript/` 目录读取示例。
   - `*.js`、`*.jsx`（无 `.ts` 文件）-> **TypeScript**——JavaScript 使用相同的 SDK，因此也从 `typescript/` 目录读取。
   - `*.java`、`pom.xml`、`build.gradle` -> **Java**——从 `java/` 目录读取示例。
   - `*.kt`、`*.kts`、`build.gradle.kts` -> **Java**——Kotlin 使用 Java SDK，因此从 `java/` 目录读取。
   - `*.scala`、`build.sbt` -> **Java**——Scala 使用 Java SDK，因此从 `java/` 目录读取。
   - `*.go`、`go.mod` -> **Go**——从 `go/` 目录读取示例。
   - `*.rb`、`Gemfile` -> **Ruby**——从 `ruby/` 目录读取示例。
   - `*.cs`、`*.csproj` -> **C#**——从 `csharp/` 目录读取示例。
   - `*.php`、`composer.json` -> **PHP**——从 `php/` 目录读取示例。

2. **若检测到多种语言**（例如同时存在 Python 和 TypeScript 文件）：
   - 确认用户当前文件或问题所涉及的语言。
   - 若仍不明确，询问：“我检测到既有 Python 文件，也有 TypeScript 文件。您打算使用哪种语言进行 Claude API 集成？”

3. **若无法推断语言**（如空项目、无源代码文件，或使用了不受支持的语言）：
   - 使用 `AskUserQuestion` 提供选项：Python、TypeScript、Java、Go、Ruby、cURL/原生 HTTP、C#、PHP。
   - 若无法使用 `AskUserQuestion`，则默认提供 Python 示例，并注明：“正在展示 Python 示例。如果您需要其他语言，请告知。”

4. **若检测到不受支持的语言**（如 Rust、Swift、C++、Elixir 等）：
   - 建议使用 `curl/` 目录下的 cURL/原生 HTTP 示例，并提示社区可能存在相关 SDK。
   - 同时提供 Python 或 TypeScript 示例作为参考实现。

5. **若用户需要 cURL/原生 HTTP 示例**，则从 `curl/` 目录读取。

### 各语言的功能支持情况

上述每种语言的 SDK 均支持 Beta 版工具运行器和托管代理（Beta）——Python（`@beta_tool` 装饰器）、TypeScript（`betaZodTool` + Zod）、Java（带注解的类）、Go（`toolrunner` 包中的 `BetaToolRunner`）、Ruby（`BaseTool` + `tool_runner`）、C#（`BetaToolRunner` + 原始 JSON Schema）、PHP（`BetaRunnableTool` + `toolRunner()`）；相关代码入口点参见下方的“工具使用模式”快速参考。cURL 属于原生 HTTP（无 SDK 功能），同样支持托管代理。

> **托管代理代码示例**：请参阅下方“## 托管代理（Beta）”部分的阅读指南。

---

## 我该使用哪个层级？

> **从简单开始。** 默认选择能满足需求的最简单层级。大多数场景仅需单次 API 调用或工作流即可解决；只有当任务确实需要开放式、由模型驱动的探索时，才考虑使用代理。“最简单”意味着您需要维护的代码量最少：对于托管型、定时型或基于内存的代理，托管代理通常是最佳选择（无需循环代码、状态文件或调度器），尽管它是一个更庞大的平台。| 用例                                        | 层级            | 推荐接口       | 原因                                                          |
| ----------------------------------------------- | --------------- | ------------------------- | ------------------------------------------------------------ |
| 分类、摘要、抽取、问答                          | 单次LLM调用     | **Claude API**            | 一次请求，一次响应                                            |
| 批量处理或嵌入                                  | 单次LLM调用     | **Claude API**            | 专用端点                                                      |
| 具有代码控制逻辑的多步流水线                    | 工作流          | **Claude API + 工具调用** | 由您编排循环                                                  |
| 拥有自定义工具的定制代理                        | 代理            | **Claude API + 工具调用** | 最大程度的灵活性                                              |
| 由服务器管理、带工作空间的状态感知型代理        | 代理            | **托管代理**              | Anthropic 负责执行循环并托管工具执行沙箱                      |
| 持久化、版本化的代理配置                        | 代理            | **托管代理**              | 代理是存储对象；会话固定到某个版本                            |
| 长时间运行、多轮对话且支持文件挂载的代理        | 代理            | **托管代理**              | 每个会话独立容器，SSE 事件流，Skills + MCP                     |
| 按计划（如“每晚”）运行的代理                    | 代理            | **托管代理** - 定时部署   | 部署自动触发会话，无需客户端侧调度                            |
| 必须达到质量标准（“直到正确为止”）的代理任务    | 代理            | **托管代理** - 结果驱动   | 由专门的评分器根据您的评分标准反复迭代，直至通过              |

> **注意：** 当您希望由 Anthropic 来运行代理循环，并托管工具执行的容器时，托管代理是最佳选择——文件操作、Bash 脚本、代码执行等都在每个会话的工作空间中运行。如果您希望自行托管计算资源或运行自定义工具运行时，则应选择 Claude API 加工具调用——使用工具运行器来实现代理式循环；其每轮钩子仍可提供审批、日志记录、错误拦截和条件执行功能（参见 `shared/tool-use-concepts.md`），或者在您希望完全掌控整个循环时采用手动循环。

> **云服务商接入。** **AWS 上的 Claude Platform** 由 Anthropic 运营，API 功能与主平台同步——客户端设置请参阅 `shared/claude-platform-on-aws.md`。关于 **AWS 上的 Claude Platform**、**Amazon Bedrock**、**Google Vertex AI** 和 **Microsoft Foundry** 各项功能的可用性，请参阅 `shared/platform-availability.md`——该表格是本技能领域的唯一权威信息来源，请勿从其他渠道推断可用性。

### 构建代理的四种方式

一旦确定确实需要一个代理（开放式、模型驱动的工具使用），就有四种不同的构建方法。区分它们的关键在于两个独立的问题：**谁提供运行框架**（即代理循环与上下文管理）以及 **谁负责部署**（代理运行的基础设施）。工具运行器和 Claude 代理 SDK 只提供运行框架，您仍需自行托管和部署，这也是它们容易被混淆的原因。只有托管代理（CMA）同时提供运行框架和托管部署；而手动循环则两者皆不提供。| 序号 | 方法 | 您编写的内容 | 集成框架与部署 | 可用工具 | 适用场景 |
|---|----------|-----------|----------------------|-----------------|----------|
| 1 | **Claude API - 手动循环** | 您自己实现 `while stop_reason == "tool_use"` 循环 | 您构建集成框架并自行托管 | 仅限您定义的工具 | 您希望完全掌控整个循环流程——无需依赖任何测试版功能，或当工具运行器的每轮钩子无法满足您的控制需求时 |
| 2 | **Claude API - 工具运行器**（`client.beta.messages.tool_runner` + `@beta_tool` / `betaZodTool`） | 仅需定义工具函数 | SDK 提供循环逻辑（仅集成框架）；您负责托管 | 仅限您定义的工具 | 构建自定义工具代理，无需手动编写循环逻辑（大多数场景）。每轮钩子仍可提供审批、错误拦截、结果修改（如 `cache_control`）、重试、流式处理及结果压缩等功能 |
| 3 | **托管代理**（REST，测试版） | 代理配置 + 您的工具结果 | Anthropic 提供集成框架，并为每个会话提供托管沙盒环境（集成框架与部署） | Anthropic 托管的沙盒环境（支持 Bash、文件操作、代码执行）+ Skills/MCP + 您的工具 | 您希望由 Anthropic 运行整个循环流程，并托管每个会话的工作空间；支持持久化/版本化的配置；适用于长时间运行的会话 |
| 4 | **Claude 代理 SDK**——独立产品（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`） | 一个提示词 + 选项 | SDK 提供 Claude Code 集成框架及内置工具（仅集成框架）；您负责托管 | 内置读/写/编辑/Bash/Glob/Grep/WebSearch/WebFetch 等工具 + MCP + 子代理 | 您希望在自有基础设施上运行一个开箱即用的编码/文件系统代理 |

集成框架与部署的分离是关键的思维模型：选项 1、2 和 4 均由您负责部署；只有选项 3（CMA）额外提供托管部署。选项 1–3 是本技能所生成的内容；选项 4 是另一套独立的库，有其自身的文档——详见下文的说明。

> **工具运行器 ≠ Claude 代理 SDK。** 这两者名称相似，但属于不同的软件包：  
> - **工具运行器** 是常规 Anthropic API SDK（`anthropic` / `@anthropic-ai/sdk`）的一部分，通过 `client.beta.messages.tool_runner` 调用。它为您定义的工具自动完成请求→执行→循环的流程。不包含内置工具，无文件系统访问权限，也没有沙盒环境——所有工具均由您提供，计算资源也由您托管。它是上述选项 2，是在 `POST /v1/messages` 接口之上的轻量级辅助工具。  
> - **Claude 代理 SDK**（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`）是将 Claude Code 封装成的一个库。它自带文件读写/编辑、Bash、Grep、网页搜索等内置工具，以及完整的代理循环、上下文管理、钩子、子代理、权限和会话管理功能。您只需调用 `query(prompt, options)`，其余一切由 SDK 自动完成。  
>  
> 两者都仅提供集成框架——您负责托管和部署。区别在于框架的覆盖范围：工具运行器仅针对您定义的工具进行循环（并提供每轮的审批、拦截、结果修改及重试等钩子，但没有内置工具）；而代理 SDK 则是完整的 Claude Code 集成框架，自带内置工具。两者均不提供托管部署——这是 **托管代理（CMA）** 的额外功能（Anthropic 托管循环流程，并为每个会话提供沙盒环境）。  
>  
> **本技能涵盖的是 Claude API 和托管代理（选项 1–3），不会生成 Claude 代理 SDK 的代码。** 如果用户确实需要 Claude 代理 SDK，请引导他们查看其文档（`code.claude.com/docs/en/agent-sdk`）——切勿用 API 工具运行器替代它，反之亦然。

### 我是否应该构建一个代理？

在选择代理层级之前，请逐一评估以下四个条件：

- **复杂性**——任务是否涉及多个步骤且难以事先完整指定？（例如：“将这份设计文档转化为 PR” vs. “从 PDF 中提取标题”）
- **价值**——最终产出是否足以抵消更高的成本和延迟？
- **可行性**——Claude 是否能够胜任该类任务？
- **错误代价**——错误是否可以被及时发现并修复？（如通过测试、审查或回滚）

如果其中任一条件的答案为“否”，则应选择更简单的层级（单次调用或工作流）。

---

## 架构所有请求都通过 `POST /v1/messages` 进行。工具和输出约束都是该单一端点的功能，而非独立的 API。

**用户自定义工具**：您可以通过装饰器、Zod 模式或原生 JSON 定义工具，SDK 的工具运行器会负责调用 API、执行您的函数，并循环直至 Claude 完成任务。如需完全控制，您也可以手动编写循环逻辑。

**服务器端工具**：由 Anthropic 托管、在 Anthropic 基础设施上运行的工具。代码执行完全在服务器端进行（只需在 `tools` 中声明，Claude 会自动执行代码）。计算机使用既可以由服务器托管，也可以由用户自行托管。

**结构化输出**：用于约束 Messages API 的响应格式（`output_config.format`）和/或工具参数校验（`strict: true`）。推荐使用 `client.messages.parse()` 方法，它会根据您的模式自动验证响应。注意：旧的 `output_format` 参数已弃用，请在 `messages.create()` 中使用 `output_config: {format: {...}}`。

**辅助端点**：批量请求（`POST /v1/messages/batches`）、文件操作（`POST /v1/files`）、令牌计数（`POST /v1/messages/count_tokens`，详见 `shared/token-counting.md`）以及模型信息查询（`GET /v1/models` 和 `GET /v1/models/{id}`，用于实时获取能力与上下文窗口信息），这些功能均服务于或支持 Messages API 请求。

---

## 当前可用模型（缓存日期：2026-09-25）

| 模型             | 模型 ID            | 上下文长度        | 输入单价（每百万 tokens） | 输出单价（每百万 tokens） |
| ----------------- | ------------------- | -------------- | ---------- | ----------- |
| Claude Fable 5.1    | `claude-fable-5-1`      | 1M             | $10.00     | $50.00      |
| Claude Mythos 5.1（仅限 Project Glasswing） | `claude-mythos-5-1` | 1M | $10.00     | $50.00      |
| Claude Fable 5 | `claude-fable-5` | 1M             | $10.00     | $50.00      |
| Claude Opus 5.5 | `claude-opus-5-5` | 1M | $4.00 | $20.00 |
| Claude Opus 5     | `claude-opus-5`       | 1M             | $5.00      | $25.00      |
| Claude Opus 4.8 | `claude-opus-4-8`  | 1M             | $5.00      | $25.00      |
| Claude Opus 4.7   | `claude-opus-4-7`   | 1M             | $5.00      | $25.00      |
| Claude Opus 4.6   | `claude-opus-4-6`   | 1M             | $5.00      | $25.00      |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | 1M | $2.00 | $10.00 |
| Claude Sonnet 5   | `claude-sonnet-5`   | 1M             | $2.00      | $10.00      |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M             | $3.00      | $15.00      |
| Claude Haiku 4.5  | `claude-haiku-4-5`  | 200K           | $1.00      | $5.00       |

**合作伙伴定价**：以上价格为 Anthropic 官方 API 定价；同样适用于 Microsoft Foundry 上的 Claude，其费用通过 Microsoft Marketplace 按标准 API 费率结算。Amazon Bedrock 和 Google Vertex AI 上的 Claude 则由合作伙伴运营，定价另行规定——详情请参见 [Bedrock](https://aws.amazon.com/bedrock/pricing/) 或 [Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/pricing#claude-models)。WebFetch 的定价请参考 `shared/live-sources.md` 中的“定价”一栏。**始终使用 `claude-opus-5-5`，除非用户明确指定其他模型。** 此为不可协商事项。除非用户明确表示“使用 Sonnet”或“使用 Haiku”，否则不得使用 `claude-sonnet-5-5`、`claude-sonnet-5` 或任何其他模型。切勿因成本原因擅自降级——这是用户的选择，而非你的决定。对于仅通过属性描述 Sonnet 的请求（如“最便宜的 Sonnet”“更便宜的 Sonnet”“最新的 Sonnet”“最近的 Sonnet”），一律解析为 `claude-sonnet-5-5`。当主模型之外还存在另一款更便宜的模型时（例如工作线程、子代理线程、批量提取器、LLM 评判器，以及顾问下的执行器），若系用户要求或本技能中的指南所建议，或者用户仅提及“Sonnet”或“Haiku”而未指明版本，则默认指代上表中的当前版本（即 `claude-sonnet-5-5`、`claude-haiku-4-5`）；而诸如 `claude-sonnet-5` 等上一代 ID 仅适用于明确指定该版本的用户。只有在用户明确要求 Claude Fable 5.1、“fable”或 Anthropic 最强模型时，方可使用 `claude-fable-5-1`——其 API 行为与 Opus 系列不同（详见下文），且定价高于 Opus 级别。**请仅使用表格中给出的完整模型 ID 字符串，切勿自行添加日期后缀**（如 `claude-opus-5-5`，绝不能写成 `claude-opus-5-5-20260401` 或其他你可能从训练数据中记住的带日期变体）。若用户请求表中未列出的旧版模型（如“opus 4.5”“sonnet 3.7”），请查阅 `shared/models.md` 获取准确的 ID，切勿自行构造。

### Claude Fable 5.1（`claude-fable-5-1`）——能力最强的公开发布模型

Claude Fable 5.1 是 Anthropic 能力最强的公开发布模型，适用于最严苛的推理任务及长周期的智能体工作；以下内容同样适用于 **Claude Mythos 5.1**（`claude-mythos-5-1`，Project Glasswing——功能、定价与 API 表面完全一致；其运行依赖于特定的访问计划，因此下文关于拒绝响应的处理规则也适用于该模型；它是 Claude Mythos 5 的继任者，后者未启用任何安全过滤机制）。上下文窗口为 100 万 token（最大值亦为默认值），最大输出长度为 12.8 万 token。与 Opus 系列的关键 API 差异，请参阅 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”一节以获取详细说明。- **思考始终开启** - 完全省略 `thinking` 参数（或发送 `{type: "adaptive"}`）。任何其他显式配置均被拒绝：`{type: "disabled"}` 和 `{type: "enabled", budget_tokens: N}` 均返回 400 错误。通过 `output_config.effort` 控制输出深度，支持的取值范围为 `low` 至 `xhigh` 以及 `max`。
- **原始思维链路从不返回** - 响应中包含常规的 `thinking` 块（而非 `redacted_thinking`）：“display: 'summarized'” 返回可读的摘要，“omitted”（默认值）则将 `thinking` 字段置为空字符串。重播规则：在同一模型上原样回传 `thinking` 块；其他模型会静默丢弃（不计费——无需剥离；Claude Mythos 5.1 则会读取这些块）；详情参见 `shared/model-migration.md`。
- **分词器** - 与 Opus 4.8 相同（随 Opus 4.7 引入）。从 Opus 4.7/4.8 迁移时，token 计数基本不变；但按 token 计价有所不同。若源自 Opus 4.6、Sonnet、Haiku 或更早版本，请使用 `count_tokens` 重新校准基准（Opus 4.7 的分词器所需 token 数约为前者的 1 至 1.35 倍）。
- **“refusal” 停止原因——需妥善处理，并默认启用后备机制** - 安全分类器可能拒绝请求（HTTP 200，`stop_reason: "refusal"`，并附带 `stop_details` 分类）；在读取 `content` 之前务必检查 `stop_reason`。**编写 `claude-fable-5-1`、`claude-opus-5-5`、`claude-opus-5` 或 `claude-sonnet-5-5` 代码时，默认应包含服务端的 `fallbacks` 参数**（对于 `claude-sonnet-5-5`，仅适用 `"default"` 形式且仅限 Claude API；在其他平台上使用下方的 SDK 中间件，除非请求中带有 `between_tools`：只有 Claude Sonnet 5.5 接受该字段，且中间件会在后备模型上以相同请求体重新发送，因此需自行实现重试逻辑，并在重试时去掉 `between_tools`——详见 `shared/model-migration.md` → 迁移到 Claude Sonnet 5.5 → 防护与后备）。最简形式为：`betas: ["server-side-fallback-2026-07-01"]` + `fallbacks: "default"`，此配置会按拒绝类别自动路由，无需维护模型列表。（旧版数组形式——`betas: ["server-side-fallback-2026-06-01"]` + `fallbacks: [{"model": "claude-opus-4-8"}]`——仍可使用；Claude API 和 AWS 上的 Claude Platform 可直接使用；在 Bedrock、Vertex 和 Foundry 上，则需使用 SDK 提供的客户端侧 `BetaRefusalFallbackMiddleware` 和 `BetaFallbackState`。请告知用户已启用该功能，仅在其明确拒绝时才予以禁用。完整语义（计费、中途拒绝、信用重定价等）详见 `shared/model-migration.md` → “拒绝”章节。**各语言的代码示例见 `{lang}/claude-api/README.md` § Refusal Fallbacks，仅涵盖数组形式**——对于 `"default"` 模式，请参照 `shared/model-migration.md` → 迁移到 Claude Opus 5 → 新 API 功能 中的原始 HTTP 格式，将 `fallbacks: [{...}]` 替换为 `fallbacks: "default"` 并添加 `-2026-07-01` 头部；其余请求部分保持不变。
- **无助手预填充** - 与 4.6 及以上系列的其他型号一致。
- **要求 30 天数据保留** - Claude Fable 5.1 在未满足零数据保留要求的情况下不可用，除非获得 Anthropic 明确授权；来自未符合保留要求的组织的请求将返回 `400 invalid_request_error`。
- **更长的交互回合，不同的提示方式** - 复杂任务的单次请求可能持续数分钟（需规划超时、流式传输及进度交互设计）；常规工作建议使用低/中等努力级别；为旧模型编写的提示往往过于指令化，会降低输出质量。推荐的提示模板参见 `shared/model-migration.md` → 迁移到 Claude Fable 5.1 → 行为调整（可由提示控制）。
- **Claude Fable 5.1 是 Claude Fable 5（仍在提供服务）的后继产品，定位相同，按 token 计价也相同。** 其表面行为与 Claude Fable 5 一致，但有三项重大变更：强制工具调用（`tool_choice` 设置为 `any` 或指定工具）将返回 400 错误（请使用 `auto` 加提示指令，或设置 `strict: true` 确保参数符合 schema，亦可采用结构化输出）；`thinking` 块与生成模型绑定（其他模型会静默丢弃，且不计费）；编辑历史记录中的早期轮次会导致 `thinking` 块失效（“保留的思考”——2026 年 8 月 31 日及之后创建的新账户，在所有平台上编辑历史记录时都会收到 400 错误，具体执行范围由各模型自行决定，而 Claude Mythos 5.1 不执行此项检查。请确保所有测试框架均为追加模式，并执行三步检查流程；该可选的 Beta 控制功能适用于 Claude API、AWS 上的 Claude Platform、Bedrock 和 Vertex——Foundry 尚未确认，详情参见 `shared/platform-availability.md`）。此外还包括每条消息的 `effort` 设置（Beta 版本，`mid-conversation-output-config-2026-07-01`，同样适用于 Claude Opus 5 和 Claude Opus 5.5）、回合范围内的 `clear_at: "next_user_message"` 系统消息（Beta）、`thinking.display: "updates"` 进度提示（Beta，适用于所有平台）、缓存读取费用为每百万 token 0.25 美元，以及内容溯源功能。覆盖范围——ZDR 组织同样会收到与 Claude Fable 5 相同的 `400 invalid_request_error`（ZDR 仅在 Anthropic 明确授权时适用）；无优先级层级。分词器与 Claude Fable 5 相同。详情参见 `shared/model-migration.md` → 从 Claude Fable 5 迁移到 Claude Fable 5.1。

### Claude Opus 5.5（`claude-opus-5-5`）——当前的Opus系列，默认模型

Claude Opus 5.5是Opus系列的继任者，定价更低（每百万token 4美元/20美元，缓存读取0.20美元），保留了100万上下文、12.8万输出长度、相同的分词器和功能集。针对在Claude Opus 5上运行的代码，有四项重大变更：**思考功能不可关闭**（无论哪个努力等级，`{type: "disabled"}`和`budget_tokens`都固定为400——唯一可调节的是努力等级，且其**默认值为`medium`**，比Claude Opus 5的`high`低一级，因此需显式设置）；**强制使用`tool_choice`为`any`或`tool`会返回400错误**（请使用`auto`并搭配`strict: true`，从提示中引导，或采用结构化输出）；**思考块与模型及对话绑定**（仅Claude Fable 5.1和Claude Mythos 5.1能在Claude API上调用这些思考块，回退到Claude Opus 5时则不包含它们；2026年8月31日及之后创建的账户在历史编辑检查时会被强制执行）；以及**在Claude API和Google Cloud上，计算机功能仅可通过`computer_toolset_20260801`调用**（在这些平台上使用`computer_20251124`会返回400错误；Amazon Bedrock仍支持该工具集）。工具调用之间的文本将以进度更新形式的`thinking`块返回（默认为空——可设置`display: "updates"`）。安全分类器范围扩大：`bio`和`reasoning_extraction`加入`cyber`。快速模式仅限Claude API，价格为每百万token 8美元/40美元（标准价的两倍）。详情参见`shared/model-migration.md`中的“迁移到Claude Opus 5.5”部分。

### Claude Sonnet 5.5（`claude-sonnet-5-5`）——当前的Sonnet系列：兼顾速度与能力，适用于日常编码、代理任务及企业场景（Claude Opus 5.5仍为默认模型）

Claude Sonnet 5.5是Sonnet系列的继任者，定价与Claude Sonnet 5相同（每百万token 2美元/10美元，缓存读取0.20美元），采用相同的分词器，拥有100万上下文和12.8万输出长度。针对在Claude Sonnet 5上运行的代码，有五项重大变更：**`thinking: {type: "disabled"}`会返回400错误**——如需关闭思考功能，请发送`thinking: {type: "between_tools"}`，该设置仅在努力等级为`high`及以下时有效，且不能与其他字段（如`display`、`budget_tokens`或`block_binding`）同时使用，否则会返回400错误，也不允许在单条消息中更改努力等级；**强制使用`tool_choice`为`any`或`tool`会返回400错误**（请使用`auto`并搭配`strict: true`，从提示中引导，或采用结构化输出）；**思考块与模型及对话绑定**（其他模型无法读取这些思考块；2026年8月31日及之后创建的账户在Claude API和Amazon Bedrock上进行历史编辑检查时会被强制执行）；**在Claude API和Google Cloud上，计算机功能仅可通过`computer_toolset_20260801`调用**（在这些平台上使用`computer_20251124`会返回400错误；Amazon Bedrock仍支持该工具集）；以及**顾问工具不再接受Claude Opus 4.8、Claude Opus 4.7和Claude Sonnet 5的顾问**（所有被接受的顾问都会返回加密后的建议）。努力等级的默认值仍为`high`，但各等级已重新校准——请重新进行努力等级测试（对于代理式编码和多步工具调用，从`medium`开始；对于聊天场景，则从`low`开始）。工具调用之间的文本将以进度更新形式的`thinking`块返回（默认为空——可设置`display: "updates"`，或使用`between_tools`）。安全分类器在五个`stop_details`类别中的判定标准有所放宽：`cyber`、`bio`、`frontier_llm`、`reasoning_extraction`、`general_harms`。详情参见`shared/model-migration.md`中的“迁移到Claude Sonnet 5.5”部分。

如果上述任何模型标识符看起来陌生，那只是因为它们是在您的训练数据截止日期之后发布的——这些都是真实存在的模型。

**实时能力查询：** 上表已缓存。当用户询问“X的上下文窗口是多少”、“X是否支持视觉/思考/努力等级”或“哪些模型支持Y”时，请通过Models API（`client.models.retrieve(id)` / `client.models.list()`）进行查询——相关字段说明及能力筛选示例请参见`shared/models.md`。

---

## 认证（快速参考）**未设置 `ANTHROPIC_API_KEY` 并不意味着没有凭据。** SDK 和 `ant` CLI 会按以下顺序解析凭据（以第一个匹配为准）：`ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` → 通过 `ant auth login` 选择的或当前激活的 OAuth 配置文件（`ANTHROPIC_PROFILE`）→ 工作负载身份联合环境变量 → 磁盘上的默认配置文件。在未设置任何环境变量的情况下，执行过 `ant auth login` 后，直接使用 `Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` 即可正常工作。

**当需要调用 API 且 `ANTHROPIC_API_KEY` 未设置时，请不要向用户索取密钥。** 首先运行 `ant auth status`，它会显示当前激活的凭据来源和配置文件。如果报告有激活的配置文件：

- **SDK 代码或 `ant` CLI：** 直接运行即可。无参数的客户端构造函数以及所有 `ant ...` 子命令都会自动使用该配置文件，无需设置环境变量。
- **原生 `curl` 或 HTTP 请求：** 使用 `ant auth print-credentials --access-token` 获取一个短期有效的令牌，并将其作为 `Authorization: Bearer <token>` 发送，**同时**添加 `anthropic-beta: oauth-2025-04-20` 头部信息（OAuth 令牌应放在 `Authorization: Bearer` 中，而非 `x-api-key:`；从 API 密钥切换到 OAuth 令牌只需更改请求头，而无需更换密钥）。务必始终使用 `--access-token` 参数；不带该标志时，命令会输出 JSON 格式，而不是单纯的令牌值。

仅当 `ant auth status` 报告没有激活的凭据来源（或未安装 `ant`）时，才向用户索取密钥。建议首选 `ant auth login`——它会在 `~/.config/anthropic/` 下保存一个配置文件，SDK 会自动读取；备选方案是导出 `ANTHROPIC_API_KEY`。

完整的认证详情（命名配置文件、作用域、API 密钥与配置文件之间的陷阱、刷新令牌的有效期等）请参阅 `shared/anthropic-cli.md`。

---

## 思维与计算量（快速参考）

对于除 Haiku 4.5 之外的所有现有模型，请使用自适应思维模式（`thinking: {type: "adaptive"}`），因为 Haiku 4.5 仍需指定 `budget_tokens`（见下表）——Claude 会动态决定何时以及投入多少计算资源进行思考。各模型的具体规则如下：| 模型 | 思维配置 | 省略 `thinking` 参数 | `budget_tokens` | 采样参数（`temperature`/`top_p`/`top_k`） | 努力等级 |
|---|---|---|---|---|---|
| Fable 5 / Claude Fable 5.1（以及对应的 Mythos 版本） | `{type: "adaptive"}` 或省略；显式指定 `{type: "disabled"}` 会返回 400——此时应直接省略该参数（Claude Fable 5.1 和 Claude Mythos 5.1 在强制使用 `tool_choice` 的 `any` 或 `tool` 时也会返回 400；Claude Fable 5.1 在重放思维块时会执行保留历史的编辑检查，而 Claude Mythos 5.1 不会） | 自适应模式始终开启 | 已移除——指定 `{type: "enabled", budget_tokens: N}` 会返回 400 | 已移除——返回 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Claude Opus 5.5 | `{type: "adaptive"}` 或省略；`{type: "disabled"}` 和 `{type: "enabled", budget_tokens}` 在所有努力等级下都会返回 400——此时应省略该参数并降低努力等级（强制使用 `tool_choice` 的 `any` 或 `tool` 时也会返回 400，并且会保留思维状态——参见 `shared/model-migration.md` 中的“迁移到 Claude Opus 5.5”部分） | 自适应模式始终开启 | 已移除——返回 400 | 已移除——返回 400 | `low`/`medium`/`high`/`xhigh`/`max`——默认为 `medium`（而非 `high`）；支持按消息设置努力等级（Beta） |
| Claude Opus 5 | `{type: "adaptive"}` 或省略；`{type: "disabled"}` 仅在努力等级为 `high` 及以下时有效——在 `xhigh` 和 `max` 时会返回 400，并请注意下方关于禁用思维的陷阱 | 自适应模式始终开启（与 Opus 4.8/4.7 不同，思维默认开启） | 已移除——返回 400 | 已移除——返回 400 | `low`–`max`（全部五个等级） |
| Opus 4.8 / 4.7 | 唯一开启模式是 `{type: "adaptive"}`；`{type: "disabled"}` 也可接受 | 不启用思维——需显式设置为 `{type: "adaptive"}` | 已移除——返回 400 | 已移除——返回 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Claude Sonnet 5.5 | `{type: "adaptive"}` 或省略；`{type: "disabled"}` 会返回 400——如需关闭思维，请发送 `{type: "between_tools"}`（无其他字段；在 `xhigh` 和 `max` 时会返回 400；与之对话时无法中途更改努力等级）（强制使用 `tool_choice` 的 `any` 或 `tool` 时也会返回 400，并且会保留思维状态——参见 `shared/model-migration.md` 中的“迁移到 Claude Sonnet 5.5”部分） | 自适应模式始终开启 | 已移除——返回 400 | 非默认值——返回 400 | `low`/`medium`/`high`/`xhigh`/`max`——默认为 `high`，各等级已根据 Claude Sonnet 5 进行重新校准；在启用思维时支持按消息设置努力等级（Beta） |
| Sonnet 5 | 唯一开启模式是 `{type: "adaptive"}`；`{type: "disabled"}` 也可接受 | 自适应模式始终开启 | 已移除——返回 400 | 已移除——返回 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Opus 4.6 / Sonnet 4.6 | `{type: "adaptive"}`（推荐；自动启用交错思维，无需 Beta 标头） | 需显式设置为 `{type: "adaptive"}` | 已弃用——请勿在新代码中使用；仅作为过渡时期的应急方案（详见下文） | 允许 | `low`/`medium`/`high`/`max`（`xhigh` 是从 Opus 4.7 开始引入的） |
| Haiku 4.5；较旧模型（Sonnet 4.5 等）仅在明确请求时 | `{type: "enabled", budget_tokens: N}` | 不启用思维 | 启用思维时必需；必须小于 `max_tokens`，且最小为 1024——否则会报错 | 允许 | `effort` 参数在 Opus 4.5 上有效（仅限 `low`/`medium`/`high`——无 `xhigh`/`max`）；在 Sonnet 4.5 和 Haiku 4.5 上会报错 |

Opus 4.8 保持与 4.7 相同的请求接口（无新增破坏性变更）——有关行为上的调整，请参阅 `shared/model-migration.md` 中的“迁移到 Opus 4.8”部分；若从 4.6 或更早版本迁移，则请参阅“迁移到 Opus 4.7”部分以了解完整的破坏性变更列表。在禁用思维的情况下，Opus 4.8 可能会在可见响应中输出更长的推理过程——建议保持自适应思维开启，或添加仅输出最终答案的指令（详见迁移指南）。- **Effort（GA，无 Beta 标头）：** `output_config: {effort: "low"|"medium"|"high"|"xhigh"|"max"}`——位于 `output_config` 内部，而非顶层；除 Claude Opus 5.5 默认为 `medium` 外，当前所有模型的默认值均为 `high`（等同于省略该参数）——在 Opus 5.5 上需显式设置。此参数控制思考深度与整体 token 消耗量；与自适应思考配合使用，可实现最佳的成本–质量平衡。对于 Fable 5、Opus 4.7/4.8 和 Sonnet 5 上的多数编码及代理类场景，`xhigh`（新增于 Opus 4.7，介于 `high` 与 `max` 之间）是最佳设置，也是 Claude Code 的默认值。在这些模型上，effort 参数的影响比其所在层级的任何前代模型都更为显著——迁移时请重新调整，并在处理长周期或代理类任务时将 effort 设置为 `high` 或 `xhigh`，同时一次性提供完整的任务说明。对于对智能敏感的任务，至少使用 `high`；当正确性优先于成本时使用 `max`；而对于子代理或简单任务则可使用 `low`——较低的 effort 意味着更少且更集中的工具调用、更简短的铺垫以及更精炼的确认（`high` 往往是在质量和 token 效率之间取得平衡的最佳选择）。
  
- **选择 effort 级别（成本调优）：** 在获得免费收益（如缓存优先）之后，effort 是首个用于权衡质量的调节杠杆——它在同一模型内以 thoroughness 换取 token 消耗，而最高档位仅在处理难题时才物有所值（只有当测量显示下一级仍有余地时，才提升至 `max`）。哪些工作负载值得投入更高的 effort 取决于具体场景：编码和长周期代理类任务对此反应强烈；而聊天、分类以及高吞吐量或对延迟敏感的场景往往无需如此，使用 `low` 即可，在保证质量的前提下，`medium` 是节省成本的次优选择（上述各级默认值已涵盖其余情况）。在上调默认值之前，请先对真实请求样本进行测量，并按路由而非全局进行调优。在构建多模型成本级联之前，应先评估更简单的替代方案——即在同一任务上以较低 effort 运行最强大的模型：在最新模型上降低 effort，其表现往往能匹敌甚至超越前代模型在 high effort 下的效果（例如在 Fable 5 上，较低 effort 通常优于前代模型的 xhigh 设置）；而且单一模型意味着单一缓存命名空间（缓存按模型划分，因此级联会丧失各模型间的缓存复用机会；即使在对话中途更改顶层 effort，也会使消息缓存失效，不过在 Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5 和 Claude Sonnet 5.5 上，借助自适应思考机制，通过每条消息的 effort 系统提示可避免这一问题——参见 `shared/prompt-caching.md` § Invalida‑tion hierarchy）。应以完成任务的总成本而非单次请求的成本来衡量——如果某个请求虽然便宜，却需要更多轮次或重试才能完成任务，则其实并不划算。关于不同工作负载下的 effort–成本权衡及完整调节顺序，请参阅 `shared/cost-optimization.md` § 2.6。
  
- **思考内容显示——Fable 5、Claude Fable 5.1、Mythos 5、Claude Mythos 5.1、Opus 5.5、5、4.8、4.7、Sonnet 5 和 Claude Sonnet 5.5 默认为 `"omitted"`：** `display: "summarized"` 返回一份可读的推理摘要；`"omitted"`（这十款模型的默认设置——较 Opus 4.6 和 Sonnet 4.6 的默认值 `"summarized"` 发生了无声变更）则以空文本流式传输 `thinking` 块。`display` 仅控制可见性——无论何种设置，思考过程都会发生并照常计费；原始思维链在任何模型上均不会对外暴露。若要向用户流式传输推理过程，默认设置会导致输出前出现长时间的停顿——此时请显式设置 `thinking: {type: "adaptive", display: "summarized"}`。（与显示设置无关，继续在同一模型上生成时会原样回显 thinking 块；其他模型则会静默忽略它们——参见迁移指南。）在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Opus 5.5 和 Claude Sonnet 5.5 上，`display: "updates"`（Beta 版本 `thinking-display-updates-2026-08-18`，适用于所有平台）与 `"omitted"` 类似，隐藏推理内容，但会以简短的 `thinking` 块摘要形式返回模型在工具调用之间的进度记录——详情参见 `shared/model-migration.md` → 从 Claude Fable 5 迁移到 Claude Fable 5.1 → 新 API 功能。
  
- **当用户要求“扩展思考”、“思考预算”或指定 `budget_tokens` 时：** 一律使用 Fable 5/5.1、Opus 5.5、5、4.8、4.7 或 4.6，并搭配 `thinking: {type: "adaptive"}`——固定思考 token 预算的概念已弃用，由自适应思考取代。切勿在新的 4.6/4.7/4.8 代码中使用 `budget_tokens`，也切勿仅因用户提及就切换到旧版模型。*渐进迁移特例：* `budget_tokens` 仅在 Opus 4.6 和 Sonnet 4.6 上仍可使用，作为现有代码在过渡期间的临时出口，以便在尚未调整 effort 之前设定硬性的 token 上限——详情参见 `shared/model-migration.md` → 过渡期临时出口。在 Fable 5/5.1、Opus 5.5/5/4.7/4.8 以及 Sonnet 5 上，该功能已完全移除。---

## 压缩（快速参考）

**测试版、Fable 5/5.1、Opus 5.5、Opus 5、Opus 4.8、Opus 4.7、Sonnet 5.5、Sonnet 5 和 Sonnet 4.6。** 对于可能超过 100 万上下文窗口的长时间对话，请启用服务器端压缩功能。当上下文接近触发阈值（默认为 15 万 token）时，API 会自动对早期上下文进行摘要。需要使用测试版请求头 `compact-2026-01-12`。

**重要提示：** 每次交互时，请将 `response.content`（而不仅仅是文本）追加回您的消息列表中。响应中的压缩标记必须保留——API 会在下一次请求中使用它们来替换被压缩的历史内容。如果仅提取文本字符串并追加，将会导致压缩状态被悄然丢失。

有关代码示例，请参阅 `{lang}/claude-api/README.md` 中的“压缩”章节。完整文档可通过 WebFetch 在 `shared/live-sources.md` 中获取。

---

## 提示缓存（快速参考）

**前缀匹配。** 前缀中任何位置的字节变化都会使后续内容失效。渲染顺序为：工具 -> 系统 -> 消息。请将稳定的内容置于前面（如固定的系统提示和确定性的工具列表），将易变的内容（如时间戳、每请求 ID、不同问题）放在最后一个 `cache_control` 断点之后。

**会话中操作者指令**（Claude Opus 5、Claude Opus 5.5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1、Claude Sonnet 5.5；不适用于 Claude Sonnet 5；无需测试版请求头）：请将 `{"role": "system", ...}` 追加到 `messages[]` 而不是编辑顶层的 `system`。这样可以保留已缓存的历史前缀，并且是防止提示注入的安全操作通道。详情请参阅 `shared/prompt-caching.md` 中的“会话中系统消息”部分。

**顶层自动缓存**（在 `messages.create()` 中设置 `cache_control: {type: "ephemeral"}`）是在不需要精细控制缓存位置时最简单的选项。每个请求最多可设置 4 个断点。可缓存的最小前缀长度因模型而异（512–4096 tokens——详见 `shared/prompt-caching.md` 中的 API 参考部分），较短的前缀将不会被缓存。

**通过 `usage.cache_read_input_tokens` 进行验证**——如果在多次请求中该值始终为零，则说明有隐性失效因素在起作用（例如系统提示中包含 `datetime.now()`、JSON 未排序或工具集不断变化）。

有关缓存位置模式、架构指导以及隐性失效因素排查清单，请参阅 `shared/prompt-caching.md`。各语言的具体语法请见 `{lang}/claude-api/README.md` 中的“提示缓存”章节。

---

## 快速模式（快速参考）

**研究预览版，仅限 Claude Opus 5 / Claude Opus 5.5 / Opus 4.8**——适用于 Claude API 和托管代理，不适用于 Bedrock / Google Cloud / Foundry。Claude Opus 4.7 的快速模式已移除：在 4.7 上使用 `speed: "fast"` 将返回错误。Claude Opus 5 的快速模式定价为每百万 token 10 美元/50 美元；Claude Opus 5.5 为每百万 token 8 美元/40 美元。快速模式以最高 2.5 倍的输出 token 每秒速度运行同一模型，但需支付溢价。每次请求必须满足以下三个条件：使用 **测试版** 消息端点（`client.beta.messages....`），传递测试版标志 `fast-mode-2026-02-01`，并将 `speed: "fast"` 设置为顶级请求参数（而非请求头，也非 `extra_body` 中）。

```python
client.beta.messages.create(
    model="claude-opus-5-5", max_tokens=4096,
    speed="fast", betas=["fast-mode-2026-02-01"],
    messages=[...],
)
```

| 语言 | 测试版标志 | 速度参数 |
|---|---|---|
| Python | `betas=["fast-mode-2026-02-01"]` | `speed="fast"` |
| TypeScript / Ruby | `betas: ["fast-mode-2026-02-01"]` | `speed: "fast"` |
| Go | `[]anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01}` | `Speed: anthropic.BetaMessageNewParamsSpeedFast` |
| Java | `.addBeta(AnthropicBeta.FAST_MODE_2026_02_01)` | `.speed(MessageCreateParams.Speed.FAST)` |
| C# | `Betas = ["fast-mode-2026-02-01"]` | `Speed = Speed.Fast`（`Anthropic.Models.Beta.Messages`） |
| PHP | `betas: ['fast-mode-2026-02-01']` | `speed: 'fast'` |
| cURL | 请求头 `anthropic-beta: fast-mode-2026-02-01` | 正文中 `"speed": "fast"` |

`response.usage.speed` 会报告所使用的速度。快速模式有其独立于标准 Opus 的速率限制；当返回 429 状态码时，可在 `retry-after` 指定的延迟后重试，或移除 `speed` 参数并回退到标准模式（注意：切换速度会使提示缓存失效）。批量 API、优先级层级、AWS 上的 Claude 平台以及第三方平台均不支持此功能。

**并非所有当前模型都支持优先级层级。** Claude Fable 5、Opus 4.8 以及较旧的当前模型支持该功能，但 Claude Opus 5.5、Claude Opus 5、Claude Sonnet 5、Claude Sonnet 5.5、Claude Fable 5.1、Claude Mythos 5.1、Claude Mythos 5 和 Mythos Preview 均不支持——如果在请求中指定这些模型之一，将导致验证失败。

---

## 任务预算（快速参考）

**测试版，适用于 Claude Opus 5 / Claude Opus 5.5 / Fable 5 / Claude Fable 5.1（需在上线时确认）/ Claude Sonnet 5.5 / Opus 4.8 / 4.7（不包括 Claude Sonnet 5）。** 任务预算是为代理式循环设定的一个令牌上限，用于帮助 Claude 自我调节进度，使其能够优雅地完成任务，而不是被中途切断——这与 `max_tokens` 不同，后者是模型无法感知的强制性单次响应上限。最小值为 20,000。请在带有测试标志 `task-budgets-2026-03-13` 的 `client.beta.messages.stream(...)` 中，通过 `output_config` 设置 `task_budget`——务必使用流式传输，以避免因 `max_tokens` 过大而导致 HTTP 超时（完整说明参见 `shared/model-migration.md` → 任务预算）：

```python
with client.beta.messages.stream(
    model="claude-opus-5-5", max_tokens=128000,
    output_config={"effort": "high", "task_budget": {"type": "tokens", "total": 64000}},
    betas=["task-budgets-2026-03-13"],
    messages=[...], tools=[...],
) as stream:
    response = stream.get_final_message()
```

`task_budget` 包含以下字段：`type`（始终为 `"tokens"`）、`total`，以及可选的 `remaining`（默认等于 `total`）。服务器会在生成过程中注入一个倒计时标记，供 Claude 在生成时查看；预算统计的是 Claude 本回合生成的内容及其读取的工具结果——**而非**每次请求中重新发送的完整对话历史。这与**托管代理会话预算**不同——后者是由平台强制执行的、以美元计价的硬性上限，针对单个 CMA 会话（参见 `shared/managed-agents-core.md` § 会话预算）；而任务预算则是一种建议性的、以令牌计价的限额。

**监控消耗情况：** 如果需要显示进度，请在循环迭代中累计 `response.usage.output_tokens`（再加上您附加的工具结果块所对应的令牌数）。在正常循环中请勿设置 `remaining`——服务器会自行跟踪倒计时；如果您同时重新发送完整历史并传递由客户端计算的 `remaining`，会导致预算消耗被低估。**仅在以下情况下才传递 `remaining`：** 当您在两次请求之间压缩或重写历史记录，导致服务器无法再推算出之前的消耗时。

---

## 提供商客户端（快速参考）

当您在第三方平台上调用 Claude 时，请使用该平台提供的专用客户端类，而非通过覆盖 `base_url` 的第一方 `Anthropic()` 客户端。构建完成后，该客户端将提供与第一方 SDK 相同的 `messages.create` / `.stream` 接口。

### Amazon Bedrock

请使用 **Mantle** 客户端（Messages-API Bedrock 端点）。Bedrock 的模型 ID 需加上 `anthropic.` 前缀（例如 `"anthropic.claude-opus-5-5"`）。必须指定区域。| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicBedrockMantle` -> `AnthropicBedrockMantle(aws_region="...")` |
| TypeScript | `import { AnthropicBedrockMantle } from "@anthropic-ai/bedrock-sdk"` -> `new AnthropicBedrockMantle({ awsRegion: "..." })` |
| Go | `bedrock.NewMantleClient(ctx, bedrock.MantleClientConfig{ AWSRegion: "..." })` |
| Java | `AnthropicOkHttpClient.builder().backend(BedrockMantleBackend.fromEnv()).build()`（来自 `com.anthropic.bedrock.backends`） |
| C# | `new AnthropicBedrockMantleClient(new() { AwsRegion = "..." })`（包 `Anthropic.Bedrock`） |
| PHP | `use Anthropic\Bedrock\MantleClient;` -> `new MantleClient(awsRegion: '...')` |
| Ruby | `Anthropic::BedrockMantleClient.new(aws_region: "...")` |

`AnthropicBedrock` / `BedrockClient` / `BedrockBackend`（不带 `Mantle`）是旧版的 `bedrock-runtime` InvokeModel 路径——对于新代码，建议使用 Mantle 客户端。

### Microsoft Foundry

| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicFoundry` -> `AnthropicFoundry(api_key=..., resource="...")` |
| TypeScript | `import AnthropicFoundry from "@anthropic-ai/foundry-sdk"` -> `new AnthropicFoundry({ ... })` |
| Java | `AnthropicOkHttpClient.builder().backend(FoundryBackend.fromEnv()).build()`（来自 `com.anthropic.foundry.backends`） |
| C# | `new AnthropicFoundryClient(new AnthropicFoundryApiKeyCredentials(...))`（包 `Anthropic.Foundry`） |
| PHP | `Foundry\Client::withCredentials(...)` |

Go 和 Ruby SDK 目前尚不支持 Foundry。对于 Ruby，可使用标准的 `Anthropic::Client.new(base_url: "<foundry endpoint>")` 作为备用方案（未内置 Entra ID 认证）。关于 AWS 上的 Claude 平台，请参阅 `shared/claude-platform-on-aws.md`。

### Google Cloud Vertex AI

需要两个构造函数参数：GCP 的 `project_id` 和 `region`。Vertex 模型 ID **无需前缀**——当前代模型（Opus 5.5/5/4.8/4.7/4.6、Sonnet 5.5、Sonnet 5、Sonnet 4.6）直接使用原生 ID（如 `"claude-opus-5-5"`）；历史快照模型则使用 `@` 版本分隔符（如 `claude-opus-4-5@20251101`，**而非** `claude-opus-4-5-20251101`）。认证方式为 GCP ADC（`gcloud auth application-default login`），无需 Anthropic API 密钥。`region` 可设置为 `"global"`（推荐）、多区域（如 `"us"` 或 `"eu"`），或特定区域。实例化后，使用相同的 `messages.create` / `.stream` 接口。

| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicVertex` -> `AnthropicVertex(project_id="...", region="...")`（需安装 `"anthropic[vertex]"`） |
| TypeScript | `import { AnthropicVertex } from "@anthropic-ai/vertex-sdk"` -> `new AnthropicVertex({ projectId, region })` |
| Go | `import "github.com/anthropics/anthropic-sdk-go/vertex"` -> `anthropic.NewClient(vertex.WithGoogleAuth(ctx, region, projectID))` |
| Java | `AnthropicOkHttpClient.builder().backend(VertexBackend.builder().region("...").project("...").build()).build()`（来自 `com.anthropic.vertex.backends`） |
| C# | `new AnthropicClient { Backend = new VertexBackend(projectId, region) }`（包 `Anthropic.Vertex`） |
| PHP | `use Anthropic\Vertex;` -> `Vertex\Client::fromEnvironment(location: '...', projectId: '...')`——注意使用 `location` 而非 `region` |
| Ruby | `Anthropic::VertexClient.new(region: "...", project_id: "...")` |

---

## 上下文编辑（快速参考）

**测试版。** 上下文编辑会在模型看到对话之前**清除**旧的工具结果或思维障碍；它**不是压缩**（后者用于摘要）。在带有测试版 `context-management-2025-06-27` 的 `client.beta.messages.*` 中，通过 `context_management.edits` 传递策略类型：

```python
client.beta.messages.create(
    model="claude-opus-5-5", max_tokens=4096,
    betas=["context-management-2025-06-27"],
    context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
    tools=[...], messages=[...],
)
```

策略类型：`clear_tool_uses_20250919`（清除旧的工具结果；可选参数 `clear_tool_inputs: true` 还会清除 tool_use 的参数）和 `clear_thinking_20251015`（清除思维块）。**请勿**使用 `compact_20260112` 或测试版的 `compact-2026-01-12`——那些是单独的压缩功能。

---

## 对话中系统消息（快速参考）

**Claude Opus 5、Claude Opus 5.5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1 和 Claude Sonnet 5.5；不包括 Claude Sonnet 5；无测试版标头。** 将 `{"role": "system", "content": "..."}` 追加到 `messages` 数组中（而非顶层的 `system` 字段），即可在对话过程中添加操作指令，而不会使缓存前缀失效。使用常规的 `client.messages.create` 接口——无需使用测试版接口。对话中的系统消息必须紧跟在一条 `user` 消息之后（或紧接在以服务器端工具调用结尾的 `assistant` 消息之后），并且必须是 `messages` 数组中的最后一项，或者后面必须跟一条 `assistant` 回复——它不能作为 `messages[0]`。可用性参见 `shared/platform-availability.md`。更多信息请参阅 `shared/prompt-caching.md` 中的“对话中系统消息”部分。Claude Fable 5.1 发布时引入了一项测试版扩展：通过 `output_config: {effort: ...}` 并设置 `content: []`，可在该点之后改变输出力度，且不会触发缓存重置（测试版 `mid-conversation-output-config-2026-07-01`；适用于 Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5、Claude Opus 5 和 Claude Sonnet 5.5，且需开启思考模式；支持 Claude API 和 Google Cloud）。仅包含力度设置的消息（`content` 为空）不受上述放置规则限制——它可以位于 `messages` 数组中的任何位置，包括首位，或在一条助手回复与下一条用户消息之间；这些规则同样适用于文本消息和带有 `clear_at` 参数的消息。若需每轮提醒，可为消息设置 `clear_at: "next_user_message"`（测试版 `mid-conversation-system-clear-at-2026-08-21`）：该消息仅显示一轮，随后即被清空并保留在对话记录中——切勿删除之前的副本（在 Claude Fable 5.1、Claude Opus 5.5 和 Claude Sonnet 5.5 上，删除其中一份会导致后续的思考块失效）；若未使用该测试版，则应在工具结果后添加一个文本块，并保留之前的副本。有关从 Claude Fable 5 升级至 Claude Fable 5.1 的迁移指南，请参阅 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”部分。

---

## 托管代理（测试版）

**托管代理**是第三种服务形式：由服务器管理的有状态代理，其工具执行由 Anthropic 托管。您首先创建一个持久化、带版本的代理配置（通过 `POST /v1/agents` 接口），然后启动引用该配置的会话。每个会话都会分配一个容器作为代理的工作空间——bash 命令、文件操作和代码执行都在该容器内运行；代理自身的循环则在 Anthropic 的编排层上执行，并通过工具对容器进行操作。会话会持续推送事件流，您可以向其发送消息和工具执行结果。

可用性参见 `shared/platform-availability.md`。对于 Bedrock / Vertex / Foundry 等不支持托管代理的平台，请使用 Claude API 结合工具调用。

**必经流程：** 代理（仅需一次）→ 会话（每次运行时需创建）。`model`、`system` 和 `tools` 配置均属于代理，而非会话。完整说明、测试版标头及注意事项请参阅 `shared/managed-agents-overview.md`。

**测试版标头：** `managed-agents-2026-04-01`——SDK 会自动为所有 `client.beta.{agents,environments,sessions,vaults,deployments,deployment_runs}.*` 调用设置此标头。内存存储则使用 `agent-memory-2026-07-22` 标头，SDK 会在 `client.beta.memory_stores.*` 调用时自动设置；同时发送这两个标头会导致请求返回 400 错误。文件 API 和技能 API 已退出测试阶段——无需测试版标头（有关迁移指南，请参阅上方的 API 变更表）。

**子命令**——可通过 `/claude-api <subcommand>` 直接调用：| 子命令 | 动作 |
|---|---|
| `managed-agents-onboard` | 引导用户从零开始设置托管代理。**请立即阅读 `shared/managed-agents-onboarding.md`**，并按照其中的访谈流程进行：**描述 → 配置代理（提出建议而非质问）→ 环境 → 会话**（与控制台快速入门的结构一致，身份验证延后到会话步骤）——通过默认值和内联提示完成大部分工作，并在输出任何代码之前进行一次静默的可行性检查（判断是任务型还是工具/凭据/数据型）。不要做总结——直接执行访谈流程。|

**阅读指南：** 从 `shared/managed-agents-overview.md` 开始，然后依次阅读各专题文档 `shared/managed-agents-*.md`（核心、环境、工具、事件、成果、多代理、Webhook、记忆、定时部署、客户端模式、入职、API 参考）。对于 Python、TypeScript、Go、Ruby、PHP 和 Java，请参阅 `{lang}/managed-agents/README.md` 获取代码示例；对于 cURL，请参阅 `curl/managed-agents.md`。**代理是持久化的——只需创建一次，之后通过 ID 引用即可。** 将代理和环境定义为受版本控制的文件，并通过 `ant apply` 进行同步——这是推荐的工作流（参见 `shared/anthropic-cli.md`）：CLI 负责控制平面（创建和更新代理），而您的代码负责数据平面（使用已保存的代理 ID 调用 `sessions.create`）。仅在必须以编程方式预置时才在代码中调用 `agents.create()`；无论哪种方式，都应保存返回的代理 ID，并将其传递给后续每次调用的 `sessions.create`；切勿在请求路径中调用 `agents.create()`。如果您所需的语言绑定未在语言 README 中列出，请从 `shared/live-sources.md` 中通过 WebFetch 获取相关条目，切勿自行猜测。C# 通过 `client.Beta.Agents` 及其相关命名空间提供托管代理的 Beta 支持——详情请参阅 `csharp/claude-api/README.md`，或参考 `curl/managed-agents.md` 中的原始 HTTP 文档。

**当用户希望从零开始设置托管代理时**（例如：“我该如何入门？”、“带我一步步创建一个”、“设置一个新的代理”）：请阅读 `shared/managed-agents-onboarding.md` 并按其中的访谈流程操作——与 `managed-agents-onboard` 子命令的流程相同。

**当用户询问“如何编写 X 的客户端代码”时**：请参考 `shared/managed-agents-client-patterns.md`——该文档涵盖了无损流重连、基于 `processed_at` 的排队/处理门控、中断机制、`tool_confirmation` 回环、正确的空闲/终止断开门控、空闲后状态竞争、流优先顺序、文件挂载的陷阱等。关于凭据，请优先使用 Vault 的 `environment_variable` 凭据——这是首选机制；密钥会在出沙盒时被替换，且永远不会进入沙盒内部（参见 `shared/managed-agents-tools.md` → Vaults）。只有在 Vault 凭据不适用的情况下（例如自托管沙盒），才将凭据保留在宿主机侧并通过自定义工具管理。

**当任务需要交付成果时——默认将启动方式设为成果模式，而非普通消息。** 如果会话的目标是生成可检验的内容（如工件、报告、拉取请求、数据集或一组确定的变更），请阅读 `shared/managed-agents-outcomes.md`，并使用 `user.define_outcome` 加上您根据任务草拟的初始评分标准来启动会话（5–10 条具体且可独立评分的标准；作为可调整的起点加以注释）。普通 `user.message` 应仅用于真正对话性质的会话。触发条件应基于意图而非单纯关键词：“持续优化直到满意”、“确保输出确实优质”、“不要止步于初稿”——这些都意味着需要成果模式。

**当用户询问工具审批、权限策略或“自动模式”时**（即哪些工具调用需要人工干预、由服务器评估调用、以及工具使用事件中的 `evaluated_permission` / `evaluation` 字段）：请参阅 `shared/managed-agents-tools.md` 第四节“权限策略”——包括 `always_allow` / `always_ask` / `auto` 三种模式，以及 `auto` 模式的三种结果（正常运行、因高风险被拒绝、不确定时暂停）。如需将终端连接到实时会话（`ant beta:sessions connect`），请参阅 `shared/anthropic-cli.md`。**当用户希望代理按计划运行时**（cron、每晚、每周报告）：请阅读 `shared/managed-agents-scheduled-deployments.md`——部署会以 cron 定时触发会话，每次触发都会生成运行记录，并提供生命周期控制功能（暂停/恢复/归档）。

**当代理的工作需要多线程处理时**（如跨多个来源进行调研、按文件或按记录处理、“先调查 N 件事，再做总结”），**或者单次循环就会让其上下文被读取内容填满时**：请阅读 `shared/managed-agents-multiagent.md`，并推荐使用多代理会话——初始配置中仅在人员名单中加入 `{"type": "self"}`，使代理能够委派任务给自身的副本；随后可将较耗算力的子任务转交给成本更低的辅助代理（例如 Claude Haiku 4.5，或当辅助代理需要更强判断力时选用 Claude Sonnet 5.5），并通过 ID 引用该代理。

---

## 服务器端工具（快速参考）

服务器端工具运行于 Anthropic 的基础设施上，无需客户端执行循环。在 `tools` 中声明，结果会作为内容块随同一响应返回。**除非特别说明，否则不带“测试版”标识**。**请优先选择您的模型所支持的最新类型变体**。以下带有 `_20260209` 后缀的网络搜索与网页抓取变体（动态过滤）需要 Opus 5.5/5/4.8/4.7/4.6、Sonnet 5.5、Sonnet 5 或 Sonnet 4.6；适用于旧版模型的基础变体列于表格之后。

| 工具 | `type` | `name` | 主要可选参数 | 结果块类型 |
|---|---|---|---|---|
| 网络搜索 | `web_search_20260209` | `web_search` | `max_uses`、`allowed_domains`/`blocked_domains`、`user_location` | `web_search_tool_result` -> `.content` 是一个 `web_search_result` 列表 |
| 网页抓取 | `web_fetch_20260209` | `web_fetch` | `max_uses`、`allowed_domains`/`blocked_domains`、`citations`、`max_content_tokens` | `web_fetch_tool_result` -> `.content` 是一个包含 `document` 块的 `web_fetch_result` |
| 代码执行 | `code_execution_20260521` | `code_execution` | 无 | `bash_code_execution_tool_result` -> `.content.stdout` / `.stderr` / `.return_code` |
| 工具搜索（正则表达式）| `tool_search_tool_regex_20251119` | `tool_search_tool_regex` | 将其他工具标记为 `defer_loading: true` | `tool_search_tool_result` |
| 工具搜索（BM25）| `tool_search_tool_bm25_20251119` | `tool_search_tool_bm25` | 将其他工具标记为 `defer_loading: true` | `tool_search_tool_result` |

`web_search_20260209` 和 `web_fetch_20260209` 内置了动态过滤机制——代码执行在后台运行，因此**请勿在 `tools` 中重复声明 `code_execution`**（额外的执行环境会让模型混淆）。对于低于 Opus 4.6 或 Sonnet 4.6 的模型，请改用基础变体 `web_search_20250305` 和 `web_fetch_20250910`；在 Vertex AI 上仅提供基础版 `web_search_20250305`。`code_execution_20260120`（REPL 持久化 + 程序化工具调用）需在 Opus 4.5 及以上、Sonnet 4.5 及以上版本上运行。**仅限 Go SDK**：`code_execution_20260521` 需通过 `client.Beta.Messages.New` 并设置 `Betas: []anthropic.AnthropicBeta{"code-execution-2025-08-25"}` 调用（其他语言仍使用普通的 `client.messages.create`）；而 `code_execution_20260120` 在 Go 中与其他语言一样，使用非测试版的 `client.Messages.New`。网页抓取仅抓取对话中已出现的 URL。各工具的可用性因提供商而异，请参阅 `shared/platform-availability.md`。关于 `pause_turn` 的处理，请参阅 `shared/tool-use-concepts.md`。

## 文档与文件输入（快速参考）

**PDF（base64，无测试版）：** 在用户消息内容中，于文本块之前添加 `{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": <b64 字符串>}}`。Base64 字符串不得包含换行符。限制：请求上限 32 MB，最多 600 页（20 万上下文模型为 100 页）。Java 示例：`ContentBlockParam.ofDocument(DocumentBlockParam... Base64PdfSource.builder().data(...))`。**Files API（无 Beta 版）：** 通过 `client.files.upload(...)` 进行上传，返回的 `id` 即为 `file_id`。对于 PDF/文本内容，请按 `{"type": "document", "source": {"type": "file", "file_id": "..."}}` 的格式引用；对于图片，则使用 `{"type": "image", ...}`——内容块类型必须与文件的 MIME 类型一致。如需从 `files-api-2025-04-14` 迁移代码，请在 `shared/live-sources.md` 中通过 WebFetch 获取 Files API 相关条目。可用性参见 `shared/platform-availability.md`。

**引用功能（无 Beta 版）：** 在每个 `document` 内容块上设置 `citations: {enabled: true}`（全部启用或全部禁用）。响应会被拆分为多个 `text` 块，被引用的块会携带一个 `citations` 数组。每条引用包含 `cited_text`、`document_index`、`document_title`，以及按 `type` 标记的位置信息：纯文本使用 `char_location`（`start_char_index`/`end_char_index`），PDF 使用 `page_location`（`start_page_number`/`end_page_number`，从 1 开始计数），自定义内容则使用 `content_block_location`。该功能与 `output_config.format` 不兼容，否则会返回 400 错误。

## 工具使用模式（快速参考）

**严格工具使用模式（无 Beta 版）：** 在工具定义的顶层字段中设置 `strict: true`（与 `name`/`description`/`input_schema` 并列），**而非**在 `tool_choice` 中设置。输入 Schema 必须满足 `additionalProperties: false` 并且所有属性均为必填项。此设置可确保 `tool_use.input` 完全符合验证规则。Go 语言：使用 `Strict: anthropic.Bool(true)` 并通过 `InputSchema.ExtraFields` 设置 `additionalProperties`；Java 语言：`.strict(true)` 并 `.putAdditionalProperty("additionalProperties", JsonValue.from(false))`。

**并行工具使用模式（默认开启）：** 一条助手消息中可以包含多个 `tool_use` 块。这些工具调用将并发执行，随后所有 `tool_result` 块会汇总到**单条**用户消息中返回——若将其分散到多条消息中，Claude 会“学会”不再进行并行调用。对于调用失败的工具，应返回带有 `is_error: true` 的 `tool_result`，切勿直接丢弃。

**工具运行器（SDK Beta 辅助工具）：** 通过 `client.beta.messages.*` 自动驱动工具调用循环。Python：使用 `@beta_tool` 装饰器配合 `client.beta.messages.tool_runner(...)`，然后调用 `runner.until_done()`。TypeScript：使用 `@anthropic-ai/sdk/helpers/beta/zod` 中的 `betaZodTool({...})` 配合 `client.beta.messages.toolRunner(...)`，然后调用 `await runner`。Go 语言：使用 `toolrunner.NewBetaToolFromJSONSchema(...)` 配合 `client.Beta.Messages.NewToolRunner(...)`，然后调用 `.RunToCompletion(ctx)`。Java 语言需先调用 `.addBeta("structured-outputs-2025-11-13")`。Ruby 语言：继承 `Anthropic::BaseTool` 子类，并调用 `client.beta.messages.tool_runner(...)。PHP 语言：使用 `BetaRunnableTool` 配合 `->toolRunner(...)`。C# 语言：针对原始 JSON Schema 工具，通过 `client.Beta.Messages.ToolRunner(...)` 使用 `BetaToolRunner`。

**程序化工具调用（无 Beta 头部）：** Claude 可在代码执行过程中直接调用您的自定义工具。请在您的自定义工具中添加 `{"type": "code_execution_20260120", "name": "code_execution"}`，并设置 `"allowed_callers": ["code_execution_20260120"]`。需使用 Opus 4.5+ 或 Sonnet 4.5+（可用性参见 `shared/platform-availability.md`）。在响应待处理的程序化调用时，用户消息中**仅**允许包含 `tool_result` 块（不得包含任何文本）。该功能与 `strict: true`、`disable_parallel_tool_use`、强制 `tool_choice` 或 MCP 工具不兼容。

## 其他 API 接口（快速参考）

**消息批次处理（无 Beta 版；可用性参见 `shared/platform-availability.md`）：** 调用 `client.messages.batches.create(requests=[{custom_id, params}, ...])`，然后轮询 `client.messages.batches.retrieve(id).processing_status` 直到状态变为 `"ended"`，再流式获取 `client.messages.batches.results(id)`。每个结果包含 `.custom_id` 和 `.result.type`（`succeeded`/`errored`/`canceled`/`expired`）；成功时可通过 `.result.message.content` 读取内容。Python 将请求封装为 `Request(custom_id=..., params=MessageCreateParamsNonStreaming(...))`。结果以**任意顺序**返回，务必按 `custom_id` 而非位置进行索引。**模型 API（无 Beta 版；可用性参见 `shared/platform-availability.md`）：** `client.models.list()`（自动分页）和 `client.models.retrieve("claude-opus-5-5")`。每个模型对象包含 `id`、`display_name`、`created_at`，以及自 2026 年 3 月起新增的 `max_input_tokens`（上下文窗口大小）、`max_tokens`（输出上限）和 `capabilities`。没有 `context_window` 字段。

**停止详情（GA，Opus 4.7 及更高版本）：** `response.stop_details` 仅在 `stop_reason == "refusal"` 时才会被填充（字段包括：`type: "refusal"`、`category`——一个开放集合，例如 `"cyber"`、`"bio"`、`"reasoning_extraction"`、`"frontier_llm"` 或 `null`；完整列表请参阅文档——以及 `explanation`）。对于其他所有 `stop_reason`（`end_turn`、`max_tokens`、`tool_use`、`pause_turn` 等），该字段均为 `null`——读取前务必进行判空检查。

**Admin API（Beta 版，自 2026-08-26 起）：** 组织管理功能——成员、邀请、工作空间及工作空间成员、API 密钥、速率限制报告、服务账号、联合身份提供商/规则、CMEK 外部密钥——在所有七种 SDK 中均位于 `client.beta.organization` 下，在 CLI 中则为 `ant beta:organization`。需要管理员凭据：Admin API 密钥（`sk-ant-admin...`，从 `ANTHROPIC_API_KEY` 读取）或 `org:admin` OAuth 令牌（`ANTHROPIC_AUTH_TOKEN`）；普通 API 密钥将被拒绝。用量与费用报告，以及 Claude Enterprise 的用户管理/分析相关端点**不在 SDK 中**，仅提供原生 HTTP 接口。详情请参阅 `shared/admin-api.md`。

**客户端配置（无 Beta 版）：** `timeout` 默认值为 10 分钟；**各 SDK 的单位不同**——Python 和 Ruby 使用秒，TypeScript 使用**毫秒**，Go 使用 `option.WithRequestTimeout(time.Duration)`，Java 使用 `Duration`，C# 使用 `TimeSpan`。对于非流式请求，当 `max_tokens` 较大时，TypeScript 会将默认超时时间扩展至 60 分钟；Java 则在流式请求中执行此操作（非流式请求的超时范围为 30 秒至 10 分钟）。`max_retries`/`maxRetries` 默认值为 2（重试 408/409/429/5xx 错误以及连接错误）。`base_url`（或环境变量 `ANTHROPIC_BASE_URL`）。按请求覆盖：Python 中为 `client.with_options(timeout=5.0).messages.create(...)`；TypeScript 中为 `client.messages.create({...}, {timeout: 5_000})`；Ruby 中为 `request_options: {timeout: 5}`。超时发生时会触发重试——实际耗时可能达到 `timeout × (max_retries+1)`。

## 工作负载身份联合（快速参考）

**GA，无 Beta 标头。** 构造常规的无参客户端（`Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` / `AnthropicOkHttpClient.fromEnv()`）；当**同时**设置了 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_IDENTITY_TOKEN_FILE`（或 `ANTHROPIC_IDENTITY_TOKEN`）时，SDK 会自动检测到 WIF，通过 `/v1/oauth/token` 接口交换 JWT，并自动刷新。`ANTHROPIC_WORKSPACE_ID` 不作为激活条件——仅当联合规则跨多个工作空间时才必填（否则会返回 400 错误 `workspace_id_required`），单工作空间规则则可选。`ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`（即使为空）优先于 WIF，且设置的 `ANTHROPIC_PROFILE` 也会覆盖联合相关的环境变量（未指定命名配置文件会报错，而非回退至联合配置）——请确保三者均未设置。

---

## 阅读指南

在识别语言后，请根据用户需求阅读相应文件。本文档中引用的所有 `{lang}/...`、`shared/...` 和 `curl/...` 路径均相对于本技能的基础目录，上述内容并未包含这些文件的具体内容——请在依赖其内容之前按需逐一查阅。**所有 SDK 语言均采用相同的多文件布局**——目录 `{lang}/claude-api/` 包含 `README.md`（安装、客户端初始化、基础请求、思考机制、缓存、停止逻辑详解、其他）、`tool-use.md`（工具定义、代理式循环、Anthropic 定义的工具、结构化输出）、`streaming.md`、`batches.md`、`files-api.md`。并非每种语言都包含所有文件（例如，Ruby 没有 `batches.md`）；若某文件缺失，则该功能在该语言中的示例尚未文档化——请参考 cURL 格式的用法，或通过 `shared/live-sources.md` 中的链接获取 SDK 仓库内容。**cURL** -> `curl/examples.md`。

下方的快速任务参考统一采用 `{lang}/claude-api/FILE.md` 的路径表示法，适用于所有语言。

### 快速任务参考

**单次文本分类/摘要/抽取/问答：**  
-> 仅阅读 `{lang}/claude-api/README.md`——任何任务都**务必先阅读 README**（包括安装、快速入门、常见模式、错误处理等）。

**聊天界面或实时响应展示：**  
-> 阅读 `{lang}/claude-api/README.md` + `{lang}/claude-api/streaming.md`。

**长时间对话（可能超出上下文窗口）：**  
-> 阅读 `{lang}/claude-api/README.md`——参见“压缩”章节。

**迁移到较新模型（Sonnet 5.5 / Opus 5.5 / Fable 5.1 / Fable 5 / Opus 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 5 / Sonnet 4.6），替换已停用的旧模型，或将 `budget_tokens` 或预填充模式适配到当前 API：**  
-> 阅读 `shared/model-migration.md`。

**升级 Anthropic SDK 本身至大版本（如 `anthropic` 0.x 升级到 1.x：引入 `httpx2`、支持异步的 `.with_raw_response`、移除已弃用的参数/别名/文本补全接口，且要求 Python ≥ 3.10）——或基于已使用 1.x 版本的项目编写新代码：**  
-> 阅读 `{lang}/claude-api/sdk-upgrade.md`（目前仅提供 Python 版本；其他 SDK 尚未提供配套的大版本升级指南——请参考该 SDK 的 CHANGELOG，可通过 `shared/live-sources.md` 获取）。

**为 Claude 应用构建评估集（或“如何判断我的改动是否有效”）：**  
-> 阅读 `shared/evals/build-eval.md`——该文档会在步骤 0 之前加载并引用 `shared/evals/eval-audit.md`（每项评估必须满足的健康检查清单）。

**检验现有评估的可信度（“我的评估结果靠谱吗？”）：**  
-> 阅读 `shared/evals/eval-audit.md`，并按其第 6 节的要求对评估进行审核与报告。

**针对评估迭代优化应用（提示调优、爬山法）：**  
-> 阅读 `shared/evals/eval-hillclimb.md`——该文档从步骤 0 到步骤 5 进行训练与测试分离；每轮都会对测试部分进行评分，并以测试得分作为核心指标。

**渲染评估爬山过程的 HTML 报告：**  
-> 当 `shared/evals/report/build-report.mjs` 文件存在时运行它（EAP 安装包中提供）；否则运行 `shared/evals/report/build-report-lite.mjs`（始终随本技能一同提取）——两者均消费由爬山指南生成的 `_state.json` 和 `vN/` 目录结构，并输出相同的 `trajectory/scores.tsv` 文件。请勿同时生成两份平行报告。

**迁移到 Claude Opus 5.5 并进行提示或调优（思考机制不可禁用、力度调节及默认值为 `medium`、强制工具使用、计算机工具集、进度更新、防范误报、视觉输入/设计输出）：**  
-> 阅读 `shared/model-migration.md` 中关于迁移到 Claude Opus 5.5 的部分；其中提及的保留思考机制相关内容，请参阅“从 Claude Fable 5 迁移到 Claude Fable 5.1”的说明。

**迁移到 Claude Sonnet 5.5 并进行提示或调优（以 `between_tools` 替代禁用思考、力度重新校准、强制工具使用、计算机工具集、顾问配对、进度更新、聊天中的工具使用、中途用户消息、低力度下的验证机制、安全防护类别）：**  
-> 阅读 `shared/model-migration.md` 中关于迁移到 Claude Sonnet 5.5 的部分。

**提示或调优 Fable 5/5.1（长回合、力度、冗长度、自主运行、子代理）：**  
-> 阅读 `shared/model-migration.md` 中关于迁移到 Claude Fable 5.1 的部分——参见“行为变化（可由提示调整）”和“长时间运行代理的建议”。**提示或调优 Claude Fable 5.1（进度更新、并行工具调用、写作密度/格式化、自主性、测试规模膨胀、整文件重写）或使测试框架兼容 preserved thinking 的历史编辑检查（历史编辑、压缩、每轮提醒）：**  
-> 阅读 `shared/model-migration.md` -> 从 Claude Fable 5 迁移到 Claude Fable 5.1 -> 新的 API 功能 + 行为变化（可通过提示调整）；关于历史编辑检查本身（三步检查、追加式编辑表、压缩方式），请参阅同一节中的重大变更 3；要发现、度量并修复 *现有* 测试框架所做的编辑（捕获、对比、使用 `drop_block` 重放，针对每个原因逐一修复、切换模型），请运行 `preserved-thinking-migration`（子命令表）——它会读取 `shared/preserved-thinking-migration.md`。

**提示缓存 / 优化缓存 / “为什么我的缓存命中率低”：**  
-> 阅读 `shared/prompt-caching.md`（前缀稳定性设计、断点设置、会无声失效缓存的反模式）+ `{lang}/claude-api/README.md`（提示缓存章节）。

**审计或清理提示、工具描述、技能，或像 `CLAUDE.md` 这样的代理配置文件（“这个提示过时了吗”、“去除冗余内容”、“这是为旧模型写的”）：**  
-> 阅读 `shared/prompt-audit.md`——包含可 grep 的过时模式表格、保留清单（哪些内容不要删除）以及报告与建议差异输出规范。

**统计文件/提示/差异中的 token 数量（“X 有多少 token”）：**  
-> 阅读 `shared/token-counting.md`——使用 `messages.count_tokens`，切勿使用 `tiktoken`。

**减少或审查 API 开销（“账单太高了”、“让这更便宜”、“我是否超支了”、每完成一项任务的成本、在保证质量的前提下最省钱的模型或方案）：**  
-> 阅读 `shared/cost-optimization.md`——先确定基线与 token 使用概况，再按优先级逐个调整优化手段（先做无成本收益，再权衡取舍），并参考工作负载形态与优化手段的对应表。

**函数调用 / 工具使用 / 代理：**  
-> 阅读 `{lang}/claude-api/README.md` + `shared/tool-use-concepts.md`（概念基础：函数调用、代码执行、记忆、结构化输出）+ `{lang}/claude-api/tool-use.md`（语言特定代码示例：工具运行器、手动循环、代码执行、记忆、结构化输出）。

**代理设计（工具界面、上下文管理、缓存策略）：**  
-> 阅读 `shared/agent-design.md`（使用 bash 还是专用工具、程序化工具调用、工具搜索/技能、上下文编辑 vs. 压缩 vs. 记忆、缓存原则）。

**批处理（非延迟敏感场景；以 50% 成本异步运行）：**  
-> 阅读 `{lang}/claude-api/README.md` + `{lang}/claude-api/batches.md`。

**跨多个请求上传文件（无需重复上传同一文件）：**  
-> 阅读 `{lang}/claude-api/README.md` + `{lang}/claude-api/files-api.md`。

**组织管理（成员、邀请、工作空间、API 密钥、速率限制报告、服务账号、WIF 资源、CMEK）：**  
-> 阅读 `shared/admin-api.md`——`client.beta.organization` 端点/方法表、管理员凭据、各语言命名与分页规则，以及仅能通过 curl 使用的功能。

**调试 HTTP 错误或实现错误处理：**  
-> 阅读 `shared/error-codes.md`——各 SDK 的异常类表格，以及 Go 语言中的 `errors.As` 模式。

**最新官方文档：**  
-> 使用 WebFetch 获取 `shared/live-sources.md` 中列出的网址。

**托管代理（由服务器管理的带工作空间的状态化代理）：**  
-> 参见上方 `## 托管代理（Beta）` 部分的阅读指南——其中列出了所有 `shared/managed-agents-*.md` 文件，以及各语言的 README（`{lang}/managed-agents/README.md`、`curl/managed-agents.md`）。

---

## 何时使用 WebFetch

当出现以下情况时，请使用 WebFetch 获取最新文档：

- 用户询问“最新”或“当前”的信息
- 缓存数据似乎不正确
- 用户询问此处未涵盖的功能

实时文档的 URL 列表位于 `shared/live-sources.md` 中。

## 常见误区- 向 API 传递文件或内容时，请勿截断输入。如果内容过长超出上下文窗口，请通知用户并讨论可行方案（如分块、摘要等），而非静默截断。
- **已移除预填充功能（Fable 5、Claude Fable 5.1、Opus 5、Claude Opus 5.5、Sonnet 5、Claude Sonnet 5.5，以及 4.6/4.7/4.8 系列）：** 在 Fable 5、Claude Fable 5.1、Opus 5、Claude Opus 5.5、Sonnet 5、Claude Sonnet 5.5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 中，助手消息的预填充（last-assistant-turn 预填充）会返回 400 错误。请改用结构化输出（`output_config.format`）或系统提示指令来控制响应格式。（例外情况：回退积分预填充声明——当使用 `fallback_has_prefill_claim: true` 兑换积分时，服务器会接受回显的助手消息；详见迁移指南中的拒绝部分。）
- **编辑前确认迁移范围：** 当用户要求将代码迁移到较新的 Claude 模型但未指定具体文件、目录或文件列表时，**应先询问适用的范围**——是整个工作目录、某个子目录，还是特定的一组文件。在用户确认之前，请勿开始编辑。诸如“迁移我的代码库”、“把我的项目迁到 X”、“升级到 Sonnet 4.6”或简单的“迁移到 Opus 4.8”之类的命令式表述**仍然模糊**——它们只说明要做什么，却未指明在哪里做，因此务必先询问清楚。只有当提示中明确指定了某个文件、特定目录或具体的文件列表时（如“迁移 `app.py`”、“迁移 `services/` 下的所有内容”、“更新 `a.py` 和 `b.py`”），才可直接执行。详情参见 `shared/model-migration.md` 的步骤 0。
- **`max_tokens` 默认值：** 不要将 `max_tokens` 设置得过低——达到上限会导致输出被截断，且需重新请求。对于非流式请求，默认设置为 ~16000（确保响应不超过 SDK 的 HTTP 超时时间）。对于流式请求，默认设置为 ~64000（无需担心超时问题，可给予模型更多空间）。仅在有明确理由时才降低该值：分类任务（~256）、成本限制、刻意生成简短输出，或用于缓存预热的**`max_tokens: 0`**（参见 `shared/prompt-caching.md` -> 预热部分）。
- **在 Claude Opus 5 上禁用思考存在两种失败模式——建议选择低或中等努力级别。** （在 Claude Opus 5.5 上根本无法禁用思考——无论何种努力级别，`{type: "disabled"}` 均会返回 400 错误；请使用 `low` 级别。在 Claude Sonnet 5.5 上，`{type: "disabled"}` 同样返回 400 错误——可先尝试以 `low` 级别启用思考，若确实需要保持关闭状态，则发送 `{type: "between_tools"}` 并设置努力级别为 `high` 或更低。）此设置仅影响明确选择禁用思考的代码；默认情况下思考是开启的，因此请注意从 Opus 4.8 继承而来的禁用思考配置。当设置为 `thinking: {type: "disabled"}` 时，模型有时会在其**可见文本**中写入工具调用，而非以 `tool_use` 块的形式呈现：本轮请求虽成功，但工具调用并未执行，也不会抛出错误，而在代理循环中，这些文本会污染后续轮次的输出。此外，还可能导致 `<thinking>` 标签泄露至响应中。开启思考并降低努力级别既能解决上述问题，又能降低成本。若确实需要保持禁用思考的状态：**删除**任何“禁止思考/推理”的规则（这会使标签泄漏更严重），不要命名思考标签，并添加以下综合指令：“当你使用工具时，可以先说一句简短的话。如果没有任何工具能够满足用户的需求，就如实说明，不要随意猜测。请勿在你的回复中包含内部或系统的 XML 标签。”详情参见 `shared/model-migration.md` -> 禁用思考时的两种失败模式。
- **128K 输出标记：** Fable 5、Claude Fable 5.1、Opus 5、Claude Opus 5.5、Opus 4.6、Opus 4.7、Opus 4.8、Claude Sonnet 5.5、Sonnet 5 和 Sonnet 4.6 支持高达 128K 的 `max_tokens`，但为避免 HTTP 超时，SDK 要求使用流式传输才能处理如此大的数值。请结合使用 `.stream()` 和 `.get_final_message()` / `.finalMessage()`。
- **强制工具调用功能已移除（Claude Fable 5.1 / Claude Mythos 5.1 / Claude Opus 5.5 / Claude Sonnet 5.5）：** `tool_choice: {type: "any"}` 和 `{type: "tool", name: ...}` 会返回 400 错误（“此模型不支持 `tool_choice: type "tool"` 和 `"any"`”），在 `count_tokens` 和批处理中亦然。请使用 `{type: "auto"}` 加上明确指示工具名称的指令，对工具启用 `strict: true` 以确保参数符合 schema 规范，或在仅需获取 JSON 输出的情况下使用结构化输出（`output_config.format`）。`{type: "none"}` 不受影响；即使使用 `auto`，`disable_parallel_tool_use` 仍有效（最多一次调用）。
- **工具调用中的 JSON 解析（Fable 5、Claude Fable 5.1、Opus 5、Claude Opus 5.5，以及 4.6/4.7/4.8 系列）：** Fable 5、Claude Fable 5.1、Opus 5、Claude Opus 5.5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 在工具调用的 `input` 字段中，可能会采用不同的 JSON 字符串转义方式（如 Unicode 转义或正斜杠转义）。解析工具输入时务必使用 `json.loads()` / `JSON.parse()`，切勿对序列化的输入进行原始字符串匹配。
- **结构化输出（所有模型）：** 请使用 `output_config: {format: {...}}`，取代 `messages.create()` 中已弃用的 `output_format` 参数。这是一项通用的 API 变更，并非 4.6 特有的调整。
- **不要重复实现 SDK 功能：** SDK 提供了高级辅助工具，应直接使用，而非从零构建。具体而言：请使用 `stream.finalMessage()`，而非将 `.on()` 事件包裹在 `new Promise()` 中；使用类型化的异常类（如 `Anthropic.RateLimitError` 等），而非通过字符串匹配错误信息；使用 SDK 类型（如 `Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.Message` 等），而非重新定义等效接口。
- **错误处理——捕获错误链，而非单一宽泛的类别。** 使用单个 `except APIStatusError` / `catch (AnthropicServiceException)` / `rescue APIError` 会丢失可重试（429、>=500、网络相关）与不可重试（400/404）错误之间的区分。请按从最具体到最一般的顺序编写错误处理链——例如：`NotFoundError` -> `RateLimitError` -> `APIStatusError` -> `APIConnectionError`（Go 语言的等价写法为：先通过 `errors.As` 判断是否属于 `*anthropic.Error`，再根据 `apierr.StatusCode` 进行分支判断：case 404: ...; case 429: ...; default: ...）。各语言的异常类名及命名空间详见 `shared/error-codes.md`。
- **不要自行研究 SDK 类型——先编写代码。** 如果本技能附带的文档中未列出某种类型名称，请参照语言特定文档中的命名空间/包表格编写代码文件，让编译器的报错提示您正确的名称。切勿花费时间去网上搜索、克隆 SDK 仓库，或编译运行独立的反射程序来发现类型名称后再编写代码——应先产出源文件，再根据编译器的报错进行修正。针对已安装的 SDK，快速执行 `strings` / `jar tf` / `javap` 来查找名称是可以接受的（几秒钟即可完成），但不应进一步深入。含有错误类型名称的文件尚可修复；若耗费一整段时间用于探索却未写出任何代码，则难以挽回。
- **Bash 和文本编辑器工具由 Anthropic 定义，无 schema。** 请声明为 `{"type": "bash_20250124", "name": "bash"}` / `{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}`——无需提供 `input_schema`。若自定义工具并命名为 `"bash"`，则属于另一款工具。处理器路径和安全检查详见 `shared/tool-use-concepts.md` 第 § 客户端工具 部分。
- **顾问工具的模型配对。** 顾问工具的 `model` 至少应与请求的顶级 `model` 同等甚至更强——例如，执行器使用 `claude-sonnet-5-5` 时，顾问应使用 `claude-opus-5-5`。无效配对会返回 400 错误；`claude-sonnet-5-5` 执行器仅接受配对表中与其对应的顾问（不包括 Claude Opus 4.8/4.7/4.6、Claude Sonnet 5 或 Sonnet 4.6）。配对表（以及哪些顾问会返回明文或加密的 `advisor_redacted_result` 建议）详见 `shared/tool-use-concepts.md` 第 § 顾问 部分。可用性信息参见 `shared/platform-availability.md`。
- **Agent Skills ≠ Managed Agents。** 若要通过 Agent Skills 让 Claude 生成 `.pptx`/`.xlsx` 等文件，请调用 `client.beta.messages.使用 `container={"skills": [...]}`、工具 `code_execution_20260521` 以及 Beta 版 `code-execution-2025-08-25` 创建（技能已退出 Beta，无需 `skills-2025-10-02` 标头）。此处请勿使用 `client.beta.agents` / `sessions` / `environments`——这些属于托管代理的接口，而非代理技能。
- **MCP 连接器需要同时指定两部分。** 单独提供 `mcp_servers=[{type:"url", url, name}]` 会因验证错误而被拒绝——还需添加 `tools=[{type:"mcp_toolset", mcp_server_name:<相同名称>}]`，并使用 Beta 版 `mcp-client-2025-11-20`。可用性参见 `shared/platform-availability.md`。
- **`inference_geo` 是一个直接的顶级请求参数**——`client.messages.create(..., inference_geo="us")` 或 `.inferenceGeo("us")`。请勿将其放入 `extra_body` 或通过 `putAdditionalBodyProperty` 设置。（仅适用于 Messages API；在托管代理中，`inference_geo` 嵌套在代理的 `model` 对象内，而非顶层，请参阅 `shared/managed-agents-core.md` § 锁定推理地域。）支持 Opus 4.6 及更高版本、Sonnet 4.6 及更高版本；可用性参见 `shared/platform-availability.md`。`response.usage.inference_geo` 会报告推理运行的地域。
- **细粒度工具流式传输并非 Beta 功能；该技能的默认设置是为流式传输和客户端工具启用此功能（API 本身仍默认使用缓冲模式）。** 在工具定义中设置 `eager_input_streaming: true`，并调用常规的 `client.messages.stream(...)`。无需 Beta 标头，也无需通过 `client.beta.*` 路径。请勿同时发送旧版的 `fine-grained-tool-streaming-2025-05-14` Beta 标头。Python 的 `@beta_tool(eager_input_streaming=True)` 可直接接受该参数；TypeScript 的 `betaZodTool()` 则不支持，需通过展开语法设置：`{ ...betaZodTool({...}), eager_input_streaming: true }`。启用该字段后，API 不再对输入进行强制转换或验证，因此累积的 `partial_json` 可能不完整（受 `max_tokens` 限制）或无效——务必做好解析防护（参见 `shared/tool-use-concepts.md` -> 立即输入流式传输）。
- **缓存诊断功能为 Beta 版。** 使用带有 Beta 标头 `cache-diagnosis-2026-04-07` 的 `client.beta.messages.*`。首次调用时传入 `diagnostics: {previous_message_id: null}`，后续调用时传入 `diagnostics: {previous_message_id: <上一条响应 ID>}`；结果可在 `response.diagnostics` 中获取。可用性参见 `shared/platform-availability.md`。
- **内存工具类型为 `memory_20250818`。** 声明 `{"type": "memory_20250818", "name": "memory"}`。Go 语言在 `client.Beta.Messages.New` 中使用 Beta 命名空间类型 `{OfMemoryTool20250818: &anthropic.BetaMemoryTool20250818Param{}}`；Python/TypeScript/Ruby/PHP/C# 使用非 Beta 的 `client.messages.create`；Java 同时提供非 Beta 的 `MemoryTool20250818` 和 Beta 工具执行路径。Python/TypeScript 提供 `BetaAbstractMemoryTool` / `betaMemoryTool` 辅助类用于实现后端逻辑。
- **使用实际支持相应功能的模型。** 部分功能仅限于特定模型层级——快速模式仅支持 Claude Opus 5 / Claude Opus 5.5 / Opus 4.8（且仅限 Claude API）；任务预算（仅限 Messages API——托管代理的会话预算无模型层级限制）仅支持 Claude Opus 5 / Claude Opus 5.5 / Fable 5 / Claude Fable 5.1（发布时确认）/ Claude Sonnet 5.5 / Opus 4.8 / 4.7（不支持 Claude Sonnet 5）；顾问工具则要求有效的执行者与顾问配对。若用户提示中指定了功能不支持的模型，请改用支持的模型，并在输出中注明替换情况。
- **不要为 SDK 数据结构定义自定义类型：** SDK 已导出所有 API 对象的类型。请使用 `Anthropic.MessageParam` 表示消息，`Anthropic.Tool` 表示工具定义，`Anthropic.ToolUseBlock` / `Anthropic.ToolResultBlockParam` 表示工具结果，`Anthropic.Message` 表示响应。自行定义 `interface ChatMessage { role: string; content: unknown }` 会与 SDK 已提供的类型重复，并丧失类型安全性。
- **生成并记录输出：** 对于生成报告、文档或可视化内容的任务，代码执行沙盒已预装 `python-docx`、`python-pptx`、`matplotlib`、`pillow` 和 `pypdf`。Claude 可以生成格式化文件（DOCX、PDF、图表）并通过 Files API 返回——对于“报告”或“文档”类型的请求，可考虑这种方式，而非仅返回纯文本标准输出。
- **服务器端工具错误不会抛出异常。** 网络搜索和网页抓取的错误会返回 HTTP 200 状态码，并附带一个 `web_search_tool_result` 或 `web_fetch_tool_result` 块，其 `content` 是一个单一的错误对象（如 `{error_code: "max_uses_exceeded"}`），而非抛出的异常。对于网络搜索，成功响应的 `content` 是一个列表；错误响应的 `content` 则是一个对象——在索引前请先根据类型分支处理。
- **托管代理的网络工具会忽略环境的 `networking` 配置。** `web_search` / `web_fetch` 在 Anthropic 的云端及自托管环境中均可运行，而控制台组织级别的网络设置仅适用于 Messages API。可通过工具集的 `configs` 条目中的 `allowed_domains` 或 `blocked_domains` 来逐个工具地进行限制（二者不可同时使用；每个列表最多 1–64 个普通主机名，子域名包含在内；IP 地址、裸顶级域名、单标签域名及 `localhost` 类型的名称在这两种工具上均不被允许；路径后缀仅允许在 `web_search` 中使用）——参见 `shared/managed-agents-tools.md` § 网络搜索与网页抓取设置。
- **评估与爬坡工作有专门指南：** 若用户提出“爬坡”“提升我的评估分数”“针对评估迭代我的提示”或“为我构建一个评估”等需求，请加载 `shared/evals/eval-hillclimb.md` 或 `shared/evals/build-eval.md`，而非临时 improvisation。内置的 HTML 报告生成器在 EAP 安装时为 `shared/evals/report/build-report.mjs`（当该文件存在于磁盘时）；否则为 `shared/evals/report/build-report-lite.mjs`（始终随本技能一同提取）——请勿另行编写。
- **代码执行输出块类型：** `code_execution_20260521` 返回的是 `bash_code_execution_tool_result`（含 `.content.stdout`），**而非**旧版的裸体 `code_execution_tool_result`。请遍历 `response.content` 并匹配正确的类型。
- **工具搜索：切勿将所有内容都延迟加载。** 搜索工具本身不得设置 `defer_loading: true`，且 `tools` 中必须至少有一个工具未设置延迟加载，否则 API 将返回 400 错误：“所有工具均已设置 defer_loading”。