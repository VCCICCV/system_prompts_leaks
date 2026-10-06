---
name: workflow-authoring
description: 在为研究、评审、迁移或其他涉及多智能体协作的复杂工作流进行设计时使用，尤其是在任务需要多个证据来源、验证或综合处理的情况下。
user-invocable: false
---
# 工作流编写

在第一个非空的工作流之前，每个父会话仅加载此参考一次。成功加载后，在后续的工作流编写中复用该结果。验证错误或重试与恢复时，请勿再次调用 `read_skill`。从当前启用的工作流指引中精确选择一个配置节。切勿跨配置混用符号，也绝不能使用当前工具规范未声明的调用。

## 共享研究合约

固定的批次数量永远无法证明已完成。根据请求调整子任务的数量与多样性，然后在出现证据状态、明确的调用方指示或运行时边界时停止。

对于用户已识别的变更，首先在流程内统计其涉及的文件及规模，再据此规划工作流：小规模变更只需几个聚焦的子任务加上一个验证投票，而非完整的调研形态。

当相关主体可读时，发现指针本身不作为已检证据。要求每个研究子任务在调用 `submit_result` 前打开其所引用的每一个实现或测试主体；搜索与 grep 的输出仅用于定位候选对象。为每项分配的主张提供经检验的证据，否则将其标记为未决。向研究子任务索取 `complete:boolean`、`evidence:string[]` 和 `unresolved:string[]`。在检查数据缺失前，先处理下方的 V2 溢出标记。对于内联结果，若存在特定配置下的失败结果封装、数据缺失或类型错误、`complete !== true`，或 `unresolved` 非空，则视为未完成。

绝不可将失败封装中的数据用作证据或缺口处置依据。评审者不可用是合成注记，而非研究缺口。在合成阶段保留精简证据、溯源引用以及所有未决项。应基于精简的 `result.data` 进行合成，而非未经检验的摘要。披露被省略的范围。

当一项主张存在显著不同的失效模式时，应采用独立验证。重复相同的提示并不构成独立覆盖。

在符合需求时，保留以下可复用模式：

- 多角度扫描：将初始研究人员分散到真正不同的证据来源上，如实现代码、测试、设计文档和运行时轨迹；若不同角色名采用相同的检索方案，则不能增加覆盖度。
- 对抗性验证：为怀疑论者针对每一项重要主张设定具体的证伪目标；仅保留经检验反证后仍成立的主张，并将无法得出或无效的结论标记为未决。
- 评审团：对于开放的解决方案空间，从不同角度生成候选方案，由独立评审员按明确标准打分，并通过溯源引用综合优胜方案及有价值的备选思路。失败或无效的评审意见不应被视为肯定票。

## V2 溢出的子任务结果

符合 Schema 规范且自定义数据超过 4096 个规范 UTF-8 字节（不包括 `notes`）的结果将被完整接收并存储。恰好 4096 字节的结果保留在内联字段中。超大结果会携带 `dataSpilledForSize: true`、`submittedPayloadBytes`、`submittedPayloadChars` 和 `ref`，而无内联 `data`。尺寸信息描述的是完整的规范提交内容，包括非空的 `notes`；它们并非仅针对自定义数据的溢出度量。

脚本必须将 `dataSpilledForSize`、两项尺寸信息以及 `ref` 传递至返回结果中。请保留封装结构，或如下面的 V2 示例所示，显式复制这些字段。绝不可将结果简化为 `data ?? null`。严禁将溢出结果视为空值或失败，也不得仅因缺少内联数据而重复已完成的工作。务必继续尊重封装的实际状态与错误信息。提交完成并不意味着未经检验的证据已完备；应保留现有的未决缺口。

目前，无论是脚本端还是父端的 API 都无法返回完整值。完整的提交内容会以其 `ref` 保存在持久化会话存储中，并通过 MSP 子代理视图公开。重复进行结果观测或将该 `ref` 传递给其他子任务时，均无法实现无损获取；`ref` 的上下文是受限的。

请要求子任务将自定义数据控制在不超过 4096 个规范字节以内。当具备文件工具时，应将大型工件（如测试模块、报告）写入文件，并返回文件路径及一份精简摘要。否则，请拆分工作或请求提供精简的模式定义。对于超出限制而被溢出的提交，应将其标记为“已完成但体积较大”，附上其 `ref` 和大小信息，并说明完整提交的存放位置，同时明确指出该 Workflow 尚未对其中内容进行检查。除非子任务确实返回了该工件的文件路径，否则不得声称已成功写入文件。

## 两条收敛规则

开放式探索与填补已知证据缺口是两种不同的任务，不应将同一条循环规则同时应用于两者。

### 示例：开放式探索

在确定性的 Workflow 状态中维护 `seen` 和 `dryRounds`。每一轮都向互补的发现者询问尚未收录于 `seen` 中的项目，并在判断之前将每个报告的项目加入 `seen`：

- 与所有已见项目（包括已被拒绝的发现）进行去重；
- 如果某一轮新增了任何新项目，则将空转计数重置为零；
- 若未新增任何项目，则将空转计数加一；
- 连续两轮均无新增后，停止探索。

调用方的上限、容量约束或运行时预算可能会提前终止探索；除非已覆盖全部请求范围，否则此次终止属于部分终止。

### 示例：明确缺口的后续跟进

跟踪每个具体未解决缺口的来源链路。

- 仅针对该缺口的来源链路派遣一次有针对性的后续跟进。
- 在该次尝试之后，将经细化、改述或仍未解决的子缺口原样带入综合环节。
- 不得使其新的表述看起来像是一个新的缺口并再次派遣跟进。

### 示例：验证与遗漏范围

对于涉及行为、抗滥用能力及已报告失败的主张：

- 分别使用正确性、安全性及复现性三个视角进行验证；
- 为每个验证者设定独立的证伪目标；
- 对于前 N 项、抽样、不重试、容量、调用方上限及运行时预算等边界条件，应明确声明遗漏的范围；
- 切勿将有界样本描述为“穷举”。

## Workflow API V1

API 以裸全局变量的形式提供——agent、parallel、pipeline、phase、log、args、budget——并通过旧版宿主对象访问。请使用 V1 全局变量或由当前 ToolSpec 定义的其 `host` 别名。调用方输入应从 `host.args` 读取，这是唯一被公开的调用方输入字段名称。

例如，可使用 `host.parallel` 执行独立研究的扇出，并用 `host.agent` 负责评论、单次缺口跟进及最终综合：

```javascript
export default async function workflow(host) {
  const evidenceSchema = {
    type: "object",
    required: ["complete", "evidence", "unresolved"],
    properties: {
      complete: { type: "boolean" },
      evidence: { type: "array", items: { type: "string" } },
      unresolved: { type: "array", items: { type: "string" } },
    },
  };
  const compact = (result, scope, missing = `${scope}: 缺失完整证据结果`) => {
    const failed = result === null || result.error_kind;
    const data = !failed && result.data && typeof result.data === "object" ? result.data : null;
    const evidence = !failed && Array.isArray(data?.evidence) ? data.evidence.filter(Boolean) : [];
    const declared = !failed && Array.isArray(data?.unresolved) ? data.unresolved.filter(Boolean) : [];
    const complete = data?.complete === true && evidence.length > 0 && declared.length === 0;
    return {
      scope,
      ref: result?.ref ?? null,
      complete,
      evidence,
      unresolved: complete ? [] : (declared.length ? declared : [missing]),
    };
  };

  const reports = await host.parallel([
    { input: "检查实现主体；返回 complete/evidence/unresolved。", schema: evidenceSchema },
    { input: "检查测试；返回 complete/evidence/unresolved。", schema: evidenceSchema },
  ]);
  const compactReports = reports.map((result, index) => compact(result, `primary-${index}`));
  const synthesisNotes = [];
  const critic = await host.agent({
    input: `找出这些精简证据中的具体漏洞：${JSON.stringify(compactReports)}。`,
    schema: evidenceSchema,
  });
  const criticData = critic !== null && !critic.error_kind && critic.data && typeof critic.data === "object" ? critic.data : null;
  const criticEvidence = Array.isArray(criticData?.evidence) ? criticData.evidence.filter(Boolean) : [];
  const criticUnresolved = Array.isArray(criticData?.unresolved) ? criticData.unresolved.filter(Boolean) : [];
  const criticHasUsableDisposition = (criticData?.complete === true && criticEvidence.length > 0 && criticUnresolved.length === 0)
    || criticUnresolved.length > 0;
  const compactCritic = criticHasUsableDisposition
    ? compact(critic, "critic")
    : (synthesisNotes.push("完整性评审不可用"), { scope: "critic", ref: critic?.ref ?? null, complete: true, evidence: [], unresolved: [] });
  const open = [...compactReports, compactCritic].flatMap((report) => report.unresolved);
  const firstGap = open[0];
  const followup = firstGap ? await host.agent({
    input: `解决这个确切的漏洞一次，或者原样返回：${firstGap}。`,
    schema: evidenceSchema,
  }) : null;
  const followupReport = firstGap ? compact(followup, firstGap, firstGap) : null;
  const all = followupReport ? [...compactReports, compactCritic, followupReport] : [...compactReports, compactCritic];
  const unresolved = firstGap ? [...open.slice(1), ...followupReport.unresolved] : open;
  const evidence = all.flatMap((report) => report.evidence.map((value) => ({ source: report.scope, ref: report.ref, value })));
  const refs = [...new Set(all.map((report) => report.ref).filter(Boolean))];
  const synthesis = await host.agent({
    input: `仅对这些精简证据进行综合：${JSON.stringify({ evidence, refs, unresolved, notes: synthesisNotes })}。`,
    schema: evidenceSchema,
  });
  const synthesisFailed = synthesis === null || synthesis.error_kind;
  const synthesisData = !synthesisFailed && synthesis.data && typeof synthesis.data === "object" ? synthesis.data : null;
  const synthesisUnresolved = Array.isArray(synthesisData?.unresolved) ? synthesisData.unresolved.filter(Boolean) : [];
  const synthesisComplete = synthesisData?.complete === true
    && Array.isArray(synthesisData.evidence)
    && synthesisData.evidence.length > 0
    && synthesisUnresolved.length === 0;
  if (!synthesisComplete) synthesisNotes.push("综合结果不可用或不完整");
  return { status: unresolved.length || synthesisUnresolved.length || synthesisNotes.length > 0 ? "partial" : "complete", ref: synthesis?.ref ?? null, unresolved: [...unresolved, ...synthesisUnresolved], notes: synthesisNotes };
}
```

当请求需要时，将共享发现与漏洞谱系状态机封装在这些调用周围。使用精简的结构化结果和引用；遵循 V1 ToolSpec 中关于模式、预算、隔离、失败处理及返回格式的规定。

## 诊断工作流 API V2

每个 V2 的 `input` 长度应控制在 4096 个 UTF-8 字节以内，包括任务文本和引用。延迟命令也需遵守其整体命令大小的限制。以下引用会请求有限的先前结果上下文；若所需证据缺失，则应检查相关主体并返回 `complete: false` 及未解决的漏洞。引用中不携带无损的 `result.data` 或父级局部的漏洞状态。请将已知未解决的项目和综合说明作为必要的精简上下文一并提供。

诊断脚本的接口完全由 Agent、Phase、Pipeline、ParallelGroup、WorkflowCommandError、log、args 和 budget 组成。此配置为全新运行且仅限终端使用。请在诊断容器内使用延迟任务来实现并行化，然后读取不可变的结果：

```javascript
const evidenceSchema = {
  type: "object",
  required: ["complete", "evidence", "unresolved"],
  properties: {
    complete: { type: "boolean" },
    evidence: { type: "array", items: { type: "string" } },
    unresolved: { type: "array", items: { type: "string" } },
  },
};
const synthesisNotes = [];
const spilledResults = [];
const recordSpill = (result, scope) => {
  if (result?.dataSpilledForSize !== true) return false;
  spilledResults.push({
    scope, ref: result.ref, status: result.status, ok: result.ok, error: result.error,
    dataSpilledForSize: result.dataSpilledForSize,
    submittedPayloadBytes: result.submittedPayloadBytes,
    submittedPayloadChars: result.submittedPayloadChars,
  });
  synthesisNotes.push(`${scope}: 大量提交数据在会话存储/ MSP 子代理视图中被保留；内容尚未检查。`);
  return result.status === "completed" && result.ok === true && result.error == null;
};
const group = await ParallelGroup.start({
  members: [
    Agent.defer.start({ input: "检查实现主体；返回 complete/evidence/unresolved。", schema: evidenceSchema }),
    Agent.defer.start({ input: "检查测试；返回 complete/evidence/unresolved。", schema: evidenceSchema }),
  ],
});
const reports = await group.result();
const fromAttemptOutcome = (outcome, scope) => {
  if (outcome?.kind !== "attempt" || !outcome.result?.ref) {
    return { scope, ref: null, complete: false, evidence: [], unresolved: [`${scope}: 没有完成的尝试结果`] };
  }
  const result = outcome.result;
  const ref = outcome.result.ref;
  if (recordSpill(result, scope)) {
    return { scope, ref: result.ref, complete: false, evidence: [], unresolved: [] };
  }
  const data = result?.data && typeof result.data === "object" ? result.data : null;
  const terminalOk = result.status === "completed" && result.ok === true && result.error == null && data !== null;
  const evidence = terminalOk && Array.isArray(data.evidence) ? data.evidence.filter(Boolean) : [];
  const declared = terminalOk && Array.isArray(data.unresolved) ? data.unresolved.filter(Boolean) : [];
  const complete = terminalOk && data.complete === true && evidence.length > 0 && declared.length === 0;
  return {
    scope,
    ref,
    complete,
    evidence: terminalOk ? evidence : [],
    unresolved: complete ? [] : (declared.length ? declared : [`${scope}: 没有完成的尝试结果`]),
  };
};
const compactReports = reports.map((outcome, index) => fromAttemptOutcome(outcome, `parallel-${index}`));
const pipeline = await Pipeline.start({
  items: reports,
  stages: [{
    title: "检查",
    run: ({ item, index }) => {
      if (item?.kind !== "attempt" || !item.result?.ref) {
        return { complete: false, evidence: [], unresolved: [`parallel-${index}: 缺少结果引用`] };
      }
      return Agent.defer.start({ input: `验证 ${item.result.ref} 背后的证据；返回 complete/evidence/unresolved。`, schema: evidenceSchema });
    },
  }],
});
const checked = await pipeline.result();
const checkedReports = checked.map((item, index) => {
  if (item?.kind !== "completed" || !item.output?.ref) {
    return { scope: `pipeline-${index}`, ref: null, complete: false, evidence: [], unresolved: [`pipeline-${index}: 尚未完成`] };
  }
  if (recordSpill(item.output, `pipeline-${index}`)) {
    return { scope: `pipeline-${index}`, ref: item.output.ref, complete: false, evidence: [], unresolved: [] };
  }
  const data = item.output.data && typeof item.output.data === "object" ? item.output.data : null;
  const outputOk = item.output.status === "completed" && item.output.ok === true && item.output.error == null && data !== null;
  const evidence = outputOk && Array.isArray(data.evidence) ? data.evidence.filter(Boolean) : [];
  const declared = outputOk && Array.isArray(data.unresolved) ? data.unresolved.filter(Boolean) : [];
  const complete = outputOk && data.complete === true && evidence.length > 0 && declared.length === 0;
  return {
    scope: `pipeline-${index}`,
    ref: item.output.ref,
    complete,
    evidence: outputOk ? evidence : [],
    unresolved: complete ? [] : (declared.length ? declared : [`pipeline-${index}: 缺少结构化输出`]),
  };
});
const unresolved = [...compactReports, ...checkedReports].flatMap((report) => report.unresolved);
const refs = [...compactReports, ...checkedReports].map((report) => report.ref).filter(Boolean);
const evidence = [...compactReports, ...checkedReports].flatMap((report) => report.evidence.map((value) => ({ source: report.scope, ref: report.ref, value })));
const critic = await Agent.start({ input: `检查相关主体，并在报告 ${refs.join(" ")} 中找出具体漏洞。已知未解决事项：${JSON.stringify(unresolved)}。返回 complete/evidence/unresolved；当无法提供证据时，将 complete 设置为 false 并列出未解决的漏洞。`, schema: evidenceSchema });
const gaps = await critic.latestAttempt.result();
const criticSpilled = recordSpill(gaps, "critic");
const criticData = gaps?.data && typeof gaps.data === "object" ? gaps.data : null;
const criticTerminalOk = gaps?.status === "completed"
  && gaps?.ok === true
  && gaps?.error == null
  && criticData !== null;
const criticGaps = criticTerminalOk && Array.isArray(criticData.unresolved) ? criticData.unresolved.filter(Boolean) : [];
if (criticTerminalOk && Array.isArray(criticData.evidence)) {
  evidence.push(...criticData.evidence.filter(Boolean).map((value) => ({ source: "critic", ref: gaps.ref, value })));
}
const criticComplete = criticTerminalOk
  && criticData?.complete === true
  && Array.isArray(criticData.evidence)
  && criticData.evidence.length > 0
  && Array.isArray(criticData.unresolved)
  && criticData.unresolved.length === 0;
if (!criticComplete && criticGaps.length === 0 && !criticSpilled) {
  synthesisNotes.push("完整性评估者不可用");
}
const open = [...unresolved, ...criticGaps];
const firstGap = open[0];
let gapResult = null;
let gapDescendants = [];
if (firstGap) {
  const resolver = await Agent.start({ input: `一次性解决此确切漏洞，或原样返回：${firstGap}`, schema: evidenceSchema });
  gapResult = await resolver.latestAttempt.result();
  recordSpill(gapResult, firstGap);
  const gapData = gapResult?.data && typeof gapResult.data === "object" ? gapResult.data : null;
  const gapTerminalOk = gapResult?.status === "completed"
    && gapResult?.ok === true
    && gapResult?.error == null
    && gapData !== null;
  const reportedDescendants = gapTerminalOk && Array.isArray(gapData.unresolved) ? gapData.unresolved.filter(Boolean) : [];
  if (gapTerminalOk && Array.isArray(gapData.evidence)) {
    evidence.push(...gapData.evidence.filter(Boolean).map((value) => ({ source: firstGap, ref: gapResult.ref, value })));
  }
  const gapComplete = gapTerminalOk
    && gapData?.complete === true
    && Array.isArray(gapData.evidence)
    && gapData.evidence.length > 0
    && reportedDescendants.length === 0;
  if (!gapComplete) gapDescendants = reportedDescendants.length > 0 ? reportedDescendants : [firstGap];
}
const finalUnresolved = firstGap ? [...open.slice(1), ...gapDescendants] : open;
const synthesisRefs = [...refs, gaps?.ref, gapResult?.ref].filter(Boolean);
const synthesisAgent = await Agent.start({
  input: `仅使用已检查的主体合成报告 ${synthesisRefs.join(" ")}。保留以下已知状态：${JSON.stringify({ unresolved: finalUnresolved, notes: synthesisNotes })}。将缺失评估者的备注与研究中的漏洞分开记录。根据需要检查相关主体；保留未解决事项及遗漏的范围。当缺少必要证据时，将 complete 设置为 false。`,
  schema: evidenceSchema,
});
const synthesis = await synthesisAgent.latestAttempt.result();
const synthesisSpilled = recordSpill(synthesis, "synthesis");
const synthesisData = synthesis?.data && typeof synthesis.data === "object" ? synthesis.data : null;
const synthesisTerminalOk = synthesis?.status === "completed”
  && 合成?.ok === true
  && 合成?.error == null
  && 合成数据 !== null;
const 合成缺口 = 合成终端状态正常且合成数据.unresolved 是数组时，取其非空元素组成的数组，否则为空数组；
const 合成已完成 = 合成终端状态正常
  && 合成数据?.complete === true
  && 合成数据.evidence 是数组
  && 合成数据.evidence 的长度大于 0
  && 合成缺口的长度为 0;
if (!合成已完成 && !合成溢出) 合成备注.push("合成不可用或不完整");
return { 状态: 最终未解决事项.length || 合成缺口.length || 合成备注.length > 0 ? "部分完成" : "完成", 引用: 合成?.ref ?? null, 报告: 合成引用列表, 溢出结果, 未解决事项: [...最终未解决事项, ...合成缺口], 备注: 合成备注 };
```在需要时，将共享发现与差距谱系状态机封装在这些调用的外部。本规范中不得引入持久化的控制或恢复机制。

## 实时工作流 API V2

在运行时添加前置结果上下文之前，应将调用方提供的完整 `input` 或 `message` 字符串限制为不超过 4096 个 UTF-8 字节。此限制适用于 `Agent.start`、`agent.followup` 和 `agent.send`。任务文本、JSON 语法、证据、引用及未解决项应合并计算；JavaScript 的 `string.length` 并非 UTF-8 字节数。

构建评论与综合的交接时，应以简短的任务和可访问的 `result.ref` 引用令牌为基础，并仅保留必要的紧凑上下文。运行时会为这些引用添加受限的上下文；子任务必须检查所需的内容，并在缺少必要证据时返回 `complete: false` 以及未解决的缺口信息。
引用不应携带无损的 `result.data` 或父级本地的缺口状态。应将已知的未解决项和综合说明作为必要的紧凑上下文一并传递。
在工作流状态及最终结果中，应保持每个未解决项原样不变。
如果必要上下文超出预算，应在剩余预算内拆分工作，或明确披露被省略的范围；不得截断 JSON 或静默丢弃缺口列表。

在本激活片段中，实时工作流 API V2 的主要接口是 `Agent` 和 `AgentAttempt` 路径。应先启动每个独立的工作进程，然后观察其确切的执行尝试。仅当某个显式缺口谱系尚未收到后续处理时，才使用 `follow-up`：
```javascript
const evidenceSchema = {
  type: "object",
  required: ["complete", "evidence", "unresolved"],
  properties: {
    complete: { type: "boolean" },
    evidence: { type: "array", items: { type: "string" } },
    unresolved: { type: "array", items: { type: "string" } },
  },
};
const synthesisNotes = [];
const spilledResults = [];
const recordSpill = (result, scope) => {
  if (result?.dataSpilledForSize !== true) return false;
  spilledResults.push({
    scope, ref: result.ref, status: result.status, ok: result.ok, error: result.error,
    dataSpilledForSize: result.dataSpilledForSize,
    submittedPayloadBytes: result.submittedPayloadBytes,
    submittedPayloadChars: result.submittedPayloadChars,
  });
  synthesisNotes.push(`${scope}: 大量提交数据在会话存储/MSP 子代理视图中被保留；内容未检查。`);
  return result.status === "completed" && result.ok === true && result.error == null;
};
const agent = await Agent.start({ input: "检查实现主体；返回 complete/evidence/unresolved。", schema: evidenceSchema });
const tests = await Agent.start({ input: "检查测试；返回 complete/evidence/unresolved。", schema: evidenceSchema });
const workers = [agent, tests];
const reports = await Promise.all([
  agent.latestAttempt.result(),
  tests.latestAttempt.result(),
]);
const compact = (result, scope) => {
  if (recordSpill(result, scope)) {
    return { scope, ref: result.ref, complete: false, evidence: [], unresolved: [] };
  }
  const data = result?.data && typeof result.data === "object" ? result.data : null;
  const terminalOk = result.status === "completed" && result.ok === true && result.error == null && data !== null;
  const evidence = terminalOk && Array.isArray(data.evidence) ? data.evidence.filter(Boolean) : [];
  const declared = terminalOk && Array.isArray(data.unresolved) ? data.unresolved.filter(Boolean) : [];
  const complete = terminalOk && data.complete === true && evidence.length > 0 && declared.length === 0;
  return {
    scope,
    ref: result?.ref ?? null,
    complete,
    evidence: terminalOk ? evidence : [],
    unresolved: complete ? [] : (declared.length ? declared : [`${scope}: 缺少 complete evidence 结果`]),
  };
};
const compactReports = reports.map((result, index) => compact(result, `primary-${index}`));
const primaryRefs = compactReports.map((report) => report.ref).filter(Boolean);
const evidence = compactReports.flatMap((report) => report.evidence.map((value) => ({ source: report.scope, ref: report.ref, value })));
const primaryGaps = compactReports.flatMap((report) => report.unresolved);
const critic = await Agent.start({
  input: `检查相关主体，并在报告 ${primaryRefs.join(" ")} 中找出具体缺口。已知未解决事项：${JSON.stringify(primaryGaps)}。返回 complete/evidence/unresolved；当证据不足时，将 complete 设置为 false 并列出未解决的缺口。`,
  schema: evidenceSchema,
});
const criticResult = await critic.latestAttempt.result();
const criticSpilled = recordSpill(criticResult, "critic");
const criticData = criticResult?.data && typeof criticResult.data === "object" ? criticResult.data : null;
const criticTerminalOk = criticResult?.status === "completed"
  && criticResult?.ok === true
  && criticResult?.error == null
  && criticData !== null;
const criticGaps = criticTerminalOk && Array.isArray(criticData.unresolved) ? criticData.unresolved.filter(Boolean) : [];
if (criticTerminalOk && Array.isArray(criticData.evidence)) {
  evidence.push(...criticData.evidence.filter(Boolean).map((value) => ({ source: "critic", ref: criticResult.ref, value })));
}
const criticComplete = criticTerminalOk
  && criticData?.complete === true
  && Array.isArray(criticData.evidence)
  && criticData.evidence.length > 0
  && criticGaps.length === 0;
if (!criticComplete && criticGaps.length === 0 && !criticSpilled) {
  synthesisNotes.push("完整性审查者不可用");
}
const open = [...primaryGaps, ...criticGaps];
const gapOwner = compactReports.findIndex((report) => report.unresolved.length > 0);
const firstGap = gapOwner >= 0 ? compactReports[gapOwner].unresolved[0] : (criticGaps[0] ?? null);
const gapAgent = gapOwner >= 0 ? workers[gapOwner] : critic;
const gapDescendants = [];
let followupRef = null;
if (firstGap) {
  const followupAttempt = await gapAgent.followup({ input: `解决这个确切的缺口一次，或原样返回：${firstGap}` });
  const followupResult = await followupAttempt.result();
  recordSpill(followupResult, firstGap);
  followupRef = followupResult?.ref ?? null;
  const data = followupResult?.data && typeof followupResult.data === "object" ? followupResult.data : null;
  const followupTerminalOk = followupResult?.status === "completed"
    && followupResult?.ok === true
    && followupResult?.error == null
    && data !== null;
  const reportedDescendants = followupTerminalOk && Array.isArray(data.unresolved) ? data.unresolved.filter(Boolean) : [];
  if (followupTerminalOk && Array.isArray(data.evidence)) {
    evidence.push(...data.evidence.filter(Boolean).map((value) => ({ source: firstGap, ref: followupResult.ref, value })));
  }
  const descendants = reportedDescendants.length > 0 ? reportedDescendants : [firstGap];
  const followupComplete = followupTerminalOk
    && data?.complete === true
    && Array.isArray(data.evidence)
    && data.evidence.length > 0
    && reportedDescendants.length === 0;
  if (!followupComplete) gapDescendants.push(...descendants);
}
const unresolved = firstGap ? [...open.slice(1), ...gapDescendants] : open;
const synthesisRefs = [...primaryRefs, criticResult?.ref, followupRef].filter(Boolean);
const synthesis = await Agent.start({
  input: `仅使用已检查的主体，综合报告 ${synthesisRefs.join(" ")}。保留以下已知状态：${JSON.stringify({ unresolved, notes: synthesisNotes })}。将不可用的审查者备注与研究缺口分开。根据需要检查相关主体；保留未解决事项和遗漏的范围。当所需证据缺失时，将 complete 设置为 false。`,
  schema: evidenceSchema,
});
const final = await synthesis.latestAttempt.result();
const finalSpilled = recordSpill(final, "synthesis");
const finalData = final?.data && typeof final.data === "object" ? final.data : null;
const finalTerminalOk = final?.status === "completed"
  && final?.ok === true
  && final?.error == null
  && finalData !== null;
const finalUnresolved = finalTerminalOk && Array.isArray(finalData.unresolved) ? finalData.unresolved.filter(Boolean) : [];
const finalOk = finalTerminalOk
  && finalData?.complete === true
  && Array.isArray(finalData.evidence)
  && finalData.evidence.length > 0
  && Array.isArray(finalData.unresolved)
  && finalData.unresolved.length === 0;
if (!finalOk && !finalSpilled) synthesisNotes.push("综合结果不可用或不完整");
return { status: unresolved.length || finalUnresolved.length || synthesisNotes.length > 0 || !finalOk ? "partial" : "complete", ref: final?.ref ?? null, spilledResults, unresolved: [...unresolved, ...finalUnresolved], notes: synthesisNotes };
```首先检查 `dataSpilledForSize`。对于内联结果，应从 `result.data` 中读取结构化子数据，包括 `data.unresolved`；顶层结果对象仅作为封装容器。若需从父对话中终止整个已启动的运行，请在该工具可用时调用 `work_stop`，并将 `work_id` 设置为启动结果中的 `workId`。`interrupt()` 仅会中断一个正在执行的子任务尝试：在等待 `agent.latestAttempt.result()` 之前，先检查 `agent.latestAttempt.getStatus()`，并调用 `agent.latestAttempt.interrupt()`。当 `result()` 解析完成后，该尝试即为最终状态。

为实现后续所有者的恢复，请使用返回的 `scriptPath` 和 `resumeFromRunId` 调用 Workflow 工具；请勿将这两个字段添加到脚本 API 中。将共享的发现与间隙谱系状态机封装在 Agent 调用的外部，并在最终边界处保持未解析的后代不变。