# PromptCenter

我的提示词库。一个提示词一个 md 文件，按用途分类存放，复制全文即可交给 AI 使用。

## 目录

```
PromptCenter/
├─ README.md                    # 本文件
└─ 01-scaffolding/              # 项目脚手架：从 0 搭框架，不写业务内容
   └─ game/
      ├─ android-libgdx-kotlin-2d-game-framework.md
      └─ web-phaser-react-2d-game-framework.md
```

## 收录

| 文件 | 说明 | 技术栈 |
|---|---|---|
| `01-scaffolding/game/android-libgdx-kotlin-2d-game-framework.md` | 在已有 Android 空 Activity 项目上搭大型 2D 游戏客户端框架 | Android 17 / libGDX / KTX / Kotlin / Compose |
| `01-scaffolding/game/web-phaser-react-2d-game-framework.md` | 从 0 搭网页版 2D 复古像素游戏框架 | Phaser 4 / React / TypeScript / Vite |

这两篇是同一套架构方法论（纯净逻辑层 + 端口适配器 + 三级实现深度 + ADR + 需求落脚点表）在两个技术栈上的落地，加新平台时只需替换「平台特有约束」与「跨端通信」两节。

## 分类

目录前缀为编号，便于排序。只建当前有内容的目录，其余等到有提示词时再建。

| 编号 | 目录 | 放什么 |
|---|---|---|
| 01 | `01-scaffolding` | 项目脚手架、骨架搭建 |
| 02 | `02-feature` | 在既有项目上加功能模块 |
| 03 | `03-refactor` | 重构、迁移、代码治理 |
| 04 | `04-debug` | 调试排障、修缺陷 |
| 05 | `05-review` | 代码审查、质量评估 |
| 06 | `06-docs` | 文档、注释、知识梳理 |
| 07 | `07-content` | 文案、剧本、世界观 |
| 08 | `08-analysis` | 调研、选型、方案对比 |
| 09 | `09-ops` | 构建、发布、运维 |

需要时可再加二级目录细分领域（如 `game/`），不预先拆分。

## 新增一个提示词

1. 按用途选分类目录；现有分类不合适就新建 `NN-名称/`，并在上表补一行。
2. 文件名用英文小写连字符，如 `web-phaser-react-2d-game-framework.md`。
3. 文件正文直接是提示词原文，开头用 `# 任务：...` 一行说清产出物。
4. 在上面的「收录」表里加一行。
