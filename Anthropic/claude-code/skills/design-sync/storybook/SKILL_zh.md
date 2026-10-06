# Storybook 源码形态

Storybook 是**保真度的“裁判”，而非运行时环境**。转换器会将包的已编译 `dist/` 目录打包成 `_ds_bundle.js`——与 claude.ai/design 代理构建所用的同一份包——并通过**直接编译故事源模块本身**（包括 hooks、fixtures、本地辅助函数等，整个闭包都会被带上）来生成每个预览；其中，所有组件的导入都会解析到该发布的包中（`lib/story-imports.mjs` 会将包内及相对路径的组件导入重定向至 `window.<Global>`）。仓库自身的 Storybook 渲染结果是这些预览必须匹配的“金标准”：对比工具会并排截取参考 Storybook 和对应预览渲染中的每个故事，并持续迭代直至两者一致。不会上传任何来自 storybook-static 的内容，且在构建时也从不执行任何故事代码——故事仅在浏览器中、基于真实产物运行。

此形态**需要** React 18+，并且 Playwright + Chromium 是**必需**的（对比循环即验证环节），并非可选项。

**首次同步还是重新同步？** 如果配置中 `projectId` 和 `pkg` 在本次运行开始前均已存在，则视为重新同步——此时本文大部分内容不适用，请直接跳转至 §7，由一次驱动运行统筹工作，未触及的组件无需额外成本。其余情况则需走完整流程（§2 构建 → §3 自我修复 → §4 对齐 → 规范说明头（基础 SKILL.md，在上传前）→ §6 上传），其中每个组件都会被验证并评分一次——这同样适用于因中断而遗留的部分配置，以及本次运行刚记录在基础技能文档 §1 中的固定版本。（仅存在旧版 `design-sync.config.json`？请先将其移动并提交：`mkdir -p .design-sync && mv -n design-sync.config.json .design-sync/config.json`，然后执行同样的检测。）

## 2. 先构建，再运行转换器

1. **构建 DS 包及其工作区依赖**。转换器会将 `dist/` 打包到 `window.<Global>` 中。执行 `<pm> run build`；在 monorepo 中可使用 `turbo run build --filter=<pkg>` 或 `pnpm -F "<pkg>..." build`（务必加上尾部的 `...`——仅写 `-F <pkg>` 会跳过依赖，导致出现 `Cannot find module '@scope/tokens'` 错误）。如果 `package.json` 的 `module`/`exports['.']` 指向的是 TS 源码，请找到实际的已编译入口，并通过 `--entry` 参数指定。**务必在步骤 2 之前完成此操作**——Storybook 经常会从其兄弟包的已编译 `dist/` 中导入模块。
2. **将参考 Storybook 构建一次，输出至 `.design-sync/sb-reference/`**——切勿放在 `ds-bundle/` 下（转换器每次重建都会清空 `--out` 目录，而 Storybook 构建耗时数分钟；参考版本必须在修复循环中保留）：

 
```bash
   npx storybook build -c <storybookConfigDir> -o .design-sync/sb-reference
 
```

   请在包含 Storybook 开发依赖的目录下运行此命令——通常是带有 `.storybook/` 的那个目录；monorepo 中往往有多个 Storybook，应选择覆盖当前待同步包的那个。**确保 `-o` 使用的是仓库根路径**（例如：`-o "$(git rev-parse --show-toplevel)/.design-sync/sb-reference"`）：转换器和对比工具均以仓库根目录为基准解析 `.design-sync/`，若在子包中使用相对于当前目录的 `-o`，参考版本将被放置在无人能找到的位置。请直接使用 `npx storybook build`，**不要**使用仓库的 `npm run build-storybook` 脚本（输出目录错误）。随后检查 `.design-sync/sb-reference/iframe.html` 是否存在且大于 10KB——即使构建失败，也可能仅留下 `index.json` 文件。

   长时间构建任务：请**仅通过 shell 工具的后台模式**将其置于后台，并等待完成通知。切勿直接使用 `&`（无法跟踪，通知永远不会到达），也不要使用 `pgrep -f '<script>'` 的轮询循环（它会匹配自身命令行并持续运行直至超时）。对于无头或 `-p` 会话，请改为同步执行长时间命令——此类环境中没有任务通知的重新调用机制，因此置于后台的任务将永远无法恢复。`.gitignore` 增加项：`.design-sync/sb-reference/`、`.design-sync/learnings/`、`.design-sync/.cache/`、`.design-sync/node_modules`（分叉符号链接——每次克隆时重新创建）、`.ds-sync/`、`ds-bundle/`——构建产物、临时工作区、验证工作状态、符号链接、暂存脚本、重新生成的输出。已提交：持久化集合（非 Storybook §2 中的规则，此处相同：`.design-sync/` 下的所有内容均未被 .gitignore 排除——`previews/` 仅存放您编写的文件；生成的 story 模块包装位于 `.design-sync/.cache/previews/`，每次构建都会重新生成；转换器从不在 `previews/` 中写入或删除任何内容）。验证状态从不提交——跨机器的传递依赖于上传项目中的 `_ds_sync.json`。仅当故事或 DS 源文件发生变化时才重建参考。
3. **编写 `.design-sync/config.json`**——仅需指定 `pkg` 和 `globalName`。**如果该文件已存在，请先读取并保留原有内容**——`titleMap`、`overrides` 和 `provider` 会累积之前同步的修复。同时请先阅读 `.design-sync/NOTES.md`——其“**重新同步风险**”部分列出了上一次运行时的关注清单；应针对这些条目重新验证，而不要假定它们会被自动继承。`../non-storybook/SKILL.md` §2.6 中的包结构字段表在此完全适用；其中最重要的字段如下：

   | 字段 | 值 |
   |---|---|
   | `pkg` / `globalName` | `pkg` 必填；省略时会根据 `pkg` 自动推导出 `globalName` |
   | `shape` | `"storybook"`——用于固定检测方式 |
   | `storybookStatic` | `".design-sync/sb-reference"`——使重新同步和比对无需额外标志即可找到参考 |
   | `storybookConfigDir` | `.storybook/` 目录（适用于 monorepo） |
   | `buildCmd` | 重新同步前需要在转换器运行前重新执行的命令 |
   | `titleMap` | 当故事标题与导出名不一致时使用 `{title: ExportName}`；若某个非视觉或内部组件完全不需要参与同步，则使用 `{title: null}` |
   | `overrides` | `{<Name>: {skip: [storyIds], cardMode: "single"|"column", primaryStory: "<Export>", viewport: "WxH"}}`——`skip` 用于无法静态渲染的故事；`cardMode: "single"` 用于叠加组件（§4a.5、§5），`"column"` 用于宽度超过网格单元的故事（参见 §3 中的 `[GRID_OVERFLOW]` 行） |
   | `provider` | 对于 **预览** 通常无需设置——`.storybook/preview` 中的装饰器会自动打包；仅在自动打包失败时才需手动设置。在进行 §6 上传之前，应将装饰器提供的上下文提炼至 `cfg.provider` 中——README/prompt.md 中的封装说明仅基于配置生成（仅使用装饰器的封装会附带一条通用提示）。**设置 `provider` 还会在下一次构建时替换装饰器作为预览的封装层**：切换后对比带有主题的组件时要特别注意——若提炼不完整，会导致原本由装饰器正常渲染的预览出现回归，而后续的评分机制可能无法发现此类问题。格式为：`{"component": "ThemeProvider", "props": {...}, "inner": {...}}`——嵌套链式结构，最外层优先；每个 `component` 必须是打包导出的内容。字面量 `props` 适用于小型标量值（如 `"theme": "light"`）和稳定的片段。对于仓库中已存在的数据——例如 locale JSON 或主题对象——**优先使用 `{"$ref": "<export>"}`**，并通过 `cfg.extraEntries` 添加一个两行模块来支持（例如 `export { default as previewI18n } from '../locales/en.json'`）：`$ref` 会注入 `window.<Global>.<export>`，因此数据仅在打包时存储一次，并在每次构建时从源文件重新读取。对于非常小且稳定的值，内联复制也是可以接受的，但需了解其代价——字面量会复制到每张卡片的 HTML 中，且当源文件变更时会悄然失效，因此任何较大或动态的数据都应使用 `$ref` 引用。`extraEntries` 的路径形式：裸名称从 `node_modules` 解析；属于本仓库的模块则需要显式的 `./` 或 `../` 相对路径（受工作区限制——若超出范围，构建日志会显示 `! extraEntries: ... skipped`）。|

4. **准备脚本并安装转换器依赖**（隔离在 `.ds-sync/` 中，不影响仓库的锁文件）：
```bash
mkdir -p .ds-sync && cp -r "<skill-base-dir>"/package-build.mjs "<skill-base-dir>"/package-validate.mjs "<skill-base-dir>"/resync.mjs "<skill-base-dir>"/lib "<skill-base-dir>"/storybook "<skill-base-dir>"/non-storybook .ds-sync/
echo '{"name":"ds-sync-deps","private":true}' > .ds-sync/package.json
(cd .ds-sync && npm i esbuild ts-morph @types/react playwright && npx playwright install chromium)
```

如果安装 Chromium 失败，先运行 `npx playwright install-deps chromium`；如果环境无法安装 Chromium，则设置 `DS_CHROMIUM_PATH=<system-chromium>`。
5. **运行转换器、验证器并进行对比**——同步执行，遇到第一个非零退出码即停止（只有在构建和验证都通过后才会运行一次对比——见第3节）。对于大型 DS（约100个以上组件），构建时可能需要使用 `NODE_OPTIONS=--max-old-space-size=<MB>`；**切勿将构建过程通过 `head`/`tail` 管道**（管道会掩盖退出码——内存溢出看起来像是成功）；应将其重定向到文件并读取：

```bash
node .ds-sync/package-build.mjs --config .design-sync/config.json --node-modules <pkg-node-modules> \
  --entry <built-dist-entry> --out ./ds-bundle
node .ds-sync/package-validate.mjs ./ds-bundle
node .ds-sync/storybook/compare.mjs --out ./ds-bundle --storybook-static .design-sync/sb-reference \
  --components <solo-phase picks>   # 将第一次对比的范围限定为第4b节中的单独组件
```

在 monorepo 中，`--node-modules` 应指向 DS 包自身的 `node_modules` 目录——除非通过 hoisting 机制该目录为空（例如，Yarn 的 `node-modules` 链接器仅在仓库根目录保留 `react`）。如果内部缺少 `react/` 或 `react-dom/`，则应传入仓库根目录的 `node_modules`。在 DS 自己的源码仓库中，`node_modules/<pkg>` 并不存在，因此需要指定 `--entry`。构建过程中会自动检测并记录 `[ICON_PKG]` 和 `[TOKENS_PKG]`，同时将 `.storybook/preview` 中的装饰器打包为预览包装器（`preview-decorators.js`），以确保预览与故事拥有相同的 Provider 链。

首次对比的范围应加以限制：对大型 DS 进行完整截图会触发数千次 Chromium 导航，在单独阶段未解决全局问题之前这样做毫无意义（每次修复全局问题都会使所有截图失效）。第一次全量组件对比应在第4b节第3步时进行；而对于超过20个有故事的组件的 DS，即使如此也应按第4c节的要求分批次进行，因此唯一必须执行的全量对比是第4d节的验收对比，它会将已完成的工作带入下一阶段，而不是重新捕获。对于拥有超过100个有故事组件的 DS，还应在展开对比前告知用户预期的规模（组件数 × 故事数），并允许用户根据需要缩小范围。
  
## 3. 自愈循环（构建 + 验证）

修复 `[TAG]` 错误 -> 重新构建 -> 重新验证，直到两者都返回退出码 0，**然后再开始第4节中的对比循环**——在包本身存在问题时，对预览进行像素级比对毫无意义。共享转换器标签（`[NO_DIST]`、`[WORKSPACE_SIBLING]`、`[CSS_*]`、`[FONT_*]`、`[TOKENS_MISSING]`、`[DTS_*]`、`[RENDER*]` 等）的行为与包结构保持一致，请参考 `../non-storybook/SKILL.md` 第3节中的表格。错误下方以 `hypothesis:` 标注的行是线索而非指令：应先执行其验证步骤，若未能确认，则放弃该假设，并直接根据错误信息进行诊断。Storybook 特有的：
| 标签 | 症状 | 修复方法 |
|---|---|---|
| `[SB_REFERENCE_MISSING]` | 比较工具找不到 `iframe.html` | 构建参考项目（§2.2）；设置 `cfg.storybookStatic`。 |
| `[SB_BUILD_FAIL]` | 转换器自身的 Storybook 构建失败 | 您跳过了 §2.2——请自行构建参考项目，并设置 `cfg.storybookStatic`，这样转换器就无需再执行此操作。 |
| `[ZERO_MATCH]`（Storybook 风格） | 没有匹配的 Story 条目 | 检查 Storybook 配置中的 `stories` glob；然后检查 `titleMap`。 |
| `[TITLE_UNMAPPED]` | N 个标题未与任何导出项匹配 | 使用 `cfg.titleMap {<title-name>: <export-name>}` 进行映射。 |
| `(预览：`<Name>`...没有配对的 Story 导出...)` | 无法将索引 Story 的名称与模块导出键匹配（配对时会先尝试显示名称，再尝试 Story ID 的尾部） | 组件会显示“地板卡片”；请修复配对问题——通常是因为某个 `.tsx` 文件重新导出了 Story，且使用了可匹配的名称。 |
| 预览单元格报错，提示“未定义的组件”或“上下文错误” | Story 导入解析方式错误——相对路径、tsconfig 别名和裸模块导入均遵循同一策略（参见 `lib/story-imports.mjs` 中的规则） | 使用 `cfg.storyImports.shim` 或 `cfg.storyImports.bundle` 中的子字符串模式，按解析后的路径强制指定解析方式——这是在修改核心逻辑之前的临时解决方案。 |
| `! 预览构建失败：<Name>` | Story 模块未能编译（顶层 await、esbuild 无法解析的包导入、缺少加载器的资源扩展名） | 查看错误信息上方的 esbuild 错误详情。未知资源扩展名——在 `cfg.storyImports.loaders` 中进行配置（覆盖默认值，例如 `{".yaml": "text"}`）；无法解析的导入——接管该 `.tsx` 文件并将其移除。修复前，组件会显示“地板卡片”。 |
| Story 的本地样式表在其单元格中缺失 | Story 本地的 `.css`/`.scss` 副作用导入会被编译为空（组件样式通过 bundle css 提供）。例外：`.module.css` 会被编译——类名会被解析，并自动链接 `_preview/<Name>.css` | 通常无需处理——这些样式只是 Storybook 页面添加的装饰。如果 Story 确实依赖它们，请将样式内联到自有的 `.tsx` 文件中。 |
| `[BUNDLE_EXPORT]` | 组件未作为函数挂载到 `window.<Global>` 上 | 对于子路径或图标导出，使用 `extraEntries`；确保 dist 入口是完整构建。 |
| `[SCHEDULER_MISSING]` | dist 中引入了 `scheduler` | react-dom 泄漏到了 DS 的 dist 中——检查其构建的 externals 配置。 |
| `! 预览装饰器打包失败` | 装饰器无法被打包 | 手动设置 `cfg.provider`，或运行 `node .ds-sync/storybook/probe.mjs --storybook-static .design-sync/sb-reference`，从实时 Storybook 中推断依赖链（将每个 `$hint` 替换为实际值）。 |
| 预览在加载 `_vendor/preview-decorators.js` 时出错（Storybook API 报“未定义”错误） | `.storybook/preview` 的导入图触达了一个 stub 未覆盖的 Storybook 运行时模块 | `manager-api` 和 `preview-api` 被替换成功能空操作的钩子，其他所有 `@storybook/*` 和 `msw` 模块则被替换成惰性可调用对象（如 `fn()`、`action()`、`setupWorker()` 在模块作用域中都会无害地求值）；如果仍有其他 API 报错，请显式设置 `cfg.provider`——它会完全跳过装饰器的打包。 |
| 比较工具中的 `[ASSETS_BLOCKED]` | 截图浏览器继承了一个网络沙盒环境——Story 资源（CDN 图片/字体）在**两个**面板上均加载失败，因此评分可能会误判为通过，而最终用户看到的内容却不同 | 请在具有对外访问权限的 shell 中重新运行 `package-validate.mjs` + `compare.mjs --force`：在提示时批准在不启用沙盒的情况下运行命令，或将相关主机加入沙盒白名单。在此期间，请勿对包含图片的组件进行评分。**增量路径（基础 SKILL.md §3）——这是打开通道的入口。** 第一次构建并验证时，两个出口都为“关闭”，在开始 §4 之前先打开上传通道：用户在此处确认一次，然后随着分级流程的推进，观察各组件逐步到位。在第一批经过分级的批次完成之前，不会有任何内容上传——共享的基础文件会随该批次一同推送——而后续的批次推送则来自 §4b/§4c。（原子路径：直到 §6 才会有任何内容上传。）

## 4. 将预览与 Storybook 对照

`compare.mjs` 是一个**捕获框架——它负责拍照，你负责分级**。它不会计算任何相似性指标（像素、文本或字体的评分在构图存在合理差异时都会产生误导）；判断完全基于两张真实的截图。编译后的预览是按**每个故事**单独捕获的——每个故事都会通过 `?story=<Export>` 在完整的捕获视口中单独渲染，其方式与 Storybook 呈现参考侧时完全一致，因此同级故事之间不会相互干扰（如 Portal 堆叠、共享的单选组名称、焦点状态、容器尺寸等）。输出分为两层：
- **临时层**（位于 `ds-bundle/` 目录下，重建时会被清除）：`_screenshots/compare/<group>__<Name>.png`——每张表包含按故事排列的一行：**真实 Storybook 渲染 | 真实预览渲染**，并排放置。表格中的图片会被缩放以适应显示；完整分辨率的原始图存放在 `.../compare/raw/` 目录下（`...__sb.png` 和 `...__ds.png`）——当缩略图太小而难以准确判断时，请查看这些原图。
- **活动状态层**（位于 `.design-sync/.cache/compare/` 目录下，已加入 .gitignore）：`<Name>.grade.json`——你的评级结果；以及 `<Name>.json`——记录捕获事实：故事与单元格的对应关系、截图路径、`previewKind`、组件的 `srcSha`（即故事文件的指纹）、抽查锚点等信息。这些数据均可重新生成——如果缺失，只需再次捕获即可。脚本仅输出事实性信息：`sb-error`（故事在 Storybook 中无法渲染）、`unpaired`（该故事没有对应的预览单元格）、`error`（单元格抛出错误）；所有成功渲染的配对均标记为 `needs-grade`。

默认情况下，每次比较最多支持 6 个故事——日志中会用 `[STORY_CAP]` 标注超出此限制的组件；使用 `--max-stories <n>` 可提高上限。请注意，此上限并不属于评级契约的一部分：提高上限只是为了在增量分级时捕获尾部故事，而已有评级结果仍会保留。需要了解的一个后果是：对于那些已完整评为 `match` 或 `close` 的组件，即使其尾部故事从未单独分级，在未来的同步中也会因上传而被整体验证——如果这些尾部故事包含值得单独验证的不同变体，请提高上限。分发子代理在一轮处理过程中不得更改此设置（否则生成的表格所涵盖的故事集将与总控的工作列表预期不符）。

**跨运行的状态**——首次运行会一次性验证所有内容；此后遵循一条规则：**等级跟随你的源文件**——即故事文件、你拥有的预览、故事集、影响预览的配置（`provider`/`storyImports`/`extraEntries`/`overrides`/`titleMap`），以及已提交的 `.design-sync/overrides/` 分支。流水线变动（如技能或工具链更新导致所有内容重新渲染）会通过采样的 `[SPOT_CHECK]` 自动验证，且等级得以保留；你的编辑仅对其所触及的部分重新评级。像素抖动绝不会触发等级变动。
- *源文件未变* 且已完全评级为 `match`/`close` -> **直接跳过**（“沿用”）：不重新捕获，也不重新评级——即使包、样式、Storybook 或转换器本身被重建。`--force` 会重新捕获所有内容 **并清空所有等级**——这是系统性的重新验证，而非随意的面板再生。
- *源文件已变*（故事编辑、`.tsx` 编辑、配置/分支编辑）-> 重新捕获，清空等级，并从新生成的面板重新评级。`[STORY_CHANGED]` 标记那些代码发生移动的故事——这些情况下，你拥有的 `.tsx` **必须更新**（生成的预览会自动重新推导）；如果没有 `[STORY_CHANGED]` 的重新捕获，通常只需重新评级即可。
- *[SPOT_CHECK]* -> 重新捕获指定组件 **但不清空其等级**；阅读新生成的面板，确认其仍与记录的等级一致。它可能在流水线变动后由驱动程序触发——这是对技能/工具链更新的常规验证，而非错误。差异修复的范围随变动规模而定：若仅少数组件受影响，则仅对这些组件重新评级；若影响广泛，则应先停止、诊断，再使用 `--force` 进行全量检查。`--spot-check N` 可调整全量运行时的随机抽样比例（0 表示禁用）；`--spot-check-components A,B` 则可显式指定要检查的组件，该选项在限定范围的运行中同样适用（参见第7步的审计环节）。
- *[REFERENCE_STALE?]* -> 包发生了变化，但参考 Storybook 并未更新。若 DS 源发生变化，请在评级前重建 `.design-sync/sb-reference`——过时的参考会导致所有等级都与*旧*设计进行比较。
- *每次捕获时故事呈现不同*（包含 `new Date()` 或 `Math.random()` 的内容）-> 故事的指纹是其文件，因此契约是稳定的——但像素并不稳定，而评级正是基于像素来判断的。冻结捕获时间可以稳定日期的呈现；对于真正随机的内容，可在你拥有的 `.tsx` 中固定相关值，或在 `cfg.overrides.<Name>.skip` 中通过 NOTES.md 注释跳过该故事。

捕获过程会进行稳定性处理，以确保评级的可比性（动画快进、减少动态效果、冻结时间——两侧面板显示的是同一稳定帧、同一渲染日期）。此处理仅用于验证目的：最终交付的预览不受影响，保持完整动画效果。
  
**评级由负责相应组件的人员执行**——单人阶段由你完成，分叉协作时则由各子代理分别负责自己的组件。每次对比运行结束后：阅读评级表（如有疑问，同时查看原始 PNG 文件），仅凭图像对每个故事作出判断，并将结果写入 `.design-sync/.cache/compare/<Name>.grade.json`（这是项目本地的工作状态——使评级持久化的关键在于上传：上传后的 `_ds_sync.json` 会在每次后续同步时锚定已上传的验证结果，无论在哪台机器上同步）：

```json
{"stories": {"Default": {"verdict": "match"}, "Compact": {"verdict": "match", "basis": "sibling-trusted"}}}
{"stories": {"Loading": {"verdict": "mismatch", "note": "缺少加载动画——该故事使用了 MSW 模拟数据"}}}
```

（两个组件的文件：一个按照下文的采样规则进行了干净评级——`Default` 是根据图像判定的主要故事，且在无警告的情况下评为 `match`，这使得“兄弟信任”的条目生效；另一个则存在不匹配，其备注将指导下一步的修复。）评分标准——针对设计师关注的方面，对比两份渲染结果进行评分：
- `match`：内容、布局与样式完全一致。可忽略抗锯齿带来的模糊、滚动条细缝、小于5像素的偏移以及框架差异（Storybook 画布与预览页面的框架不同——应以组件本身为准，而非其周边环境）。
- `close`：可识别为同一渲染，但存在细微差异（如内边距略有不同、焦点环、占位符文本等）。**`close` 仍属待修复项，而非通过标准**：若能明确指出差异点，通常也能定位到对应的配置项——继续迭代。仅当某次迭代未能改善，且已无可行的修复方向时，方可接受 `close` 等级；此时备注须同时说明 *哪些部分不一致* 以及 *尝试过哪些方法 / 为何无法修复*（例如：“焦点环颜色不同——Storybook 应用了一个全局焦点插件，不属于设计系统的一部分”）。
- `mismatch`：内容错误或缺失、未应用样式、变体错误、图标/图片缺失、使用了默认字体。备注须明确指出 *具体哪些部分不同*——这将指导后续修复。

当 REFERENCE 一侧为产物时——即 Storybook 在 UI 饰面（如主题或控件切换提示）之后才渲染故事，而预览则直接呈现真实组件——应仅就组件本身的渲染效果进行评判，并注明该遮罩情况；若预览呈现的内容 *多于* 被遮罩的参考，则不应评为 `close`。

**只评主故事，其余可信。** 同一组件的兄弟故事共用同一流程——相同的导入、相同的 Provider 链、相同的 CSS——因此，只要其中一个故事渲染正确，其余几乎也都会正确。首次比对时，仅根据图片评判组件的 **主故事**（若设置了 `cfg.overrides.<Name>.primaryStory`，则以此故事为准；否则以 Sheet 的第一个故事为准）。若主故事评分为 `match`，且组件状态良好——无 `sb-error`/`unpaired`/`error` 单元格，无 `[PORTAL?]`、无 `[RENDER_BLANK]`，无空白或尺寸异常的截图——则可为其余故事标注基于主故事的 `match`，并附上依据标记，如 `{"verdict": "match", "basis": "sibling-trusted"}`，以便记录每项判定的依据（比对工具仅读取 `verdict` 字段）。一个组件的所有判定——经图像比对的主故事，加上所有基于兄弟故事可信的判定——均汇总在其唯一的 `grade.json` 中。注意：基于兄弟故事可信的判定无需额外打开图片，也不需逐个故事确认。当组件涉及 Portal/Overlay、对主题或 Provider 敏感、拥有独立预览，或出现任何警告时，应逐个故事彻底比对；对于 §4b 的单独测试集，更需全面比对，唯有如此才能建立信任基础。

无论何种情况，每个故事都应拍摄照片——抽样是为了节省比对精力，而非拍摄时间，且这些截图可供日后随时查阅（§7 第4步的 carried-grade 审核会沿用同样的抽查路径）。这与 `[STORY_CAP]` 中未评级的尾部故事属于同一类“可信”范畴，只是有意识地加以应用。抽样检查绝不能放松对 `[FONT_MISSING]` 的核查（§4a）——因为这一问题在比对图片中始终不可见。

### 4a. 修复决策树——全局优先

自顶向下推进；全局性修复可一次性解决所有组件的问题，而单组件修复则仅针对某一组件。1. **大多数/所有组件都以相同方式出错** -> 全局性问题，需在配置中修复并进行完整重建：
   - 单元格中的上下文/提供者错误（`use<X> 必须位于 <Provider> 内部`）-> 装饰器未被打包（见第3节 `! 预览装饰器打包失败` 行）-> 检查 `cfg.provider`。
   - 所有内容无样式或使用默认字体 -> 检查 `cfg.cssEntry`（查看构建日志中的 `[CSS_FROM_STORYBOOK]`）、`cfg.tokensPkg`、`cfg.extraFonts`。
   - **`[FONT_MISSING]` - 对比循环无法识别此类问题。** 当双方均未提供该字体时，两个面板都会渲染相同的 Chromium 回退字体，因此视觉上看似“匹配”，但所有 claude.ai/design 用户看到的却是错误的字体——切勿将“双方回退到同一字体”视为通过。请按照 `../non-storybook/SKILL.md` 第3节中的 `[FONT_MISSING]` 行进行处理；针对 Storybook 的额外注意事项：`cfg.extraFonts` 的路径受限于包含 `dirname(--node-modules)` 的 Git 仓库——同属 monorepo 的排版包可直接使用；只有当没有 `.git` 父级时，路径范围才会缩窄至 `dirname(--node-modules)`。若添加了引用缺失的字体，请将相同的 `@font-face` 注入到 `.design-sync/sb-reference/iframe.html` 中，以便验证工具在两端均使用真实字体进行校验。
   - 各处图标缺失 -> 检查 `cfg.extraEntries`（查看 `[ICON_PKG]`）。
2. **某个组件显示为 `unpaired` 或 `fallback preview`** -> 该组件的 `.tsx` 文件缺少对应故事的单元格。预览会完整编译故事模块（包括钩子、固定装置及局部辅助函数——闭包并非导致失败的原因），因此可能的原因有：配对失败（`storyName` 被覆盖）、包装器构建失败（构建日志中显示 `! preview build failed`），或模块在加载时抛出异常——请查看该页面的 `(page)` 错误行以定位实际异常（可能是模块作用域内调用了桩代码未覆盖的包）。打开包装器文件（生成的：`.design-sync/.cache/previews/<Name>.tsx`；自有的：`.design-sync/previews/<Name>.tsx`），添加或重命名导出，或移除引发问题的导入——若为生成文件，请将修复保存为 `.design-sync/previews/<Name>.tsx`，且不要保留首行标记（原地修改的缓存仅在此机器上保留，但会被 Git 忽略——克隆后即消失，且后续重新编译时不会再次评级；只有自有的副本才会更新评级契约，而重建时会提示存在已编辑的缓存副本）。故事导入采用与位置无关的 `@ds-stories/<repo-relative path>` 格式，因此无论从哪个目录运行，文件均可正常工作。
3. **某个组件被判定为 `mismatch`** -> 属性或组件结构有误。请阅读故事源码，并将其镜像复制到自有的 `.design-sync/previews/<Name>.tsx` 文件中（从缓存包装器中复制过来，但去掉首行标记）。这是针对已编译故事预览的唯一调整手段。
4. **`sb-error`** -> 该故事在 Storybook 中也无法渲染（如数据获取或交互相关问题）。将其 ID 添加至 `cfg.overrides.<Name>.skip`，并在 NOTES.md 中注明原因。
5. **`[PORTAL?]` / 叠加层组件**（Dialog/Tooltip/Toast）-> 评级已按故事隔离捕获，但产品卡片会渲染整个网格 HTML，因此打开叠加层的故事也会在其中覆盖相邻单元格。设置 `cfg.overrides.<Name>.cardMode: "single"`——卡片将以全屏模式渲染单个故事（由 `primaryStory` 指定；否则使用首个导出），并使用一个包含 `position:fixed` 子元素的包装器，同时在卡片上声明评级视口，使产品以您验证过的尺寸呈现。对于那些宽度超出网格单元格的故事（如数据表格、全宽条形图——这些会被标记为 `[GRID_OVERFLOW] ... wide`），则使用 `cardMode: "column"`：每个故事保持卡片的完整宽度，不会被裁剪。对该组件执行定向重建（`preview-rebuild.mjs --components <Name>`，耗时数秒）——**评级参数会沿用**（`cardMode` 和 `primaryStory` 不在评级键或打标后的配置片段中）；只有更改视口才会触发重新评级（因为视口是捕获基准），此时需要完整构建（以移动配置片段）。**重建规则——仅重建受变更影响的部分。** 样式变更（CSS/字体/变量）会重新渲染所有预览，但不会改变任何评分契约——评分会沿用。Provider、`storyImports`、`extraEntries` 以及分支编辑都属于评分契约的一部分（它们会改变预览挂载的内容）——受影响的评分会被清空，并在重建时重新评分。

| 您更改了 | 重建 | 对比 |
|---|---|---|
| 仅更改了一个预览 `.tsx` 文件 | 下方的目标循环（几秒） | 作用域为 `--components <Name>`——其评分被清空并重新评分 |
| `overrides`（`skip`/`viewport`）/ `titleMap` | 执行完整的 `package-build.mjs` + `package-validate.mjs`（重新打上配置键，用于目标重建的检查） | 执行完整的 `compare.mjs`——被触及的组件会重新评分；已匹配或接近匹配的组件直接跳过，而尚未处理的组件则会获取新的评估表（完整构建已清空这些表——下一轮会读取这些表） |
| 仅更改 `overrides` 中的 `cardMode`/`primaryStory` | **目标循环**（`preview-rebuild.mjs --components <Name>`，几秒）——展示相关键不在已打标的配置片段中，因此不会触发 `[CONFIG_STALE]`；循环会重新输出卡片 HTML 并更新其 renderHash | **不重新评分**：仅涉及展示的键不属于评分契约——评分沿用；更改后的卡片 HTML 会重新下发，后续的同步可能会对其进行抽查 |
| `provider` / `storyImports` / `.design-sync/overrides/` 分支 | 执行完整构建 + 验证 | 执行完整的 `compare.mjs`——受影响的评分按上述规则重新评分 |
| CSS / 字体 / 变量 | 执行 `package-build.mjs --skip-dts` + 验证 | 执行完整的 `compare.mjs`——成本较低：已匹配或接近匹配的组件直接跳过，只有待处理的组件才会根据新样式重新评估。评分沿用——并非完全无操作：变更的字节仍会重新下发，后续同步可能会将其作为 `verification.canary` 抽检出来 |
| `entry` / `extraEntries` | 执行完整构建 + 验证——绝不使用 `--skip-dts`（它们会改变打包和导出接口） | 执行完整的 `compare.mjs`——受影响的评分重新评分 |

战役中期——§4c 的批次仍在等待处理——此时该表中的“执行完整的 `compare.mjs`”应理解为*最终通过批次逐步完成*：重建无论如何都会清空受影响的评分，下一轮的局部运行会重新评估这些组件，而 §4d 的收尾则是全名单的最终结算（§4c 跨批次步骤 2）。只有在没有未处理批次时，才需要立即进行全名单对比。

`--skip-dts` 会跳过每组件的类型提取——这是大型 DS 构建中最耗时的环节——并输出占位的 `.d.ts` 内容，因此其验证阶段会因设计原因报错 `[DTS_STUBBED]`（渲染检查仍然会回答“修复是否生效？”）；§4d/§6 的验证必须以退出码 0 结束，这要求最终构建必须在不使用该选项的情况下进行。请注意，占位构建生成的卡片和 README 简介可能会显得简略——最终构建会将其恢复。`--skip-dts` 仅适用于修复迭代流程：任何会被上传读取的构建——无论是增量批次推送（基础 SKILL.md §3），还是 §6 的收尾——都必须是完整构建，因此如果 `.ds-build-meta.json` 中仍标记有 `dtsStubbed`，在推送前请移除该选项并重新构建（批次推送会上传磁盘上的 `.d.ts` 文件）。

**将多个配置修改合并到一个周期内。** 在触发重建之前，请先梳理所有待处理的评估结果及已知问题，涵盖它们所涉及的所有配置修改（`skip`、`titleMap` 条目、`cardMode` 等），并将这些修改一并应用——即使两处修改相隔仅数分钟，也不应为此付出两次重建+验证+对比的代价。

**对比运行中途失败**（浏览器崩溃、内存溢出）：它所捕获的评估结果仍然有效——请先对这些结果进行评分，然后再重新运行；评分会延续到中断的部分。切勿使用 `--force` 重启已崩溃的运行（这会清空您刚刚获得的评分）。**在大型 DS 环境中，请务必在执行完整重建之前验证修复是否正确**：先在一个受影响的组件上运行下方的定向循环（或预览其渲染后的页面）——如果判断失误并进行了完整重建，将浪费整个构建周期。**中间验证可采用抽样方式**：全局性问题本质上具有系统性，因此使用 `--render-sample 10` 即可以较低的成本快速确认“修复是否生效”；但只要涉及任何可能影响渲染的内容发生变化，在 §4d/§6 的上传阶段仍需执行完整的渲染检查——在锚定式重新同步时，§7 驱动程序会自动应用该规则（相关层级规则也在此处定义）。

仅针对 `.tsx` 文件的定向循环：
  ```bash
  node .ds-sync/lib/preview-rebuild.mjs --config .design-sync/config.json --node-modules <nm> --out ./ds-bundle --components <Name>
  node .ds-sync/storybook/compare.mjs --out ./ds-bundle --storybook-static .design-sync/sb-reference --components <Name>
  ```

该定向循环仅重新编译预览，不会从源码重新计算合约等级：如果仅对故事文件进行编辑并只运行此循环，旧的等级将持续存在，直到下一次完整构建或驱动程序运行时才会重新计算等级——因此，请通过完整构建来处理故事文件的修改（驱动程序会自动完成这一过程）。

### 4b. 单独阶段——先一个，再几个

切勿立即扩展到多个子代理。全局性问题必须先沉淀到配置中，否则每个子代理都会再次发现这些问题。1. **单个组件。** 选择一个结构简单、故事完整的组件（类似 Button：多个故事，无门户）。运行 §4a 循环，直到根据其截图对每个故事的匹配度都打分完毕——只有在某次迭代无法进一步提升时才接受“接近”（参照上文评分标准）。**每次修复都要记录在 `.design-sync/NOTES.md` 中**：症状 → 根因 → 修复方案；若非特定于某个组件，则标注为 `[GENERAL]`。
2. **再选三个具有多样性的组件：** 一个复合或叠加型组件（如 Dialog/Tabs），一个图标或资源密集型组件——其故事会加载远程图片（这是 `[ASSETS_BLOCKED]` 的预警信号——参见 §3 的说明：在网络沙盒环境下，外壳会在两个面板上均屏蔽资源，导致评分误判；若在此阶段未发现，后续全量检查时则需重新捕获整个组件）；以及一个主题或 Provider 敏感型组件。同时确保这组组件中包含一个**文本密集型**组件（仅针对按钮的测试容易遗漏字体/排版问题，进而导致整轮评分作废）。同样采用单独循环进行检查。*渐进路径：* 当这组组件的所有故事都达到“匹配”（或按评分标准达到“接近”）时，即为首个验证通过的批次——将其推送至基础 SKILL.md 第三部分。
3. **首次全量捕获——按有故事组件的数量设限。**
   - **20 个及以下：** 对整个组件列表运行一次完整的 `compare.mjs`。通过 Shell 工具的后台模式启动，并等待完成通知——§2.2 的规则在此重申，因为此处最容易被违反：前台的 `sleep` 轮询会阻塞唤醒你的通知，而 `pgrep -f` 循环又会匹配自身命令行并持续运行直至超时。（无头 / `-p` 会话：改用同步方式执行——无头模式下不会触发任务通知的重新调用，因此后台运行的任务永远不会被恢复。）如果至少 30% 的组件因**同一原因**失败，则说明存在你遗漏的全局性问题——先在配置中修复，再重新运行，然后展开检查。**在重建之前，将清单中显示的所有跳过和配对修复逐一处理**——每次重建加比较都会耗费数分钟；逐项修复则只需付出每项的成本。
   - **超过 20 个：不要进行一次性全量捕获。** 捕获应在 §4c 的批次中进行——每个子代理运行一次限定范围的 `compare.mjs --components <its batch>`，并对刚捕获的截图进行评分。这样做有三点好处：限定范围的捕获可并发执行（整个组件列表的渲染时间远小于串行扫描的总耗时）；评分可在第一个批次的截图生成后立即开始，而非等到最后一个组件渲染完毕；当某一轮检查发现 `[GENERAL]` 类问题时，受影响的只是已评分的少数几个批次，而不是整个列表的捕获与评分。关于 ≥30% 组件因同一原因失败的检查也随捕获流程推进——它将成为第一轮的学习总结（§4c 各轮之间的环节）。而你**不会跳过的**那一次全量运行，就是 §4d 的验收：此时所有组件均已评分，因此只需将它们向前传递，无需重新捕获，耗时以秒计，而非分钟。

### 4c. 扩展——并行子代理
将仍需改进的组件按 5–8 个一组划分成若干批次——在大型设计系统（§4b 第三步的 >20 门槛）中，这些是除独立试验集之外的所有组件，其中大部分尚未捕获任何截图；而在小型设计系统的全量捕获之后，这些则是未匹配的组件集合。将相关组件归为一组（共用 Provider、共用 Fixture——一次诊断即可覆盖整组）。每轮最多启动 4 个子代理（通过 Agent 工具，一条消息即可让它们并发运行）。4 也是浏览器并发数的上限：每个子代理的限定范围比较都会启动自己的 Chromium 实例，超过约 4 个并发捕获可能会因机器层面的竞争而导致启动失败。对于每个子代理，请填入此提示中的所有 `{...}`，并将**当前**的 NOTES.md 内容粘贴进去（子代理会通过该文件继承独立试验阶段的学习成果）：

```text
修复 design-sync 预览，使其与仓库自身的 Storybook 渲染一致。
仓库：{REPO_ROOT}。您的组件（仅限您负责的部分）：{COMPONENT_LIST}。

为何重要：该设计系统正被同步到 claude.ai/design，在那里，一个设计代理将基于这份完全相同的编译产物构建真实 UI。Storybook 渲染是每个组件应有的外观证明；预览与其一致，说明组件完好无损地到达；不一致则意味着代理使用该组件构建的每一个设计都会出现同样的错误。

每个组件的相关产物（请优先阅读）：
- {OUT}/_screenshots/compare/<group>__<Name>.png —— 真实 Storybook 渲染（左）与真实预览渲染（右），按每个故事分别展示。完整分辨率原图位于 {OUT}/_screenshots/compare/raw/。
- .design-sync/.cache/compare/<Name>.json —— 配对信息 + 截图路径（不含相似度分数——请凭肉眼判断）。
- 预览源码（实际 JSX，从 '{PKG}' 导入）：自有组件为 .design-sync/previews/<Name>.tsx，否则为生成的 .design-sync/.cache/previews/<Name>.tsx。您的修复将写入 .design-sync/previews/<Name>.tsx（步骤 2）。
- {OUT}/.stories-map.json —— 将组件映射到故事 ID；可通过 .design-sync/sb-reference/index.json 中的 ID 查找每个故事的源文件（`importPath`）。故事源文件是预期 Props/组合的权威依据。
- .ds-sync/storybook/SKILL.md 第四部分——评分标准与修复决策树。

第一步，全批统一操作：若您的组件中尚有未生成对比截图的，请运行
  node .ds-sync/storybook/compare.mjs --out {OUT} --storybook-static {SB_REF} --components {COMPONENT_LIST}
一次限定范围的运行即可捕获该批次中所有缺失的截图（仅需启动一次浏览器，而非每个组件单独运行）；源码未变且已评分的组件会自动跳过。

针对每个组件（最多 3 次迭代）：
1. 阅读截图；根据两张图片（当截图过小时使用原始 PNG）按照 §4 抽样规则判断主故事的表现；若组件包含门户、主题/Provider 敏感性、自有预览或任何警告标志，则需全面分析；并通过决策树诊断问题。
2. 将 .design-sync/.cache/previews/<Name>.tsx 复制到 .design-sync/previews/<Name>.tsx，并删除首行的 `// @ds-preview generated ...` 标记（自有文件存于 previews/ 目录，优先于生成副本，且持久化并提交版本控制；就地修改缓存可在本机重建时保留，但会被 Git 忽略并在全新克隆时消失）。`@ds-stories/...` 导入语句在新位置保持不变。复制故事的 JSX，并内联故事本地的 Fixture 数据。
3. 运行 node .ds-sync/lib/preview-rebuild.mjs --config .design-sync/config.json --node-modules {NM} --out {OUT} --components <Name>
4. 再次运行 node .ds-sync/storybook/compare.mjs --out {OUT} --storybook-static {SB_REF} --components <Name>（您的修改改变了组件的契约，因此本次运行会清除其旧评分——这是预期行为）。
5. 重新阅读最新截图，并将您的判定结果写入 .design-sync/.cache/compare/<Name>.grade.json（{"stories": {"<story>": {"verdict": "match|close|mismatch", "note": "..."}}）。若其他组件符合 §4 抽样规则且值得信赖，则可直接标注 {"verdict": "match", "basis": "sibling-trusted"}——在同一份 grade.json 中注明，无需再次打开截图查看。当所有故事均被评为匹配时即告完成。被评为“接近”的故事仍需修复——若能明确差异所在，可尝试调整相关参数；仅当某次迭代未能改善，或无可行的修复方向时才接受“接近”，且备注中必须说明偏差的具体内容及已尝试的解决方法。若三次迭代后仍未突破，则如实评定为“不匹配”或“接近”，并记录确切的阻碍因素，继续下一个组件。
硬性规则——违反这些规则会破坏其他代理的工作：
- 只编辑 .design-sync/previews/{<你的组件>}.tsx、你组件的 .design-sync/.cache/compare/*.grade.json 文件，以及 .design-sync/learnings/{BATCH_ID}.md。
- 绝不编辑 .design-sync/config.json、.design-sync/NOTES.md、.ds-sync/，或任何其他组件的文件。
- 绝不运行 package-build.mjs 或 package-validate.mjs——它们会重写共享包。通过 --components 限定范围的 preview-rebuild.mjs 和 compare.mjs 是你唯一可用的构建命令。
- 绝不对本轮未读取的图片打“图像判定”等级。只有兄弟信任的裁决才可标注 “basis”: “sibling-trusted”，且仅当该图片的主要故事等级匹配、组件无警告（§4 抽样规则）时才允许。
- 在 Storybook 中无法渲染的故事（sb-error）需在配置中添加 cfg.overrides.<Name>.skip；同理，[PORTAL?] 需将 cfg.overrides.<Name>.cardMode 设置为 “single”。这两项配置修改你不得自行进行——应在学习文件和最终报告中记录，由协调器统一应用。绝不可通过在 .tsx 中强制关闭故事的打开状态来“修复”遮罩溢出问题——这会破坏待验证的真实性。
- 如果同一根本原因出现在你负责的两个及以上组件中，或即便只出现一次但根源是配置层面（provider/css/font/token/import 解析），则立即停止处理这些组件：这是全局性问题。将其记入学习文件的 “[GENERAL]” 部分，上报，切勿逐个组件绕过。针对全局性问题的局部修复不仅无益，反而有害：由于 .design-sync/previews/ 中的内容不会被机器自动删除，你为此留下的预览将持续存在，并在每次后续构建中遮挡正确的生成预览。

学习记录：随工作进展追加到 .design-sync/learnings/{BATCH_ID}.md 中——每项发现一条，格式为：<组件>: <症状> -> <根本原因> -> <解决方案>；若适用于多个组件，则加前缀 [GENERAL]。

已知仓库陷阱（开始前请阅读）：
{CURRENT_NOTES_MD_CONTENT}

最终报告：按组件列出 match/close/blocked 状态及简要原因；随后以原文形式列出所有 [GENERAL] 学习内容。
```

**波次之间（协调器）——学习记录合并为必选项，而非可选：**
1. 阅读所有 .design-sync/learnings/*.md。将 [GENERAL] 条目提升至 .design-sync/NOTES.md（去重并保持简洁），然后删除已合并的学习文件。只要还有学习文件存在，完整 compare.mjs 运行就会输出 [LEARNINGS_UNMERGED]，且 §4d 驱动验收也会因这一条件而判定失败——遗漏合并可能导致问题悄然上线。
2. **在下一波启动前立即落实所有 [GENERAL] 学习成果，无论有多少组件受到影响。** 即使仅有 2/24 的组件出现该问题，它仍是全局性的；若未落实便推进下一波，每个受影响的组件都会再次遭遇此问题，而等到配置修正到位时，之前的等级记录早已失效。应用配置修正，**删除所有子代理为绕过该问题而创建的自有预览**（自有文件不会被机器删除，留在原地会掩盖修复效果），然后执行完整重建（必须是真正的重建——步骤 3 的批次推送会上传磁盘上的文件，绝不能使用 --skip-dts 的空壳版本）并验证。随后，通过限定范围的 compare.mjs --components 对实际受影响的 1–2 个组件进行验证，证明修复有效——**切勿在战役中途对整个清单进行比较。** 重建已经清除了修复所涉及的所有等级记录；这些组件只需重新加入队列，下一轮限定范围的运行会再次捕获它们，最后的 §4d 验收会一次性确认整个清单。如果在战役中途对大量组件进行了全量比较，那只是症状，而非常规步骤：要么这些组件从未被评级（每批次都应评级其捕获的所有内容），要么全局性的配置变更清除了原本已获得的等级——务必先诊断，再为渲染时间买单。
3. *渐进路径：* 将本波中已达到 §4d 等级标准的组件（所有故事均为 match，或符合评分细则的 close）作为已验证批次提交（基于 SKILL.md §3）——在完成步骤 1–2 后进行，以便本波的全局性修复首先应用于这些组件。
4. 下一波将收到更新后的 NOTES.md 内容，以及仍未通过的组件。最后一波结束后，对剩余组件重复步骤 1，并删除 .design-sync/learnings/。

### 4d. 完成标准 + 报告

- **一次 §7 驱动运行即为最终验收——无论采用何种路径。** 将本次会话的最后一次构建设为驱动（resync.mjs）；若无锚点（首次同步或恢复项目），则省略 --remote 参数——对已有锚点的项目进行完整复核仍视为通过。验收门限是驱动的判定结果：ok: true 且 verification.pendingGrade 为空。其覆盖范围为其工作列表中可捕获的部分——首次同步时为所有有故事的组件，重新同步时为 changed+added 组件——已携带等级的组件会被跳过，因此验收只需限定范围的通过，而非全面重新捕获（不可捕获的成员仅通过上传分区重新发送，无需评级；通过上传验证的组件不在验收范围内）。驱动会检查 .design-sync/learnings/ 自身，若有未合并的学习文件存在，则判定失败并显示 [LEARNINGS_UNMERGED]（.compare-report.json 的汇总仅支持完整运行）。在此次最终运行中，所有纳入范围的组件均应显示 “carried forward”，且 “grade cleared” 为零——这一条正是下次同步将很快完成的证明。若在无变化的运行中出现等级被清除的情况，说明存在非确定性来源输入（如不稳定的故事内容），应立即排查；驱动触发的 [SPOT_CHECK] 并非如此（流水线变动已自动验证——确认相关文档即可继续）。
- 所有纳入范围的有故事组件均应拥有最新的 .grade.json 文件，且所有故事均为 match——或符合评分细则接受标准的 close（§4）——或通过 cfg.overrides.<Name>.skip 并附 NOTES.md 理由予以跳过。机械检查依据驱动的 verification.pendingGrade：若组件出现在该列表中，则说明仍有故事未获得当前判定，尚未完成（通过上传验证的组件除外）。
- 最终重建后，package-validate.mjs 的退出码仍为 0，且不存在未解决的 [FONT_MISSING] 警告（§4a——这是比较引擎无法识别的唯一警告）。
- 从最终的 ds-bundle/.render-check.json（由 package-validate.mjs 写入；iterations 表示完整重建的次数）调用 DesignSync({method: 'report_validate', counts: {total, bad, thin, variantsIdentical, iterations}})。对于驱动限定范围的验收（§7），该文件可能不存在（跳过层级）或仅包含样本——若需要完整计数，应先以 --render-sample 0 重新运行驱动；若为无变化的重新同步且未上传任何内容，则跳过该调用。
- NOTES.md 应包含最新的“重新同步风险”章节，此时撰写，因为你还清楚这些风险：哪些内容可能悄然失效（内联至配置的数据、被禁用的故事导出、依赖上游 API 的自有预览），哪些仅被部分验证（故事上限、被接受的 close 理由），以及构建阶段做了哪些假设（工具链版本、CDN 获取的资源）。修复措施记录了你所做的工作；本节则告知下一次运行需要注意什么。
- 告知用户：N/M 个组件评级为 match，哪些为 close（及其合理性），哪些被跳过及原因。

## 5. 当仓库异常时——应急出口

首次处理特殊仓库时，难免会遇到默认设置无法覆盖的情况。每种启发式方法都有对应的可配置覆盖选项——原则是：**绝不要手动修补生成的输出，而应将修复写入下一次运行会读取的文件中。** 根据失败类型选择相应参数：

| 仓库的特殊性 | 配置项 | 存放位置 |
|---|---|---|
| 非标准构建/入口（`module` 指向 TypeScript 源码，分发布局另类） | `cfg.entry`、`cfg.buildCmd` | 配置 |
| CSS 由独立流水线构建 / 无 dist 旁路文件 / CSS-in-JS | 若有文件则使用 `cfg.cssEntry`；否则依赖 `[CSS_FROM_STORYBOOK]`——转换器会从 `sb-reference` 中提取**已编译**的 CSS，后者是通用兜底：无论构建流程多么奇特，其输出都在 Storybook 构建产物中 | 配置 |
| 以独立包形式发布的 tokens | `cfg.tokensPkg` | 配置 |
| 字体来自运行时服务 / 专有 CDN | `cfg.extraFonts`、`cfg.runtimeFontPrefixes` | 配置 |
| 子路径导出的图标或组件 | `cfg.extraEntries` | 配置 |
| 命名约定（故事标题 ≠ 导出名） | `cfg.titleMap`；故事与单元的对应关系也可退而求其次按顺序匹配 | 配置 |
| 不会被打包的装饰器/提供者（仅 Vite 插件、MDX、别名等） | `cfg.provider`——明确指定的链式组合优先于装饰器打包；`probe.mjs` 可从实时 Storybook 中推断；或者直接在组件自身的 `.tsx` 文件中**内联组合**提供者（自有预览可导入并包裹包所导出的一切） | 配置 / 预览 |
| 无法静态渲染的故事（MSW、数据获取、交互测试） | `cfg.overrides.<Name>.skip` + 在 NOTES.md 中注明原因。跳过会移除该故事的单元，但包装器仍会导入整个故事模块——若文件在导入时崩溃（模块作用域内的 fetch/worker），则应接管 `.tsx` 并直接移除该导入 | 配置 |
| `[PORTAL?]`——覆盖/门户故事会在网格卡片中绘制到单元之外 | `cfg.overrides.<Name>.cardMode: "single"`（还可选 `primaryStory`、`viewport: "WxH"`）——单故事卡片，固定定位的容器，声明产品视口。对比仍会通过 `?story=` 对每条故事进行评分 | 配置 |
| `[GRID_OVERFLOW]`——校验网格卡片的几何尺寸：`wide` 表示故事渲染宽度超过其单元（单元裁剪会在产品中截断）；`escape` 表示固定/门户内容的位置超出了所有单元 | 应用该覆盖，并记录警告名称——`wide` -> `cardMode: "column"`（每行一个故事，占满卡片宽度，保留所有故事）；`escape` -> `cardMode: "single"` + `primaryStory`。将结构化信息写入 `.render-check.json`（`gridOverflow`、`gridOverflowCells`、`suggestedOverride`）。将所有被标记的组件批量纳入一次定向重建（`preview-rebuild.mjs --components A,B,C`）——仅展示层的修改不会触发 `[CONFIG_STALE]`，且评分结果会沿用。无需为了确认而重新执行一次完整的校验：已应用的修复措施不会再次触发标记（single 完全豁免；column 无法再次标记为 `wide`——escape 仍受监控，因此后续新增的门户故事仍会被发现）；如需视觉确认，可查看 `.review.html` | 配置 |
| `[EXPORT_COLLISION]`——兄弟包（如图标等）导出了主包也导出的名称 | 主包在全局合并中胜出，因此从兄弟包导入该冲突名称的故事会渲染错误的内容 | 日志会给出修复方案：`cfg.storyImports.bundle: ["<sibling>"]` |
| `[FILE_TOO_LARGE]`——构建产物超出上传的单文件 12 MB 限制 | 通常是仅开发环境使用的重型资源被打包进了预览或装饰器包（语法高亮、代码形式的图标） | 在评分前立即精简——对自有预览进行事后瘦身会重新对该组件进行评分 |
| `[PROVIDER_UNEXPORTED]`——`cfg.provider` 指定的组件未作为包的导出 | 构建会在发出任何组件预览或文档之前提前退出一步，输出目录将处于不完整状态；修复后需重新构建 | 使用确切的导出名称，或通过 `cfg.extraEntries` 再次导出。检查会读取包本身的导出列表，因此缺失情况可靠；隐藏在 CommonJS 打包再导出背后的名称无法枚举——此类会附带 `[PROVIDER_UNVERIFIED]` 警告；若所有预览均报“元素类型无效”，则说明名称有误 |
| 故事导入解析错误（本应打包却打成了 shim，反之亦然——任何导入方式） | `cfg.storyImports.shim` / `cfg.storyImports.bundle`——基于解析后路径的子字符串匹配模式（裸包导入按**说明符**打成 shim，不经过解析——针对此类应匹配说明符）。未知的包子路径（如 `<pkg>/utils`）默认打包；若应走全局，则将其加入 `cfg.extraEntries`。在包的源码仓库中，自身打包导入无需解析——先将 `node_modules/<pkg>` 符号链接至构建后的 `dist/` | 配置 |
| 故事文件导入了默认加载器无法处理的资产类型（`.yaml`、`?raw`、作为组件的 SVG） | `cfg.storyImports.loaders`——在默认配置上合并的 esbuild 加载器映射（如 `{".yaml": "text"}`） | 配置 |
| 生成的预览 props 或组合错误 | 将 `.design-sync/.cache/previews/<Name>.tsx` 复制到 `.design-sync/previews/<Name>.tsx`，并删除其中的标记行（永久归属） | 预览 |
| 源码/文档发现遗漏（仓库布局异常） | `cfg.componentSrcMap`、`cfg.docsMap`、`cfg.dtsPropsFor`、`cfg.srcDir` | 配置 |
| 更深层次的需求——自定义故事格式、特殊的参数提取、CSS 变换 | 分支适配器：将打包的库模块复制到 `.design-sync/overrides/<name>.mjs`，并在 `cfg.libOverrides` 中声明，附上一行理由（构建会双向校验：防止 `[OVERRIDE_UNDECLARED]` 和 `[OVERRIDE_MISSING]`）。分支会被提交，因此后续同步会自动使用它们。**`emit.mjs` 和 `bundle.mjs` 是应用契约的表面——切勿对其进行分支。** | `.design-sync/overrides/`具体到**故事处理**，按关注点划分的分支点如下：`story-imports.mjs`（用于预览构建的所有导入解析策略——为每个仓库的定制化而设计的接口，完整构建和 `preview-rebuild.mjs` 都会遵循）、`source-storybook.mjs`（index.json 的发现、标题到组件的映射、故事源的解析与导出配对）、`preview-gen-storybook.mjs`（包装模板与 composeStories 语义）、`css-fallback.mjs`（从 Storybook 构建中提取 CSS/字体）。请 fork 那个“最窄”的、负责该问题的模块，保留其导出签名，并在 NOTES.md 中记录该仓库的不同之处——下次同步时这些内容都会被继承。fork 会从 `.design-sync/overrides/` 加载，而其兄弟模块仍位于暂存脚本中；同时将 fork 的相对导入（如 `./common.mjs` 等）指向 `../../.ds-sync/lib/`。如果 fork 引入了原生转换器依赖（如 `esbuild`），还需执行 `ln -sfn ../.ds-sync/node_modules .design-sync/node_modules`，以便 Node 能从 fork 的位置解析该依赖——只需在克隆时执行一次，而非每次：该符号链接会被 Git 忽略（遵循 `node_modules` 规则），而需要它的已提交 fork 会在克隆后依然存在，因此在全新克隆时需重新创建。

对于那些确实超出转换器覆盖范围的仓库，梯子的最后一级是：**上传格式才是契约，而非转换器本身**（参见基础技能）。无论仓库以何种方式生成布局，`package-validate.mjs` 以及比对/分级门控都会原封不动地应用于你所产出的内容。Oracle 永远不会被 fork。

表格中的所有内容都是已提交的文件，且 §2.3 要求在动手之前先阅读现有配置和 NOTES.md——因此，针对第 N 次所做的每项决策，都要进行 N+1 次回放验证。当你在某个特殊仓库上修复问题时，应自问：“哪份已提交的文件能让下一次自动解决这个问题？”若答案是否定的，至少应在 NOTES.md 中记录下来，而且很可能意味着此处缺少一条值得上报的记录。

## 在上传前编写规范头文件

当预览已通过验证——无论是新编写的还是通过重新同步带过来的——都应按照基础 SKILL.md 中的“编写规范头文件”步骤来操作：它会将你在让预览正常渲染过程中学到的内容提炼成 `.design-sync/conventions.md`，并通过 `readmeHeader` 配置键将其关联起来。顺序很重要：务必先编写该文件并设置该键，然后再按照基础步骤中的“重建规则”进行重建（对每条路径都执行一次全新的 DRIVER 运行——首次同步时省略 `--remote` 参数），这样生成的 README 才能真正包含该头文件，而 §4d 的确认也会描述 §6 的构建上传内容。随后即可进入下方的“上传”环节。

## 6. 上传

两条路径中的哪一条适用，由基础技能 §1 的路由决定（运行开始时已固定 -> 原子式；否则为空 -> 增量式，非空 -> 原子式）：

**增量路径**（首次同步到一个空项目）：计划自本文件 §3 的门控起就已公开，且经过验证的批次也已落地。在 §4d 通过并规范头文件步骤完成后（基础 SKILL.md ——必须先于其重建所驱动的上传），再执行基础 SKILL.md §3 中的收尾工作——哨兵围栏 -> 全量内容写入 -> 对账删除 -> 哨兵重置 -> 最后写入 `_ds_sync.json`。这些写入操作同样适用本节的分块、卫生及本地化规则；`projectId` 已在 §1 中记录；本节末尾的手动审计依然有效。跳过本节剩余的流程——那是原子路径。

**原子路径**（重新同步，或任何非空目标——可能正在使用中，因此在一切验证无误后一次性更新）：以下所有步骤。仅在 §4d 和规范头文件步骤（基础 SKILL.md）之后执行。调用 `DesignSync(finalize_plan)`，并指定 `localDir: "./ds-bundle"`。- **写入——始终写入所有内容**（包括完整的重新验证和重新同步）：`writes: ["components/**", "tokens/**", "fonts/**", "_vendor/**", "_preview/**", "guidelines/**", "_ds_bundle.js", "_ds_bundle.css", "styles.css", "README.md", "_ds_sync.json", "_ds_needs_recompile"]`。重复上传未更改的文件是幂等且成本极低的。如果 `writes` 列表范围过小，会导致项目在后台悄然且永久性地不同步——因此，全量写入是安全的默认选项。
- **删除。** 带锚点的重新同步：完全按照差异中的内容执行——直接复制 `.sync-diff.json` 中的 `upload.deletePaths`；切勿手动推导该列表，也切勿在差异中列出路径时传入 `[]`。无锚点（即重新采用或恢复的非空项目正在进行完整重新验证）：此时差异无法查看项目的历史，因此请在调用 `finalize_plan` 之前立即审查其 `list_files`，找出本次构建不生成的文件，并将这些已审查的路径加入计划的 `deletes` 中（计划中未列出的删除操作将被拒绝）。
- **第4d节的最终收据同时作为上传的真实依据。** 会话的最终构建本身就是一次第7节的驱动运行（第4d节）；仅运行 `package-build.mjs` 会清空 `.sync-diff.json`，而驱动的差异阶段会重新生成它，因此 `deletePaths` 和 `upload.any` 准确描述了您上传的字节内容——一次运行既是验证收据，也是上传清单，无需在其后进行额外的完整比对。
- **`upload.any === false` -> 完全跳过上传**——此时项目已与本次构建完全一致。（下方的手动交接审计仍需执行。）
- **`_ds_sync.json` 是最后的绝对写入操作**——在所有内容写入、所有删除以及哨兵重置之后，通过单独的 `write_files` 调用完成。若过早上传，计划中途失败会导致锚点为项目中不存在的文件背书，而确定性的重建意味着后续同步也无法修复这些问题。
- **本地保留的内容**：`_sb/**`（storybook-static 是参考内容，从不上传）、以点开头的条目（`.stories-map.json`、`.compare-report.json`、`.ds-build-meta.json`、`.sb-static/`、`.sync-diff.json`）以及 `_screenshots/`。`_vendor/` 和 `_preview/` 需要上传——预览卡片会从中加载 React 和编译后的预览内容。
  
如果 `finalize_plan` 被拒绝，**请立即停止**——拒绝表示当前会话无法批准，而非参数有误。应告知用户具体被拒绝的内容，并询问其希望如何处理：再次尝试批准，或获取已验证的 `ds-bundle/` 并由用户自行交互式地执行上传。
  
计划获批后，上传按固定顺序进行：

1. **先设置哨兵**：`DesignSync(write_files, [{path: "_ds_needs_recompile", localPath: "_ds_needs_recompile"}])`——用于防止应用的清单及复制机制处于半上传状态。
2. **所有内容写入**，分批以不超过256个文件的 `write_files` 调用完成，且使用相同的 `planId`。服务器不仅限制文件数量，还对有效载荷的字节数加以约束——对于二进制文件较多的目录（如 fonts/、images/），应拆分成更小的批次；若遇到 500 MB 的限制，则将批次大小减半后重试。
3. **所有删除操作**：针对 `upload.deletePaths` 中的每个路径调用 `DesignSync(delete_files)`。（无锚点时，指的是在 `finalize_plan` 时已审查并加入计划 `deletes` 的那些路径——参见上方的“删除”部分。）如果 `delete_files` 拒绝了远程不存在的路径（例如地板卡片组件没有 `_preview/` 文件），则可剔除这些被拒绝的条目后重试——只有这种“未找到”的拒绝可以继续处理，其他任何写入或删除失败都必须停止。
4. **重置哨兵，最后写入 `_ds_sync.json`。** 即使在删除之后也要设置锚点——若删除失败，可能会导致刷新后的锚点无法再识别某些远程文件。

任何其他写入或删除失败，即使重试也无法解决，都必须**停止**——不得重置哨兵，也不得写入 `_ds_sync.json`。对于未锚定的项目，下次同步时只会重新验证；而对于已建立新锚点的项目，半途而废的上传将造成永久性问题。**上传时的卫生习惯**：将文件列表和分块清单保存在 `.design-sync/` 目录下，切勿直接使用 `/tmp` 路径，否则其他仓库同步时遗留的过时清单可能会上传错误的设计系统。在上传前立即从实时的 `ds-bundle/` 重新生成清单，并进行合理性检查：组件名称必须属于当前设计系统，且包中的 `window.<globalName>` 应与之匹配。最后调用 `DesignSync(list_files)` 确认数量。

仅当上传后的 `list_files` 计数通过验证后，**如果 `.design-sync/config.json` 中缺少或项目 ID 不同时，才将其记录到该文件中**（这是最后一道防线——§1 在每次路由的目标结算时都会记录项目 ID，因此通常它已经存在；绝不能在上传验证之前就在此处记录项目 ID，以免将配置固定到内容尚未真实存在的项目上）——这会锁定未来重新同步所依据的项目。完成后，向用户告知：项目 URL（`https://claude.ai/design/p/<projectId>`）、组件数量、对比结果摘要，以及验证已顺利结束。持久化集合（即下文交接审计中的规则：所有未被 Git 忽略的 `.design-sync/` 下的内容）必须留在仓库中，以便后续重新同步复用所有修复；经过验证的状态应与已上传的 `_ds_sync.json` 一同保存，而不提交到 Git。下文的交接审计会涵盖是否提交的提示。

**最后一步——交接审计**。未来的运行速度和正确性完全取决于本次运行留下的成果；务必验证，切勿假设：

1. 执行 `git status`——持久化集合（`.design-sync/` 下所有未被 Git 忽略的文件——目前包括 config.json、NOTES.md、conventions.md、预览目录、覆盖目录；规则即契约，因此未来新增的持久化文件也自动纳入此集合）是本次同步在仓库中的足迹；而 `sb-reference/`、`learnings/`、`.cache/`、`.ds-sync/` 均被忽略。如果本次运行创建或修改了任何持久化文件，**应提示用户提交并打开 PR**（单次提交，仅包含同步状态，不得混入无关文件）。未提交的修复意味着下一次同步将无法获得该修复。
2. 再次阅读 NOTES.md，仿佛你是下一位执行者，对本次会话一无所知：仅凭文档内容，你是否能跳过今天的调试？每一个自有的预览、跳过项、配置开关和库分支都应在文档中找到对应条目，且“重新同步风险”部分应保持最新（§4d）。若发现遗漏，立即补充——今天花一分钟，未来可省一次重新推导的时间。
3. 无论重新同步带来了多少变化或重新校准，除非本次运行产生了下一次运行需要知晓的新信息，否则应将 NOTES.md 和 Git 状态恢复为初始状态；只有当提交确实能为未来的同步带来价值时，才让用户提交。

## 7. 重新同步——一条命令即可完成全部工作

仓库中保存着同步的输入（配置、自有预览、NOTES.md）；已上传的项目则承载着锚点（_ds_sync.json）。首先阅读 NOTES.md（“重新同步风险”部分为关注清单），然后：

1. **刷新输入**。重新复制暂存的脚本（参见 §2.4 的 `cp -r` 命令——即时完成；过时的 `.ds-sync/` 会使用旧版转换器处理这些指令）。只要设计系统源可能发生变化，就重新运行 `buildCmd` 并重建 `.design-sync/sb-reference`——两者必须同步更新；如有疑问，两者均需重建（确定性构建使得不必要的重建不会产生任何影响；捕获日志中的 `[REFERENCE_STALE?]` 表示你遗漏了这一步）。额外的初始化操作包括：§2.4 的依赖安装 + Chromium 浏览器，§2.2 的 sb-reference 构建，以及——如果仓库中存在带有裸导入的 `.design-sync/overrides/` 分支——`ln -sfn ../.ds-sync/node_modules .design-sync/node_modules`。
2. **获取锚点**：调用 `DesignSync(get_file, path: "_ds_sync.json")`，并将结果保存至 `.design-sync/.cache/remote-sync.json`。若项目中无辅助文件，则视为首次同步范围（此时省略下方的 `--remote` 参数）。
3. **在仓库根目录运行驱动程序**：

   ```sh
   node .ds-sync/resync.mjs --config .design-sync/config.json --node-modules <nm> \
     [--entry <dist-entry>] --out ./ds-bundle --remote .design-sync/.cache/remote-sync.json
   ```它按顺序执行构建 → 差异分析 → 验证 → 捕获（仅限新增及契约变更的组件），并输出一份结果 JSON（同时写入 `ds-bundle/.resync-verdict.json`）。各阶段的日志会流式输出到标准错误。该驱动程序具有幂等性——修复后可重复运行。若需对单个组件进行预览迭代，请改用 §4a 中的定向循环（耗时以秒计，而非完整构建与渲染检查）；驱动程序的再次运行则作为最终确认。

驱动程序还会根据差异分析的结果来限定验证阶段的渲染检查范围（显式指定的 `--render-sample` 或 `--no-render-check` 标志优先生效）。在锚点健康且包与样式未发生变化的情况下，每个未变更预览的渲染输入与上次上传时经验证（或被明确接受）的完全一致——差异分析将锚点固定至最新的侧载文件，[SYNC_STALE]/包 SHA 的重新计算则将渲染表面固化至磁盘（样式由刚刚写入两者的构建过程锁定），而对相同字节的重新渲染仅用于测试您的 Chromium 安装，而非构件本身。因此：若一切未变，则渲染检查会被**跳过**（该次运行中出现的 [RENDER_SKIPPED] 警告是驱动程序主动发出的预期信息，无需进一步排查）；若有内容仍需发布但不影响渲染的部分发生变动（如文档或指南的编辑、锚点刷新），则进行**采样检查**（使用 `--render-sample 10`）；凡可能改变渲染的内容发生变动——组件变更、新增或频繁变动，或包与样式发生变化（例如 `.d.ts`/`.prompt.md` 的修改会导致包重新发布，其头部会嵌入这些文件的哈希值）——或者没有健康的锚点，则始终执行**完整检查**。文件结构检查（[SYNC_STALE]、包头、CSS/字体、`.d.ts` 解析）在每一层级都会完整执行；若要强制执行完整渲染流程，可传入 `--render-sample 0`。

4. **根据结果采取行动**——针对所有需要您处理的字段：

| 字段 | 您的工作 |
|---|---|
| `ok: false` | 失败的阶段（`stages.<name>`）已记录其 `[TAG]` 日志——请参照该阶段的说明进行修复并重新运行。若所有阶段均通过，则检查 `learningsUnmerged` |
| `learningsUnmerged` 非空 | 展开并整理分支学习成果——归入 NOTES.md，删除相关文件（§4c 第 1 步），然后重新运行；仅此一项即会使 `ok` 失败，且本次运行会保留用于重试的参考漂移信标 |
| `verification.pendingGrade` | 对这些新提交的页面进行评分（参见 §4 的评分标准）。在捕获日志中：若出现 `[STORY_CHANGED]`，请先在对应的 `.tsx` 文件中同步故事内容；若为 `unpaired`，请添加导出声明；若出现 `extraCells` 并指向某个自有导出，请将其移除 |
| `verification.canary` | 若流水线发生变动（或参考 Storybook 发生变化），而您的源码保持稳定，则评分应保持不变；请对照记录的评分核验指定的 `[SPOT_CHECK]` 页面。若有少数页面评分不一致，重新评估这些组件；若大面积偏离，则使用 `--force` 强制执行完整检查 |
| 验证日志中的警告行（如 `[RENDER_THIN]` 等） | 查看 NOTES.md 中的已知列表——若警告已在其中记录，则表明该问题已在之前的同步中得到妥善处理（某些组件确实较短，会被持续判定为“thin”）；若未记录，则为新问题——请检查该组件，并予以修复或将其记录在 NOTES.md 中 |
| `verification.removed` | 若有组件已在上游被移除，请确认这些删除操作确属有意 |
| `upload.styling: true` | 样式会自动重新发布，评分保持不变 |
| `upload.any: false` | 本次结果无需上传任何内容——继续执行第 5 步；只有在完成第 5 步后才算真正结束（届时会生成一个包含相应信息的头部，从而触发驱动程序的再次运行） |
| `upload.any: true` | 执行 §6 的上传步骤——默认执行全量写入，`deletes` 则严格按照 `upload.deletePaths` 中的路径进行删除（切勿依据验证分区来限制写入范围） |

评分机制的设计决定了评分会沿用既有的来源——DS 源、CSS 和 bundle 的变更都会被继承，而流水线的变动则以 `verification.canary` 的形式体现，而非重新评分。若需在 DS 版本大幅升级后或存在疑虑时，主动对已继承的评分进行审计，可执行命令：`node .ds-sync/storybook/compare.mjs --out ./ds-bundle --components <A,B> --spot-check-components <A,B>`——这将生成全新的评分表，并保留原有评分，随后确认这些评分表是否仍与记录中的评分一致。
5. **执行“编写规范头”步骤**（基于 SKILL.md 中的“编写规范头”任务）——在根据评审结果采取行动之后、上传之前，且无论评审结果如何，都必须执行此步骤。在重新同步时，该步骤会将现有的 `.design-sync/conventions.md` 文件与最新构建进行比对，并报告偏差；对于在该步骤引入之前就已同步的仓库，它将首次创建该文件。如果该步骤新建或修改了规范头，则应按照基础步骤的**重建规则**（此处为驱动程序运行）重新构建，并根据新的评审结果采取相应措施——因为之前的评审结果是在规范头出现之前作出的。
6. 在执行 `finalize_plan` 之前，请再次获取 sidecar 文件；如果 sidecar 发生了变化（例如因并发同步），请重新运行驱动程序，并根据最新的评审结果采取行动。