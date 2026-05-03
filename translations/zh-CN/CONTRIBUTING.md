# 为 gstack 做贡献

感谢你想让 gstack 变得更好。无论你是修复技能提示中的拼写错误,还是构建全新的工作流,本指南都能让你快速上手。

## 快速开始

gstack 技能是 Markdown 文件,Claude Code 从 `skills/` 目录中发现它们。通常它们位于 `~/.claude/skills/gstack/`(你的全局安装)。但当你开发 gstack 本身时,你希望 Claude Code 使用*工作树中*的技能——这样编辑立即生效,无需复制或部署任何东西。

这就是开发模式的作用。它将你的仓库符号链接到本地 `.claude/skills/` 目录,这样 Claude Code 直接从你的检出版本读取技能。

```bash
git clone https://github.com/garrytan/gstack.git && cd gstack
bun install                    # 安装依赖
bin/dev-setup                  # 激活开发模式
```

> **完整克隆 vs 浅克隆。** README 面向用户的安装使用 `--depth 1` 以提高速度。作为贡献者,使用完整克隆(不带 `--depth` 参数)——你需要历史记录来使用 `git log`、`git blame`、`git bisect` 以及针对早期版本审查 PR。如果你已经按照 README 进行了 `--depth 1` 克隆,可以使用 `git fetch --unshallow` 将其提升为完整克隆。

现在编辑任何 `SKILL.md`,在 Claude Code 中调用它(例如 `/review`),就能看到你的更改生效。完成开发后:

```bash
bin/dev-teardown               # 停用——回到全局安装
```

## 操作性自我改进

gstack 自动从失败中学习。在每个技能会话结束时,代理会反思出了什么问题(CLI 错误、错误的方法、项目特性),并将操作性学习记录到 `~/.gstack/projects/{slug}/learnings.jsonl`。未来的会话会自动呈现这些学习内容,因此 gstack 会随着时间推移在你的代码库上变得更智能。

无需设置。学习内容会自动记录。使用 `/learn` 查看它们。

### 贡献者工作流

1. **正常使用 gstack**——操作性学习会自动捕获
2. **检查你的学习内容:** `/learn` 或 `ls ~/.gstack/projects/*/learnings.jsonl`
3. **Fork 并克隆 gstack**(如果你还没有这样做)
4. **将你的 fork 符号链接到遇到 bug 的项目中:**
   ```bash
   # 在你的核心项目中(遇到 gstack 问题的那个)
   ln -sfn /path/to/your/gstack-fork .claude/skills/gstack
   cd .claude/skills/gstack && bun install && bun run build && ./setup
   ```
   Setup 创建包含 SKILL.md 符号链接的每个技能目录(`qa/SKILL.md -> gstack/qa/SKILL.md`)
   并询问你的前缀偏好。传递 `--no-prefix` 跳过提示并使用短名称。
5. **修复问题**——你的更改在此项目中立即生效
6. **通过实际使用 gstack 进行测试**——做让你烦恼的事情,验证它已修复
7. **从你的 fork 开启 PR**

这是最好的贡献方式:在做实际工作时修复 gstack,在你真正感受到痛苦的项目中。

### 会话感知

当你同时打开 3 个以上的 gstack 会话时,每个问题都会告诉你是哪个项目、哪个分支以及正在发生什么。不再盯着问题想"等等,这是哪个窗口?"所有技能的格式都是一致的。

## 在 gstack 仓库内开发 gstack

当你编辑 gstack 技能并想通过在同一仓库中实际使用 gstack 来测试它们时,`bin/dev-setup` 会进行配置。它创建指向你工作树的 `.claude/skills/` 符号链接(被 gitignore),这样 Claude Code 使用你的本地编辑而不是全局安装。

```
gstack/                          <- 你的工作树
├── .claude/skills/              <- 由 dev-setup 创建(被 gitignore)
│   ├── gstack -> ../../         <- 指向仓库根目录的符号链接
│   ├── review/                  <- 真实目录(短名称,默认)
│   │   └── SKILL.md -> gstack/review/SKILL.md
│   ├── ship/                    <- 或 gstack-review/、gstack-ship/ 如果使用 --prefix
│   │   └── SKILL.md -> gstack/ship/SKILL.md
│   └── ...                      <- 每个技能一个目录
├── review/
│   └── SKILL.md                 <- 编辑这个,用 /review 测试
├── ship/
│   └── SKILL.md
├── browse/
│   ├── src/                     <- TypeScript 源代码
│   └── dist/                    <- 编译的二进制文件(被 gitignore)
└── ...
```

Setup 在顶层创建真实目录(不是符号链接),内部有 SKILL.md 符号链接。这确保 Claude 将它们发现为顶层技能,而不是嵌套在 `gstack/` 下。名称取决于你的前缀设置(`~/.gstack/config.yaml`)。短名称(`/review`、`/ship`)是默认的。如果你更喜欢命名空间名称(`/gstack-review`、`/gstack-ship`),运行 `./setup --prefix`。

## 日常工作流

```bash
# 1. 进入开发模式
bin/dev-setup

# 2. 编辑技能
vim review/SKILL.md

# 3. 在 Claude Code 中测试——更改立即生效
#    > /review

# 4. 编辑 browse 源代码?重新构建二进制文件
bun run build

# 5. 今天完成了?拆除
bin/dev-teardown
```

## 测试与评估

### 设置

```bash
# 1. 复制 .env.example 并添加你的 API 密钥
cp .env.example .env
# 编辑 .env → 设置 ANTHROPIC_API_KEY=sk-ant-...

# 2. 安装依赖(如果你还没有)
bun install
```

Bun 自动加载 `.env`——无需额外配置。Conductor 工作区自动从主工作树继承 `.env`(参见下面的"Conductor 工作区")。

### 测试层级

| 层级 | 命令 | 成本 | 测试内容 |
|------|---------|------|---------------|
| 1 — 静态 | `bun test` | 免费 | 命令验证、快照标志、SKILL.md 正确性、TODOS-format.md 引用、可观测性单元测试 |
| 2 — E2E | `bun run test:e2e` | ~$3.85 | 通过 `claude -p` 子进程完整执行技能 |
| 3 — LLM 评估 | `bun run test:evals` | ~$0.15 独立 | 对生成的 SKILL.md 文档进行 LLM-as-judge 评分 |
| 2+3 | `bun run test:evals` | ~$4 组合 | E2E + LLM-as-judge(两者都运行) |

```bash
bun test                     # 仅层级 1(每次提交运行,<5秒)
bun run test:e2e             # 层级 2:仅 E2E(需要 EVALS=1,不能在 Claude Code 内运行)
bun run test:evals           # 层级 2 + 3 组合(~$4/次运行)
```

### 层级 1:静态验证(免费)

使用 `bun test` 自动运行。不需要 API 密钥。

- **技能解析器测试**(`test/skill-parser.test.ts`)——从 SKILL.md bash 代码块中提取每个 `$B` 命令,并针对 `browse/src/commands.ts` 中的命令注册表进行验证。捕获拼写错误、已删除的命令和无效的快照标志。
- **技能验证测试**(`test/skill-validation.test.ts`)——验证 SKILL.md 文件仅引用真实的命令和标志,并且命令描述符合质量阈值。
- **生成器测试**(`test/gen-skill-docs.test.ts`)——测试模板系统:验证占位符正确解析,输出包含标志的值提示(例如 `-d <N>` 而不仅仅是 `-d`),关键命令的丰富描述(例如 `is` 列出有效状态,`press` 列出按键示例)。

### 层级 2:通过 `claude -p` 进行 E2E(~$3.85/次运行)

将 `claude -p` 作为子进程生成,使用 `--output-format stream-json --verbose`,流式传输 NDJSON 以实现实时进度,并扫描 browse 错误。这是最接近"这个技能实际上端到端工作吗?"的测试。

```bash
# 必须从普通终端运行——不能嵌套在 Claude Code 或 Conductor 内
EVALS=1 bun test test/skill-e2e-*.test.ts
```

- 由 `EVALS=1` 环境变量控制(防止意外的昂贵运行)
- 如果在 Claude Code 内运行则自动跳过(`claude -p` 不能嵌套)
- API 连接预检——在消耗预算之前快速失败于 ConnectionRefused
- 实时进度到 stderr:`[Ns] turn T tool #C: Name(...)`
- 保存完整的 NDJSON 记录和失败 JSON 以供调试
- 测试位于 `test/skill-e2e-*.test.ts`(按类别拆分),运行器逻辑在 `test/helpers/session-runner.ts`

### E2E 可观测性

当 E2E 测试运行时,它们在 `~/.gstack-dev/` 中生成机器可读的工件:

| 工件 | 路径 | 目的 |
|----------|------|---------|
| 心跳 | `e2e-live.json` | 当前测试状态(每次工具调用更新) |
| 部分结果 | `evals/_partial-e2e.json` | 已完成的测试(在终止后保留) |
| 进度日志 | `e2e-runs/{runId}/progress.log` | 仅追加的文本日志 |
| NDJSON 记录 | `e2e-runs/{runId}/{test}.ndjson` | 每个测试的原始 `claude -p` 输出 |
| 失败 JSON | `e2e-runs/{runId}/{test}-failure.json` | 失败时的诊断数据 |

**实时仪表板:** 在第二个终端中运行 `bun run eval:watch` 以查看显示已完成测试、当前运行测试和成本的实时仪表板。使用 `--tail` 还可以显示 progress.log 的最后 10 行。

**评估历史工具:**

```bash
bun run eval:list            # 列出所有评估运行(轮次、持续时间、每次运行的成本)
bun run eval:compare         # 比较两次运行——显示每个测试的增量 + Takeaway 评论
bun run eval:summary         # 跨运行的聚合统计 + 每个测试的效率平均值
```

**评估比较评论:** `eval:compare` 生成自然语言 Takeaway 部分,解释运行之间的变化——标记回归、注意改进、指出效率提升(更少的轮次、更快、更便宜),并生成总体摘要。这由 `eval-store.ts` 中的 `generateCommentary()` 驱动。

工件永远不会被清理——它们在 `~/.gstack-dev/` 中累积,用于事后调试和趋势分析。

### 层级 3:LLM-as-judge(~$0.15/次运行)

使用 Claude Sonnet 在三个维度上对生成的 SKILL.md 文档进行评分:

- **清晰度**——AI 代理能否无歧义地理解指令?
- **完整性**——是否记录了所有命令、标志和使用模式?
- **可操作性**——代理能否仅使用文档中的信息执行任务?

每个维度评分 1-5。阈值:每个维度必须评分**≥ 4**。还有一个回归测试,将生成的文档与 `origin/main` 的手动维护基线进行比较——生成的文档必须评分相等或更高。

```bash
# 需要 .env 中的 ANTHROPIC_API_KEY——包含在 bun run test:evals 中
```

- 使用 `claude-sonnet-4-6` 以保证评分稳定性
- 测试位于 `test/skill-llm-eval.test.ts`
- 直接调用 Anthropic API(不是 `claude -p`),因此可以从任何地方工作,包括在 Claude Code 内

### CI

GitHub Action(`.github/workflows/skill-docs.yml`)在每次推送和 PR 时运行 `bun run gen:skill-docs --dry-run`。如果生成的 SKILL.md 文件与提交的内容不同,CI 失败。这在合并之前捕获过时的文档。

测试直接针对 browse 二进制文件运行——它们不需要开发模式。

## 编辑 SKILL.md 文件

SKILL.md 文件是从 `.tmpl` 模板**生成**的。不要直接编辑 `.md`——你的更改将在下次构建时被覆盖。

```bash
# 1. 编辑模板
vim SKILL.md.tmpl              # 或 browse/SKILL.md.tmpl

# 2. 为所有主机重新生成
bun run gen:skill-docs --host all

# 3. 检查健康状况(报告所有主机)
bun run skill:check

# 或使用监视模式——保存时自动重新生成
bun run dev:skill
```

有关模板编写最佳实践(自然语言优于 bash 风格、动态分支检测、`{{BASE_BRANCH_DETECT}}` 使用),请参阅 CLAUDE.md 的"编写 SKILL 模板"部分。

要添加 browse 命令,将其添加到 `browse/src/commands.ts`。要添加快照标志,将其添加到 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS`。然后重新构建。

## 术语列表(V1 写作风格)

gstack 的写作风格部分(注入到每个层级≥2 技能的前言中)在每次技能调用时首次使用时对技术术语进行注释。符合注释条件的术语列表位于 `scripts/jargon-list.json`——约 50 个精选的高频术语(幂等、竞态条件、N+1、背压等)。不在列表中的术语被假定为足够通俗易懂的英语。

**添加或删除术语:** 打开 PR 编辑 `scripts/jargon-list.json`。编辑后运行 `bun run gen:skill-docs`——术语在生成时被烘焙到每个生成的 SKILL.md 中,因此更改仅在重新生成后生效。没有运行时加载;没有用户端覆盖。仓库列表是真实来源。

适合添加的候选术语:非技术用户在审查输出中遇到的高频术语,没有上下文(常见的数据库/并发术语、安全术语、前端框架概念)。不要添加仅出现在一两个小众技能中的术语——成本与价值的权衡不值得审查开销。

## 多主机开发

gstack 从一组 `.tmpl` 模板为 8 个主机生成 SKILL.md 文件。每个主机是 `hosts/*.ts` 中的类型化配置。生成器读取这些配置以生成适合主机的输出(不同的前言、路径、工具名称)。

**支持的主机:** Claude(主要)、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw。

### 为所有主机生成

```bash
# 为特定主机生成
bun run gen:skill-docs                    # Claude(默认)
bun run gen:skill-docs --host codex       # Codex
bun run gen:skill-docs --host opencode    # OpenCode
bun run gen:skill-docs --host all         # 所有 8 个主机

# 或使用 build,它会为所有主机生成 + 编译二进制文件
bun run build
```

### 主机之间的变化

每个主机配置(`hosts/*.ts`)控制:

| 方面 | 示例(Claude vs Codex) |
|--------|---------------------------|
| 输出目录 | `{skill}/SKILL.md` vs `.agents/skills/gstack-{skill}/SKILL.md` |
| 前言 | 完整(名称、描述、钩子、版本) vs 最小(名称 + 描述) |
| 路径 | `~/.claude/skills/gstack` vs `$GSTACK_ROOT` |
| 工具名称 | "use the Bash tool" vs 相同(Factory 重写为"run this command") |
| 钩子技能 | `hooks:` 前言 vs 内联安全建议文本 |
| 抑制的部分 | 无 vs Codex 自调用部分被剥离 |

有关完整的 `HostConfig` 接口,请参阅 `scripts/host-config.ts`。

### 测试主机输出

```bash
# 运行所有静态测试(包括所有主机的参数化冒烟测试)
bun test

# 检查所有主机的新鲜度
bun run gen:skill-docs --host all --dry-run

# 健康仪表板涵盖所有主机
bun run skill:check
```

### 添加新主机

有关完整指南,请参阅 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)。简短版本:

1. 创建 `hosts/myhost.ts`(从 `hosts/opencode.ts` 复制)
2. 添加到 `hosts/index.ts`
3. 将 `.myhost/` 添加到 `.gitignore`
4. 运行 `bun run gen:skill-docs --host myhost`
5. 运行 `bun test`(参数化测试自动覆盖它)

不需要更改生成器、设置或工具代码。

### 添加新技能

当你添加新的技能模板时,所有主机都会自动获得它:
1. 创建 `{skill}/SKILL.md.tmpl`
2. 运行 `bun run gen:skill-docs --host all`
3. 动态模板发现会捕获它,无需更新静态列表
4. 提交 `{skill}/SKILL.md`,外部主机输出在设置时生成并被 gitignore

## Conductor 工作区

如果你使用 [Conductor](https://conductor.build) 并行运行多个 Claude Code 会话,`conductor.json` 会自动配置工作区生命周期:

| 钩子 | 脚本 | 作用 |
|------|--------|-------------|
| `setup` | `bin/dev-setup` | 从主工作树复制 `.env`,安装依赖,符号链接技能 |
| `archive` | `bin/dev-teardown` | 删除技能符号链接,清理 `.claude/` 目录 |

当 Conductor 创建新工作区时,`bin/dev-setup` 自动运行。它检测主工作树(通过 `git worktree list`),复制你的 `.env` 以便 API 密钥传递,并设置开发模式——无需手动步骤。

**首次设置:** 在主仓库的 `.env` 中放入你的 `ANTHROPIC_API_KEY`(参见 `.env.example`)。每个 Conductor 工作区都会自动继承它。

## 需要知道的事情

- **SKILL.md 文件是生成的。** 编辑 `.tmpl` 模板,而不是 `.md`。运行 `bun run gen:skill-docs` 重新生成。
- **TODOS.md 是统一的待办事项列表。** 按技能/组件组织,具有 P0-P4 优先级。`/ship` 自动检测已完成的项目。所有计划/审查/回顾技能都读取它以获取上下文。
- **Browse 源代码更改需要重新构建。** 如果你修改 `browse/src/*.ts`,运行 `bun run build`。
- **开发模式会遮蔽你的全局安装。** 项目本地技能优先于 `~/.claude/skills/gstack`。`bin/dev-teardown` 恢复全局安装。
- **Conductor 工作区是独立的。** 每个工作区都是自己的 git 工作树。`bin/dev-setup` 通过 `conductor.json` 自动运行。
- **`.env` 在工作树之间传播。** 在主仓库中设置一次,所有 Conductor 工作区都会获得它。
- **`.claude/skills/` 被 gitignore。** 符号链接永远不会被提交。

## 在真实项目中测试你的更改

**这是开发 gstack 的推荐方式。** 将你的 gstack 检出版本符号链接到你实际使用它的项目中,这样你的更改在你做实际工作时是实时的。

### 步骤 1:符号链接你的检出版本

```bash
# 在你的核心项目中(不是 gstack 仓库)
ln -sfn /path/to/your/gstack-checkout .claude/skills/gstack
```

### 步骤 2:运行 setup 创建每个技能的符号链接

仅有 `gstack` 符号链接是不够的。Claude Code 通过单独的顶层目录(`qa/SKILL.md`、`ship/SKILL.md` 等)发现技能,而不是通过 `gstack/` 目录本身。运行 `./setup` 创建它们:

```bash
cd .claude/skills/gstack && bun install && bun run build && ./setup
```

Setup 会询问你是否想要短名称(`/qa`)或命名空间(`/gstack-qa`)。你的选择保存到 `~/.gstack/config.yaml` 并在未来运行时记住。要跳过提示,传递 `--no-prefix`(短名称)或 `--prefix`(命名空间)。

### 步骤 3:开发

编辑模板,运行 `bun run gen:skill-docs`,下一次 `/review` 或 `/qa` 调用立即捕获它。无需重启。

### 回到稳定的全局安装

删除项目本地符号链接。Claude Code 回退到 `~/.claude/skills/gstack/`:

```bash
rm .claude/skills/gstack
```

每个技能目录(`qa/`、`ship/` 等)包含指向 `gstack/...` 的 SKILL.md 符号链接,因此它们会自动解析到全局安装。

### 切换前缀模式

如果你使用一个前缀设置安装了 gstack 并想切换:

```bash
cd .claude/skills/gstack && ./setup --no-prefix   # 切换到 /qa、/ship
cd .claude/skills/gstack && ./setup --prefix       # 切换到 /gstack-qa、/gstack-ship
```

Setup 自动清理旧的符号链接。无需手动清理。

### 替代方案:将全局安装指向分支

如果你不想要每个项目的符号链接,可以切换全局安装:

```bash
cd ~/.claude/skills/gstack
git fetch origin
git checkout origin/<branch>
bun install && bun run build && ./setup
```

这会影响所有项目。要恢复:`git checkout main && git pull && bun run build && ./setup`。

## 社区 PR 分类(波次流程)

当社区 PR 累积时,将它们批量分组为主题波次:

1. **分类**——按主题分组(安全、功能、基础设施、文档)
2. **去重**——如果两个 PR 修复同一问题,选择更改行数较少的那个。关闭另一个,并附上指向获胜者的注释。
3. **收集器分支**——创建 `pr-wave-N`,合并干净的 PR,解决脏 PR 的冲突,使用 `bun test && bun run build` 验证
4. **带上下文关闭**——每个关闭的 PR 都会收到一条评论,解释原因以及(如果有的话)什么取代了它。贡献者做了真正的工作;用清晰的沟通尊重这一点。
5. **作为一个 PR 发布**——单个 PR 到 main,在合并提交中保留所有归属。包括合并内容和关闭内容的摘要表。

有关第一波的示例,请参阅 [PR #205](../../pull/205)(v0.8.3)。

## 升级迁移

当发布更改磁盘状态(目录结构、配置格式、过时文件)的方式使得仅 `./setup` 无法修复时,添加迁移脚本,以便现有用户获得干净的升级。

### 何时添加迁移

- 更改了技能目录的创建方式(符号链接 vs 真实目录)
- 重命名或移动了 `~/.gstack/config.yaml` 中的配置键
- 需要删除先前版本的孤立文件
- 更改了 `~/.gstack/` 状态文件的格式

不要为以下情况添加迁移:新功能(用户自动获得它们)、新技能(setup 发现它们)或仅代码更改(无磁盘状态)。

### 如何添加

1. 创建 `gstack-upgrade/migrations/v{VERSION}.sh`,其中 `{VERSION}` 与需要修复的发布的 VERSION 文件匹配。
2. 使其可执行:`chmod +x gstack-upgrade/migrations/v{VERSION}.sh`
3. 脚本必须是**幂等的**(可以安全地多次运行)和**非致命的**(失败被记录但不会阻止升级)。
4. 在顶部包含一个注释块,解释发生了什么变化、为什么需要迁移以及哪些用户受到影响。

示例:

```bash
#!/usr/bin/env bash
# Migration: v0.15.2.0 — Fix skill directory structure
# Affected: users who installed with --no-prefix before v0.15.2.0
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")/../.." && pwd)"
"$SCRIPT_DIR/bin/gstack-relink" 2>/dev/null || true
```

### 如何运行

在 `/gstack-upgrade` 期间,在 `./setup` 完成后(步骤 4.75),升级技能扫描 `gstack-upgrade/migrations/` 并运行每个版本