# 公式、重新计算与现有工作簿的编辑

## 为什么必须进行重新计算

openpyxl 将公式以纯字符串形式写入，且不缓存计算结果。在真正的引擎对文件进行重新计算之前，所有包含公式的单元格在预览器、pandas 以及 `load_workbook(data_only=True)` 中都会显示为空：未重新计算的交付物对用户而言是空白的。每次保存时若涉及公式，请运行  
`python3 "/opt/hatch/skills/artifacts/scripts/recalc_xlsx.py" output.xlsx`；该脚本会就地重写文件，并返回一个 JSON，其中包含 `status`（`success` 或 `errors_found`）、`total_formulas`、`total_errors`，以及一个 `error_summary`，后者列出出错单元格的位置（每种错误类型最多记录 100 个位置，并附有 `locations_truncated` 计数，因此应以 `total_errors` 为准，而非列表长度）。如果 `errors_found` 的值为 0，则表示成功；若返回的是 `error` 而非 `status`，则意味着未执行任何重新计算。只要报告存在错误，切勿交付；在未证实错误确属原有问题之前，切勿将其归咎于既有错误：请先使用 `data_only=True` 加载原始文件，并检查相关单元格。

链接到其他文件的工作簿属于特殊情况：被链接的文件不在本机上，因此其单元格中仅保留了缓存值，而 openpyxl 再次保存时会清除这些值；随后的重新计算会使每个单元格都变为错误。此类文件在未使用 `--force` 参数时，重新计算脚本将拒绝处理。在覆盖保存之前，请先复制被链接单元格的值。

## 如何选择能够保留的公式

此处用于重新计算的引擎是 LibreOffice，其支持的函数数量少于 Excel；对于无法识别的函数，LibreOffice 会将其原样输出为 `#NAME?`。

- 优先使用经典函数集：`SUM`、`SUMIFS`、`INDEX`、`MATCH`、`IFERROR`、`SUMPRODUCT` 及其衍生版本。
- 2007 年之后引入的新函数名在 XML 中带有前缀存储，而 openpyxl 会原样写入您提供的字符串，因此请直接写 `_xlfn.TEXTJOIN`、`_xlfn.CONCAT`、`_xlfn.IFS`、`_xlfn.SWITCH`、`_xlfn.MAXIFS`、`_xlfn.MINIFS`；若直接写成裸字符串，则会显示为 `#NAME?`。
- 切勿使用动态数组函数：`XLOOKUP`、`XMATCH`、`SORT`、`FILTER`、`UNIQUE`、`SEQUENCE`。即使某些引擎能够识别这些函数，由 openpyxl 写入的文件也不包含溢出元数据，因此只有左上角单元格会获得值，而重新计算后系统会将截断的结果视为无错误。查找时请使用 `INDEX`/`MATCH`，并在写入单元格之前先在 Python 中完成排序、筛选和去重操作。
- 在跨工作表引用中，含空格的工作表名称需用单引号括起：`='Assumptions Inputs'!$B$5`；若未加引号，则会报错。

## openpyxl 的注意事项
- 读取模型需要两次加载：`data_only=True` 返回的是已计算的值，公式已被移除；默认设置则返回公式字符串，但无计算结果。一次遍历无法同时获取两者。
- 使用 `data_only=True` 进行保存时具有破坏性：内存中的工作簿不再包含任何公式，因此保存时会将所有公式替换为文本。
- 对于 openpyxl 刚刚写入的文件，使用 `data_only=True` 打开时所有单元格都会显示为 `None`；请务必先进行重新计算。结果为空字符串的公式也会被读作 `None`，因此仅凭 `None` 无法得出任何结论。
- 合并单元格：只需写入左上角的锚点单元格；合并范围内的其他单元格均为只读。
- `.xlsm` 文件若未使用 `keep_vba=True` 加载，将丢失宏代码。

## 编辑现有工作簿
文件自身的约定优先于此处的所有指导原则。首先找到指定的输入单元格（通常通过不同的字体颜色、填充或底纹加以标识），仅在这些单元格中进行编辑，并保持所有现有公式不变。格式和字体应与现有内容一致，避免重新设置样式。