# 任务：在我已有的 Android 空 Activity 项目上，搭建「基于 Android 17 + libGDX + Kotlin」的大型 2D 游戏项目框架（空壳）

## 0. 背景与目标

我要做 Android 游戏，**游戏类型尚未确定**（可能是横版闯关、模拟养成、剧情、RPG 或混合；偏 2D 复古像素风）。
现在只需要你搭建**项目框架**，不要写任何游戏内容（没有关卡、角色设定、剧情、数值）。

这个框架会长期使用：我之后会一个需求一个需求地交给 AI 在此基础上开发。所以请**以超大型游戏项目为目标**设计模块边界与目录，尽量覆盖未来可能出现的需求（时间线/过场、对话、任务、背包、存档、战斗、日历、本地化、Play 服务等），让每个未来需求都有明确的"落脚点"，避免日后推翻重构。

请以**资深 Android 游戏客户端架构师**的视角工作，主动做出专业决策并用 ADR 记录。除非遇到无法继续的阻塞，否则**不要向我提问**。

语言约定：对话与文档用中文；代码标识符用英文；代码注释用简洁中文。

## 1. 开工前先做的事

1. 检查我现有的项目（包名/applicationId、Gradle 配置、是否已用 Compose、已有依赖），**沿用现有包名作为根包名**。
2. 先 `git status`；若没有仓库则 `git init`，并把现状提交一次作为基线，之后每个阶段提交一次。
3. 核对并采用以下**最新稳定版**（用官方文档 / Maven / Gradle 插件门户确认，**严禁凭记忆写版本号**，也**不要使用 alpha/beta/RC 依赖**，除非无替代并写 ADR）：Android Gradle Plugin、Gradle、Kotlin、Compose BOM、libGDX（目前 1.14.x）、KTX、kotlinx-coroutines、kotlinx-serialization、AndroidX 相关库、JDK（≥17）。
4. 版本集中放入 `gradle/libs.versions.toml`（version catalog）。

## 2. 技术栈（硬性要求）

- 语言：Kotlin（唯一语言，不写 Java），启用 explicit API mode（对库型模块）与 `allWarningsAsErrors`（可对个别告警豁免并注明原因）。
- **游戏框架：libGDX + KTX**（Android 后端 + LWJGL3 桌面后端）。Kotlin 与 libGDX 的兼容性以 libGDX 官方说明为准，遇到问题写 ADR。
- 外壳 UI：Jetpack Compose（Material 3，仅用于游戏之外的页面：首页、设置、关于、隐私说明等）。
- 游戏内 UI：libGDX Scene2D（用 KTX scene2d DSL），像素风皮肤**在运行时用 Pixmap 生成**（不依赖外部图片）。
- 异步：Kotlin Coroutines（注意线程模型，见 §6.4）。序列化：kotlinx-serialization（JSON）。
- 设置持久化：Jetpack DataStore。存档：自建文件存储（见 §5 Tier 1）。
- 依赖注入：**不使用 Hilt/Dagger**；用轻量的手写 `ServiceContainer`（与 Web 版框架理念一致），并写 ADR 说明理由。
- ECS：**v1 不引入**，采用 Entity + Component 组合；ADR 中说明何时考虑引入 Fleks 等 ECS 库及迁移方式。
- 测试与质量：kotlin-test、JUnit、Turbine、MockK、Konsist（架构规则测试）、detekt、Spotless（ktlint）。
- 除以上外新增依赖必须在 ADR 里写明理由；**不要引入任何广告/统计 SDK**（只留接口）。
- 最低版本：`minSdk` 由你按 Play 覆盖率与 libGDX 要求决定（倾向 26，写 ADR）；`compileSdk` / `targetSdk` = Android 17（API 37）及其最新稳定 SDK 修订，以官方为准。

## 3. Android 17 必须处理的事项（请先阅读官方 developer.android.com/about/versions/17 的 behavior changes 并核对；以官方为准，下列清单是我已知的重点）

1. **大屏强制自适应**：targetSdk≥37 时，sw≥600dp 的设备上 `screenOrientation`、`resizeableActivity`、宽高比限制会被忽略。因此：不得依赖方向锁定；游戏必须能在**旋转、折叠/展开、分屏、自由窗口缩放**下正确重算视口，不重启、不拉伸、不崩溃（处理好 libGDX 的 `resize()`、`configChanges`、`androidx.window` 窗口信息）。
2. **按设备 RAM 的应用内存上限**：建立纹理/音频的**内存预算机制**；分包懒加载资源；实现 `onTrimMemory` 响应与低内存降级（释放可再生缓存、降低画质档位）；禁止一次性加载全部资源。
3. **16 KB 内存页大小**：所有 native 库（libGDX 的 .so 等）必须 16 KB 对齐；在构建检查中验证，并记录到 `docs/`。
4. **反射修改 `static final` 被封锁**：不使用此类 hack；检查所选依赖是否兼容。
5. **后台音频/媒体限制**：游戏音频接入音频焦点；应用进入后台自动暂停并正确恢复；不做后台常驻播放。
6. 其余：Edge-to-edge 与系统栏沉浸式、刘海/挖孔（display cutout）、**预测性返回手势**（`OnBackPressedCallback`）、最小权限原则（**默认不申请 INTERNET**，需要时再加并写 ADR）。
7. Google Play 的 target API 政策（当前要求与未来节点）请查官方并写入 `docs/PLAY_POLICY.md`，注明查询日期。
8. 注意：Robolectric 等测试库对 API 37 的支持可能滞后，**不要把关键测试建立在其上**；核心逻辑放纯 JVM 模块测试。

## 4. Gradle 模块与依赖规则（最重要，用 Gradle 模块边界 + Konsist 强制执行）

```
:app               # Android 应用壳：Application、Manifest、Compose 外壳（Home/Settings/About）、GameActivity 启动、依赖组装
:engine            # libGDX 层：唯一允许依赖 libGDX 的业务模块
:platform-android  # Android 专属适配（实现 core 中定义的 ports）：存储、触觉、窗口信息、热状态、电量、GameMode、Play 服务占位
:domain            # 纯 Kotlin/JVM：游戏规则与数据模型（零 Android、零 libGDX）
:core              # 纯 Kotlin/JVM：ServiceContainer、类型化 EventBus、Logger 接口、Result/错误类型、Lifecycle，以及 ports（接口）
:content           # 纯 Kotlin/JVM：内容数据 schema、registry、校验、本地化文本
:desktop           # JVM 桌面启动器（LWJGL3）：不开模拟器也能运行/调试/截图，并提供 ports 的桌面实现
:testing           # 测试夹具与假实现（FakeClock、FakeStorage、FakeInput）
build-logic/       # convention plugins（统一 Kotlin/Android/detekt/spotless 配置）
```

**依赖规则：**
1. 依赖方向：`app → engine, platform-android, domain, content, core`；`engine → domain, content, core`；`platform-android → core, domain`；`desktop → engine, domain, content, core`；`domain → core`；`content → core, domain`；`core` 不依赖任何项目模块。
2. `core / domain / content` 是**纯 Kotlin/JVM 模块**（`kotlin("jvm")`），从构建层面就无法引用 Android 与 libGDX。
3. **ports（接口）定义在 `:core`**，实现放在 `:platform-android` 与 `:desktop`（六边形架构）。例如 `StoragePort`、`ClockPort`、`HapticsPort`、`SystemInfoPort`、`PlayServicesPort`、`BillingPort`、`AdsPort`、`AnalyticsPort`。
4. `engine` 不得依赖 Compose / AndroidX UI；`app` 的 Compose 代码不得直接操作 libGDX 对象，只能通过 §6.4 的桥接接口。
5. 每个模块通过明确的公共 API 包暴露内容，内部实现用 `internal`；禁止跨模块访问对方内部类。
6. 用 Konsist 写架构测试：包依赖、命名约定、`internal` 使用、禁止 `domain` 出现 `android.*`/`com.badlogic.*` 等；**违规时 `./gradlew check` 必须失败**。

## 5. 模块清单与实现深度（分三级，严格按级别执行，不要越级膨胀）

每个模块都要有 `README.md`（职责、公共 API、扩展点、与其他模块的关系、当前状态）和测试目录。状态标记：✅ 可用并有测试 / 🟡 最小可运行内核 / ⚪ 仅骨架。

### Tier 1 —— 必须真正可运行（✅）
- **core**：ServiceContainer（类型化注册/获取、销毁顺序）、类型化 EventBus、Logger、Lifecycle/Disposable、Result/错误类型、ports 接口全集
- **engine/boot**：`GameApp : KtxGame`，Screen 管理器（含淡入淡出转场）、`BaseScreen`（统一注入 services、自动清理订阅与资源、统一生命周期钩子）；屏幕流 `BootScreen → PreloadScreen → SandboxScreen`
- **engine/rendering**：**像素完美渲染**：基准分辨率（默认 480×270，集中配置）、整数倍缩放 Viewport（含黑边/填充策略可配置）、`TextureFilter.Nearest`、无 mipmap、相机坐标吸附到像素网格；`resize` 全流程可靠
- **engine/assets**：基于 `AssetManager` 的封装：资源清单、分包（`core` / `ui` / 按场景懒加载）、加载进度、**内存预算与卸载策略**、资源缺失时优雅降级并打日志；纹理图集打包流程（libGDX TexturePacker 的 Gradle 任务，输入 `assets-src/`，输出 `assets/`）
- **engine/input**：动作映射（action map，不直接读键码）、键盘 + 手柄 + 触屏虚拟摇杆/按键（基础）、可改键数据结构、输入设备热插拔
- **engine/audio**：BGM / SFX 分总线、音量、淡入淡出/交叉淡入、**音频焦点**、进后台暂停/恢复
- **engine/ui（Scene2D）**：像素风皮肤（运行时生成）、Button/Panel/Dialog/ProgressBar/Toast 基础控件、界面栈（MenuStack）、手柄/方向键焦点导航基础
- **engine/text**：CJK 像素字体方案：使用 gdx-freetype 加载开源像素字体（如方舟像素 / 缝合像素，需自行确认许可证，记录到 `docs/CREDITS.md`），按需生成字形并缓存，避免一次生成海量字形；字体缺失时回退系统字体并告警
- **engine/debug**：仅 debug 构建启用的调试覆盖层（FPS、帧时间、当前 Screen、事件日志、内存/纹理统计、flags 查看、时间倍率），三指点按或桌面 F2 开关；release 构建必须完全剔除
- **platform-android**：窗口信息（`androidx.window`，折叠状态/窗口尺寸）、热状态与电量监听（→ 自动降低画质档位的基础机制）、触觉反馈、GameMode 读取、`onTrimMemory` 转发、Edge-to-edge 与沉浸式、预测性返回
- **settings**：音量、语言、显示（整数缩放/画质档位/帧率偏好）、键位、无障碍（减弱震屏等）；DataStore 持久化，**与存档相互独立**
- **save**：存档信封 `{meta:{version, savedAt, playtimeMs, slot, appVersion}, state}`；各模块通过 `Saveable` 接口（`saveKey`/`serialize`/`deserialize`/`version`/`migrate`）贡献数据；多槽位、自动存档、版本迁移链、**原子写入（临时文件 + 重命名）与损坏数据容错**、通过 SAF 导入/导出；存储适配器接口（先实现内部存储文件版）；云存档只留接口。**需要完整单元测试**。同时配置 `backup_rules` / `data_extraction_rules` 并写 ADR 说明存档是否参与系统备份
- **i18n**：外壳用 Android 资源（含按应用语言 `LocaleManager` 支持）；游戏内文本用 `content` 里的本地化表；默认 zh-CN，附带 en；文本 key 集中管理
- **desktop 启动器**：在 PC 上运行同一套 `engine`，用于快速迭代与自动截图
- **构建与发布基础**：debug/release 变体、R8 规则（含 libGDX/反射相关 keep）、AAB 产物、签名配置通过 `local.properties` / 环境变量注入（**仓库里绝不放密钥**）、16 KB 对齐检查任务、`.editorconfig`、`.gitignore`

### Tier 2 —— 纯逻辑最小内核 + 单元测试（🟡，全部在 `:domain`）
- **timeline（时间线/过场序列）**：数据模型（轨道、片段/关键帧、标记、并行/串行/等待/循环）、`TimelineRunner`（由外部 delta 驱动，支持暂停/跳转/倍速/跳过/序列化）、**动作注册表**（`registerAction(type, handler)`，数据与表现解耦）；`engine/systems/timeline` 放 libGDX 侧动作处理器骨架（相机、补间、动画、音频、对话、设置 flag 等）
- **dialogue**：对话图（节点、选项、条件、变量、说话人、立绘引用、本地化 key）+ `DialogueRunner`
- **flags / variables**：全局/场景/存档作用域，支持变更订阅
- **events & triggers**：条件表达式（可序列化数据，禁止 eval）、触发器、区域触发数据模型
- **quest**、**inventory**（定义/实例分离、容器、堆叠、装备槽）、**stats**（属性、修饰器栈、Buff 持续时间）
- **calendar（游戏内时间）**：游戏时钟、昼夜、日/周/季、时间缩放；**与 timeline 是两个不同概念**
- **fsm**（含分层）、**scheduler**（受游戏时间缩放、可暂停）、**rng**（可播种 + 权重/掉落表）、**pool**
- **content registry**：`id → 定义` 注册表，kotlinx-serialization 解析 + 校验（id 唯一、跨表引用有效），提供 Gradle 任务 `validateContent`，并纳入 `check`

### Tier 3 —— 仅骨架（⚪：README + 接口/类型 + 空实现，保证编译与 detekt 通过）
combat、ai（行为树/效用 AI）、crafting、shop/economy、relationship、progression、loot、spawner、interaction、achievements、codex/journal、tutorial/hints、tilemap 加载（Tiled TMX 约定）、pathfinding（网格 A*）、camera 控制器、fx 预设（震屏/闪白/粒子）、weather、lighting、day-night 叠加、shader 管理、**Play Games 服务（成就/排行榜/云存档）**、**Play Billing**、**广告**、**Play Integrity**、**Play Asset Delivery（资产包：install-time / fast-follow / on-demand）**、分析与崩溃上报适配器、本地通知、触屏控制自定义布局、ECS 适配层（仅评估文档）、`engine/ui` 下的 HUD/DialogueBox/InventoryPanel/SaveSlots/PauseMenu（空壳容器）。

> 本清单之外但明显必要的基础模块，可以补充，并在 `docs/MODULES.md` 与 ADR 中说明。

## 6. 首页与空壳场景（本阶段唯一的"可见内容"）

1. **Compose 首页（Home）**：像素风排版，居中展示项目名（默认 `pixel-game-framework`，集中配置可改）、一段介绍（说明这是一个**基于 libGDX 的 Android 2D 复古像素游戏框架，目前还是一个空壳**，并列出技术栈与未来可扩展模块）、**实时读取并显示 libGDX / Kotlin / compileSdk / targetSdk 的实际版本**、语言切换（zh-CN / en）、「进入空场景」按钮。
2. **游戏 Activity**：独立的 `GameActivity`（libGDX `AndroidApplication`），沉浸式全屏，由 Compose 外壳启动；如你评估后认为"Fragment 嵌入 Compose"更优，可偏离，但必须写 ADR。
3. **SandboxScreen**：只显示一行像素风文字（如"libGDX 空壳"）和一个**运行时用 Pixmap 生成的占位精灵**（不依赖外部图片），用来肉眼验证像素渲染不模糊、旋转/缩放窗口后依然正确。
4. **外壳 ↔ 游戏的桥接（`:core` 中定义接口，`:app` 与 `:engine` 实现）**：UI 线程（Compose）与 libGDX 渲染线程是**不同线程**，所有跨线程调用必须经线程安全队列（如 `Channel`/`Gdx.app.postRunnable`），**严禁在渲染线程直接改 Compose 状态、也严禁在 UI 线程直接操作 libGDX 对象**。演示：Compose 层（或游戏内 Scene2D 按钮）发出命令让占位精灵变化，游戏回传事件让界面显示计数，证明双向通道可用。
5. 生命周期：正确处理 `pause/resume/dispose`、Activity 重建、旋转/窗口变化时不泄漏 GL 资源、不出现黑屏。

## 7. 文档（给未来的 AI 和我自己看，质量与代码同等重要）

- `AGENTS.md`（`CLAUDE.md` 内容仅写 `@AGENTS.md`），必须包含：
  1. 项目简介、技术栈、常用 Gradle 命令；
  2. 模块与依赖规则精炼版；
  3. **「需求 → 落脚点」对照表**：至少 30 类常见需求（过场/时间线、对话、任务、背包、存档、战斗、昼夜/日历、新增 Screen、新增 UI 面板、新增音效、新增本地化文本、新增物品/敌人数据、改键、调试工具、接入 Play 服务、内购、资源分包…）分别改哪些模块/文件；
  4. **术语表**：区分易混概念（timeline ≠ calendar ≠ 事件日志）；需求含糊时先对照术语表并写明理解；
  5. 编码规范：不使用 `!!`、不滥用 `lateinit`、domain 中禁止可变全局状态；单文件尽量不超过 300 行；魔法数字进配置；用户可见文本走 i18n；每个新模块必须带 README 与测试；
  6. **工作流**：先读 `docs/ARCHITECTURE.md` 与相关模块 README → 给出计划 → 小步实现 → 跑 `./gradlew check` → 更新模块 README 与 `docs/MODULES.md` 状态 → 必要时补 ADR。
- `docs/ARCHITECTURE.md`：模块图（Mermaid）、数据流、线程模型、Screen 流、存档流程、内容加载流程、生命周期处理。
- `docs/MODULES.md`、`docs/CONVENTIONS.md`、`docs/REQUEST_TEMPLATE.md`（我给 AI 提需求的模板）、`docs/ROADMAP.md`、`docs/CREDITS.md`、`docs/PLAY_POLICY.md`、`docs/ANDROID17.md`（§3 每项的落实方式与验证方法）。
- `docs/ADR/`：至少覆盖：选择 libGDX 的理由与备选对比、Kotlin/AGP/libGDX 版本与兼容结论、模块划分与 ports、Compose 外壳 + libGDX 游戏 Activity 的决策、线程模型、存档与备份方案、不用 Hilt、不引入 ECS、像素渲染与分辨率策略、字体方案、minSdk 选择。

## 8. 阶段与工作方式

每阶段结束都要跑通验证并 `git commit`：
- **阶段 A｜调研与脚手架**：核对版本与 Android 17 官方说明；建立多模块结构、version catalog、convention plugins、质量工具；ADR-001/002。
- **阶段 B｜Tier 1**：§5 Tier 1 + §6 的首页、游戏 Activity、桥接演示、桌面启动器。
- **阶段 C｜Tier 2**：纯逻辑内核 + 单元测试。
- **阶段 D｜Tier 3 骨架与文档**、CI（`.github/workflows`：assemble → lint → detekt → spotless → test → validateContent）。
- **阶段 E｜总验收**。

遇到不确定的 libGDX / Android 17 API：先查官方文档与源码；查不到就做适配层并在 ADR 标注风险，**不要猜，不要编造不存在的库或 API**。不要写游戏内容，不要引入有版权风险的素材。

## 9. 验收标准（全部满足才算完成）

1. `./gradlew assembleDebug` 成功，应用可在 **API 37 模拟器**与真机上运行；首页正常，进入游戏后像素场景清晰不模糊。
2. **大屏自适应**：在可调整大小的大屏/平板模拟器上旋转、分屏、拖动窗口缩放，游戏视口始终正确、不重启不崩溃。
3. 外壳 ↔ 游戏双向通信演示可用；反复进入/退出游戏、旋转 20 次后无明显内存增长或 GL 资源泄漏。
4. `./gradlew check`（含 detekt、spotless、Konsist 架构测试、单元测试、`validateContent`）全部通过；故意制造一次违反模块边界的引用，必须构建失败（验证后还原）。
5. 存档模块单测覆盖：多槽位、版本迁移、损坏数据容错、原子写入、导入/导出。
6. `assembleRelease`（开启 R8）产物可正常启动；**16 KB 对齐检查通过**；release 中不包含调试覆盖层代码。
7. `:desktop` 可在 PC 上运行同一场景。
8. §5 每个模块都有 README，`docs/MODULES.md` 状态与实际一致。

## 10. 最终报告（完成后输出）

用中文列出：实际采用的各依赖版本；关键决策与偏离本提示词之处及原因；每个模块的最终状态（✅🟡⚪）；Android 17 各项的落实与验证结果；已知风险与待办；如何运行、测试、构建；建议我下一步让 AI 做的 3 个需求。
