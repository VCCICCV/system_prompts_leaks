# 在 Muse 虚拟机上安装 magic-moment

无需任何安装操作。该技能的 tar 包中已包含代码和字体（磁盘占用约 13MB，tar 包约 11MB——不含头像；头像数据按用户存储，并在容器构建时从虚拟机读取），而渲染所需的所有其他内容均已随虚拟机镜像一同提供：

| 组件 | 部署位置 |
|---|---|
| 图像处理（PIL 10.2） | 容器镜像中（`python3-pil`） |
| ffmpeg / ffprobe | 容器镜像中（`/usr/bin`） |
| 截图浏览器驱动 | 技能包中的 playwright-core，位于 `/opt/hatch/skills/spaces/ts-runtime/dist/node_modules`（遵循包契约），运行于容器内的 Node.js 环境 |
| 浏览器 | 镜像预置的 `/opt/meta-chromium/chrome` |
| 语音转文字 | 守护进程沙箱 API → 推理代理 → 主机 ASR 服务；容器内的 ffmpeg 提取 16kHz 单声道音频，ffprobe 获取视频片段时长 |

## 验证

只需一条命令，由将负责执行渲染的主体运行——通常为容器内的 Agent 自身。无需 sudo、无需 root 权限，也不需要网络连接：

```bash
bash <skill-dir>/install.sh          # 数秒完成，幂等操作
bash <skill-dir>/install.sh --check  # 几百毫秒的前置检查，绝不产生任何变更
```

完整执行会验证已部署栈的各个环节，通过真实截图流程进行一次烟雾测试，并清理用户主目录中遗留的旧版依赖层（从 WeasyPrint 时代到 pip-playwright 时代，最多可回收约 500MB 的冗余数据）。

无需重启守护进程：技能发现基于内容哈希，Agent 下次轮询时即可识别新技能。

规则制定的原因：
- **切勿手动安装任何组件。** 已部署的完整栈即为流水线所适配的契约。若 `install.sh` 报告栈不完整，则说明该虚拟机镜像版本过旧——应报告此环境无法支持该技能，切勿通过 pip、npm 或自行下载浏览器等方式临时补救。
- **开发机环境**（无容器镜像）需显式覆盖各环节：`MM_NODE`、`MM_PLAYWRIGHT_MODULES`、`JARVIS_CHROMIUM_BINARY`、`MM_FFMPEG`。

## 更新

请通过 Jarvis 支持的部署流程更新打包的技能。切勿在用户工作区中解压出单独的技能副本。标准运行时路径为 `/opt/hatch/skills/magic-moment/`。

快速检查用于确认依赖项是否存在；完整检查则会执行静态截图流程。至于语音转文字、动态截图以及最终的音视频构建，仍需在具体运行时进行验证。