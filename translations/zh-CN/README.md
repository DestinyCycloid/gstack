# gstack

> "我觉得自从去年12月以来,我基本上没有打过一行代码。这是一个极其巨大的变化。" — [Andrej Karpathy](https://fortune.com/2026/03/21/andrej-karpathy-openai-cofounder-ai-agents-coding-state-of-psychosis-openclaw/),No Priors 播客,2026年3月

当我听到 Karpathy 这么说时,我想知道他是怎么做到的。一个人如何像二十人团队一样交付?Peter Steinberger 基本上是独自用 AI 代理构建了 [OpenClaw](https://github.com/openclaw/openclaw) — 24.7万 GitHub 星标。革命已经到来。一个拥有正确工具的构建者可以比传统团队更快地推进。

我是 [Garry Tan](https://x.com/garrytan),[Y Combinator](https://www.ycombinator.com/) 的总裁兼 CEO。我与数千家初创公司合作过 — Coinbase、Instacart、Rippling — 当时他们还只是车库里的一两个人。在 YC 之前,我是 Palantir 最早的工程师/PM/设计师之一,联合创立了 Posterous(被 Twitter 收购),并构建了 Bookface,YC 的内部社交网络。

**gstack 是我的答案。** 我构建产品已经二十年了,而现在我交付的产品比以往任何时候都多。在过去60天里:3个生产服务,40多个已发布功能,兼职完成,同时全职运营 YC。在逻辑代码变更上 — 不是 AI 会膨胀的原始代码行数 — 我2026年的运行速率是**我2013年速度的约810倍**(11,417 vs 14 逻辑行/天)。年初至今(截至4月18日),2026年已经产生了**2013年全年的240倍**。在包括 Bookface 在内的40个公开+私有 `garrytan/*` 仓库中测量,排除了一个演示仓库。大部分是 AI 写的。重点不是谁打的字,而是交付了什么。

> 批评代码行数的人说 AI 会膨胀原始行数,这没错。但他们错在认为经过通胀调整后,我的生产力降低了。我的生产力更高了,高很多。完整方法论、注意事项和复现脚本:**[关于代码行数争议](docs/ON_THE_LOC_CONTROVERSY.md)**。

**2026年 — 1,237次贡献并持续增长:**

![GitHub 贡献 2026 — 1,237次贡献,1-3月大幅加速](docs/images/github-2026.png)

**2013年 — 当我在 YC 构建 Bookface 时(772次贡献):**

![GitHub 贡献 2013 — 772次贡献在 YC 构建 Bookface](docs/images/github-2013.png)

同一个人。不同的时代。区别在于工具。

**gstack 就是我的做法。** 它将 Claude Code 变成一个虚拟工程团队 — 一个重新思考产品的 CEO,一个锁定架构的工程经理,一个捕捉 AI 粗制滥造的设计师,一个发现生产 bug 的审查员,一个打开真实浏览器的 QA 负责人,一个运行 OWASP + STRIDE 审计的安全官,以及一个发布 PR 的发布工程师。二十三个专家和八个强大工具,全部是斜杠命令,全部是 Markdown,全部免费,MIT 许可证。

这是我的开源软件工厂。我每天都在使用它。我分享它是因为这些工具应该对所有人开放。

Fork 它。改进它。让它成为你自己的。如果你想批评免费开源软件 — 欢迎,但我更希望你先试试。

**适合谁:**
- **创始人和 CEO** — 尤其是仍想交付的技术型创始人
- **首次使用 Claude Code 的用户** — 结构化角色而不是空白提示
- **技术负责人和资深工程师** — 每个 PR 都有严格的审查、QA 和发布自动化

## 快速开始

1. 安装 gstack(30秒 — 见下文)
2. 运行 `/office-hours` — 描述你正在构建什么
3. 对任何功能想法运行 `/plan-ceo-review`
4. 对任何有变更的分支运行 `/review`
5. 对你的预发布 URL 运行 `/qa`
6. 到此为止。你会知道这是否适合你。

## 安装 — 30秒

**要求:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Git](https://git-scm.com/)、[Bun](https://bun.sh/) v1.0+、[Node.js](https://nodejs.org/)(仅 Windows)

### 步骤1:在你的机器上安装

打开 Claude Code 并粘贴这个。Claude 会完成其余工作。

> 安装 gstack:运行 **`git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`** 然后在 CLAUDE.md 中添加一个 "gstack" 部分,说明对所有网页浏览使用 gstack 的 /browse 技能,永远不要使用 mcp\_\_claude-in-chrome\_\_\* 工具,并列出可用技能:/office-hours、/plan-ceo-review、/plan-eng-review、/plan-design-review、/design-consultation、/design-shotgun、/design-html、/review、/ship、/land-and-deploy、/canary、/benchmark、/browse、/connect-chrome、/qa、/qa-only、/design-review、/setup-browser-cookies、/setup-deploy、/setup-gbrain、/retro、/investigate、/document-release、/codex、/cso、/autoplan、/plan-devex-review、/devex-review、/careful、/freeze、/guard、/unfreeze、/gstack-upgrade、/learn。然后询问用户是否也想将 gstack 添加到当前项目,以便团队成员获得它。

### 步骤2:团队模式 — 共享仓库的自动更新(推荐)

从你的仓库内部,粘贴这个。切换到团队模式,引导仓库以便团队成员自动获得 gstack,并提交更改:

```bash
(cd ~/.claude/skills/gstack && ./setup --team) && ~/.claude/skills/gstack/bin/gstack-team-init required && git add .claude/ CLAUDE.md && git commit -m "require gstack for AI-assisted work"
```

你的仓库中没有供应商文件,没有版本漂移,没有手动升级。每个 Claude Code 会话都以快速自动更新检查开始(限制为每小时一次,网络故障安全,完全静默)。

如果你更愿意提示团队成员而不是阻止他们,将 `required` 换成 `optional`。

### OpenClaw

OpenClaw 通过 ACP 生成 Claude Code 会话,因此当 Claude Code 安装了 gstack 时,每个 gstack 技能都能正常工作。将此粘贴到你的 OpenClaw 代理:

> 安装 gstack:运行 `git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup` 为 Claude Code 安装 gstack。然后在 AGENTS.md 中添加一个 "编码任务" 部分,说明:为编码工作生成 Claude Code 会话时,告诉会话使用 gstack 技能。包括这些示例 — 安全审计:"加载 gstack。运行 /cso",代码审查:"加载 gstack。运行 /review",QA 测试 URL:"加载 gstack。运行 /qa https://...",端到端构建功能:"加载 gstack。运行 /autoplan,实现计划,然后运行 /ship",构建前规划:"加载 gstack。运行 /office-hours 然后 /autoplan。保存计划,不要实现。"

**设置后,只需自然地与你的 OpenClaw 代理对话:**

| 你说 | 发生什么 |
|---------|-------------|
| "修复 README 中的拼写错误" | 简单 — Claude Code 会话,不需要 gstack |
| "对这个仓库运行安全审计" | 生成带有 `运行 /cso` 的 Claude Code |
| "给我构建一个通知功能" | 生成带有 /autoplan → 实现 → /ship 的 Claude Code |
| "帮我规划 v2 API 重新设计" | 生成带有 /office-hours → /autoplan 的 Claude Code,保存计划 |

参见 [docs/OPENCLAW.md](docs/OPENCLAW.md) 了解高级调度路由和 gstack-lite/gstack-full 提示模板。

### 原生 OpenClaw 技能(通过 ClawHub)

四个方法论技能直接在你的 OpenClaw 代理中工作,不需要 Claude Code 会话。从 ClawHub 安装:

```
clawhub install gstack-openclaw-office-hours gstack-openclaw-ceo-review gstack-openclaw-investigate gstack-openclaw-retro
```

| 技能 | 作用 |
|-------|-------------|
| `gstack-openclaw-office-hours` | 带有6个强制问题的产品质询 |
| `gstack-openclaw-ceo-review` | 带有4种范围模式的战略挑战 |
| `gstack-openclaw-investigate` | 根本原因调试方法论 |
| `gstack-openclaw-retro` | 每周工程回顾 |

这些是对话技能。你的 OpenClaw 代理通过聊天直接运行它们。

### 其他 AI 代理

gstack 适用于10个 AI 编码代理,不仅仅是 Claude。设置会自动检测你安装了哪些代理:

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup
```

或使用 `./setup --host <name>` 针对特定代理:

| 代理 | 标志 | 技能安装到 |
|-------|------|-------------------|
| OpenAI Codex CLI | `--host codex` | `~/.codex/skills/gstack-*/` |
| OpenCode | `--host opencode` | `~/.config/opencode/skills/gstack-*/` |
| Cursor | `--host cursor` | `~/.cursor/skills/gstack-*/` |
| Factory Droid | `--host factory` | `~/.factory/skills/gstack-*/` |
| Slate | `--host slate` | `~/.slate/skills/gstack-*/` |
| Kiro | `--host kiro` | `~/.kiro/skills/gstack-*/` |
| Hermes | `--host hermes` | `~/.hermes/skills/gstack-*/` |
| GBrain (mod) | `--host gbrain` | `~/.gbrain/skills/gstack-*/` |

**想添加对另一个代理的支持?** 参见 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)。
只需一个 TypeScript 配置文件,零代码更改。

## 看它工作

```
你:    我想为我的日历构建一个每日简报应用。
你:    /office-hours
Claude: [询问痛点 — 具体例子,不是假设]

你:    多个 Google 日历,事件信息过时,位置错误。
        准备工作耗时很长,结果也不够好...

Claude: 我要对这个框架提出质疑。你说"每日简报应用"。
        但你实际描述的是个人首席助理 AI。
        [提取你没意识到自己在描述的5个能力]
        [挑战4个前提 — 你同意、不同意或调整]
        [生成3种实现方法及工作量估算]
        建议:明天发布最窄的楔子,从实际使用中学习。完整愿景
        是一个3个月的项目 — 从真正有效的每日简报开始。
        [编写设计文档 → 自动输入到下游技能]

你:    /plan-ceo-review
        [阅读设计文档,挑战范围,运行10节审查]

你:    /plan-eng-review
        [数据流、状态机、错误路径的 ASCII 图表]
        [测试矩阵、故障模式、安全问题]

你:    批准计划。退出计划模式。
        [在11个文件中编写2,400行。约8分钟。]

你:    /review
        [自动修复] 2个问题。[询问] 竞态条件 → 你批准修复。

你:    /qa https://staging.myapp.com
        [打开真实浏览器,点击流程,发现并修复一个 bug]

你:    /ship
        测试:42 → 51 (+9个新测试)。PR:github.com/you/app/pull/42
```

你说"每日简报应用"。代理说"你在构建首席助理 AI" — 因为它倾听了你的痛点,而不是你的功能请求。八个命令,端到端。这不是副驾驶。这是一个团队。

## 冲刺

gstack 是一个流程,不是工具集合。技能按冲刺运行的顺序执行:

**思考 → 规划 → 构建 → 审查 → 测试 → 发布 → 反思**

每个技能都输入到下一个。`/office-hours` 编写 `/plan-ceo-review` 读取的设计文档。`/plan-eng-review` 编写 `/qa` 采用的测试计划。`/review` 捕获 `/ship` 验证已修复的 bug。没有任何东西会漏掉,因为每一步都知道之前发生了什么。

| 技能 | 你的专家 | 他们做什么 |
|-------|----------------|--------------|
| `/office-hours` | **YC 办公时间** | 从这里开始。六个强制问题在你编写代码之前重新构建你的产品。对你的框架提出质疑,挑战前提,生成实现替代方案。设计文档输入到每个下游技能。 |
| `/plan-ceo-review` | **CEO / 创始人** | 重新思考问题。找到隐藏在请求中的10星产品。四种模式:扩展、选择性扩展、保持范围、缩减。 |
| `/plan-eng-review` | **工程经理** | 锁定架构、数据流、图表、边缘情况和测试。强制将隐藏假设公开。 |
| `/plan-design-review` | **资深设计师** | 对每个设计维度评分0-10,解释10分是什么样子,然后编辑计划以达到目标。AI 粗制滥造检测。交互式 — 每个设计选择一个 AskUserQuestion。 |
| `/plan-devex-review` | **开发者体验负责人** | 交互式 DX 审查:探索开发者角色,对比竞争对手的 TTHW 基准,设计你的神奇时刻,逐步追踪摩擦点。三种模式:DX 扩展、DX 打磨、DX 分类。20-45个强制问题。 |
| `/design-consultation` | **设计合作伙伴** | 从零开始构建完整的设计系统。研究领域,提出创意风险,生成逼真的产品模型。 |
| `/review` | **资深工程师** | 找到通过 CI 但在生产中崩溃的 bug。自动修复明显的问题。标记完整性差距。 |
| `/investigate` | **调试器** | 系统化根本原因调试。铁律:没有调查就没有修复。追踪数据流,测试假设,3次失败修复后停止。 |
| `/design-review` | **会编码的设计师** | 与 /plan-design-review 相同的审计,然后修复发现的问题。原子提交,前后截图。 |
| `/devex-review` | **DX 测试员** | 实时开发者体验审计。实际测试你的入门:浏览文档,尝试入门流程,计时 TTHW,截图错误。与 `/plan-devex-review` 分数比较 — 显示你的计划是否符合现实的回旋镖。 |
| `/design-shotgun` | **设计探索者** | "给我看选项。"生成4-6个 AI 模型变体,在浏览器中打开比较板,收集你的反馈并迭代。品味记忆学习你喜欢什么。重复直到你喜欢某个东西,然后交给 `/design-html`。 |
| `/design-html` | **设计工程师** | 将模型转换为真正有效的生产 HTML。预文本计算布局:文本重排,高度调整,布局是动态的。30KB,零依赖。检测 React/Svelte/Vue。每种设计类型的智能 API 路由(着陆页 vs 仪表板 vs 表单)。输出是可发布的,不是演示。 |
| `/qa` | **QA 负责人** | 测试你的应用,找到 bug,用原子提交修复它们,重新验证。为每个修复自动生成回归测试。 |
| `/qa-only` | **QA 报告员** | 与 /qa 相同的方法论,但仅报告。纯 bug 报告,不更改代码。 |
| `/pair-agent` | **多代理协调器** | 与任何 AI 代理共享你的浏览器。一个命令,一次粘贴,连接。适用于 OpenClaw、Hermes、Codex、Cursor 或任何可以 curl 的东西。每个代理获得自己的标签页。自动启动有头模式,所以你可以看到一切。为远程代理自动启动 ngrok 隧道。作用域令牌、标签页隔离、速率限制、活动归因。 |
| `/cso` | **首席安全官** | OWASP Top 10 + STRIDE 威胁模型。零噪音:17个误报排除,8/10+置信度门槛,独立发现验证。每个发现都包含具体的利用场景。 |
| `/ship` | **发布工程师** | 同步 main,运行测试,审计覆盖率,推送,打开 PR。如果你没有测试框架,则引导测试框架。 |
| `/land-and-deploy` | **发布工程师** | 合并 PR,等待 CI 和部署,验证生产健康。从"已批准"到"在生产中验证"的一个命令。 |
| `/canary` | **SRE** | 部署后监控循环。监视控制台错误、性能回归和页面故障。 |
| `/benchmark` | **性能工程师** | 基线页面加载时间、Core Web Vitals 和资源大小。在每个 PR 上比较前后。 |
| `/document-release` | **技术作家** | 更新所有项目文档以匹配你刚发布的内容。自动捕获过时的 README。 |
| `/retro` | **工程经理** | 团队感知的每周回顾。每人细分、发布连续性、测试健康趋势、成长机会。`/retro global` 在你所有的项目和 AI 工具(Claude Code、Codex、Gemini)上运行。 |
| `/browse` | **QA 工程师** | 给代理眼睛。真实的 Chromium 浏览器,真实的点击,真实的截图。每个命令约100毫秒。`/open-gstack-browser` 启动带有侧边栏、反机器人隐身和自动模型路由的 GStack Browser。 |
| `/setup-browser-cookies` | **会话管理器** | 从你的真实浏览器(Chrome、Arc、Brave、Edge)导入 cookie 到无头会话。测试已认证页面。 |
| `/autoplan` | **审查流水线** | 一个命令,完全审查的计划。自动运行 CEO → 设计 → 工程审查,编码决策原则。仅显示品味决策供你批准。 |
| `/learn` | **记忆** | 管理 gstack 跨会话学到的内容。审查、搜索、修剪和导出项目特定的模式、陷阱和偏好。学习在会话间累积,因此 gstack 在你的代码库上变得更智能。 |

### 我应该使用哪个审查?

| 构建对象... | 计划阶段(代码前) | 实时审计(发布后) |
|-----------------|--------------------------|----------------------------|
| **最终用户**(UI、Web 应用、移动端) | `/plan-design-review` | `/design-review` |
| **开发者**(API、CLI、SDK、文档) | `/plan-devex-review` | `/devex-review` |
| **架构**(数据流、性能、测试) | `/plan-eng-review` | `/review` |
| **以上所有** | `/autoplan`(运行 CEO → 设计 → 工程 → DX,自动检测哪些适用) | — |

### 强大工具

| 技能 | 作用 |
|-------|-------------|
| `/codex` | **第二意见** — 来自 OpenAI Codex CLI 的独立代码审查。三种模式:审查(通过/失败门槛)、对抗性挑战和开放咨询。当 `/review` 和 `/codex` 都运行时的跨模型分析。 |
| `/careful` | **安全护栏** — 在破坏性命令(rm -rf、DROP TABLE、force-push)之前警告。说"小心"激活。覆盖任何警告。 |
| `/freeze` | **编辑锁定** — 将文件编辑限制在一个目录。在调试时防止范围外的意外更改。 |
| `/guard` | **完全安全** — 一个命令中的 `/careful` + `/freeze`。生产工作的最大安全性。 |
| `/unfreeze` | **解锁** — 移除 `/freeze` 边界。 |
| `/open-gstack-browser` | **GStack Browser** — 启动带有侧边栏、反机器人隐身、自动模型路由(Sonnet 用于操作,Opus 用于分析)、一键 cookie 导入和 Claude Code 集成的 GStack Browser。清理页面,智能截图,编辑 CSS,并将信息传回你的终端。 |
| `/setup-deploy` | **部署配置器** — `/land-and-deploy` 的一次性设置。检测你的平台、生产 URL 和部署命令。 |
| `/setup-gbrain` | **GBrain 入门** — 从零到运行 gbrain 不到5分钟。PGLite 本地、Supabase 现有 URL,或通过管理 API 自动配置新的 Supabase 项目。Claude Code 的 MCP 注册 + 每个仓库的信任三元组(读写/只读/拒绝)。[完整指南](USING_GBRAIN_WITH_GSTACK.md)。 |
| `/gstack-upgrade` | **自我更新器** — 升级 gstack 到最新版本。检测全局 vs 供应商安装,同步两者,显示更改内容。 |

### 新二进制文件(v0.19)

除了斜杠命令技能,gstack 还提供独立 CLI,用于不属于会话内部的工作流:

| 命令 | 作用 |
|---------|-------------|
| `gstack-model-benchmark` | **跨模型基准测试** — 通过 Claude、GPT(通过 Codex CLI)和 Gemini 运行相同的提示;比较延迟、令牌、成本和(可选)LLM 评判质量分数。每个提供商检测认证,不可用的提供商干净跳过。输出为表格、JSON 或 markdown。`--dry-run` 验证标志 + 认证而不花费 API 调用。 |
| `gstack-taste-update` | **设计品味学习** — 将 `/design-shotgun` 的批准和拒绝写入持久的每项目品味配置文件。每周衰减5%。反馈到未来的变体生成,因此系统学习你实际选择的内容。 |

### 连续检查点模式(选择加入,默认本地)

设置 `gstack-config set checkpoint_mode continuous`,技能会在你工作时自动提交,带有 `WIP:` 前缀加上结构化的 `[gstack-context]` 正文(决策、剩余工作、失败方法)。在崩溃和上下文切换中幸存。`/context-restore` 读取这些提交以重建会话状态。`/ship` 在 PR 之前过滤压缩 WIP 提交(保留非 WIP 提交),因此 bisect 保持干净。推送通过 `checkpoint_push=true` 选择加入 — 默认仅本地,因此你不会在每个 WIP 提交上触发 CI。

### 域技能 + 原始 CDP 逃生舱

两个新的浏览器原语随时间复合 gstack 代理:

- **`$B domain-skill save`** — 代理保存每个站点的注释(例如,"LinkedIn 的申请按钮位于 iframe 中"),下次访问该主机名时自动触发。隔离 → 3次成功使用后激活 → 通过 `$B domain-skill promote-to-global` 可选的跨项目推广。存储与 `/learn` 的每项目学习文件一起。完整参考:**[docs/domain-skills.md](docs/domain-skills.md)**。
- **`$B cdp <Domain.method>`** — 原始 Chrome DevTools Protocol 逃生舱,用于精选命令遗漏的罕见情况。默认拒绝:方法必须明确添加到 `browse/src/cdp-allowlist.ts`,并附上一行理由。两层互斥锁将浏览器范围的 CDP 调用与每个标签页的工作序列化。数据外泄方法的输出包装在 UNTRUSTED 信封中。

> 想要没有护栏、没有允许列表、没有守护进程的原始 CDP — 只是从代理到 Chrome 的薄传输?[browser-use/browser-harness-js](https://github.com/browser-use/browser-harness-js) 是不同的哲学(代理编写的助手 vs gstack 的精选命令),如果你不想要 gstack 的安全堆栈,它很合适。两者可以共存:gstack 的 `$B cdp` 和 harness 都可以通过 Playwright 的 `newCDPSession` 附加到同一个 Chrome。

**[每个技能的深入探讨、示例和哲学 →](docs/skills.md)**

### Karpathy 的四种失败模式?