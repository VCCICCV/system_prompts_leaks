---
name: artifact_testing
metadata: { "不包含在提示中": 假 }
description: 在交付工件之前对其进行验证——无论是文件类交付物（PDF、PPTX、DOCX、XLSX、CSV），还是Web类工件。每当构建即将输出链接，或构建任务需要验证、质量保证（QA）或视觉检查时，均可使用。该功能涵盖各类工件的准入检查脚本、渲染后比对验证，以及残留占位符扫描。
---
# 工件验证

在工件命名空间内实现统一的验证机制。所有类型共用一条规则：通过查看用户将看到的、全新渲染的内容来验证交付物，而不是依赖生成它的代码。当下游门控未通过时，绝不返回链接；在尝试修复三次失败后，应报告具体失败原因并寻求指导，而非反复迭代。

## 文件类型：先门控，后目视

| 类型 | 门控 |
|---|---|
| PDF | 先运行 `render_audit.mjs`，使用 PDF 技能的 `workflow.md` 中列出的参数，再执行 `/opt/hatch/skills/artifacts/scripts/validate_pdf.sh` |
| 演示文稿 | 先运行 `assemble_deck.mjs`，再运行 `render_audit.mjs`，使用 `workflow.md` 中列出的参数 |
| DOCX | 使用无头 LibreOffice 渲染为 PDF（`soffice --headless --convert-to pdf`），然后进行光栅化（`pdftoppm -jpeg -r 100`） |
| XLSX | 执行 `/opt/hatch/skills/artifacts/scripts/validate_xlsx.py` |
| CSV / MD | 重新解析文件（CSV：使用 Python 的 `csv` 模块读取；MD：直接读取文件） |

共享的渲染引擎是 `/opt/hatch/skills/artifacts/scripts/render_audit.mjs`；其渲染时间会超过 `muse.exec` 的默认超时值，因此请将 `yield_ms` 设置为预期运行时间的数倍，并读取由参数指定的报告文件，而不要仅凭后台命令的静默状态来判断。

**然后仔细检查。** 以全新的视角阅读每一张验证用的 PNG 或页面图像——生成上下文往往只看到它所期望的内容，而非实际渲染结果。首先检查文本是否溢出或被截断，接着检查重叠、碰撞、间距过紧或不均、低对比度文本，以及遗留的模板装饰元素。

**占位符扫描。** 在交付前，务必在交付物的文本中搜索残留的临时内容：TODO、lorem、placeholder，以及用户未曾要求的 [insert]、xxx 运行记录和示例行。任何发现的内容都必须修复，不得直接交付。

## Web 工件

Web 构建拥有独立的审计工具（`web_artifacts.build` 和 `web_artifacts.audit` 使用相同的抓取引擎并强制执行相关规则）；本技能中的“全新渲染”和“占位符”规则同样适用于其输出。