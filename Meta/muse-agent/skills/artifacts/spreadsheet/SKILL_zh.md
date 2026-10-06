---
name: artifact_spreadsheet
metadata: { "不包含在提示中": 假 }
description: 创建、读取、编辑、修复或清理电子表格文件（.xlsx、.xlsm、.csv、.tsv）。当构建任务的工件类型为电子表格，或任务指定了一个电子表格文件并需要对其进行处理或从中生成内容时使用，包括将杂乱的表格数据重新整理为规范的工作簿。涵盖 openpyxl 的生成、公式与重新计算、现有工作簿的编辑以及验证检查。不适用于其交付物仅为包含表格的文档、报告或网页的任务。
---
# 电子表格工件

使用预装的 `openpyxl` 库，通过一个生成脚本创建了一个 `.xlsx` 文件。请将该生成脚本保留在 `.src/` 目录下：它是未来修订时可编辑的源文件，二进制文件始终由此重新生成。

| 任务 | 路径 |
|---|---|
| 创建或使用公式与格式进行编辑 | `openpyxl` |
| 快速查看现有工作表 | `muse.read`（转换为 Markdown 并分页显示）；它不携带单元格坐标，因此切勿计划在此基础上进行编辑 |
| 读取工作簿的模型（包含公式及其值） | 需两次调用 `load_workbook`；参见 `/opt/hatch/skills/artifacts/spreadsheet/references/formulas.md` |
| 大量或杂乱的表格数据的导入导出 | 使用 Python 标准库中的 `csv` 模块；当任务确实需要时，可执行 `pip install --break-system-packages pandas` |
| 设计、结构、数字格式、模型约定 | `/opt/hatch/skills/artifacts/spreadsheet/references/visual.md` |
| 公式、重新计算、编辑现有工作簿 | `/opt/hatch/skills/artifacts/spreadsheet/references/formulas.md` |

对每份交付的工作簿的要求：

- 设置非空的 `workbook.properties.title`，并指定一个便于人类阅读的文件名。数据仅来自构建过程中收集的内容；对于您无法获取的值，应留空单元格或以问号表示，绝不能凭空捏造数值。
- 编写公式，而非预先计算的结果：例如 `sheet["B10"] = "=SUM(B2:B9)"`，而不是由 Python 计算出的总和。工作表必须在其输入发生变化时重新计算。
- 严格按照用户规格执行：包括准确的标签名称、精确的列标题，以及用户明确给出的公式。即使重新设计实现了更优雅的功能，只要计算结果不同，也算不合格。
- 交付时确保公式无错误：运行下方的重新计算检查，并修复报告中指出的问题。
- 在读者可见的位置（如单元格注释或相邻的标注单元格）记录假设和硬编码数值；如有真实来源，请注明出处；若数值来自用户，则应明确说明。
- 为供他人填写而创建的工作簿，需附带简短说明，标明可编辑的单元格及一行具有实际意义的示例数据；切勿在要求您编辑的文件中添加示例行。

## 脚本

| 脚本 | 功能 |
|---|---|
| `/opt/hatch/skills/artifacts/scripts/recalc_xlsx.py` | 通过无头模式的 LibreOffice 对工作簿进行就地重新计算，并以 JSON 格式报告所有公式错误的单元格；只要文件包含公式，此脚本为必选。`errors_found` 返回 0 表示有错误：请查看 JSON 输出，而非退出码 |
| `/opt/hatch/skills/artifacts/scripts/validate_xlsx.py` | 打开并回读工作簿，检查 ZIP 文件完整性、工作表各部分及已填充单元格数量；若检查未通过，不得提供下载链接。其 CSV 输出无需额外校验 |

## 验证流程

先运行 `recalc_xlsx.py`（若有公式），再运行 `validate_xlsx.py`，然后按照 `/opt/hatch/skills/artifacts/testing/SKILL.md` 的要求进行后续操作。成功的重新计算仅证明公式能够正确求值，但并不保证其结果正确；在构建完整网格之前，应抽查两到三个公式，确认其返回预期值。