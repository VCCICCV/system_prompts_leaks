# ReportFindings 工具

将代码审查结果以类型化列表的形式报告，以便宿主界面进行渲染。仅当当前的代码审查说明要求使用此工具报告结果时才使用；否则，请按照那些说明中指定的输出格式进行操作。在报告一次审查的结果时，只需调用一次该工具，并按严重程度从高到低排序已验证的发现（如果没有发现通过验证，则传入空数组），且不要同时以文本形式打印这些发现。在应用修复后重新报告结果时（仅当应用说明要求这样做时），请将每个发现的 `outcome` 设置为实际发生的情况。

## 输入模式

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "level": {
      "description": "审查执行的力度级别",
      "type": "string",
      "enum": [
        "low",
        "medium",
        "high",
        "xhigh",
        "max"
      ]
    },
    "findings": {
      "description": "已验证的发现，按严重程度从高到低排列；若无发现通过验证，则为空数组",
      "maxItems": 32,
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": {
            "description": "发现所在文件的仓库相对路径",
            "type": "string"
          },
          "line": {
            "description": "发现所定位的行号（从1开始计数）",
            "type": "integer",
            "minimum": -9007199254740991,
            "maximum": 9007199254740991
          },
          "summary": {
            "description": "缺陷的一句话概述",
            "type": "string"
          },
          "short_summary": {
            "description": "用于紧凑型界面的简短标签（≤60字符）：仅包含问题描述，不含原因或后果部分",
            "type": "string",
            "maxLength": 60
          },
          "failure_scenario": {
            "description": "具体的输入/状态 → 错误的输出/崩溃",
            "type": "string"
          },
          "category": {
            "description": "发现类型的短小kebab-case标识符，例如：'correctness'、'simplification'、'efficiency'、'test-coverage'",
            "type": "string",
            "maxLength": 40
          },
          "verdict": {
            "description": "仅在执行过验证通过步骤时设置；对于仅内联进行的审查则不存在",
            "type": "string",
            "enum": [
              "CONFIRMED",
              "PLAUSIBLE"
            ]
          },
          "outcome": {
            "description": "仅在应用修复后重新报告时设置：该发现的实际处理结果",
            "type": "string",
            "enum": [
              "fixed",
              "skipped",
              "no_change_needed"
            ]
          }
        },
        "required": [
          "file",
          "summary",
          "failure_scenario"
        ],
        "additionalProperties": false
      }
    }
  },
  "required": [
    "findings"
  ],
  "additionalProperties": false
}
```