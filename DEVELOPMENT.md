# 二次开发与上游同步要求

> 本仓库是基于 [milind-soni/OpenMausBot](https://github.com/milind-soni/OpenMausBot) 的二次开发 fork。
> 目标：**只改样式与插件、尽量不碰核心，同时持续跟进官方更新**。
> 本文档是每次开发前必读的开发要求。任何改动前先对照第 4 节判断自己该落在哪一层。

---

## 1. 目标与原则

- 只改「样式」和「插件」，核心代码（server 运行逻辑、驱动、协议）能不动就不动。
- 持续同步官方最新版（upstream），不长期累积与上游的差异。
- 每一处改动都要满足：**改动面最小、可 rebase、不污染上游文件**。

## 2. 关键事实：这个项目已有的官方扩展点

动手前先看这张表——能用官方入口的，绝不 fork。

| 想改的东西 | 官方入口 | 要不要碰核心 |
|---|---|---|
| 插件 / Skill | `OMB_SKILLS_DIR` 环境变量；`~/.claude/skills`（需开启 `claudeUserSkills`）；skill 作者/库机制 | ✅ 零改动 |
| 自定义模型引擎 | `~/.openmausbot/config.json` 的 `instances` | ✅ 零改动 |
| 自定义 MCP | `config.json` 的 MCP 注册 | ✅ 零改动 |
| 用户数据 / 配置 | 全部在 `~/.openmausbot` 目录，不碰仓库 | ✅ 零改动 |
| 皮肤 / 配色 | **无外置机制**，11 个皮肤硬编码在 `src/lib/skins.ts` + `src/styles.css` + `electron/skin-overlay.cjs` 三处 | ❌ 目前只能改源码 |

**结论**：插件需求全部能落到官方外置入口，零 fork。真正需要动源码的只有「样式（皮肤）」这一块。

## 3. Git 拓扑与同步策略

```
upstream/main   ←  只镜像官方 milind-soni/OpenMausBot，永远 clean
      │
   main         ←  本仓库主线，所有自定义改动都在这条线，基于 upstream rebase
```

- `upstream` remote 指向 `https://github.com/milind-soni/OpenMausBot.git`。
- 同步用 **rebase**，不用 merge：让自定义改动保持线性，官方更新直接「垫底」，冲突集中处理一次。
- 开启 `git rerere`，让 git 记住反复解决的同类冲突，rebase 时自动重放。

## 4. 改动分层（从上到下，能不用就不用）

### L0 — 零改动（优先吃官方）
- 皮肤：先用内置 11 个（midnight / daylight / meadow / pop 等）。
- 插件：走 `OMB_SKILLS_DIR` / `~/.claude/skills`，自定义 skill 放独立目录，**一个字节都不改仓库**。

### L1 — 最小侵入（改样式的主力方案）
- 新皮肤：在 `skins.ts` 的 `SKIN_IDS`、`styles.css`、`skin-overlay.cjs` 三处各加一小段，**各加几行**，冲突面极小。
- 全局样式微调：集中在 `src/styles.css` 末尾的一块「自定义覆盖区」。

### L2 — fork 内新增独立文件
- 自定义 skill / connector 做成独立新文件放进 `skills/`、`connector/`。新文件名官方不会碰，几乎零冲突。

### L3 — 尽量避免的核心改动
- 非改核心不可时，优先**向上游提 PR** 让官方吸收，避免长期维护自己的补丁。

## 5. 同步操作流程（标准步骤）

每次要跟进官方更新时：

```bash
git fetch upstream
git checkout main
git rebase upstream/main          # 有冲突逐个解决，git rerere 会自动重放
pnpm install                      # 依赖可能变了
pnpm typecheck                    # 或 pnpm build
pnpm test                         # 至少跑冒烟
```

解决冲突时优先检查三个「同步热点」文件：
- `src/lib/skins.ts`
- `src/styles.css`
- `electron/skin-overlay.cjs`

## 6. 红线 / 纪律

1. **不改 `pnpm-lock.yaml` 和 `package.json` 依赖版本**，除非真的需要——这是 rebase 冲突重灾区。
2. **皮肤三处文件是同步热点**：改动要小、要集中，rebase 时优先看这三处。
3. **`electron-updater` 会拉到官方更新包**：自己 fork 打包时，要么改 update 源，要么关掉自动更新，否则用户会被官方版本覆盖。
4. **商标**：OpenMausBot / MausBot 名称与吉祥物是 Supamaus 的商标，Apache 2.0 不授权商标，二次分发必须换名。
5. **`enterprise/` 目录**是 open-core：源码可看可自建，但生产环境跑企业功能需要商业 license；删掉该目录即为纯 OSS 版。

## 7. 打包与分发

- 官方已发安装包，本 fork 需自行打包：`pnpm package:win` / `pnpm package:mac` / `pnpm package:linux`。
- 打包前先处理第 6 节的 update 源与商标问题。

## 8. 待办 / 决策点

- [ ] 若皮肤改动会持续膨胀，优先投入：**给官方提「外置皮肤 / CSS 注入」PR**，做成后加皮肤永远零冲突。
- [ ] 确认现有插件需求能否 100% 落到 L0（skill / MCP / 自定义引擎）。
