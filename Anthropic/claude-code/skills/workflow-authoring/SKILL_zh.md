---
name: workflow-authoring
description: |-
  编写工作流工具脚本的参考文档（脚本 API 及注意事项、简历、质量模式、示例）。在为用户已选择加入的工作流编写脚本之前加载；它本身并不授权运行该工作流。
---
# 工作流编写参考

工作流将工作在多个智能体之间进行结构化编排——以实现全面性（分解并并行处理）、确保可靠性（独立视角与提交前的对抗性验证），或应对单个上下文无法承载的大规模任务（迁移、审计、广泛扫描）。脚本就是用来编码这种结构的地方：哪些部分需要展开，哪些部分需要验证，哪些部分需要整合。

当你调用它时，最佳做法通常是**混合模式**：先在内联模式下进行初步侦察（列出文件、查找渠道、确定差异范围）以发现待办事项清单，然后再调用工作流对其进行流水线式处理。你无需在*任务*开始前就了解其完整形态，只需在*编排步骤*开始前明确即可。

常见的单阶段工作流可以跨轮次串联使用：
- **理解**——对相关子系统进行并行阅读 → 结构化地图
- **设计**——由N个独立方案组成的评审小组 → 打分后的综合
- **评审**——按维度查找 → 对比核查 → 对抗性验证（如“评审变更”示例）
- **研究**——多模态扫描 → 深度阅读 → 综合
- **迁移**——发现目标位置 → 分别转换（工作树隔离）→ 验证

对于更大型的工作，可按顺序执行多个工作流——在决定下一阶段之前先查看每个结果。你始终处于闭环之中；每个工作流都是一次范围明确的扇出。

**超代码模式。**当系统提醒确认已启用超代码模式时，该选项即被激活：默认情况下，为每个实质性任务编写并运行一个工作流。目标是产出尽可能详尽且正确的答案——token成本不再是约束。对于多阶段工作（理解→设计→实施→评审），这通常意味着按顺序执行多个工作流——每个阶段一个——以便在各阶段之间保持闭环。下方的质量模式（对抗性验证、多模态扫描、完整性评估、循环直至完成）都是可用工具，根据任务需求选择合适的组合。尽量通过工作流进行编排，并对所得结果进行对抗性验证——除非任务极其简单或已得到验证。仅在对话式交互或简单的机械编辑场景下才单独使用智能体。当提醒显示超代码模式已关闭时，请恢复至工作流工具说明中的默认规则。

请通过`script`参数直接传递脚本，不要先将其写入文件。每次调用都会自动将脚本持久化到会话目录下的文件中，并在工具结果中返回该文件的路径。若需迭代工作流，可在Write/Edit中编辑该文件，然后使用`{scriptPath: "<path>"}`重新调用工作流，而无需再次发送完整脚本。

每个脚本必须以`export const meta = {...}`开头：
  export const meta = {
    name: 'find-flaky-tests',
    description: '查找不稳定测试并提出修复建议',   // 单行描述，在权限对话框中显示
    phases: [                                            // 每个phase()调用对应一项
      { title: '扫描', detail: '在测试日志中grep重试标记' },
      { title: '修复', detail: '每个不稳定测试分配一个智能体' },
    ],
  }
  // 脚本主体从这里开始 — 使用agent()/parallel()/pipeline()/phase()/log()
  phase('扫描')
  const flaky = await agent('在CI日志中grep重试标记', {schema: FLAKY_SCHEMA})
  ...

`meta`对象必须是纯字面量——不得包含变量、函数调用、扩展运算符或模板插值。必填字段：`name`、`description`。可选字段：`whenToUse`（在工作流列表中显示）、`phases`。`meta.phases`中的阶段标题必须与`phase()`调用中的标题完全一致——标题是精确匹配的；若`phase()`调用没有对应的`meta`条目，则会为其创建独立的进度组。

脚本主体钩子：
- `agent(prompt: string, opts?: {label?: string, phase?: string, schema?: object, effort?: string, isolation?: 'worktree', agentType?: string}): Promise<any>` — 派生一个子代理。无模式时，返回其最终文本作为字符串。有模式（JSON Schema）时，子代理会被强制调用 StructuredOutput 工具，agent() 返回经过验证的对象——无需解析。若用户中途跳过该代理，或子代理在重试后因终端 API 错误而终止，则返回 null（可用 .filter(Boolean) 进行过滤）。opts.label 可覆盖显示标签。opts.phase 显式将该代理分配到某个进度组（在 pipeline()/parallel() 阶段内使用此选项，可避免对全局 phase() 状态的竞态——相同 phase 字符串对应同一组框）。opts.effort 可覆盖本次代理调用的推理力度（'low' | 'medium' | 'high' | 'xhigh' | 'max'），省略则继承会话力度；对廉价的机械型阶段使用 'low'，仅在最困难的验证/评判阶段使用更高档位。opts.isolation: 'worktree' 会在全新的 git 工作树中运行代理——开销较大（每次代理约需 200–500ms 的初始化及磁盘操作），仅当多个代理并行修改文件且可能产生冲突时才使用；若工作树未发生变化，则会自动删除。opts.agentType 可指定自定义的子代理类型（如 'general-purpose'、'code-reviewer'），而非默认的工作流子代理——从与 Agent 工具相同的注册表中解析；可与 schema 组合使用（自定义代理的系统提示会附加 StructuredOutput 指令）。
- `pipeline(items, stage1, stage2, ...): Promise<any[]>` — 将每个项目独立地依次通过所有阶段，各阶段之间无屏障。项目 A 可能已进入第 3 阶段，而项目 B 仍在第 1 阶段。这是多阶段工作的默认方式。实际耗时等于单个最慢项目的总时长，而非各阶段中最慢时间之和。每个阶段的回调函数接收 (prevResult, originalItem, index)——可在后续阶段使用 originalItem/index 来标记任务，而无需通过第 1 阶段的返回值传递上下文。若某阶段抛出异常，则该项目会被置为 null，并跳过剩余阶段。
- `parallel(thunks: Array<() => Promise<any>>): Promise<any[]>` — 并发执行多个任务。这是一个屏障：会等待所有 thunk 执行完毕后再返回结果。若某个 thunk 抛出异常（或其代理发生错误），则结果数组中对应的项为 null——整个调用不会拒绝，因此在使用结果前需调用 .filter(Boolean)。仅在确实需要所有结果同时可用时才使用。
- log(message: string): void — 向用户输出一条进度消息（显示为进度树上方的旁白行）
- phase(title: string): void — 开启一个新的阶段；后续的 agent() 调用将在进度显示中归入该标题下的分组
- args: any — 作为 Workflow 的 `args` 输入原样传入的值（若未提供则为 undefined）。数组或对象应在工具调用中以 JSON 值的形式直接传递，而非 JSON 编码的字符串——例如 `args: ["a.ts", "b.ts"]`，而不是 `args: "[\"a.ts\", ...]"`（被字符串化的列表在脚本中会作为一个整体字符串出现，因此 `args.filter`/`args.map` 会抛出异常）。可用于参数化命名工作流——例如直接传递研究问题、目标路径或配置对象，而非通过额外的文件通道。
- budget: {total: number|null, spent(): number, remaining(): number} — 用户 "+500k" 式指令中设定的本轮 token 目标。若未设定目标，则 `budget.total` 为 null。`budget.spent()` 返回本轮主循环及所有工作流中已使用的输出 token——配额是共享的，而非按工作流单独计算。`budget.remaining()` 返回 `max(0, total - spent())`，若无目标则返回 `Infinity`。该目标是硬性上限，而非建议值：一旦 `spent()` 达到 `total`，后续的 `agent()` 调用将抛出异常。可用于动态循环：`while (budget.total && budget.remaining() > 50_000) { ... }`，或静态缩放：`const FLEET = budget.total ? Math.floor(budget.total / 100_000) : 5`。
- `workflow(nameOrRef: string | {scriptPath: string}, args?: any): Promise<any>` — 内联运行另一个工作流作为子步骤，并返回其结果。传入名称以调用已保存的工作流（来自与 {name: "..."} 相同的注册表），或传入 {scriptPath} 以运行先前编写的脚本文件。子流程共享当前运行的并发上限、代理计数器、中断信号及 token 预算——其代理会在 /workflows 中以 "▸ name" 分组显示，且其 token 会计入 budget.spent()。args 参数将成为子流程的 `args` 全局变量。嵌套仅限一层：子流程内部再调用 workflow() 会抛出异常。若名称未知、scriptPath 无法读取或子流程存在语法错误，则会抛出异常；可捕获异常以进行优雅处理。

子代理被告知其最终文本即为返回值（而非面向人类的消息），因此它们会返回原始数据。对于结构化输出，请使用 schema 选项——验证在工具调用层进行，如果不符合则模型会重试。
Schema 在根节点必须是 {type: 'object', properties: {...}}，且 required 必须是 properties 的子集；不满足条件的会在 agent() 时抛出异常。

工作流代理可通过 ToolSearch 访问所有已连接会话的 MCP 工具——schema 会按需在每个代理启动时加载。注意：需要交互式认证的 MCP 服务器（如 claude.ai）在无头或定时运行模式下可能不可用。

子代理在启动时会注入与您相同的 CLAUDE.md 文件（内置代理类型除外，例如 Explore 和 Plan）——不要让它们重新读取这些文件或将规则粘贴到提示中；如有需要，只需指定某个阶段所需的具体规则。

脚本是纯 JavaScript，不是 TypeScript——类型注解（`: string[]`）、接口和泛型都无法解析。脚本主体在异步上下文中运行——可直接使用 await。标准 JS 内置对象（JSON、Math、Array 等）可用——但 `Date.now()`、`Math.random()` 和无参数的 `new Date()` 会抛出异常（因为它们会破坏恢复机制）；应通过 `args` 传递时间戳，在工作流转回后对结果打上时间戳，并通过索引来调整代理的提示或标签以实现随机性。无法访问文件系统或 Node.js API。

默认使用 pipeline()。只有当确实需要将所有前一阶段的结果汇总在一起时，才使用 barrier（阶段间的并行同步）。

只有在以下情况下使用 barrier 才是正确的：
- 在昂贵的下游工作之前，对整个结果集进行去重或合并；
- 如果总数量为零，则提前退出（“未发现任何漏洞 → 完全跳过验证”）；
- 阶段 N 的提示中引用了“其他发现”用于比较。

以下情况不应使用 barrier：
- “我需要先展平/映射/过滤”——应在 pipeline 的某个阶段内完成：pipeline(items, stageA, r => transform([r]).flat(), stageB)；
- “各阶段在概念上是独立的”——这正是 pipeline() 的设计意图。独立阶段 ≠ 同步阶段；
- “这样代码更整洁”——barrier 会带来实际的延迟。如果有 5 个查找器同时运行，而最慢的那个耗时是最快的 3 倍，那么 barrier 会让最快的 2/3 查找器白白浪费空闲时间。

检验方法：如果你写了
  const a = await parallel(...)
  const b = transform(a)        // 展平、映射、过滤——不存在跨项依赖
  const c = await parallel(b.map(...))
那么中间的 transform 并不需要 barrier。应将其改写为 pipeline，并把 transform 放在一个阶段内。如有疑问，就用 pipeline。

并发的 agent() 调用上限为 min(16, 可用 CPU 数 - 2) 每个工作流——超出的部分会排队，待有空位时再执行。你仍然可以向 parallel()/pipeline() 传入 100 个任务，它们都会完成；但同一时刻最多只有约 10 个在运行。一个工作流生命周期内的代理总数上限为 1000——这是远高于实际工作流需求的防溢出保护。单次 parallel()/pipeline() 调用最多接受 4096 个任务；超过此数会直接报错，而不是静默截断。

当确实需要 barrier 时——例如在昂贵的验证之前对所有发现进行去重：
  const all = await parallel(DIMENSIONS.map(d => () => agent(d.prompt, {schema: FINDINGS_SCHEMA})))
  const deduped = dedupeByFileAndLine(all.filter(Boolean).flatMap(r => r.findings))  // <-- 确实需要一次性获取全部
  const verified = await parallel(deduped.map(f => () => agent(verifyPrompt(f), {schema: VERDICT_SCHEMA})))

循环直到达到目标的模式——不断累积直到满足条件：
  const bugs = []
  while (bugs.length < 10) {
    const result = await agent("在这个代码库中查找漏洞。", {schema: BUGS_SCHEMA})
    bugs.push(...result.bugs)
    log(`${bugs.length}/10 已找到`)
  }

预算循环模式——根据用户的“+50万”指令调整深度。对budget.total进行保护：若未设置目标，remaining()将为Infinity，循环会直接达到1000个代理的上限。
  const bugs = []
  while (budget.total && budget.remaining() > 50_000) {
    const result = await agent("在这个代码库中查找缺陷", {schema: BUGS_SCHEMA})
    bugs.push(...result.bugs)
    log(`${bugs.length} 个已发现，还剩 ${Math.round(budget.remaining()/1000)}k`)
  }

模式组合——全面审查（查找→去重 vs 已见→多样性视角面板→循环至无新发现）：
  const seen = new Set(), confirmed = []
  let dry = 0
  while (dry < 2) {                                              // 循环至无新发现
    const found = (await parallel(FINDERS.map(f => () =>          // 同步屏障：收集本轮所有查找者的结果
      agent(f.prompt, {phase: 'Find', schema: BUGS})))).filter(Boolean).flatMap(r => r.bugs)
    const fresh = found.filter(b => !seen.has(key(b)))           // 去重 vs 所有已见——纯代码实现，非代理
    if (!fresh.length) { dry++; continue }
    dry = 0; fresh.forEach(b => seen.add(key(b)))
    const judged = await parallel(fresh.map(b => () =>           // 每个新发现的缺陷并发评审...
      parallel(['correctness','security','repro'].map(lens => () =>   // ...由三个不同视角分别评估
        agent(`从${lens}视角评判：“${b.desc}”——是否真实？`, {phase: 'Verify', schema: VERDICT})))
        .then(vs => ({ b, real: vs.filter(Boolean).filter(v => v.real).length >= 2 }))))
    confirmed.push(...judged.filter(v => v.real).map(v => v.b))
  }
  return confirmed
  // 去重基于 `seen`，而非 `confirmed`——否则被评审否定的发现每轮都会重现，永远无法收敛。

质量模式——常见结构；按任务选择并自由组合：
- 对抗式验证：为每个发现生成N个独立的质疑者，各自被提示去反驳。若≥多数人反驳，则剔除该发现。可防止看似合理但错误的发现存活下来。
  const votes = await parallel(Array.from({length: 3}, () => () =>
    agent(`尝试反驳：${claim}。若不确定，默认驳回。`, {schema: VERDICT})))
  const survives = votes.filter(Boolean).filter(v => !v.refuted).length >= 2
- 多视角验证：当一个发现可能以多种方式失效时，让每个验证者使用不同的视角（正确性、安全性、性能、能否复现），而不是N个完全相同的反驳者——多样性能捕捉到冗余无法发现的失效模式。
- 评审小组：从不同角度生成N个独立的尝试（如先MVP、先风险、先用户），由多个评审并发打分，并在胜者的基础上融合亚军的最佳想法。当解空间较广时，优于单次迭代。
- 循环至无新发现：对于未知规模的发现任务（缺陷、问题、边缘情况），持续生成查找者，直到连续K轮没有新发现为止。仅用简单计数器（while count < N）会遗漏尾部。
- 多模态扫描：多个代理并行，各自采用不同的搜索方式（按容器、按内容、按实体、按时序）。彼此对对方的发现结果一无所知；当单一搜索角度无法覆盖全部时尤为有用。
- 完整性批评者：最后由一个代理提出“还缺少什么——未执行的模态、未验证的主张、未读取的来源？”它发现的问题将成为下一轮的工作。
- 避免无声截断：如果工作流限制了覆盖范围（前N项、不重试、采样），则需记录被丢弃的内容——无声截断会让人误以为“已覆盖全部”，而实际上并非如此。

按用户要求调整规模。“找出任何缺陷”→少量查找者，单次投票验证。“彻底审计”或“全面覆盖”→更大的查找者池，3–5次对抗式投票，再进行综合。不确定时，对于研究/审查/审计类请求倾向于全面，而对于快速检查则倾向于简洁。

这些模式并非穷尽——当任务需要时，可组合出新的框架（如锦标赛式分组、自我修复循环、分级升级等）。

在控制流应为确定性（循环、条件分支、并行执行）而非模型驱动的情况下，使用此工具进行多步编排。

## 恢复

工具结果包含一个 runId。要在暂停、终止或脚本编辑后恢复，请使用 Workflow({scriptPath, resumeFromRunId}) 重新启动——agent() 调用中最长的未更改前缀会立即返回缓存结果；第一个被编辑或新增的调用及其之后的所有调用都会实时运行。相同脚本 + 相同参数 → 100% 缓存命中。在诊断已完成的工作流为何返回空值或意外结果之前，请先读取 `<transcriptDir>/journal.jsonl` ——它记录了每个代理的实际返回值；不要假设缓存结果一定非空。Date.now()/Math.random()/new Date() 在脚本中不可用（它们会破坏这一机制），请在工作流返回后为结果打上时间戳，或通过参数传递时间戳。当没有日志可用时的备选方案：读取转录目录中的 agent-`<id>`.jsonl 文件，并手动编写续写脚本。
