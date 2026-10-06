# JSON 模式

本文档定义了技能创建者所使用的 JSON 模式。

---

## evals.json

定义了某项技能的评估。位于技能目录下的 `evals/evals.json` 文件中。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的示例提示",
      "expected_output": "对预期结果的描述",
      "files": ["evals/files/sample1.pdf"],
      "expectations": [
        "输出包含 X",
        "该技能使用了脚本 Y"
      ]
    }
  ]
}
```

**字段说明：**
- `skill_name`：与技能的 frontmatter 中名称一致
- `evals[].id`：唯一的整数标识符
- `evals[].prompt`：要执行的任务
- `evals[].expected_output`：人类可读的成功描述
- `evals[].files`：可选的输入文件路径列表（相对于技能根目录）
- `evals[].expectations`：可验证的陈述列表

---

## history.json

记录 Improve 模式下的版本演进过程。位于工作区根目录。

```json
{
  "started_at": "2026-01-15T10:30:00Z",
  "skill_name": "pdf",
  "current_best": "v2",
  "iterations": [
    {
      "version": "v0",
      "parent": null,
      "expectation_pass_rate": 0.65,
      "grading_result": "baseline",
      "is_current_best": false
    },
    {
      "version": "v1",
      "parent": "v0",
      "expectation_pass_rate": 0.75,
      "grading_result": "won",
      "is_current_best": false
    },
    {
      "version": "v2",
      "parent": "v1",
      "expectation_pass_rate": 0.85,
      "grading_result": "won",
      "is_current_best": true
    }
  ]
}
```

**字段说明：**
- `started_at`：改进开始的 ISO 时间戳
- `skill_name`：正在改进的技能名称
- `current_best`：表现最佳的版本标识符
- `iterations[].version`：版本标识符（如 v0、v1 等）
- `iterations[].parent`：该版本的父版本
- `iterations[].expectation_pass_rate`：评分通过率
- `iterations[].grading_result`：“baseline”、“won”、“lost”或“tie”
- `iterations[].is_current_best`：是否为当前最佳版本

---

## grading.json

由评分代理生成的输出。位于 `<run-dir>/grading.json` 文件中。

```json
{
  "expectations": [
    {
      "text": "输出中包含姓名 'John Smith'",
      "passed": true,
      "evidence": "在第 3 步的转录中找到：'提取的姓名：John Smith, Sarah Johnson'"
    },
    {
      "text": "表格的 B10 单元格中有 SUM 公式",
      "passed": false,
      "evidence": "未生成任何表格，输出为文本文件。"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "execution_metrics": {
    "tool_calls": {
      "Read": 5,
      "Write": 2,
      "Bash": 8
    },
    "total_tool_calls": 15,
    "total_steps": 6,
    "errors_encountered": 0,
    "output_chars": 12450,
    "transcript_chars": 3200
  },
  "timing": {
    "executor_duration_seconds": 165.0,
    "grader_duration_seconds": 26.0,
    "total_duration_seconds": 191.0
  },
  "claims": [
    {
      "claim": "表单有 12 个可填写字段",
      "type": "factual",
      "verified": true,
      "evidence": "在 field_info.json 中统计到 12 个字段。"
    }
  ],
  "user_notes_summary": {
    "uncertainties": ["使用了 2023 年的数据，可能已过时"],
    "needs_review": [],
    "workarounds": ["对于不可填写的字段，回退到文本叠加显示"]
  },
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "输出中包含姓名 'John Smith'",
        "reason": "即使输出是一份虚构的文档且提到了该姓名，也会被视为通过。"
      }
    ],
    "overall": "断言仅检查是否存在，而未验证其正确性。"
  }
}
```

**字段：**
- `expectations[]`：带有证据的评分期望
- `summary`：通过/未通过的汇总统计
- `execution_metrics`：工具使用情况及输出大小（来自执行器的 metrics.json）
- `timing`：墙钟计时信息（来自 timing.json）
- `claims`：从输出中提取并验证的主张
- `user_notes_summary`：执行器标记的问题摘要
- `eval_feedback`：（可选）针对评估的改进建议，仅在评分者发现值得提出的问题时存在

---

## metrics.json

执行器代理的输出文件。位于 `<run-dir>/outputs/metrics.json`。

```json
{
  "tool_calls": {
    "Read": 5,
    "Write": 2,
    "Bash": 8,
    "Edit": 1,
    "Glob": 2,
    "Grep": 0
  },
  "total_tool_calls": 18,
  "total_steps": 6,
  "files_created": ["filled_form.pdf", "field_values.json"],
  "errors_encountered": 0,
  "output_chars": 12450,
  "transcript_chars": 3200
}
```

**字段：**
- `tool_calls`：各工具类型的调用次数
- `total_tool_calls`：所有工具调用的总次数
- `total_steps`：主要执行步骤的数量
- `files_created`：创建的输出文件列表
- `errors_encountered`：执行过程中遇到的错误数量
- `output_chars`：输出文件的总字符数
- `transcript_chars`：日志文本的字符数

---

## timing.json

一次运行的墙钟计时信息。位于 `<run-dir>/timing.json`。

**如何记录：** 当子代理任务完成时，任务通知中会包含 `total_tokens` 和 `duration_ms`。请立即保存这些数据——它们不会被持久化存储，事后也无法恢复。

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3,
  "executor_start": "2026-01-15T10:30:00Z",
  "executor_end": "2026-01-15T10:32:45Z",
  "executor_duration_seconds": 165.0,
  "grader_start": "2026-01-15T10:32:46Z",
  "grader_end": "2026-01-15T10:33:12Z",
  "grader_duration_seconds": 26.0
}
```

---

## benchmark.json

基准测试模式的输出文件。位于 `benchmarks/<timestamp>/benchmark.json`。

```json
{
  "metadata": {
    "skill_name": "pdf",
    "skill_path": "/path/to/pdf",
    "executor_model": "claude-sonnet-4-20250514",
    "analyzer_model": "most-capable-model",
    "timestamp": "2026-01-15T10:30:00Z",
    "evals_run": [1, 2, 3],
    "runs_per_configuration": 3
  },

  "runs": [
    {
      "eval_id": 1,
      "eval_name": "Ocean",
      "configuration": "with_skill",
      "run_number": 1,
      "result": {
        "pass_rate": 0.85,
        "passed": 6,
        "failed": 1,
        "total": 7,
        "time_seconds": 42.5,
        "tokens": 3800,
        "tool_calls": 18,
        "errors": 0
      },
      "expectations": [
        {"text": "...", "passed": true, "evidence": "..."}
      ],
      "notes": [
        "使用了2023年的数据，可能已过时",
        "对于不可填写的字段回退到文本叠加"
      ]
    }
  ],

  "run_summary": {
    "with_skill": {
      "pass_rate": {"mean": 0.85, "stddev": 0.05, "min": 0.80, "max": 0.90},
      "time_seconds": {"mean": 45.0, "stddev": 12.0, "min": 32.0, "max": 58.0},
      "tokens": {"mean": 3800, "stddev": 400, "min": 3200, "max": 4100}
    },
    "without_skill": {
      "pass_rate": {"mean": 0.35, "stddev": 0.08, "min": 0.28, "max": 0.45},
      "time_seconds": {"mean": 32.0, "stddev": 8.0, "min": 24.0, "max": 42.0},
      "tokens": {"mean": 2100, "stddev": 300, "min": 1800, "max": 2500}
    },
    "delta": {
      "pass_rate": "+0.50",
      "time_seconds": "+13.0",
      "tokens": "+1700"
    }
  },

  "notes": [
    "‘输出为PDF文件’这一断言在两种配置下均100%通过——可能无法体现技能的实际价值",
    "评估3显示出较大的波动性（50% ± 40%）——可能是不稳定性或与模型相关",
    "无技能运行在表格提取相关的期望上始终失败",
    "引入技能后平均执行时间增加了13秒，但通过率提升了50%"
  ]
}
```

**字段：**
- `metadata`: 基准测试运行的相关信息
  - `skill_name`: 技能名称
  - `timestamp`: 基准测试运行的时间
  - `evals_run`: 评估名称或 ID 的列表
  - `runs_per_configuration`: 每个配置的运行次数（例如 3）
- `runs[]`: 单次运行的结果
  - `eval_id`: 数字形式的评估标识符
  - `eval_name`: 易读的评估名称（在查看器中用作章节标题）
  - `configuration`: 必须为 `"with_skill"` 或 `"without_skill"`（查看器使用此确切字符串进行分组和颜色编码）
  - `run_number`: 整数形式的运行编号（1、2、3……）
  - `result`: 嵌套对象，包含 `pass_rate`、`passed`、`total`、`time_seconds`、`tokens`、`errors`
- `run_summary`: 按配置统计的汇总数据
  - `with_skill` / `without_skill`: 每个都包含 `pass_rate`、`time_seconds`、`tokens` 对象，其中带有 `mean` 和 `stddev` 字段
  - `delta`: 差值字符串，如 `"+0.50"`、`"+13.0"`、`"+1700"`
- `notes`: 分析器的自由文本观察记录

**重要提示：** 查看器会严格按这些字段名读取数据。如果将 `configuration` 写成 `config`，或将 `pass_rate` 放在运行结果的顶层而不是嵌套在 `result` 下，查看器就会显示为空值或零值。手动生成 benchmark.json 时，请务必参考此 schema。

---

## comparison.json

盲评比较器的输出文件，位于 `<grading-dir>/comparison-N.json`。

```json
{
  "winner": "A",
  "reasoning": "输出 A 提供了完整的解决方案，格式规范且包含所有必填字段。输出 B 缺少日期字段，且格式存在不一致之处。",
  "rubric": {
    "A": {
      "content": {
        "correctness": 5,
        "completeness": 5,
        "accuracy": 4
      },
      "structure": {
        "organization": 4,
        "formatting": 5,
        "usability": 4
      },
      "content_score": 4.7,
      "structure_score": 4.3,
      "overall_score": 9.0
    },
    "B": {
      "content": {
        "correctness": 3,
        "completeness": 2,
        "accuracy": 3
      },
      "structure": {
        "organization": 3,
        "formatting": 2,
        "usability": 3
      },
      "content_score": 2.7,
      "structure_score": 2.7,
      "overall_score": 5.4
    }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["完整的解决方案", "格式良好", "所有字段齐全"],
      "weaknesses": ["标题部分有轻微的风格不一致"]
    },
    "B": {
      "score": 5,
      "strengths": ["输出可读性好", "基本结构正确"],
      "weaknesses": ["缺少日期字段", "格式不一致", "数据提取不完整"]
    }
  },
  "expectation_results": {
    "A": {
      "passed": 4,
      "total": 5,
      "pass_rate": 0.80,
      "details": [
        {"text": "输出包含姓名", "passed": true}
      ]
    },
    "B": {
      "passed": 3,
      "total": 5,
      "pass_rate": 0.60,
      "details": [
        {"text": "输出包含姓名", "passed": true}
      ]
    }
  }
}
```

---

## analysis.json

事后分析器的输出文件，位于 `<grading-dir>/analysis.json`。

```json
{
  "比较总结": {
    "胜者": "A",
    "胜者技能": "path/to/winner/skill",
    "败者技能": "path/to/loser/skill",
    "比较理由": "简要说明比较者选择胜者的原因"
  },
  "胜者优势": [
    "提供了清晰的多页文档处理步骤说明",
    "包含可检测格式错误的验证脚本"
  ],
  "败者劣势": [
    "模糊的指令‘适当处理文档’导致行为不一致",
    "缺少验证脚本，代理只能临时应变"
  ],
  "指令遵循情况": {
    "胜者": {
      "评分": 9,
      "问题": ["次要：跳过了可选的日志记录步骤"]
    },
    "败者": {
      "评分": 6,
      "问题": [
        "未使用该技能的格式化模板",
        "未按第3步操作，自行设计了处理方法"
      ]
    }
  },
  "改进建议": [
    {
      "优先级": "高",
      "类别": "指令",
      "建议": "将‘适当处理文档’替换为明确的步骤说明",
      "预期影响": "消除导致行为不一致的歧义"
    }
  ],
  "对话记录分析": {
    "胜者执行模式": "阅读技能说明 -> 按照5步流程执行 -> 使用验证脚本",
    "败者执行模式": "阅读技能说明 -> 对处理方式不明确 ->  시도한 방법이 세 가지"
  }
}
```