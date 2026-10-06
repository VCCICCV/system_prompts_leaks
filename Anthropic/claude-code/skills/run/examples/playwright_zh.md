# 示例：浏览器驱动的 Web 应用

你有一个开发服务器，向浏览器提供 HTML。无头容器中的代理无法打开浏览器窗口——因此，“运行应用”意味着启动开发服务器，用无头 Chromium 对其进行操作，并生成一张截图来证明页面已正确渲染。

不要编写浏览器驱动程序，使用 `chromium-cli` 即可。

## 开发服务器

找到开发命令（`package.json` 中的 `scripts.dev`、`Makefile`、README），在后台启动它，并等待其真正开始服务：

```bash
npm run dev &   # 或者 yarn dev、pnpm dev、make serve、./dev.sh
timeout 30 bash -c 'until curl -sf http://localhost:3000 >/dev/null; do sleep 1; done'
```

不要简单地 `sleep 5`，而要轮询端口。在重新启动之前，先通过杀死该端口的监听进程来停止服务——`lsof -ti:3000 -sTCP:LISTEN | xargs -r kill`——否则下一次运行会报 `EADDRINUSE` 错误。（`npm run dev &` 后的 `$!` 只是 npm 的包装进程；npm 不会将 SIGTERM 信号转发给它启动的服务器，因此真正释放端口的是杀掉监听进程这一步。）避免使用带有宽泛模式的 `pkill -f`，因为它可能会匹配到代理自身的命令行并终止整个会话。

## 驱动

`chromium-cli` 是一个无头 Chromium 的 REPL。将脚本通过管道传入标准输入：

```bash
chromium-cli --session app <<'EOF'
nav http://localhost:3000
wait-for text=Dashboard
screenshot
click button:has-text("New item")
fill input[name="title"] Smoke test
press Enter
wait-for text=Smoke test
screenshot
console --errors
EOF
```

截图会保存在 `chromium_cli/sessions/app/screenshots/` 目录中（最新的一张会被软链接为 `screenshot.png`）。这就是整个流程：`nav` -> 等待所需元素出现 -> 执行操作（`click` / `fill` / `type` / `press`）-> 截图 -> 检查控制台错误信息以确保没有异常抛出。完整命令参考：`chromium-cli` 技能文档，或在提示符下输入 `help`。

对于迭代式调试，可以在 tmux 下运行，并逐条发送命令——使用相同的命令和同一个会话。

**如果 `chromium-cli` 不可用：** 可以参考 [electron.md](electron.md) 中的 REPL 驱动——结构和命令可以照搬，但它是 `_electron` 特有的：改用 `import { chromium }`，通过 `chromium.launch({ args: ['--no-sandbox'] })` 启动，通过 `(await app.newContext()).newPage()` 获取页面并调用 `goto()` 访问你的开发 URL，同时去掉仅 Electron 才有的窗口检查相关代码（`.windows()`/`.firstWindow()`/`windows` 命令）。

## 技能中应包含的内容

只放项目相关的部分，`chromium-cli` 负责处理底层逻辑。

- **开发命令 + 端口 + 停止方式。** 确切的启动命令、所需的环境变量，以及用于停止服务的 `kill` 命令。
- **认证。** 任何能获取已登录会话的方式——例如 `set-cookie` 行、登录表单的 `fill`/`click` 流程，或者一个执行 API 交互并输出 Cookie 的辅助脚本。
- **一个具有代表性的交互流程。** 不需要覆盖整个应用，只需一条能证明应用正在运行的路径，并以截图结束。
- **应用特有的陷阱。** 只记录实际遇到的问题。

## 常见问题与解决方法

- **React 控制的输入框。** 使用 `eval el.value = '...'` 并不会触发 React 的 onChange 事件。请使用 `fill` 或 `type`，它们会通过 Playwright 的输入管道处理。
- **WebSocket / 长轮询。** `wait-idle` 可能永远无法结束。请直接等待你需要的元素出现。
- **首次渲染较慢。** Vite 和 Next.js 是按需编译路由的，第一次 `nav` 可能需要 10 秒以上。使用 `wait-for` 可以应对这种情况，而简单的 `sleep` 则不行。
- **`screenshot-element <sel>`** 只截取指定元素，适用于差异仅出现在某个特定组件时，而不是整个页面。
- **在宣布成功前务必检查 `console --errors`。** 页面可能已经渲染出了外壳，但所有数据请求都返回了 500 错误。