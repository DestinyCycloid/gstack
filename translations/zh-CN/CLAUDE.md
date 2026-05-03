# gstack 开发

## 命令

```bash
bun install          # 安装依赖
bun test             # 运行免费测试(browse + snapshot + skill 验证)
bun run test:evals   # 运行付费评估: LLM 评判 + E2E (基于 diff,每次运行最多约 $4)
bun run test:evals:all  # 运行所有付费评估,不考虑 diff
bun run test:gate    # 仅运行 gate 级别测试(CI 默认,阻止合并)
bun run test:periodic  # 仅运行 periodic 级别测试(每周 cron / 手动)
bun run test:e2e     # 仅运行 E2E 测试(基于 diff,每次运行最多约 $3.85)
bun run test:e2e:all # 运行所有 E2E 测试,不考虑 diff
bun run eval:select  # 显示基于当前 diff 将运行哪些测试
bun run dev <cmd>    # 以开发模式运行 CLI,例如 bun run dev goto https://example.com
bun run build        # 生成文档 + 编译二进制文件
bun run gen:skill-docs  # 从模板重新生成 SKILL.md 文件
bun run skill:check  # 所有 skill 的健康仪表板
bun run dev:skill    # 监视模式: 变更时自动重新生成 + 验证
bun run eval:list    # 列出 ~/.gstack-dev/evals/ 中的所有评估运行
bun run eval:compare # 比较两次评估运行(自动选择最近的)
bun run eval:summary # 所有评估运行的汇总统计
bun run slop          # 完整的 slop-scan 报告(所有文件)
bun run slop:diff     # 仅在此分支上更改的文件中的 slop 发现
```

`test:evals` 需要 `ANTHROPIC_API_KEY`。Codex E2E 测试(`test/codex-e2e.test.ts`)
使用 Codex 自己的 `~/.codex/` 配置中的认证 — 不需要 `OPENAI_API_KEY` 环境变量。

**密钥在本机上的位置。** Conductor 工作空间不继承用户的交互式 shell 环境,
因此 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY` 不在默认进程环境中。在运行任何
付费评估 / E2E 之前,从 `~/.zshrc` 中获取它们(Garry 将它们保存在那里):

```bash
bash -c '
  eval "$(grep -E "^export (ANTHROPIC_API_KEY|OPENAI_API_KEY)=" ~/.zshrc)"
  export ANTHROPIC_API_KEY OPENAI_API_KEY
  EVALS=1 EVALS_TIER=periodic bun test test/skill-e2e-<whatever>.test.ts
'
```

不要在任何地方回显密钥值(stdout、日志、shell 历史)。grep+eval 模式仅将其保留
在进程环境中。当传递给测试的 Agent SDK 时,不要向 `runAgentSdkTest` 传递
`env: {...}` — 当 env 作为对象提供时,SDK 的认证管道不会以相同方式获取密钥
(已确认的失败模式)。相反,在调用之前环境地修改 `process.env.ANTHROPIC_API_KEY`,
并在 `finally` 中恢复。

E2E 测试实时流式传输进度(通过 `--output-format stream-json --verbose` 逐工具)。
结果持久化到 `~/.gstack-dev/evals/`,并自动与上次运行进行比较。

**基于 diff 的测试选择:** `test:evals` 和 `test:e2e` 根据与基础分支的 `git diff`
自动选择测试。每个测试在 `test/helpers/touchfiles.ts` 中声明其文件依赖关系。
对全局 touchfiles(session-runner、eval-store、touchfiles.ts 本身)的更改会触发
所有测试。使用 `EVALS_ALL=1` 或 `:all` 脚本变体来强制所有测试。运行 `eval:select`
预览将运行哪些测试。

**两层系统:** 测试在 `E2E_TIERS`(在 `test/helpers/touchfiles.ts` 中)中分类为
`gate` 或 `periodic`。CI 仅运行 gate 测试(`EVALS_TIER=gate`);periodic 测试通过
cron 每周运行或手动运行。使用 `EVALS_TIER=gate` 或 `EVALS_TIER=periodic` 进行
过滤。添加新的 E2E 测试时,对它们进行分类:
1. 安全防护或确定性功能测试? -> `gate`
2. 质量基准、Opus 模型测试或非确定性? -> `periodic`
3. 需要外部服务(Codex、Gemini)? -> `periodic`

## 测试

```bash
bun test             # 每次提交前运行 — 免费,<2s
bun run test:evals   # 发布前运行 — 付费,基于 diff(每次运行最多约 $4)
```

`bun test` 运行 skill 验证、gen-skill-docs 质量检查和 browse 集成测试。
`bun run test:evals` 通过 `claude -p` 运行 LLM 评判质量评估和 E2E 测试。
创建 PR 之前两者都必须通过。

## 项目结构

```
gstack/
├── browse/          # 无头浏览器 CLI (Playwright)
│   ├── src/         # CLI + 服务器 + 命令
│   │   ├── commands.ts  # 命令注册表(单一事实来源)
│   │   └── snapshot.ts  # SNAPSHOT_FLAGS 元数据数组
│   ├── test/        # 集成测试 + fixtures
│   └── dist/        # 编译的二进制文件
├── hosts/           # 类型化的主机配置(每个 AI agent 一个)
│   ├── claude.ts    # 主要主机配置
│   ├── codex.ts, factory.ts, kiro.ts  # 现有主机
│   ├── opencode.ts, slate.ts, cursor.ts, openclaw.ts  # IDE 主机
│   ├── hermes.ts, gbrain.ts  # Agent 运行时主机
│   └── index.ts     # 注册表: 导出所有,派生 Host 类型
├── scripts/         # 构建 + DX 工具
│   ├── gen-skill-docs.ts  # 模板 → SKILL.md 生成器(配置驱动)
│   ├── host-config.ts     # HostConfig 接口 + 验证器
│   ├── host-config-export.ts  # 设置脚本的 Shell 桥接
│   ├── host-adapters/     # 主机特定适配器(OpenClaw 工具映射)
│   ├── resolvers/   # 模板解析器模块(preamble、design、review、gbrain 等)
│   ├── skill-check.ts     # 健康仪表板
│   └── dev-skill.ts       # 监视模式
├── test/            # Skill 验证 + 评估测试
│   ├── helpers/     # skill-parser.ts, session-runner.ts, llm-judge.ts, eval-store.ts
│   ├── fixtures/    # 真实 JSON、植入错误 fixtures、评估基线
│   ├── skill-validation.test.ts  # 第 1 层: 静态验证(免费,<1s)
│   ├── gen-skill-docs.test.ts    # 第 1 层: 生成器质量(免费,<1s)
│   ├── skill-llm-eval.test.ts   # 第 3 层: LLM 作为评判(每次运行约 $0.15)
│   └── skill-e2e-*.test.ts       # 第 2 层: 通过 claude -p 的 E2E(每次运行约 $3.85,按类别拆分)
├── qa-only/         # /qa-only skill (仅报告 QA,不修复)
├── plan-design-review/  # /plan-design-review skill (仅报告设计审计)
├── design-review/    # /design-review skill (设计审计 + 修复循环)
├── ship/            # Ship 工作流 skill
├── review/          # PR 审查 skill
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── autoplan/        # /autoplan skill (自动审查管道: CEO → design → eng)
├── benchmark/       # /benchmark skill (性能回归检测)
├── canary/          # /canary skill (部署后监控循环)
├── codex/           # /codex skill (通过 OpenAI Codex CLI 的多 AI 第二意见)
├── land-and-deploy/ # /land-and-deploy skill (merge → deploy → canary verify)
├── office-hours/    # /office-hours skill (YC Office Hours — 创业诊断 + 构建者头脑风暴)
├── investigate/     # /investigate skill (系统化根本原因调试)
├── retro/           # Retrospective skill (包括 /retro 全局跨项目模式)
├── bin/             # CLI 实用程序(gstack-repo-mode、gstack-slug、gstack-config 等)
├── document-release/ # /document-release skill (发布后文档更新)
├── cso/             # /cso skill (OWASP Top 10 + STRIDE 安全审计)
├── design-consultation/ # /design-consultation skill (从头开始的设计系统)
├── design-shotgun/  # /design-shotgun skill (视觉设计探索)
├── open-gstack-browser/  # /open-gstack-browser skill (启动 GStack Browser)
├── connect-chrome/  # symlink → open-gstack-browser (向后兼容)
├── design/          # Design 二进制 CLI (GPT Image API)
│   ├── src/         # CLI + 命令(generate、variants、compare、serve 等)
│   ├── test/        # 集成测试
│   └── dist/        # 编译的二进制文件
├── extension/       # Chrome 扩展(侧面板 + 活动源 + CSS 检查器)
├── lib/             # 共享库(worktree.ts)
├── docs/designs/    # 设计文档
├── setup-deploy/    # /setup-deploy skill (一次性部署配置)
├── .github/         # CI 工作流 + Docker 镜像
│   ├── workflows/   # evals.yml (Ubicloud 上的 E2E)、skill-docs.yml、actionlint.yml
│   └── docker/      # Dockerfile.ci (预构建工具链 + Playwright/Chromium)
├── contrib/         # 仅贡献者工具(从不为用户安装)
│   └── add-host/    # /gstack-contrib-add-host skill
├── setup            # 一次性设置: 构建二进制文件 + 符号链接 skills
├── SKILL.md         # 从 SKILL.md.tmpl 生成(不要直接编辑)
├── SKILL.md.tmpl    # 模板: 编辑此文件,运行 gen:skill-docs
├── ETHOS.md         # 构建者哲学(Boil the Lake、Search Before Building)
└── package.json     # browse 的构建脚本
```

## SKILL.md 工作流

SKILL.md 文件是从 `.tmpl` 模板**生成**的。要更新文档:

1. 编辑 `.tmpl` 文件(例如 `SKILL.md.tmpl` 或 `browse/SKILL.md.tmpl`)
2. 运行 `bun run gen:skill-docs`(或自动执行此操作的 `bun run build`)
3. 提交 `.tmpl` 和生成的 `.md` 文件

要添加新的 browse 命令: 将其添加到 `browse/src/commands.ts` 并重新构建。
要添加 snapshot 标志: 将其添加到 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS` 并重新构建。

**Token 上限:** 生成的 SKILL.md 文件在超过 160KB(约 40K tokens)时会触发警告。
这是一个"注意功能膨胀"的防护措施,而不是硬性限制。现代旗舰模型具有 200K-1M
上下文窗口,因此 40K 是窗口的 4-20%,并且提示缓存使较大 skills 的边际成本很小。
上限的存在是为了捕获失控的 preamble/resolver 增长,而不是强制压缩精心调整的
大型 skills(`ship`、`plan-ceo-review`、`office-hours` 合理地包含 25-35K tokens
的行为)。如果你超过 40K,正确的修复通常是:(1)查看什么增长了,(2)如果一个
resolver 在单个 PR 中添加了 10K+,质疑它是否属于内联或作为参考文档,(3)仅作为
最后手段压缩精心调整的散文 — 对覆盖审计、审查军队或语音指令的削减有真正的
质量成本。

**SKILL.md 文件上的合并冲突:** 永远不要通过接受任一方来解决生成的 SKILL.md
文件上的冲突。相反:(1)解决 `.tmpl` 模板和 `scripts/gen-skill-docs.ts`(事实
来源)上的冲突,(2)运行 `bun run gen:skill-docs` 重新生成所有 SKILL.md 文件,
(3)暂存重新生成的文件。接受一方的生成输出会默默地丢弃另一方的模板更改。

## 平台无关设计

Skills 绝不能硬编码框架特定的命令、文件模式或目录结构。相反:

1. **读取 CLAUDE.md** 获取项目特定配置(测试命令、评估命令等)
2. **如果缺失,AskUserQuestion** — 让用户告诉你或让 gstack 搜索仓库
3. **将答案持久化到 CLAUDE.md**,这样我们就不必再问了

这适用于测试命令、评估命令、部署命令和任何其他项目特定行为。项目拥有其配置;
gstack 读取它。

## 编写 SKILL 模板

SKILL.md.tmpl 文件是 **Claude 读取的提示模板**,而不是 bash 脚本。每个 bash
代码块在单独的 shell 中运行 — 变量不会在块之间持久化。

规则:
- **使用自然语言表达逻辑和状态。** 不要使用 shell 变量在代码块之间传递状态。
  相反,告诉 Claude 要记住什么并在散文中引用它(例如,"步骤 0 中检测到的基础分支")。
- **不要硬编码分支名称。** 通过 `gh pr view` 或 `gh repo view` 动态检测
  `main`/`master`/等。对于 PR 目标 skills 使用 `{{BASE_BRANCH_DETECT}}`。
  在散文中使用"基础分支",在代码块占位符中使用 `<base>`。
- **保持 bash 块自包含。** 每个代码块应独立工作。如果一个块需要前一步的上下文,
  在上面的散文中重新陈述它。
- **将条件表达为英语。** 不要在 bash 中使用嵌套的 `if/elif/else`,而是编写
  编号的决策步骤:"1. 如果 X,执行 Y。2. 否则,执行 Z。"

## 写作风格 (V1)

每个 tier-≥2 skill 的默认输出遵循 `scripts/resolvers/preamble.ts` 中的写作风格
部分:首次使用时解释术语(在 `scripts/jargon-list.json` 中的精选列表,在
gen-skill-docs 时烘焙),问题以结果术语("如果...对你的用户有什么影响")而不是
实现术语框架,短句,决策以用户影响结束。想要更紧凑的 V0 散文的高级用户设置
`gstack-config set explain_level terse`(二进制开关,没有中间模式)。参见
`docs/designs/PLAN_TUNING_V1.md` 了解完整的设计理由。最初试图与写作风格一起
进行的审查节奏改革被提取到 V1.1 — 参见 `docs/designs/PACING_UPDATES_V0.md`。

## 浏览器交互

当你需要与浏览器交互(QA、dogfooding、cookie 设置)时,使用 `/browse` skill
或直接通过 `$B <command>` 运行 browse 二进制文件。永远不要使用
`mcp__claude-in-chrome__*` 工具 — 它们很慢、不可靠,并且不是这个项目使用的。

**侧边栏架构:** 在修改 `sidepanel.js`、`background.js`、`content.js`、
`terminal-agent.ts` 或侧边栏相关服务器端点之前,阅读
`docs/designs/SIDEBAR_MESSAGE_FLOW.md`。侧边栏有一个主要表面 — **Terminal**
窗格(交互式 `claude` PTY) — Activity / Refs / Inspector 作为页脚的 `debug`
切换后面的调试覆盖层。一旦 PTY 被证明,聊天队列路径就被撕掉了;
`sidebar-agent.ts` 和 `/sidebar-command` / `/sidebar-chat` /
`/sidebar-agent/event` 端点已消失。该文档涵盖了 WS 认证流程、双令牌模型和
威胁模型边界 — 这里的静默失败通常可以追溯到不理解跨组件流程。

**WebSocket 认证使用 Sec-WebSocket-Protocol,而不是 cookies。** 浏览器无法
在 WebSocket 升级上设置 `Authorization`,但它们可以通过
`new WebSocket(url, [token])` 设置 `Sec-WebSocket-Protocol`。agent 读取它,
针对 `validTokens` 验证,并且必须在升级响应中回显协议 — 没有回显,Chromium
会立即关闭连接。`Set-Cookie: gstack_pty=...` 作为非浏览器调用者的后备保留
(跨端口 `SameSite=Strict` cookie 路径不会从 chrome-extension 源存活)。

**跨窗格 PTY 注入。** 工具栏的 Cleanup 按钮和 Inspector 的"Send to Code"
操作都通过 `window.gstackInjectToTerminal(text)` 将文本管道传输到实时 claude
PTY,由 `sidepanel-terminal.js` 公开。没有 `/sidebar-command` POST — 实时
REPL 现在是侧边栏中唯一的执行表面。

**`/health` 不得暴露任何 shell-grant token。** 它已经在 headed 模式下向
localhost 调用者泄漏 `AUTH_TOKEN`(v1.1+ TODO)。不要通过在那里添加 PTY 会话
令牌来使情况变得更糟。PTY 认证仅通过 `POST /pty-session` 流动。

**传输层安全**(v1.6.0.0+)。当 `pair-agent` 启动 ngrok 隧道时,守护进程绑定
两个 HTTP 监听器:本地监听器(127.0.0.1,完整命令表面,从不转发)和隧道监听器
(锁定的允许列表:`/connect`、带有作用域令牌的 `/command` + 26 命令浏览器驱动
允许列表、`/sidebar-chat`)。ngrok 仅转发隧道端口。通过隧道的根令牌返回 403。
SSE 端点使用通过 `POST /sse-session` 铸造的 30 分钟 HttpOnly `gstack_sse`
cookie(对 `/command` 永远无效)。隧道表面拒绝通过 `tunnel-denial-log.ts` 进入
`~/.gstack/security/attempts.jsonl`。在编辑 `server.ts`、
`sse-session-cookie.ts` 或 `tunnel-denial-log.ts` 之前,阅读
[ARCHITECTURE.md](ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600) —
模块边界(从 `sse-session-cookie.ts` 到 `token-registry.ts` 没有导入)对于
作用域隔离是承重的。

**侧边栏安全堆栈**(针对提示注入的分层防御):

| 层 | 模块 | 位于 |
|-------|--------|----------|
| L1-L3 | `content-security.ts` | 服务器和 agent — 数据标记、隐藏元素剥离、ARIA 正则表达式、URL 黑名单、信封包装 |
| L4 | `security-classifier.ts` (TestSavantAI ONNX) | **仅 sidebar-agent** |
| L4b | `security-classifier.ts` (Claude Haiku 转录) | **仅 sidebar-agent** |
| L5 | `security.ts` (canary) | 两者 — 在编译中注入,在 agent 中检查 |
| L6 | `security.ts` (combineVerdict 集成) | 两者 |

**关键约束:** `security-classifier.ts` 不能从编译的 browse 二进制文件导入。
`@huggingface/transformers` v4 需要 `onnxruntime-node`,它无法从 Bun compile
的临时提取目录 `dlopen`。只有 `security.ts`(纯字符串操作 — canary、verdict
组合器、攻击日志、状态)对 `server.ts` 是安全的。参见
`~/.gstack/projects/garrytan-gstack/ceo-plans/2026-04-19-prompt-injection-guard.md`
§"Pre-Impl Gate 1 Outcome" 了解完整的架构决策。

**阈值**(在 `security.ts` 中):
- `BLOCK: 0.85` — 如果交叉确认会导致 BLOCK 的单层分数
- `WARN: 0.60` — 交叉确认阈值。当 L4 和 L4b 都 >= 0.60 → BLOCK
- `LOG_ONLY: 0.40` — 门控转录分类器(当所有层 < 0.40 时跳过 Haiku)

**集成规则:** 仅当 ML 内容分类器和转录分类器都报告 >= WARN 时才 BLOCK。单层
高置信度降级为 WARN — 这是 Stack Overflow 指令编写 FP 缓解。Canary 泄漏总是
BLOCK(确定性)。

**环境旋钮:**
- `GSTACK_SECURITY_OFF=1` — 紧急终止开关。即使预热,分类器也保持关闭。Canary
  仍然被注入;只是跳过 ML 扫描。
- `GSTACK_SECURITY_ENSEMBLE=deberta` — 选择加入 DeBERTa-v3 集成。添加
  ProtectAI DeBERTa-v3-base-injection-onnx 作为 L4c 分类器以实现跨模型一致。
  首次运行下载 721MB。启用集成后,BLOCK 需要 2-of-3 ML 分类器在 >= WARN 时
  同意(testsavant、deberta、transcript)。没有集成(默认),BLOCK 需要
  testsavant + transcript 在 >= WARN。
- 分类器模型缓存:`~/.gstack/models/testsavant-small/`(112MB,仅首次运行)
  加上 `~/.gstack/models/deberta-v3-injection/`(721MB,仅当启用集成时)
- 攻击日志:`~/.gstack/security/attempts.jsonl`(加盐 sha256 + 仅域名,
  在 10MB 时轮换,5 代)
- 每设备盐:`~/.gstack/security/device-salt` (0600)
- 会话状态:`~/.gstack/security/session-state.json`(跨进程,原子)

## 开发符号链接意识

开发 gstack 时,`.claude/skills/gstack` 可能是指向此工作目录的符号链接
(gitignored)。这意味着 skill 更改**立即生效**,非常适合快速迭代,但在大型
重构期间有风险,因为半写的 skills 可能会破坏同时使用 gstack 的其他 Claude
Code 会话。

**每个会话检查一次:** 运行 `ls -la .claude/skills/gstack` 查看它是符号链接
还是真实副本。如果它是指向你的工作目录的符号链接,请注意:
- 模板更改 + `bun run gen:skill-docs` 立即影响所有 gstack 调用
- 对 SKILL.md.tmpl 文件的破坏性更改可能会破坏并发的 gstack 会话
- 在大型重构期间,删除符号链接(`rm .claude/skills/gstack`),以便使用
  `~/.claude/skills/gstack/` 的全局安装

**前缀设置:** Setup 在顶层创建真实目录(不是符号链接),内部有 SKILL.md 符号
链接(例如,`qa/SKILL.md -> gstack/qa/SKILL.md`)。这确保 Claude 将它们发现为
顶级 skills,而不是嵌套在 `gstack/` 下。名称要么是短的(`qa`),要么是命名空间
的(`gstack-qa`),由 `~/.gstack/config.yaml` 中的 `skill_prefix` 控制。传递
`--no-prefix` 或 `--prefix` 以跳过交互式提示。

**注意:** 将 gstack 供应到项目的仓库中已被弃用。改用全局安装 + `./setup --team`。
参见 README.md 了解团队模式说明。

**对于计划审查:** 在审查修改 skill 模板或 gen-skill-docs 管道的计划时,考虑
在上线之前是否应该隔离测试更改(特别是如果用户在其他窗口中积极使用 gstack)。

**升级迁移:** 当更改修改磁盘状态(目录结构、配置格式、陈旧文件)的方式可能会
破坏现有用户安装时,将迁移脚本添加到 `gstack-upgrade/migrations/`。阅读
CONTRIBUTING.md 的"升级迁移"部分了解格式和测试要求。升级 skill 在
`/gstack-upgrade` 期间在 `./setup` 之后自动运行这些。

## 编译的二进制文件 — 永远不要提交 browse/dist/ 或 design/dist/

`browse/dist/` 和 `design/dist/` 目录包含编译的 Bun 二进制文件(`browse`、
`find-browse`、`design`,每个约 58MB)。这些仅是 Mach-O arm64 — 它们不能在
Linux、Windows 或 Intel Mac 上工作。`./setup` 脚本已经为每个平台从源代码
构建,因此签入的二进制文件是多余的。由于历史错