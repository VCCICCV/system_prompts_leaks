# 示例：Electron 桌面 GUI 应用

Electron 应用有一个窗口。而在无头容器中运行的未来智能体是无法看到这个窗口的。因此，你在这里的交付物并不是一个写着“`npm start` 会打开一个窗口”的 Markdown 文件，而是一个**驱动脚本**，它会在 xvfb 下启动应用，提供一个命令 REPL（点击、输入、截屏），并允许智能体通过发送文本行来操作 UI。

于是，该技能的 `SKILL.md` 就变成了这份驱动脚本的简明使用手册。

## 你要构建的内容

```
apps/desktop/
  .claude/skills/run-desktop/
    SKILL.md               <- 简短说明：“运行驱动脚本，以下是可用命令”
    driver.mjs             <- REPL：将标准输入的命令转换为 Playwright 操作
```

驱动脚本就是最终的产品。如果没有它，这个技能就只能描述一个智能体永远无法触碰的 GUI。

**晋升路径：** 如果驱动脚本中包含了项目的真实端到端测试套件也想复用的启动辅助功能，可以将其移至 `e2e-playwright/driver.mjs`（或 `scripts/drive.mjs`），并更新技能的路径引用。技能仍保留在 `.claude/skills/run-desktop/`；而驱动脚本则找到更合适的归属位置。

## 第一步 - 让应用在 xvfb 下至少能够启动

这通常是难度最大的一步，也是最容易遇到问题的地方。README 里可能会写着“仅支持 macOS/Windows”，请忽略这一点。安装 xvfb 和 Chromium 的共享库，找到 Electron 可执行文件并启动它：

```bash
apt-get install -y xvfb libnss3 libgbm1 libasound2t64 libgtk-3-0 \
  libxss1 libxkbcommon0 libatk-bridge2.0-0 libcups2 libdrm2

# 先构建应用。通常，“dev” 脚本是 electron-forge，它会先进行 Vite/webpack 构建，然后再启动。你只需要构建过程：
npm install
npx electron-forge start &   # 构建 .vite/build/ 或 dist/
sleep 20 && kill %1          # 构建完成后杀死进程——接下来由你手动启动

# 现在尝试直接启动
xvfb-run -a node -e "
  const { _electron } = require('playwright-core');
  _electron.launch({
    executablePath: './node_modules/electron/dist/electron',
    args: ['--no-sandbox', '.'],
    timeout: 30000,
  }).then(app => {
    console.log('已启动，窗口 URL:', app.windows().map(w => w.url()));
    return app.close();
  });
"
```

反复迭代，直到成功启动。每缺少一个 `.so` 文件，就增加一个 `apt-get` 包，并在先决条件中相应地添加一行。每次启动超时，检查 `nodeCliInspect` 的 fuse 是否被禁用，并确认构建输出文件存在。

**在容器中几乎总是需要使用 `--no-sandbox` 选项。** Electron 的沙箱机制需要 `CAP_SYS_ADMIN` 权限或用户命名空间，而这些权限在默认情况下均未启用。

## 步骤 2 - 构建 REPL 驱动程序

一旦能够成功启动应用，就把那个临时脚本改造成一个 REPL。初始阶段保持最小化——后续会根据需求逐步添加命令。**REPL 的设计是恰当的**，因为代理可以在 tmux 中运行它，并在每次交互时无需重新启动（速度较慢的）应用，从而实现快速迭代。
```javascript
// .claude/skills/run-<unit>/driver.mjs
// 用于 <app> 的 REPL 驱动程序。在无头 Linux 系统上通过 xvfb 运行。
// 专为代理设计：包裹在 tmux 中，发送命令键，捕获窗格输出。
import { _electron as electron } from 'playwright-core';
import * as readline from 'node:readline';
import * as fs from 'node:fs';
import * as path from 'node:path';

const APP_DIR = path.resolve(import.meta.dirname, '../../..');
const SHOT_DIR = process.env.SCREENSHOT_DIR || '/tmp/shots';
fs.mkdirSync(SHOT_DIR, { recursive: true });

let app = null;
let page = null;   // 您实际交互的窗口/页面

const electronBin = process.platform === 'darwin'
  ? path.join(APP_DIR, 'node_modules/electron/dist/Electron.app/Contents/MacOS/Electron')
  : path.join(APP_DIR, 'node_modules/electron/dist/electron');

const COMMANDS = {
  async launch() {
    if (app) return console.log('已启动');
    app = await electron.launch({
      executablePath: electronBin,
      args: ['--no-sandbox', APP_DIR],
      env: { ...process.env, DISPLAY: process.env.DISPLAY || ':99' },
      timeout: 30_000,
    });
    // Electron 没有明确的“加载完成”信号——此处的延时只是个猜测。
    // 一旦确定该应用的就绪状态，可用轮询替代：
    // 等待 windows() 包含预期的 URL，或对 firstWindow() 使用 waitForSelector。
    await new Promise(r => setTimeout(r, 8_000));
    // 查找真正的 UI 页面。通常不是 firstWindow()——可能是启动画面，
    // 或者实际内容位于 BrowserView 叠加层中。
    page = app.windows().find(w => !w.url().startsWith('devtools://'))
        ?? await app.firstWindow();
    console.log('已启动。共有', app.windows().length, '个窗口：');
    for (const w of app.windows()) console.log(' ', w.url());
  },

  async ss(name) {
    if (!page) return console.log('错误：请先启动');
    const f = path.join(SHOT_DIR, (name || `ss-${Date.now()}`) + '.png');
    await page.screenshot({ path: f });
    console.log('截图：', f);
  },

  // 通过 evaluate() 进行点击，而非 locator.click()。如果内容位于主窗口之上的 BrowserView 中，
  // Playwright 的坐标计算会击中错误的图层。使用 DOM 的 .click() 总是有效的。
  async click(sel) {
    if (!page) return console.log('错误：请先启动');
    const r = await page.evaluate(s => {
      const el = document.querySelector(s);
      if (!el) return '未找到';
      el.click(); return '成功';
    }, sel);
    console.log('点击', sel, '->', r);
  },

  async 'click-text'(text) {
    if (!page) return console.log('错误：请先启动');
    const r = await page.evaluate(t => {
      const els = [...document.querySelectorAll('button, a, [role="button"]')];
      const el = els.find(e => e.textContent?.trim() === t)
              ?? els.find(e => e.textContent?.includes(t));
      if (!el) return '未找到';
      el.click(); return '成功：' + el.tagName;
    }, text);
    console.log('点击文本', JSON.stringify(text), '->', r);
  },

  async type(text)  { if (page) await page.keyboard.type(text, { delay: 30 }); },
  async press(key)  { if (page) await page.keyboard.press(key); },

  async wait(sel) {
    if (!page) return console.log('错误：请先启动');
    try { await page.waitForSelector(sel, { timeout: 10_000 }); console.log('找到：', sel); }
    catch { console.log('超时：', sel); }
  },

  async eval(expr) {
    if (!page) return console.log('错误：请先启动');
    try { console.log(JSON.stringify(await page.evaluate(expr))); }
    catch (e) { console.log('错误：', e.message); }
  },

  async text(sel) {
    if (!page) return console.log('错误：请先启动');
    console.log(await page.evaluate(
      s => (s ? document.querySelector(s) : document.body)?.innerText ?? '(null)',
      sel || null));
  },

  // 内省：对于确定哪个窗口或 webContents 实际包含 UI 至关重要。Electron 应用通常会创建多个。
  async windows() {
    if (!app) return console.log('错误：请先启动');
    for (const w of app.windows()) console.log(' ', w.url());
    const wcs = await app.evaluate(({ webContents }) =>
      webContents.getAllWebContents().map(w => ({ id: w.id, type: w.getType(), url: w.getURL() })));
    console.log('webContents：');
    for (const w of wcs) console.log(` [${w.id}] ${w.type}：${w.url}`);
  },

  async quit() { if (app) await app.close().catch(()=>{}); app = null; page = null; },
  help() { console.log('命令：', Object.keys(COMMANDS).join(', ')); },
};

// 阻止 Electron 盗用标准输入——使用原始文件描述符。
const stdin = fs.createReadStream(null, { fd: fs.openSync('/dev/stdin', 'r') });
const rl = readline.createInterface({ input: stdin, output: process.stdout, prompt: 'driver> ' });

rl.on('line', async line => {
  const [cmd, ...rest] = line.trim().split(/\s+/);
  if (!cmd) return rl.prompt();
  const fn = COMMANDS[cmd];
  if (!fn) { console.log('未知命令：', cmd, '- 请尝试：help'); return rl.prompt(); }
  try { await fn(rest.join(' ')); } catch (e) { console.log('错误：', e.message); }
  if (cmd === 'quit') { rl.close(); process.exit(0); }
  rl.prompt();
});
rl.on('close', async () => { await COMMANDS.quit(); process.exit(0); });

console.log('<app> 驱动程序 - 输入 "help" 查看命令，输入 "launch" 启动');
rl.prompt();
```

**这是一个初始框架。** 在尝试进入应用的有趣部分时，你会添加一些应用特定的命令：导航到某个视图、聚焦某种特殊的输入类型、绕过认证流程，等等。这些命令都包含了你辛苦积累的经验——请保留它们。

## 第3步 - 通过 tmux 自己使用

以与下一个代理相同的方式运行驱动程序：

```bash
tmux new-session -d -s app -x 200 -y 50
tmux send-keys -t app 'cd /workspace/apps/desktop && xvfb-run -a node .claude/skills/run-desktop/driver.mjs' Enter
timeout 20 bash -c 'until tmux capture-pane -t app -p | grep -q "driver>"; do sleep 0.2; done'
tmux send-keys -t app 'launch' Enter
timeout 60 bash -c 'until tmux capture-pane -t app -p | grep -q "launched"; do sleep 0.2; done'
tmux send-keys -t app 'ss 01-landing' Enter
timeout 10 bash -c 'until tmux capture-pane -t app -p | grep -q "screenshot:"; do sleep 0.2; done'
tmux send-keys -t app 'windows' Enter    # 哪个页面才是真正的界面？
tmux capture-pane -t app -p
```

然后打开 `/tmp/shots/01-landing.png`。这是应用的界面吗？是空白的吗？还是登录页面？每种情况都会告诉你下一步该做什么。

继续操作——点击进入主要功能、填写表单、查看结果并截图。驱动程序会根据需要逐步扩展各种命令（如 `focus-input`、`goto-settings`、`login-as-test-user` 等）。当一个真实的流程能够端到端地运行时，你就完成了构建，可以开始编写了。

## 第4步 - 编写 SKILL.md

保持简短。驱动程序是核心；`SKILL.md` 是使用手册。推荐的结构如下：

> ---
> 名称：run-desktop
> 描述：构建、运行并操控 `<app>` Electron 桌面应用。当需要启动桌面应用、截取其屏幕截图、构建它，或与它的 UI 交互时使用。
> ---
>
> `<App>` 是一个 Electron 桌面应用。对于自动化代理的使用，请通过位于 xvfb 环境下的 Playwright REPL 脚本 `.claude/skills/run-desktop/driver.mjs` 来操控它。启动过程较慢（约 10 秒），且主要的有趣 UI 部分位于 BrowserView 中，而非主窗口——驱动脚本会同时处理这两者。
>
> 所有路径均以 `apps/desktop/` 为基准。
>
> ## 前置条件
>
> ```bash
> apt-get install -y xvfb libnss3 libgbm1 libasound2t64 libgtk-3-0 \
>   libxss1 libxkbcommon0 libatk-bridge2.0-0 libcups2 libdrm2
> ```
>
> ## 构建
>
> ```bash
> npm install
> npx electron-forge start   # 会构建出 .vite/build/ 目录 - 构建完成后按 Ctrl-C 停止
> # `<你可能需要应用的任何补丁：例如修改功能开关等>`
> ```
>
> ## 运行（代理模式）
>
> ```bash
> cd apps/desktop
> xvfb-run -a node .claude/skills/run-desktop/driver.mjs
> ```
>
> 可用 tmux 封装以实现交互式使用：
>
> ```bash
> tmux new-session -d -s app -x 200 -y 50
> tmux send-keys -t app 'cd apps/desktop && xvfb-run -a node .claude/skills/run-desktop/driver.mjs' Enter
> timeout 20 bash -c 'until tmux capture-pane -t app -p | grep -q "driver>"; do sleep 0.2; done'
> tmux send-keys -t app 'launch' Enter
> timeout 60 bash -c 'until tmux capture-pane -t app -p | grep -q "launched"; do sleep 0.2; done'
> tmux send-keys -t app 'ss landing' Enter
> tmux capture-pane -t app -p
> ```
>
> 截图会保存到 `/tmp/shots/` 目录（可通过设置 `SCREENSHOT_DIR` 环境变量来更改）。
>
> ### 命令说明
>
> | 命令       | 功能描述                     |
> |------------|------------------------------|
> | `launch`   | 启动应用，等待窗口出现       |
> | `ss [name]`| 截图并保存为 `/tmp/shots/<name>.png` |
> | `click <css-sel>` | 点击指定 CSS 选择器对应的元素（基于 DOM，而非坐标——详见注意事项） |
> | `click-text <text>` | 点击包含指定文本的按钮或链接 |
> | `type <text>` / `press <key>` | 输入文本或按键操作           |
> | `wait <css-sel>` | 等待指定元素出现，超时时间为 10 秒 |
> | `eval <js>` | 在页面上下文中执行 JavaScript，并打印 JSON 结果 |
> | `text [css-sel]` | 输出指定元素的 innerText     |
> | `windows`  | 列出所有窗口及 webContents（用于定位真正的 UI） |
> | `quit`     | 关闭应用并退出               |
>
> 此外，还可使用你自定义的应用特定命令：<你的命令> - <功能说明>。
>
> ## 运行（人工模式）
>
> ```bash
> npm start   # 会打开一个窗口；在无头模式下无用。按 Ctrl-C 退出。
> ```
>
> ## 注意事项
>
> - **`<你遇到的具体问题>`** - `<原因>` -> <修复方案/ workaround>
> - `<其他实际遇到的问题，仅限真实案例，不提供通用建议>`
>
> ## 故障排除
>
> - **启动超时（30 秒）：** 构建输出缺失？请重新执行构建步骤。`nodeCliInspect` 熔断未启用？会导致 Playwright 无法附加调试器；开发构建中请勿禁用该熔断。
> - **“缺少 X 服务器”：** 忘记使用 `xvfb-run`。无头 Linux 系统必须使用它。
> - **Xvfb 锁文件残留：** `rm -f /tmp/.X*-lock; pkill Xvfb`
> - **`<你实际遇到的其他问题>`**

## 你会遇到的障碍（归入“陷阱”类别）

这些都是真实 Electron 应用中的常见模式，你很可能会遇到其中的一部分：

- **`firstWindow()` 打开的是启动/加载界面，而不是应用本身。** 需要等待更长时间，或者通过 URL 定位到正确页面，又或者等待某个只有在应用真正准备好时才会出现的特定选择器。

- **真正的 UI 在 BrowserView 中，而不是 BrowserWindow 中。** Playwright 会将其识别为一个独立的“窗口”，且 URL 不同。`windows` 命令正是用来排查这种情况的。在较新的 Electron 版本中，`getBrowserViews()` 可能也会返回空；此时应改用 `webContents.getAllWebContents()`。

- **`locator.click()` 点击了错误的元素。** Playwright 是基于主窗口计算点击坐标，而如果你的内容位于 BrowserView 叠加层中，这些坐标会落在其背后的窗口上。驱动框架之所以使用 `page.evaluate(el => el.click())`，就是为了绕过坐标计算，直接触发 DOM 级别的点击。

- **功能门控会阻止你要测试的功能。** 应用可能会检查订阅等级、环境标志，或内嵌在 SSR HTML 中的特性开关。找到检查发生的代码位置（在构建产物中搜索该门控名称），并在本地运行时对其进行修补——例如对构建输出执行 `sed` 替换、覆盖环境变量，或者对于内嵌于 SSR 的标志，通过 CDP 的 `Fetch.enable` 拦截响应并实时修改。务必详细记录你做了哪些修改以及原因。

- **`contentEditable` 输入框**（如 ProseMirror、Tiptap、Slate）并非 `<textarea>`。`fill()` 方法无法正常工作。应先聚焦该元素，再使用 `keyboard.type()`。如果应用中有此类输入框，可新增一条 `focus <sel>` 命令。

- **Electron 会劫持标准输入。** 框架中的 `fs.openSync('/dev/stdin', 'r')` + `createReadStream` 技巧可以保护你的 REPL 输入不受干扰。

- **原生模块无法加载**（如钥匙串、通知等）。通常这不会导致严重问题——核心应用仍能运行，只是这些功能会无操作地跳过。只需记录下来，继续测试即可。