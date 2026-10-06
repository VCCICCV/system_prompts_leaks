# Brother 打印机：IPP 打印

**最后验证日期：** 2026-09-14，测试设备为 Brother MFC-J880DW

当通过自动发现机制识别到一台支持 IPP 的 Brother 打印机，且用户请求打印时，请参考本指南。

## 推荐流程

1. 查询打印机当前的 IPP 能力。如果支持，优先选择 `image/pwg-raster` 格式，其次才是 PDF 或 JPEG。
2. 使用标准 CUPS 工具（如 `imagetoraster` 和 `rastertopwg`）生成 PWG Raster 格式数据。
3. 根据打印机公布的参数，匹配纸张类型、分辨率、可打印区域及其他打印选项，并按该分辨率调整版面和文本大小。
4. 通过 Home Link 以 IPP `Print-Job` 作业提交，并指定文档格式为 `image/pwg-raster`。在重试前，请检查作业状态及打印输出。