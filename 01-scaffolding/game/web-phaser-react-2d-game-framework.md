# 任务：从 0 搭建「基于 Phaser 最新版 + React」的大型 2D 复古像素游戏项目框架（空壳）

## 0. 背景与目标

我要做一个网页版 2D 复古像素游戏，**游戏类型尚未确定**（可能是横版闯关、模拟养成、剧情、RPG 或混合）。
现在只需要你搭建**项目框架**，不要写任何游戏内容（没有关卡、角色设定、剧情、数值）。

这个框架将长期使用：我之后会一个需求一个需求地交给 AI 在此基础上开发。所以请**以超大型游戏项目为目标**设计目录、分层和模块边界，尽量覆盖未来可能出现的需求（例如：时间线/过场、对话、任务、背包、存档、战斗、日历、本地化等），让未来每个需求都有明确的"落脚点"，而不需要推翻重构。

请你以**资深游戏客户端架构师**的视角工作，主动做出专业决策，并把关键决策记录为 ADR。除非遇到无法继续的阻塞，否则**不要向我提问**。

语言约定：对话与文档用中文；代码标识符用英文；代码注释用简洁中文。

## 1. 技术栈（硬性要求）

- **Phaser：安装 npm 上的最新稳定版（目前为 v4.x 系列，用 `npm view phaser version` 确认）。**
  Phaser 4 重写了 WebGL 渲染器，部分 v3 的用法已变更。**严禁凭 v3 记忆写代码**：必须以 `node_modules/phaser` 内的类型定义和官方文档/迁移指南为准。官方 React 模板可能仍是 v3，只可借鉴其"React 与 Phaser 通信"的思路，不可照搬。
- Vite（最新稳定版）+ TypeScript（`strict`，开启 `noUncheckedIndexedAccess`）+ React（最新稳定版）
- 包管理器：pnpm（没有则用 npm，并在文档中说明）
- 允许的运行时依赖：`phaser` `react` `react-dom` `react-router-dom` `zustand` `zod` `i18next` `react-i18next`
- 允许的开发依赖：`vite` `@vitejs/plugin-react` `vitest` `@testing-library/react` `jsdom` `eslint`（flat config）`typescript-eslint` `prettier` `playwright` `dependency-cruiser` `tsx` `vite-tsconfig-paths`
- 除此之外新增任何依赖，必须在 ADR 里写明理由。所有版本号以实际安装为准，并写入 `docs/ADR/`。

## 2. 分层架构与依赖规则（最重要，必须用工具强制执行）

```
src/
├─ main.tsx
├─ app/        # React 应用壳：路由、Provider、页面(Home/Play/...)、错误边界、布局
├─ ui/         # React 游戏内 UI：HUD、对话框、背包、菜单栈、设置、存档槽、加载页、像素风 UI Kit、调试面板
├─ bridge/     # React↔Phaser 唯一通道：类型化 EventBus、<GameHost/> 组件、store 同步适配器
├─ stores/     # Zustand：供 UI 订阅的状态（settings、session、ui、hud...）
├─ game/       # Phaser 层（唯一允许 import phaser 的业务层）
│  ├─ boot/        # 创建 Game、Phaser 配置、平台适配
│  ├─ scenes/      # system/(Boot,Preload,...) world/ overlay/ debug/ 以及 BaseScene、SceneKeys、SceneRegistry
│  ├─ entities/    # Entity/Component 基类、prefab 工厂注册表
│  ├─ systems/     # 依赖 Phaser 的运行时系统（见 §4）
│  └─ plugins/     # 自定义 Phaser 插件
├─ domain/     # 纯 TypeScript：规则与数据模型，零 Phaser、零 React（见 §4）
├─ content/    # 游戏内容数据 + zod schema + 本地化文本（本阶段只放空数据与示例说明）
├─ core/       # 纯 TS 基础设施：ServiceContainer、类型化事件发射器、Logger、Disposable/Lifecycle、Result/错误类型
├─ shared/     # 通用 types、utils（数学/网格/时间格式化等）、constants
└─ config/     # display、input、audio、game、featureFlags、env
```

**依赖规则（用 dependency-cruiser 或 ESLint 规则写成配置，违反时 `pnpm lint` 必须失败）：**

1. `domain`、`core`、`shared`、`config`、`content` 不得 import `phaser` / `react` / `game` / `ui` / `app` / `bridge`。
2. `game` 不得 import `react` / `ui` / `app`；`ui`、`app` 不得 import `phaser` / `game`。
3. React 与 Phaser **只能通过 `bridge`** 通信：UI → 游戏用"命令事件"，游戏 → UI 用"状态/通知事件"，UI 可订阅的状态落在 `stores`。
4. 每个模块只通过自己的 `index.ts` 暴露公共 API，禁止跨模块 import 内部文件。
5. 使用路径别名（`@core` `@domain` `@game` `@ui` `@bridge` `@stores` `@content` `@shared` `@config` `@app`），禁止 `../../..` 这种深层相对路径。同步配置 tsconfig、vite、vitest。

## 3. 项目其余目录

```
public/assets/             # 随游戏发布的资源：fonts/ images/(sprites,ui,tilesets,backgrounds,portraits) atlases/ tilemaps/ audio/(bgm,sfx,voice) + 生成的 manifest.json
assets-src/                # 不发布的源文件：aseprite/ tiled/ audio-src/ + README（导出规范：像素图命名、图集、Tiled 导出设置）
scripts/                   # generate-asset-manifest、validate-content 等（tsx 运行）
e2e/                       # Playwright 冒烟测试
docs/                      # 见 §6
.github/workflows/ci.yml   # install → typecheck → lint → test → validate:content → build
AGENTS.md  CLAUDE.md       # 见 §6（AGENTS.md 为正文，CLAUDE.md 内容仅为 `@AGENTS.md`）
```
另外配置：`.editorconfig`、`.gitignore`、`.nvmrc`（Node LTS）、`prettier`、`eslint`。
`package.json` scripts 至少包含：`dev` `build` `preview` `typecheck` `lint` `format` `test` `test:e2e` `validate:content` `gen:assets`。

## 4. 模块清单与实现深度（分三级，严格按级别执行，不要越级膨胀）

每个模块都要有独立目录和 `README.md`（职责、公共 API、扩展点、与其他模块的关系、当前状态）、`index.ts`。
状态标记：✅ 可用并有测试 / 🟡 最小可运行内核 / ⚪ 仅骨架。

### Tier 1 —— 必须真正可运行（✅）
- **core**：ServiceContainer（类型化注册/获取、生命周期、销毁顺序）、类型化 EventEmitter、Logger（分级，生产环境可降噪）、Disposable、Result/错误类型
- **bridge**：类型化 `EventBus`（事件名采用 `域:动作` 命名并集中定义）、`<GameHost/>`（创建/销毁 Phaser 实例；**必须正确处理 React StrictMode 双挂载与 Vite HMR，不得泄漏第二个 Game 实例**）、store 同步适配器
- **game/boot + scenes**：`BaseScene`（统一注入 services、自动清理订阅、统一生命周期钩子）、`SceneKeys`、`SceneRegistry`；场景流 `BootScene → PreloadScene → SandboxScene`（SandboxScene 即"空壳场景"）
- **config/display**：基准分辨率（默认 480×270，集中配置，可改）、整数倍缩放选项、居中、像素渲染设置（按 Phaser 4 文档正确开启 pixelArt / roundPixels 相关配置，并给 canvas 加 `image-rendering: pixelated`）
- **assets**：资源清单（manifest）+ 分包（bundle：`core` / `ui` / 按场景懒加载）+ 加载进度上报到 bridge；资源缺失时优雅降级并打日志，不崩溃
- **input**：动作映射（action map，而非直接读键码）、键盘 + 手柄基础支持、可改键数据结构；触屏虚拟摇杆留接口
- **audio**：BGM / SFX 分总线、音量、淡入淡出/交叉淡入、浏览器自动播放限制处理（首次交互后解锁）
- **settings**：音量、语言、显示（整数缩放/全屏）、键位、无障碍（减弱震屏等）；持久化与存档**相互独立**
- **save**：存档信封 `{ meta:{version, savedAt, playtimeMs, slot, appVersion}, state }`；各模块通过 `Saveable` 接口（`saveKey` / `serialize` / `deserialize` / `version` / `migrate`）自行贡献数据；多槽位、自动存档、版本迁移链、导入/导出文件；存储适配器接口（先实现 localStorage 与 IndexedDB 两种，云存档只留接口）；**需要完整单测**
- **i18n**：基于 i18next，默认 `zh-CN`，附带 `en`；文本 key 集中管理；UI 与 Phaser 共用同一套翻译
- **ui kit（React）**：像素风设计令牌（CSS 变量）、Button / Panel / Modal / Tooltip / ProgressBar / Tabs / Toast；`MenuStack`（界面栈管理）；支持键盘/手柄焦点导航的基础
- **debug**：仅开发环境启用（生产构建中必须被摇树移除）的调试面板（FPS、当前场景、事件总线日志、flags 查看、时间倍率），快捷键 F2 开关

### Tier 2 —— 纯逻辑最小内核 + 单元测试（🟡，全部放在 `domain/`，不依赖 Phaser/React）
- **timeline（时间线/过场序列）**：数据模型（轨道 tracks、片段/关键帧、标记 markers、并行/串行/等待/循环）、`TimelineRunner`（由外部 delta 驱动，支持暂停/跳转/倍速/跳过/序列化）、**动作注册表**（`registerAction(type, handler)`，把"时间线数据"与"具体表现"解耦）。`game/systems/timeline` 中放 Phaser 侧的动作处理器骨架（相机、补间、动画、音频、对话、设置 flag 等）。
- **dialogue**：对话图模型（节点、选项、条件、变量、说话人、立绘引用、本地化 key）+ `DialogueRunner`；表现层分别留给 `ui/`（DOM 对话框）和 `game/systems`
- **flags / variables**：全局/场景/存档作用域的变量与开关，支持变更订阅
- **events & triggers**：条件表达式（可序列化 JSON，禁止 eval）、触发器、区域触发的数据模型
- **quest**：任务/目标状态机与进度
- **inventory**：物品定义/实例分离、容器、堆叠、装备槽
- **stats**：属性、修饰器栈（加/乘/覆盖）、Buff 持续时间
- **calendar（游戏内时间）**：游戏时钟、昼夜、日/周/季、时间缩放；**注意它与 timeline 是两个不同概念**
- **fsm**：有限状态机/分层状态机
- **scheduler**：受游戏时间缩放、可暂停的计时器与延迟任务
- **rng**：可播种的随机数 + 权重表/掉落表
- **pool**：通用对象池
- **content registry**：`id → 定义` 注册表，用 zod 校验、检查 id 唯一性与跨表引用（如任务引用的物品是否存在），配套 `scripts/validate-content.ts`

### Tier 3 —— 仅骨架（⚪：README + 接口/类型 + 空实现，保证编译与 lint 通过）
combat（命中/受击框、伤害管线）、ai（行为树/效用 AI）、crafting、shop/economy、relationship（好感度）、progression（经验/技能树）、loot、spawner、interaction（可交互物）、achievements、codex/journal（图鉴）、tutorial/hints、tilemap 加载器（Tiled 约定）、pathfinding（网格 A*）、camera 控制器、scene transition（淡入淡出等）、fx 预设（震屏/闪白/粒子）、weather、lighting、day-night 叠加层、cloud-save 适配器、analytics 适配器、touch 虚拟控制、`ui/` 下的 HUD、DialogueBox、InventoryPanel、SaveSlots、PauseMenu、Settings 页面（只做空壳容器）。

> 注意：遇到本清单之外但明显必要的基础模块，可以补充，并在 `docs/MODULES.md` 与 ADR 中说明。

## 5. 首页与空壳场景（本阶段唯一的"可见内容"）

- 路由：`/` 首页（Home），`/play` 游戏页；游戏页懒加载（Phaser 单独分包）。
- **首页**：像素风排版，居中展示——项目名（默认 `pixel-game-framework`，集中配置可改）、一段介绍（说明这是一个**基于 Phaser 的 2D 复古像素游戏框架，目前还是一个空壳**，列出技术栈与"未来可扩展的模块"）、**实时读取并显示 Phaser / React 的实际版本号**、语言切换（zh-CN / en）、「进入空场景」按钮。介绍文字走 i18n。
- **游戏页**：`<GameHost/>` 挂载 Phaser，画布按 §2 的显示配置缩放。`SandboxScene` 只显示一行像素风文字（如"Phaser 空壳"）和一个**运行时用 Graphics 生成的占位精灵**（不依赖任何外部图片），用来肉眼验证像素渲染不模糊。
- **通信演示**：游戏页放一个 React 按钮，点击后通过 bridge 让 Phaser 场景里的占位精灵发生变化；场景里再回传一个事件让 React 显示计数。用它证明双向通道可用。
- **中文像素字体**：使用开源且许可证允许的中文像素字体（如方舟像素 Ark Pixel / 缝合像素 Fusion Pixel，需自行确认 OFL 等许可证）放入 `public/assets/fonts`，在游戏启动前确保字体已加载完成（`document.fonts.load`）；若无法联网下载，则回退到系统等宽字体，并在 `docs/` 写明手动放置字体的步骤。资源来源与许可证记录在 `docs/CREDITS.md`。
- 同时处理：移动端 viewport、canvas `touch-action: none`、防止页面滚动/双击缩放、React 全局错误边界 + `window.onerror` 的日志上报入口。
- 构建：`vite.config` 中 `base: './'`（可静态托管到任意子路径），把 `phaser` 拆成独立 chunk。

## 6. 文档（给未来的 AI 和我自己看的，质量与代码同等重要）

- `AGENTS.md`（CLAUDE.md 仅写 `@AGENTS.md`），内容必须包含：
  1. 项目简介与技术栈、常用命令；
  2. 分层与依赖规则（§2 的精炼版）；
  3. **「需求 → 落脚点」对照表**：列出至少 30 类常见需求（如"过场/时间线""对话""任务""背包""存档""战斗""昼夜/日历""新增场景""新增 UI 面板""新增音效""新增本地化文本""新增物品/敌人数据""改键""调试工具"…）分别应该改动/新增哪些目录和文件；
  4. **术语表**：明确区分易混概念，如 timeline（过场序列）≠ calendar（游戏内日历）≠ 事件日志/历史记录；当我的需求含糊时，先对照术语表判断，再在回复里写明你的理解；
  5. 编码规范：不使用 `any`；具名导出（不用 default export，页面组件除外）；单文件尽量不超过 300 行；每个新模块必须带 README、`index.ts` 与测试；Phaser 对象不得泄漏进 `domain`、`stores`、`ui`；所有魔法数字进 `config`；所有用户可见文本走 i18n；
  6. **开发工作流**：先读 `docs/ARCHITECTURE.md` 与相关模块 README → 给出改动计划 → 小步实现 → 跑 typecheck/lint/test → 更新模块 README 与 `docs/MODULES.md` 状态 → 需要时补 ADR。
- `docs/ARCHITECTURE.md`：分层图（可用 Mermaid）、数据流（输入 → 逻辑 → 渲染 → UI）、React↔Phaser 通信约定、场景流、存档流程、内容加载流程。
- `docs/MODULES.md`：全部模块一览表（路径、职责、状态 ✅🟡⚪、依赖关系）。
- `docs/CONVENTIONS.md`：命名、目录、事件命名、资源命名与图集约定、Git 提交规范。
- `docs/ADR/`：每个关键决策一篇短文，至少包括：Phaser 版本与 v4 适配结论、分层与依赖规则、为什么用 domain 纯逻辑 + 动作注册表、存档方案、状态管理方案、为什么**暂不引入 ECS**（采用 Entity + Component 组合，并说明何时考虑引入）、像素渲染与分辨率策略、字体方案。
- `docs/REQUEST_TEMPLATE.md`：我以后向 AI 提需求时用的模板（背景 / 期望行为 / 涉及模块 / 验收标准 / 不要改动的范围）。
- `docs/ROADMAP.md`：建议的后续开发顺序（先做哪些模块的真实实现，为什么）。

## 7. 工作方式与阶段

请按阶段执行，**每个阶段结束时跑通验证并 `git commit`**（先 `git init`）：

- **阶段 A｜调研与脚手架**：核对 Phaser 最新稳定版及 v4 的 API 变化；初始化 Vite + React + TS；配置别名、lint、format、依赖边界规则；写 ADR-001/002。
- **阶段 B｜Tier 1**：按 §4 实现，并完成 §5 的首页、游戏页、通信演示。
- **阶段 C｜Tier 2**：纯逻辑内核 + 单测。
- **阶段 D｜Tier 3 骨架与文档**：骨架、README、`AGENTS.md`、`docs/*`、CI。
- **阶段 E｜总验收**（见 §8）。

遇到不确定的 Phaser v4 API：先查类型定义与官方文档，再动手；查不到就写成适配层并在 ADR 里标注风险，不要猜。
不要编造不存在的 npm 包或 API。不要写任何游戏内容，不要引入有版权风险的素材。

## 8. 验收标准（全部满足才算完成）

1. 全新克隆后 `pnpm install && pnpm dev` 即可运行；首页正常显示，点击进入后看到清晰不模糊的像素场景。
2. React ↔ Phaser 双向通信演示可用；StrictMode 与热更新下没有重复的 Game 实例或内存泄漏。
3. `pnpm typecheck`、`pnpm lint`（含依赖边界规则）、`pnpm test`、`pnpm validate:content`、`pnpm build` 全部通过；故意制造一次违反分层规则的 import，`lint` 必须报错（验证后还原）。
4. Playwright 冒烟测试：打开首页 → 进入游戏页 → 确认 canvas 存在且无控制台报错。（若环境无法安装浏览器，保留配置与用例，并在最终报告中说明。）
5. 存档模块的单测覆盖：多槽位、版本迁移、损坏数据的容错、导入/导出。
6. §4 每个模块都有 README 与 `index.ts`，`docs/MODULES.md` 状态与实际一致。
7. 生产构建中不包含调试面板代码。

## 9. 最终报告（完成后输出）

用中文列出：实际安装的各依赖版本；关键决策与偏离本提示词之处及原因；每个模块的最终状态（✅🟡⚪）；已知风险与待办；如何运行、测试、构建；建议我下一步让 AI 做的 3 个需求。
